# 검색 성능 최적화 — Filter Cache·Request Cache·shard preference

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Filter Cache(Query Cache)는 어떤 쿼리를 캐시하고 어떤 쿼리는 캐시하지 않는가?
- Request Cache는 Filter Cache와 어떻게 다르고 어떤 조건에서 효과적인가?
- `preference=_local`이 캐시 히트율을 높이는 원리는 무엇인가?
- 자주 사용되는 날짜 범위 필터가 매번 새로 계산되는 이유와 해결 방법은?
- 검색 성능 병목이 쿼리 자체인지 Fetch인지 구분하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

동일한 대시보드 쿼리를 수백 명이 동시에 요청하면 ES가 수백 번 동일한 계산을 반복하는가, 아니면 캐시를 재사용하는가? 이 차이가 대시보드 응답 시간 50ms vs 2000ms를 결정할 수 있다. Filter Cache와 Request Cache의 원리를 이해하면 코드 변경 없이 서버 설정만으로 검색 성능을 극적으로 향상시킬 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: now를 포함한 range를 filter에 사용
  쿼리:
    { "filter": { "range": { "timestamp": {
      "gte": "now-1h",
      "lte": "now"
    }}}}

  문제:
    "now"는 요청마다 다른 값
    → 동일해 보이는 쿼리가 실제로는 매 요청마다 다른 쿼리
    → Filter Cache 저장 불가 (캐시 키가 매번 다름)
    → 매 요청마다 전체 재계산

  올바른 접근:
    애플리케이션에서 구체적인 시간으로 변환:
    "gte": "2024-01-15T09:30:00", "lte": "2024-01-15T10:30:00"
    → 동일 쿼리 → Filter Cache 적용 가능

실수 2: 로드 밸런서가 요청을 분산해서 캐시 효율이 낮음
  상황:
    클라이언트 → 로드 밸런서 → node-1 또는 node-2
    Request Cache: 각 노드의 샤드 레벨에서 독립적
    node-1에서 집계 → node-1의 Request Cache에 저장
    다음 요청 → node-2로 → node-2 Request Cache 없음 → 재계산

  올바른 접근:
    preference=_local 또는 특정 preference 값 사용
    동일 쿼리 유형 → 동일 노드로 라우팅
    → 같은 노드의 Filter Cache + Request Cache 재사용

실수 3: Request Cache를 항상 무효화하는 설정
  상황:
    refresh_interval: 100ms (빠른 NRT)

  문제:
    refresh마다 Request Cache 무효화
    100ms마다 캐시 무효화 → 사실상 캐시 효과 없음
    → 캐시는 있지만 사용 못 함

  해결:
    대시보드 조회: refresh_interval 늘리기 (1분~5분)
    또는 특정 집계용 인덱스 별도 운영
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
캐시 계층 활용 전략:

  Filter Cache 활용:
    □ bool.filter에 자주 사용되는 조건 배치
    □ "now" 대신 구체적 시간 사용 (애플리케이션에서 계산)
    □ 자주 반복되는 필터 조건 식별 후 모니터링

  Request Cache 활용:
    □ size: 0 집계 요청 (hits 없는 순수 집계)
    □ refresh_interval 늘리기 (캐시 생존 시간 증가)
    □ preference 설정으로 동일 노드 라우팅

  shard preference 활용:
    □ preference=_local (로컬 샤드 우선 → 네트워크 감소, 캐시 재사용)
    □ preference=custom_key (동일 쿼리 유형 → 동일 샤드)
    □ 대시보드 쿼리: preference=dashboard_type

  _source 필드 제한:
    □ _source: ["id", "title", "price"] (필요한 필드만)
    □ _source: false (ID만 필요 시)
    → Fetch Phase 비용 감소
```

---

## 🔬 내부 동작 원리

### 1. Filter Cache (Query Cache) — 세그먼트 레벨 비트셋

```
캐시 대상:
  Filter Context에서 처리되는 쿼리 (bool.filter, bool.must_not)
  term, terms, range, match_all 등 순수 필터 쿼리

캐시 저장 위치:
  노드 메모리 (각 노드의 세그먼트 레벨)
  세그먼트별 독립적인 캐시 항목

캐시 형태:
  Roaring Bitset: 각 세그먼트에서 조건을 만족하는 문서 ID 비트맵
  예: "category=keyboard" → [1,0,1,0,1,1,0,...] (doc_id 인덱스)

캐시 조건:
  ① Filter Context 쿼리여야 함 (Query Context × → Filter Cache 미적용)
  ② 쿼리가 "비용이 충분히 높아야" 캐시 가치 있음
     (단순 term 쿼리는 빠르므로 모든 term을 캐시하지 않음)
  ③ 세그먼트 크기가 충분해야 (소형 세그먼트 캐시 효율 낮음)

캐시 무효화:
  해당 세그먼트에 변화 발생 (refresh) → 해당 세그먼트 캐시 무효화
  세그먼트가 병합되면 → 구 세그먼트 캐시 제거

  즉: refresh_interval = Filter Cache 수명
  → refresh 자주 → 캐시 자주 무효화 → 효과 감소

"now" 문제:
  "now"를 포함한 쿼리:
    "gte": "now-1h" → 매 요청마다 다른 ms 값
    → 쿼리 직렬화 결과가 다름 → 다른 캐시 키
    → 캐시 히트 불가

  해결: 애플리케이션에서 시간 범위를 계산 + 반올림
    "gte": "2024-01-15T09:00:00" (분 단위 반올림)
    → 같은 분 내 요청은 동일 쿼리 → 캐시 재사용

설정:
  indices.queries.cache.size: 10% (힙의 10%, 기본값)
  → 늘리면 더 많은 비트셋 캐시 가능 but 힙 증가

모니터링:
  GET _nodes/stats/indices/query_cache?pretty
  → hit_count, miss_count, memory_size_in_bytes
  → 히트율 = hit/(hit+miss)
```

### 2. Request Cache — 집계 결과 캐시

```
캐시 대상:
  size: 0 요청 (hits 없는 집계 전용)
  전체 샤드 레벨의 집계 결과

캐시 저장 위치:
  각 노드의 Request Cache (샤드 레벨)
  Filter Cache와 독립적

캐시 키:
  요청 JSON 전체 직렬화 → 해시값
  → 동일 JSON → 동일 해시 → 캐시 HIT

캐시 무효화:
  해당 인덱스의 세그먼트 변경 (refresh) → 해당 인덱스의 Request Cache 무효화
  → refresh_interval = Request Cache 수명

  예시:
    refresh_interval: 1s → Request Cache 최대 1초 유효
    반복 대시보드 조회(10초마다) → 매번 miss (1초마다 무효화)

    refresh_interval: 5m → Request Cache 최대 5분 유효
    반복 조회 → 5분간 hit 가능

캐시 효과 극대화:
  정적 인덱스 (과거 데이터, 변경 없음):
    → refresh 없음 → Request Cache 영구 유효
    → 매 요청이 캐시 HIT

  시계열 인덱스:
    → ILM warm 단계 (읽기 전용) → refresh 없음 → 캐시 영구 유효
    → hot 단계만 캐시 효율 낮음

설정:
  indices.requests.cache.size: 1% (힙의 1%, 기본값)
  → 집계 캐시 많이 사용한다면 2~5%로 증가

비활성화:
  특정 요청에서 캐시 비활성화:
  GET /index/_search?request_cache=false
  { "size": 0, "aggs": { ... } }

모니터링:
  GET _nodes/stats/indices/request_cache?pretty
  → hit_count, miss_count, memory_size_in_bytes
```

### 3. shard preference — 캐시 히트율 최적화

```
기본 동작 (Round-Robin):
  Coordinator → Shard 0의 Primary 또는 Replica 선택 (순서대로)
  → 요청마다 다른 노드의 샤드로 분산
  → Filter Cache/Request Cache가 각 노드에 분산
  → 같은 쿼리가 여러 노드에서 각각 캐시됨 (중복 캐시)

preference=_local:
  Coordinator (= 요청받은 노드)의 로컬 샤드 우선 선택
  → 동일 노드의 Filter Cache/Request Cache 재사용
  → 네트워크 왕복 없음 (같은 노드 내 처리)

  이득:
    캐시 히트율 높음 (같은 노드에 여러 번 요청 → 캐시 재사용)
    네트워크 감소 (로컬 처리)
  비용:
    특정 노드에 요청 집중 가능성 (로드 밸런싱 불균형)
    → 로드 밸런서가 여러 노드에 분산하면 효과적

preference=custom_value:
  동일 custom_value → 항상 동일 샤드 선택
  → 같은 쿼리 유형이 항상 같은 샤드/노드로
  → 해당 샤드의 캐시가 집중되어 히트율 향상

  사용 예시:
    대시보드 A: preference="dashboard_a"
    대시보드 B: preference="dashboard_b"
    → 각 대시보드 요청이 항상 같은 샤드로 → 캐시 재사용

  GET /orders/_search?preference=daily_report
  { "size": 0, "aggs": { "by_cat": { "terms": ... } } }
```

### 4. 검색 성능 병목 진단

```
Query Phase vs Fetch Phase 비용 분리:

  GET _nodes/stats/indices/search?pretty
  - query_time_in_millis / query_total: 평균 Query Phase 시간
  - fetch_time_in_millis / fetch_total: 평균 Fetch Phase 시간

  Query Phase 느림:
    복잡한 쿼리, wildcard, fuzzy, 대규모 Terms 집계
    → 쿼리 최적화 (filter 이동, 인덱스 최적화)

  Fetch Phase 느림:
    큰 _source 문서, 많은 결과 반환
    → _source 필드 제한, size 줄이기

profile API로 정밀 분석:
  GET /index/_search
  {
    "profile": true,
    "query": { ... }
  }
  → 어떤 절이 얼마나 걸리는지 ms 단위 측정

_source 필드 제한:
  전체 _source 반환: { "title": "...", "desc": "...", "raw": "..." }
  → Fetch Phase: 모든 필드 읽기 + 네트워크 전송

  필드 제한:
  GET /index/_search
  {
    "_source": ["title", "price"],  // 필요한 필드만
    "query": { ... }
  }
  → Fetch Phase 비용 대폭 감소

  또는:
  "stored_fields": ["_id"]  // _source 대신 stored field만
  "_source": false           // _source 비활성화
```

---

## 💻 실전 실험

```bash
# Filter Cache 히트율 확인
GET _nodes/stats/indices/query_cache?pretty
# hit_count, miss_count, memory_size_in_bytes 확인
# 히트율 = hit_count / (hit_count + miss_count)

# Request Cache 히트율 확인
GET _nodes/stats/indices/request_cache?pretty

# Filter Cache 효과 테스트
# 1. 첫 번째 요청 (miss)
GET /orders/_search
{
  "size": 0,
  "query": {
    "bool": { "filter": [
      { "term": { "category": "keyboard" } },
      { "range": { "price": { "gte": 100000 } } }
    ]}
  },
  "aggs": { "count": { "value_count": { "field": "_id" } } }
}

# 2. 통계 확인 (miss 증가했는지)
GET _nodes/stats/indices/query_cache?pretty

# 3. 동일 요청 반복 (hit 기대)
GET /orders/_search
{
  "size": 0,
  "query": {
    "bool": { "filter": [
      { "term": { "category": "keyboard" } },
      { "range": { "price": { "gte": 100000 } } }
    ]}
  },
  "aggs": { "count": { "value_count": { "field": "_id" } } }
}

# 4. 통계 재확인 (hit 증가했는지)
GET _nodes/stats/indices/query_cache?pretty

# preference 사용 예시
GET /orders/_search?preference=_local
{ "size": 0, "aggs": { "by_cat": { "terms": { "field": "category" } } } }

GET /orders/_search?preference=dashboard_main
{ "size": 0, "aggs": { "daily": { "date_histogram": { "field": "created_at", "calendar_interval": "day" } } } }

# profile로 Query/Fetch 분리 측정
GET /orders/_search
{
  "profile": true,
  "size": 10,
  "query": { "match": { "title": "keyboard" } }
}
# profile.shards[0].searches[0].query: Query Phase 시간
# profile.shards[0].searches[0].fetch 확인

# _source 필드 제한으로 Fetch Phase 최적화
GET /orders/_search
{
  "_source": ["title", "price", "category"],
  "query": { "match": { "title": "keyboard" } }
}

# 캐시 수동 초기화 (테스트 목적)
POST /orders/_cache/clear?query=true     // Filter Cache
POST /orders/_cache/clear?request=true   // Request Cache
```

---

## 📊 캐시 종류 비교

| 캐시 | 저장 내용 | 위치 | 무효화 | 효과적인 경우 |
|------|---------|------|-------|-----------|
| Filter Cache | 비트셋 (bool.filter 결과) | 노드 RAM (세그먼트별) | refresh 시 | 반복 필터 조건 |
| Request Cache | 집계 전체 결과 | 노드 RAM (샤드별) | refresh 시 | size:0 반복 집계 |
| OS Page Cache | Lucene 세그먼트 파일 | OS RAM (힙 외부) | mmap eviction | 모든 검색 (자동) |

---

## ⚖️ 트레이드오프

```
Filter Cache 크게:
  이득: 더 많은 비트셋 캐시 → 히트율 향상
  비용: 힙 증가 → GC 압박 → 다른 작업 성능 저하
  → 기본 10% 유지, 극단적 filter 반복 시 15% 고려

Request Cache 크게:
  이득: 더 많은 집계 결과 캐시
  비용: 힙 증가 (단, 기본 1%로 영향 적음)
  → 대시보드 집중 서비스: 2~5%로 증가 고려

preference=_local:
  이득: 네트워크 감소, 캐시 재사용
  비용: 특정 노드 과부하 가능성
  → 로드 밸런서로 분산된 경우 효과적

refresh_interval 늘리기:
  이득: 캐시 수명 증가 → 캐시 효과 향상
  비용: NRT 지연 증가
  → 대시보드 용 read-only 인덱스에 가장 효과적
```

---

## 📌 핵심 정리

```
Filter Cache:
  bool.filter 쿼리 결과를 Roaring Bitset으로 저장
  캐시 키: 쿼리 직렬화 → "now" 포함 시 매번 다른 키 → 캐시 불가
  무효화: refresh 시 → refresh_interval = 캐시 수명

Request Cache:
  size:0 집계 결과 전체를 노드에 캐시
  무효화: refresh 시 → 정적 인덱스(변경 없음)에서 효과 극대화
  캐시 키: 요청 JSON 전체 해시

shard preference:
  _local: 로컬 샤드 우선 → 네트워크 감소, 캐시 재사용
  custom: 동일 쿼리 유형 → 같은 샤드 → 캐시 집중

검색 병목 진단:
  _nodes/stats query_time vs fetch_time → 어느 Phase가 병목
  profile API → 절별 실행 시간
  _source 필드 제한 → Fetch 비용 감소
```

---

## 🤔 생각해볼 문제

**Q1.** "최근 1시간 데이터" 조회 쿼리에서 Filter Cache를 활용하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

"now-1h"처럼 동적 시간을 사용하면 매 요청마다 다른 쿼리가 되어 Filter Cache를 사용할 수 없다. 캐시를 활용하려면 애플리케이션 레벨에서 시간을 반올림해 고정값으로 변환해야 한다. 예를 들어 1시간 단위로 반올림하면 "2024-01-15T09:00:00 ~ 2024-01-15T10:00:00"처럼 같은 시간대의 모든 요청이 동일한 쿼리가 되어 캐시가 작동한다. 단, 반올림 단위가 클수록 캐시 효율이 높아지지만 데이터 신선도가 낮아지는 트레이드오프가 있다.

</details>

**Q2.** Request Cache가 활성화된 상태에서 동일 집계를 두 번 실행했는데 두 번째도 miss라면 이유는?

<details>
<summary>해설 보기</summary>

여러 원인이 있다. 첫째, 두 번째 요청 전에 refresh가 발생해 캐시가 무효화된 경우다. refresh_interval이 1초이면 1초 이내에 두 번 호출해야 hit가 가능하다. 둘째, 두 번째 요청이 다른 노드의 다른 샤드로 라우팅된 경우다. Request Cache는 샤드 레벨이므로 다른 샤드에는 캐시가 없다. preference를 동일하게 설정하면 같은 샤드로 라우팅된다. 셋째, size가 0이 아닌 경우(hits 있는 요청)는 Request Cache 대상이 아니다.

</details>

**Q3.** 검색 응답에서 `took`이 낮은데 사용자 체감 시간은 높은 이유는?

<details>
<summary>해설 보기</summary>

`took`은 ES 서버 내부 처리 시간으로, 이 시간은 검색 처리에만 걸린 시간이다. 사용자 체감 시간에는 클라이언트 → 로드 밸런서 → ES 노드까지의 네트워크 RTT, 응답 반환의 직렬화/역직렬화, 클라이언트 라이브러리 처리, 여러 ES 요청을 순서대로 실행하는 경우 각 요청의 대기 시간이 포함된다. 특히 응답 데이터가 큰 경우(많은 문서 또는 큰 문서) 직렬화와 네트워크 전송 시간이 ES `took`보다 훨씬 클 수 있다. APM 도구나 클라이언트 측 타이밍을 측정해 전체 경로를 분석해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: 쓰기 성능 최적화](./04-write-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 중 발생하는 문제 패턴 ➡️](./06-operational-problems.md)**

</div>
