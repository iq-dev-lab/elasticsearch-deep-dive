# 쓰기 성능 최적화 — bulk API·refresh_interval·translog 설정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bulk API가 단건 인덱싱 대비 왜 훨씬 빠른가?
- `refresh_interval: -1`로 설정하면 쓰기 성능이 왜 향상되는가?
- translog `durability: async` 설정의 내구성 위험은 무엇인가?
- 대규모 초기 적재(Bulk Load) 시 권장 설정 조합은 무엇인가?
- 쓰기 병목이 인덱싱인지 네트워크인지 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

수천만 건의 데이터를 ES에 적재해야 하는 상황에서 단건 인덱싱으로 시작했다가 완료까지 10시간 걸리는 문제를 겪는 경우가 많다. Bulk API와 최적화 설정을 적용하면 같은 데이터를 1시간 내에 완료할 수 있다. 쓰기 성능과 내구성, 실시간 검색 가능성은 트레이드오프 관계이며, 비즈니스 요구사항에 맞는 설정 선택이 필요하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 단건 인덱싱으로 대량 적재
  코드:
    for document in 10_million_documents:
      es.index(index="products", body=document)

  문제:
    각 단건마다 HTTP 요청 1회 → 10만 번 HTTP 왕복
    각 요청마다 refresh 대기 가능성
    각 요청마다 translog fsync (durability: request 기본)
    → 10만 HTTP 왕복 × 평균 10ms = 1,000초 ≈ 17분 (네트워크만)

  올바른 접근:
    es.bulk(index="products", documents=batch_of_1000)
    → 1,000건당 HTTP 1회 → 100회 왕복
    → 100회 × 10ms = 1초

실수 2: 초기 적재 시 refresh_interval 기본값 유지
  설정: refresh_interval: 1s (기본값)

  문제:
    1초마다 새 세그먼트 생성
    1,000만 건 / 1초 = 수백만 번의 refresh 가능
    → 작은 세그먼트 수천 개 생성
    → 세그먼트 병합 폭발적 증가 → I/O 포화

  올바른 접근:
    초기 적재 전: refresh_interval: -1 (비활성화)
    적재 완료 후: refresh_interval: 1s 복원 + 수동 refresh

실수 3: 레플리카 있는 상태로 초기 적재
  설정: number_of_replicas: 1

  문제:
    모든 문서가 Primary → Replica로 복제
    인덱싱 비용 2배 (Primary + Replica)
    → 초기 적재 중 네트워크 대역폭 2배 소비

  올바른 접근:
    초기 적재: number_of_replicas: 0
    완료 후: number_of_replicas: 1 복원 → ES가 자동 복제
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
초기 대량 적재(Bulk Load) 최적 설정:

  사전 설정 (적재 전):
    PUT /myindex/_settings
    {
      "index.refresh_interval": "-1",         // refresh 비활성화
      "index.number_of_replicas": 0,           // 레플리카 제거
      "index.translog.durability": "async",    // fsync 비동기
      "index.translog.sync_interval": "60s"    // 60초마다 sync
    }

  적재 방식:
    Bulk API, 배치 크기 5~15MB (경험적 최적값)
    스레드 수 = Primary Shard 수 (병렬화)

  사후 복원 (적재 완료 후):
    POST /myindex/_refresh                    // 세그먼트 생성 (검색 가능화)
    PUT /myindex/_settings
    {
      "index.refresh_interval": "1s",          // 기본 복원
      "index.number_of_replicas": 1,           // 레플리카 복원 (자동 복제 시작)
      "index.translog.durability": "request"   // 내구성 복원
    }

  운영 중 쓰기 최적화:
    Bulk API 사용 (단건 금지)
    refresh_interval: 5s ~ 30s (NRT 요구사항에 따라)
    translog: request (기본, 내구성 유지)
```

---

## 🔬 내부 동작 원리

### 1. Bulk API vs 단건 인덱싱

```
단건 인덱싱 비용:
  PUT /index/_doc/1 { "field": "value" }

  네트워크:
    HTTP 요청 헤더 + 본문 → ES 노드 → 응답 헤더 + 본문
    왕복 1회: ~5~50ms (로컬) ~ 수백 ms (원격)

  처리:
    요청 파싱 → 샤드 라우팅 → Primary 인덱싱 → Replica 복제
    → translog fsync (durability: request 기본) → 응답

  문제:
    100만 건 × 10ms/건 = 10,000초 = 2.7시간 (네트워크만)

─────────────────────────────────────────────────────────

Bulk API 비용:
  POST /index/_bulk
  { "index": { "_id": "1" } }
  { "field": "value_1" }
  { "index": { "_id": "2" } }
  { "field": "value_2" }
  ...1000건...

  네트워크:
    HTTP 요청 1회 (1000건 묶어서) → ES 노드 → 응답 1회
    왕복 1회 for 1000건: ~50~500ms (1건 처리와 비슷한 시간)

  처리:
    1000건 배치로 파싱 → 샤드별 그룹핑
    → 각 Primary에 배치 전달 → 배치 인덱싱
    → translog fsync 1회 (1000건 한번에)

  효과:
    100만 건 / 1000건 배치 = 1000회 HTTP 왕복
    1000회 × 50ms = 50초 (단건 2.7시간 vs Bulk 50초)

Bulk API 배치 크기 튜닝:
  너무 작음 (10건): HTTP 오버헤드 비율 높음
  너무 큼 (10만 건): 파싱 메모리 증가, GC 압박, 응답 지연
  권장: 5~15MB per bulk request (문서 수가 아닌 크기 기준)

  예시:
    평균 문서 크기 1KB → 배치 = 5,000~15,000건
    평균 문서 크기 10KB → 배치 = 500~1,500건

  최적 배치 크기 실험:
    100건 → 1000건 → 5000건으로 테스트
    인덱싱 처리량 (건/초) 측정 → 최댓값 지점 선택
```

### 2. refresh_interval 최적화

```
refresh 비용:
  1초마다 인메모리 버퍼 → 새 세그먼트 생성
  비용:
    새 세그먼트마다 FST(Term Dictionary) 구축
    OS Page Cache에 파일 생성
    TieredMergePolicy가 작은 세그먼트 병합 시작

  고빈도 인덱싱 시 문제:
    초당 10,000건 인덱싱 + refresh 1초마다
    1초마다 세그먼트 생성 → 분당 60개 세그먼트
    병합이 생성 속도를 따라가지 못함 → 세그먼트 폭발
    → 검색 느려짐, 병합 I/O 포화

refresh_interval: -1 (비활성화):
  자동 refresh 없음
  → 인메모리 버퍼에 문서 누적
  → 버퍼가 가득 찰 때만 세그먼트 생성 (큰 세그먼트)
  → 병합 부하 최소화

  트레이드오프:
    이득: 쓰기 성능 대폭 향상 (병합 부하 제거)
    비용: 검색 불가 (refresh 전까지)
    → 초기 적재, 배치 인덱싱에 적합
    → 실시간 검색 필요 서비스에는 부적합

refresh_interval 단계별 설정:
  실시간 검색 중요: 1s (기본)
  검색 지연 5초 허용: 5s
  검색 지연 30초 허용: 30s → 쓰기 성능 크게 향상
  초기 적재 only: -1 → 최대 성능

  권장:
    초당 인덱싱 건수 높으면 → refresh_interval 높이기
    (쓰기 성능 vs NRT 트레이드오프)
```

### 3. translog durability 설정

```
translog 역할 (Ch2-05 참고):
  모든 인덱싱 작업을 순서대로 기록
  노드 재시작 시 replay하여 데이터 복구

durability: request (기본값):
  매 인덱싱 요청마다 translog를 디스크에 fsync
  → 노드 재시작 시 마지막 요청까지 복구 가능
  → 데이터 손실 없음
  비용: 매 요청마다 fsync → I/O 집중 → 쓰기 속도 제한

durability: async:
  translog를 버퍼에 누적 → sync_interval마다 fsync
  → 기본 sync_interval: 5s
  이득: fsync 횟수 대폭 감소 → 쓰기 성능 향상
  위험: 노드 재시작 시 마지막 sync_interval 내 데이터 손실 가능
         예: sync_interval=60s → 최대 60초 데이터 손실

사용 가이드:
  durability: request (기본):
    금융, 주문 등 데이터 손실 절대 불허
    운영 중 인덱싱

  durability: async:
    초기 대량 적재 (적재 완료 후 복원)
    로그 데이터 (일부 손실 허용)
    검색 인덱스 (원본 DB에서 재인덱싱 가능)

  설정:
    PUT /myindex/_settings
    {
      "index.translog.durability": "async",
      "index.translog.sync_interval": "60s"
    }
```

### 4. 병렬 인덱싱 전략

```
병렬화 권장 방식:

  스레드 수 = Primary Shard 수

  이유:
    각 스레드가 다른 샤드를 담당
    → 샤드 간 병렬 인덱싱
    → Primary 수보다 스레드 많아도 추가 이득 없음 (병목은 샤드)

  예시:
    인덱스 Primary Shard 5개
    → 5개 스레드 병렬 Bulk 요청
    → 각 스레드가 특정 샤드 ID로 라우팅 또는 랜덤
    → 5개 샤드가 병렬로 인덱싱

  라우팅 고려:
    기본: hash(_id) % shard_count → 자동 분산
    커스텀: routing=shard_key → 특정 샤드에 집중

인덱싱 처리량 측정:
  GET _nodes/stats/indices/indexing?pretty
  nodes[*].indices.indexing.index_total
  nodes[*].indices.indexing.index_time_in_millis

  처리량 = (현재 index_total - 이전 index_total) / 측정 간격
  평균 지연 = index_time / index_total
```

---

## 💻 실전 실험

```bash
# 대규모 초기 적재 최적 설정
PUT /products/_settings
{
  "index.refresh_interval": "-1",
  "index.number_of_replicas": 0,
  "index.translog.durability": "async",
  "index.translog.sync_interval": "60s"
}

# Bulk API 기본 사용
POST /products/_bulk
{ "index": { "_id": "1" } }
{ "title": "Keyboard A", "price": 100000 }
{ "index": { "_id": "2" } }
{ "title": "Mouse B", "price": 50000 }
{ "delete": { "_id": "3" } }
{ "update": { "_id": "4" } }
{ "doc": { "price": 120000 } }

# 인덱싱 처리량 측정
GET _nodes/stats/indices/indexing?pretty

# 적재 완료 후 설정 복원
POST /products/_refresh
PUT /products/_settings
{
  "index.refresh_interval": "1s",
  "index.number_of_replicas": 1,
  "index.translog.durability": "request"
}

# 쓰기 성능 비교 실험
# refresh_interval 변화에 따른 처리량 비교
PUT /test-write/_settings { "index.refresh_interval": "1s" }
# bulk 인덱싱 1만 건 → 처리량 측정

PUT /test-write/_settings { "index.refresh_interval": "30s" }
# 동일 데이터 → 처리량 비교

# translog 크기 확인
GET _nodes/stats/indices/translog?pretty
# translog.operations: translog에 쌓인 작업 수
# translog.size_in_bytes: 현재 translog 크기

# 인덱싱 대기열 확인 (병목 징후)
GET _nodes/stats/thread_pool/write?pretty
# thread_pool.write.queue: 대기 중인 인덱싱 작업 수
# queue가 높으면 → 인덱싱 병목 (노드/샤드 증설 필요)
```

---

## 📊 쓰기 최적화 설정 조합

| 시나리오 | refresh_interval | replicas | translog | 예상 성능 |
|---------|-----------------|---------|---------|---------|
| 초기 대량 적재 | -1 | 0 | async 60s | 최대 |
| 로그 수집 (손실 허용) | 30s | 1 | async 5s | 높음 |
| 일반 운영 (NRT 1초) | 1s | 1 | request | 기본 |
| 고내구성 운영 | 1s | 2 | request | 낮음 |

---

## ⚖️ 트레이드오프

```
refresh_interval:
  짧게 → 실시간 검색 가능, 세그먼트 병합 부하 증가
  길게 → 쓰기 성능 향상, 검색 지연 증가

translog durability:
  request → 데이터 손실 없음, fsync 비용
  async   → 쓰기 성능 향상, 노드 재시작 시 최대 sync_interval 손실

레플리카:
  0개 → 최대 쓰기 성능, 장애 시 데이터 손실 위험
  1개 → 장애 내성, 쓰기 비용 2배
  → 초기 적재: 0개 → 완료 후 1개 (복원 시 자동 복제)

Bulk 배치 크기:
  너무 작음 → HTTP 오버헤드 높음
  너무 큼  → 메모리 부담, 단일 요청 실패 시 재처리 비용
  5~15MB  → 대부분 최적
```

---

## 📌 핵심 정리

```
Bulk API:
  단건 대비 10~100배 빠름 (HTTP 왕복 감소)
  배치 크기: 5~15MB 권장
  스레드 수: Primary Shard 수와 동일

초기 적재 설정:
  refresh_interval: -1 (병합 부하 제거)
  number_of_replicas: 0 (복제 비용 제거)
  translog durability: async (fsync 비용 제거)
  완료 후 복원 + POST /_refresh

운영 중 쓰기 최적화:
  refresh_interval: 5~30s (NRT 허용 범위에서 최대한 길게)
  translog: request (내구성 유지)
  Bulk API 필수 (단건 인덱싱 금지)

쓰기 병목 진단:
  thread_pool.write.queue 높음 → 인덱싱 병목
  indices.merges.current 높음 → 세그먼트 병합 병목
  힙 높음 → fielddata/GC 문제
```

---

## 🤔 생각해볼 문제

**Q1.** `refresh_interval: -1` 설정 후 인덱싱된 문서가 검색되지 않는다. 검색 가능하게 만들려면?

<details>
<summary>해설 보기</summary>

`POST /index/_refresh`를 호출하면 인메모리 버퍼에 있는 문서들이 세그먼트로 변환되어 검색 가능해진다. 또는 특정 인덱싱 요청에 `?refresh=wait_for` 또는 `?refresh=true`를 추가하면 해당 요청이 완료되고 검색 가능해진 후 응답한다. 단, `?refresh=true`는 해당 샤드에서 즉시 refresh를 트리거하므로 성능에 영향을 줄 수 있어 프로덕션에서는 피하는 것이 좋다.

</details>

**Q2.** Bulk API 요청에서 일부 문서 인덱싱이 실패하면 전체 요청이 롤백되는가?

<details>
<summary>해설 보기</summary>

아니다. Bulk API는 원자적으로 동작하지 않는다. 1,000건 중 10건이 실패하면 나머지 990건은 인덱싱되고 10건만 실패 응답을 포함한다. 응답의 `items` 배열에서 각 문서의 성공/실패 상태를 확인할 수 있다. 실패한 문서만 골라서 재시도하는 로직이 필요하다. 이것이 Bulk API가 ACID 트랜잭션이 아닌 Best-Effort 방식으로 동작하는 이유다. 실패 처리 로직 없이 Bulk 응답을 무시하면 데이터 누락이 발생할 수 있다.

</details>

**Q3.** 초기 적재 완료 후 `number_of_replicas: 1`로 복원하면 ES는 어떻게 Replica를 채우는가?

<details>
<summary>해설 보기</summary>

설정 변경 후 Master 노드가 Replica Shard를 각 노드에 배치한다. 빈 Replica Shard가 Primary로부터 데이터를 복사하는 과정(Recovery)이 시작된다. 이 과정에서 Primary의 세그먼트 파일을 네트워크로 전송한다. 인덱스 크기에 따라 수 분 ~ 수 시간이 소요되며, 이 동안 클러스터는 Yellow 상태다. Recovery 진행 상황은 `GET _cat/recovery?v&active_only=true`로 확인한다. Recovery 완료 후 Green 상태가 된다.

</details>

---

<div align="center">

**[⬅️ 이전: 힙 메모리 튜닝](./03-heap-memory-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: 검색 성능 최적화 ➡️](./05-search-performance.md)**

</div>
