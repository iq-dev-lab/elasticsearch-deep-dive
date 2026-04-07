# 집계와 메모리 — fielddata·Circuit Breaker·OOM 방지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- text 필드에 집계를 실행하면 왜 힙에 대규모 데이터가 올라오는가?
- Circuit Breaker는 OOM을 막기 위해 어떤 시점에 어떻게 작동하는가?
- `indices.fielddata.cache.size`와 `indices.breaker.fielddata.limit`의 차이는?
- 집계가 힙을 많이 쓰는 또 다른 이유(Bucket 객체 누적)는 무엇인가?
- OOM 발생 후 복구하는 방법과 사전에 방지하는 설정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

ES OOM 장애의 주요 원인 두 가지는 fielddata와 대규모 집계다. "Aggregation이 느린데 인덱스 설정을 최적화해달라"는 요청 뒤에 실제로는 텍스트 필드에 집계를 걸어 힙을 모두 잡아먹는 fielddata가 있는 경우가 많다. Circuit Breaker가 없으면 OOM으로 노드가 죽는다. 이 메커니즘을 이해하면 장애를 예방하고, 발생했을 때 빠르게 진단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: text 필드에 fielddata: true 설정 후 집계
  설정:
    "description": { "type": "text", "fielddata": true }

  집계:
    { "terms": { "field": "description" } }

  결과:
    첫 집계 시: 전체 역색인 로딩 → 힙 수백 MB ~ 수 GB 점유
    두 번째 집계: 캐시 활용으로 빠름
    하지만 힙 점유 유지 → GC 압박 → 다른 작업 지연

    더 큰 문제:
    text 필드는 형태소 분석됨 → 각 토큰이 별도 집계 버킷
    "Mechanical Keyboard Review"가 ["mechanical", "keyboard", "review"]로 분해
    → terms 집계 결과가 의미없는 개별 단어 목록

실수 2: 매우 많은 버킷 생성
  쿼리:
    { "terms": { "field": "user_id", "size": 1000000 } }

  문제:
    100만 개의 버킷을 각각 힙에 생성
    각 버킷: 키(user_id), doc_count, 서브 집계 결과
    → 100만 버킷 × 버킷 크기 = 수 GB 힙 점유

  Circuit Breaker가 없으면:
    OOM → 노드 재시작 → 데이터 손실 가능

실수 3: indices.fielddata.cache.size를 너무 크게
  설정:
    indices.fielddata.cache.size: 80%  // 힙의 80%

  문제:
    fielddata가 힙의 80%를 점유 가능
    GC가 이 영역을 회수하려면 GC stop-the-world 발생
    → 검색/인덱싱 지연, 최악 OOM
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
집계 메모리 설계 원칙:

  1. text 필드 집계는 keyword 서브필드 사용
     "description.keyword"로 집계 → doc_values (off-heap)
     → fielddata 없음 → 힙 영향 없음

  2. Circuit Breaker 설정으로 OOM 방지
     indices.breaker.fielddata.limit: 40%  (힙의 40%)
     indices.breaker.request.limit: 60%   (힙의 60%)
     → 임계값 초과 시 요청 거부 (노드 죽음 방지)

  3. fielddata 캐시 크기 제한
     indices.fielddata.cache.size: 20%   (힙의 20%)
     → 캐시 eviction 발생해도 힙 보호

  4. 버킷 수 제한
     search.max_buckets: 10000  (클러스터 기본값)
     → 버킷 폭발 방지
     너무 많은 버킷 요청 시 오류 반환

  5. 사전 모니터링
     GET _nodes/stats/indices/fielddata
     → fielddata 힙 사용량 추적
     임계값 접근 시 알림 설정
```

---

## 🔬 내부 동작 원리

### 1. fielddata가 힙에 올라오는 과정

```
text 필드 집계 요청:
  GET /products/_search
  { "size": 0, "aggs": { "by_desc": { "terms": { "field": "description" } } } }

  내부 처리:
    ① 역색인 읽기
       description 필드의 역색인:
         "mechanical" → [doc_1, doc_3, doc_7, ...]
         "keyboard"   → [doc_1, doc_2, doc_4, ...]
         "review"     → [doc_2, doc_5, ...]
         (수천~수만 개의 토큰 목록)

    ② 역방향 매핑 구성 (힙에!)
       doc_id → [토큰 목록] 형태로 변환
       doc_1 → ["mechanical", "keyboard"]
       doc_2 → ["keyboard", "review"]
       ...
       → 전체 인덱스의 역색인을 역방향으로 힙에 로딩

    ③ 집계 실행
       각 토큰별 doc_count 계산
       → 버킷 생성

  힙 사용량:
    100만 문서 × 평균 10개 토큰 × 평균 8바이트/토큰 = 800MB

  캐시 유지:
    fielddata는 힙에 캐시
    다음 동일 필드 집계 요청 → 캐시 재사용 (빠름)
    하지만 힙 계속 점유 → 다른 작업에 힙 부족

doc_values와의 차이:
  doc_values: 디스크 저장 (mmap, off-heap)
              집계 시 Page Cache에서 읽음 → 힙 무관
              keyword, 숫자, 날짜 필드 기본 활성화

  fielddata:  힙 저장
              text 필드에만 사용 (분석된 토큰)
              → 가능하면 keyword 필드(doc_values)로 대체
```

### 2. Circuit Breaker — OOM 방지 메커니즘

```
Circuit Breaker 종류와 역할:

  ┌─────────────────────────────────────────────────────────┐
  │           Circuit Breaker 계층 구조                       │
  │                                                         │
  │  Parent Breaker (indices.breaker.total.limit: 95%)      │
  │    ↓ 모든 하위 Breaker의 합산 한도                           │
  │                                                         │
  │  ├── Fielddata Breaker (indices.breaker.fielddata.limit: 40%)
  │  │   fielddata 로딩 전 힙 사용 추정 → 초과 시 거부             │
  │  │                                                      │
  │  ├── Request Breaker (indices.breaker.request.limit: 60%)
  │  │   단일 요청 처리 중 메모리 → 초과 시 거부                    │
  │  │   (집계 버킷, 정렬 버퍼 등 포함)                           │
  │  │                                                      │
  │  ├── In-flight Requests (indices.breaker.inflight_requests)
  │  │   현재 진행 중 요청들의 합산 크기                           │
  │  │                                                      │
  │  └── Script Compilation Breaker                         │
  │      Painless 스크립트 컴파일 빈도 제한                       │
  └─────────────────────────────────────────────────────────┘

Circuit Breaker 동작 흐름:

  fielddata 로딩 요청 발생
    ↓
  예상 fielddata 크기 추정
    ↓
  현재 fielddata 힙 사용량 + 예상 크기 계산
    ↓
  indices.breaker.fielddata.limit (기본 힙의 40%) 초과?
    YES → CircuitBreakingException 반환 (요청 거부)
          HTTP 429 Too Many Requests
          노드 계속 동작

    NO → fielddata 로딩 진행

  핵심: OOM이 발생하기 전에 요청을 거부
        노드 프로세스 자체는 죽지 않음
        클라이언트에 오류 반환 → 재시도 가능

Request Breaker (집계 버킷):
  terms 집계에서 100만 버킷 생성 시도
    ↓
  버킷 메모리 추정 계산
    ↓
  indices.breaker.request.limit (기본 힙의 60%) 초과?
    YES → 오류: Data too large, data for [terms] would be [...]
    NO  → 집계 진행

  search.max_buckets 설정:
    기본 65,536 (ES 버전에 따라 다름)
    이 수를 초과하는 버킷 → 오류 반환
    → "Too many buckets" 오류
```

### 3. fielddata 캐시 관리

```
indices.fielddata.cache.size (기본값: 무제한):
  fielddata가 캐시에서 차지할 최대 크기
  크기 초과 시 가장 오래 사용하지 않은 항목 eviction

권장 설정: 힙의 20~30%
  elasticsearch.yml:
    indices.fielddata.cache.size: 20%

캐시 동작:
  첫 집계: 역색인 → 힙 로딩 → 캐시 저장
  이후 동일 필드 집계: 캐시 HIT → 즉시 반환 (빠름)
  캐시 크기 초과 → LRU eviction → 오래된 fielddata 제거
                → 다음 해당 필드 집계 시 재로딩 필요

캐시 수동 제거 (OOM 임박 시 긴급 조치):
  POST /index/_cache/clear?fielddata=true   // 특정 인덱스
  POST _cache/clear?fielddata=true          // 전체

  주의: 캐시 제거 후 다음 집계 시 재로딩 → 일시적 느림

fielddata 사용량 확인:
  GET _nodes/stats/indices/fielddata?pretty
  → 노드별 fielddata 메모리 사용량

  GET _cat/fielddata?v&fields=*
  → 필드별 fielddata 크기 확인
```

### 4. 집계 버킷의 힙 사용

```
버킷 생성 비용:

  terms 집계 100만 버킷:
    각 버킷: 키(문자열), doc_count(long), 서브 집계 결과
    버킷 1개 ≈ 100~500 bytes (서브 집계 없을 때)
    100만 버킷: 100MB ~ 500MB 힙 필요

  중첩 집계의 버킷 폭발:
    { "terms": { "field": "category", "size": 100 },  // 100 버킷
      "aggs": { "date_histogram": { "calendar_interval": "day" } } }  // 365 버킷
    → 100 × 365 = 36,500 버킷
    → 서브 집계 하나 더 추가하면 36,500 × N 버킷

  버킷 수 제한 (클러스터 설정):
    PUT _cluster/settings
    {
      "transient": {
        "search.max_buckets": 10000
      }
    }

  버킷 메모리 추적:
    Request Breaker가 버킷 생성 중 메모리 추적
    한도 초과 → CircuitBreakingException

  실용적 제한:
    terms size를 필요한 만큼만 설정 (불필요하게 큰 size 지양)
    중첩 집계 깊이 제한 (2~3단계가 적절)
    대용량 집계는 사전 집계(Transform) 고려
```

### 5. OOM 발생 후 진단과 복구

```
OOM 징후 확인:

  GET _nodes/stats/jvm?pretty
  → jvm.mem.heap_used_percent: 90% 이상이면 위험
  → jvm.gc.collectors.*.collection_count: GC 빈번하면 위험

  GET _nodes/stats/indices/fielddata?pretty
  → 과도한 fielddata 메모리 확인

  GET _nodes/hot_threads
  → 현재 CPU를 많이 사용하는 스레드 확인 (GC 스레드가 많으면 위험)

OOM 발생 후 즉각 조치:

  1. 문제 인덱스의 fielddata 캐시 제거:
     POST /problem_index/_cache/clear?fielddata=true

  2. Circuit Breaker 현황 확인:
     GET _nodes/stats/breaker?pretty
     → breaker_name별 tripped_count 확인

  3. 집계 실행 금지 (임시):
     index.blocks.read: true (해당 인덱스 읽기 차단)

  4. JVM heap dump 분석 (근본 원인):
     -XX:+HeapDumpOnOutOfMemoryError JVM 옵션
     Eclipse MAT 또는 IntelliJ IDEA로 분석

OOM 사전 방지 설정:
  elasticsearch.yml:
    # fielddata Circuit Breaker
    indices.breaker.fielddata.limit: 40%

    # 전체 요청 Circuit Breaker
    indices.breaker.request.limit: 60%

    # 전체 메모리 한도
    indices.breaker.total.limit: 70%

    # fielddata 캐시 크기
    indices.fielddata.cache.size: 20%

  클러스터 설정:
    search.max_buckets: 10000
```

---

## 💻 실전 실험

```bash
# Circuit Breaker 현황 확인
GET _nodes/stats/breaker?pretty
# tripped_count: 얼마나 많이 차단했는지
# limit_size_in_bytes: 한도 설정값
# estimated_size_in_bytes: 현재 추정 사용량

# fielddata 메모리 사용량
GET _nodes/stats/indices/fielddata?pretty

# 필드별 fielddata 크기
GET _cat/fielddata?v&fields=*

# JVM 힙 상태
GET _nodes/stats/jvm?pretty
# heap_used_percent: 현재 힙 사용률
# gc.collectors: GC 통계

# text 필드 집계 시도 (오류 발생 확인)
GET /orders/_search
{
  "size": 0,
  "aggs": { "by_amount_text": { "terms": { "field": "description" } } }
}
# Fielddata is disabled on text fields 오류 확인

# keyword 필드로 집계 (doc_values 활용)
GET /orders/_search
{
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category" } } }
}
# doc_values → off-heap → 힙 영향 없음

# 버킷 수 제한 테스트
PUT _cluster/settings
{ "transient": { "search.max_buckets": 3 } }

GET /orders/_search
{
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category", "size": 10 } } }
}
# 오류: Trying to create too many buckets

# 원복
PUT _cluster/settings
{ "transient": { "search.max_buckets": null } }

# fielddata 캐시 제거
POST _cache/clear?fielddata=true

# Circuit Breaker 설정 확인
GET _cluster/settings?pretty&include_defaults=true
# indices.breaker.fielddata.limit, indices.breaker.request.limit 확인
```

---

## 📊 메모리 유형별 비교

| 메모리 유형 | 저장 위치 | 집계 사용 | GC 대상 | 크기 제어 방법 |
|-----------|---------|---------|--------|-------------|
| doc_values | 디스크(mmap) | keyword, 숫자, 날짜 집계 | X | 필드 수 × 문서 수 |
| fielddata | JVM 힙 | text 필드 집계 | O | cache.size 설정 |
| 집계 버킷 | JVM 힙 | 모든 Bucket 집계 | O | max_buckets, size |
| OS Page Cache | RAM (힙 외부) | Lucene 세그먼트 읽기 | X | 힙 크기로 간접 제어 |

---

## ⚖️ 트레이드오프

```
fielddata 활성화:
  이득: text 필드 집계 가능
  비용: 힙 점유 (수백 MB ~ GB), GC 압박 → 성능 저하, OOM 위험
  → 가능하면 keyword 필드로 대체 (doc_values 활용)

Circuit Breaker 한도 높음:
  이득: 더 큰 집계 허용
  비용: OOM 발생 가능성 증가 → 노드 죽음 → 더 큰 장애
  → 보수적으로 설정 (기본값 유지 권장)

fielddata 캐시 크기 크게:
  이득: 캐시 재사용률 높음 → 반복 집계 빠름
  비용: 힙 점유 증가 → GC 압박 → 다른 작업 성능 저하
  → 20% 이하 권장

max_buckets 낮게:
  이득: 버킷 폭발로 인한 OOM 방지
  비용: 일부 합법적인 대용량 집계도 차단
  → 서비스 요구사항에 맞게 조정
```

---

## 📌 핵심 정리

```
fielddata vs doc_values:
  fielddata: text 필드 집계 → 힙 점유 → OOM 위험
  doc_values: keyword/숫자/날짜 → off-heap → 힙 무관
  → text 집계는 .keyword 서브필드로 대체

Circuit Breaker:
  fielddata 로딩 전 힙 추정 → 한도 초과 시 요청 거부
  OOM 전에 차단 → 노드 생존 보장
  indices.breaker.fielddata.limit: 40% (기본)

메모리 모니터링:
  GET _nodes/stats/indices/fielddata → fielddata 힙 사용량
  GET _nodes/stats/breaker → 차단 횟수, 한도
  GET _nodes/stats/jvm → 힙 사용률, GC 통계

OOM 방지 설정:
  indices.fielddata.cache.size: 20% (fielddata 캐시 한도)
  indices.breaker.fielddata.limit: 40% (Circuit Breaker)
  search.max_buckets: 10000 (버킷 수 제한)
```

---

## 🤔 생각해볼 문제

**Q1.** Circuit Breaker가 요청을 거부할 때 해당 노드는 여전히 다른 요청을 처리할 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. Circuit Breaker는 특정 요청의 메모리 사용이 한도를 초과할 것으로 예상될 때 해당 요청만 거부하고 `CircuitBreakingException`을 반환한다. 노드 프로세스는 계속 실행되며 다른 요청(검색, 인덱싱)은 정상 처리된다. 이것이 OOM으로 노드가 죽는 것과 근본적으로 다른 점이다. OOM은 모든 작업이 중단되고 노드 재시작이 필요하지만, Circuit Breaker는 문제 요청만 차단하고 시스템은 유지된다. 따라서 Circuit Breaker 오류는 클라이언트가 재시도하거나 쿼리를 최적화해 해결해야 한다.

</details>

**Q2.** fielddata 캐시가 eviction되면 다음 집계에서 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

eviction된 필드의 fielddata는 힙에서 제거된다. 다음에 해당 필드에 집계 요청이 들어오면 역색인을 다시 읽어 힙으로 로딩하는 재구성 과정이 일어난다. 이 재로딩은 초기 로딩과 동일한 비용(시간, CPU, 메모리)이 들기 때문에 첫 요청이 느려진다. eviction이 너무 자주 발생한다면(캐시 크기가 작아서) 매번 재로딩이 반복되어 성능이 저하된다. 따라서 fielddata 캐시 크기는 자주 사용되는 필드의 fielddata를 충분히 수용할 만큼 설정하는 것이 좋다.

</details>

**Q3.** `search.max_buckets` 초과로 오류가 발생할 때 이를 피하는 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

두 가지 방법이 있다. 첫째, `search.max_buckets` 값 자체를 높이는 것이다. 클러스터 설정으로 동적 변경이 가능하다. 단, 이 방법은 메모리 위험을 함께 높인다. 둘째, 집계 설계를 최적화해 버킷 수를 줄이는 것이다. `terms size` 줄이기, 중첩 집계 단순화, 날짜 히스토그램 간격을 늘려 버킷 수 감소, Composite Aggregation으로 페이지네이션 처리가 그 방법이다. 세 번째로 Transform API를 사용해 주기적으로 사전 집계한 결과를 별도 인덱스에 저장하면 실시간 집계 부하를 없앨 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Date Histogram 집계](./03-date-histogram-aggregation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 집계 성능 최적화 ➡️](./05-aggregation-performance.md)**

</div>
