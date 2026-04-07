# 집계 아키텍처 — Bucket·Metric·Pipeline 계층과 분산 실행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bucket, Metric, Pipeline 집계는 각각 무엇을 하고 어떻게 조합되는가?
- 집계가 각 샤드에서 부분 결과를 계산하고 Coordinator에서 병합되는 과정은?
- 쿼리와 집계가 동시에 실행될 때 처리 순서와 `size: 0`의 의미는?
- 집계 결과가 검색 결과와 달리 `_score` 없이 반환되는 이유는?
- 중첩 집계(sub-aggregation)는 내부에서 어떻게 처리되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대시보드, 통계 리포트, 실시간 분석 — 이 모두가 ES Aggregation으로 구현된다. 집계가 느려질 때 "샤드를 늘리면 빨라지는가", "인덱스 크기를 줄여야 하는가", "캐시가 적용되고 있는가"를 판단하려면 집계 아키텍처를 알아야 한다. 분산 집계의 두 단계(샤드별 부분 계산 + Coordinator 병합)를 모르면 Terms 집계 오차, 느린 집계, OOM 원인 모두 미궁에 빠진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 집계에 size: 10000을 기대
  요청:
    { "size": 10000, "aggs": { "by_cat": { "terms": { "field": "category" } } } }

  오해:
    "size: 10000이니 10,000개 문서를 보고 집계하겠지"

  실제:
    size = 검색 hits 반환 수 (기본 10)
    집계는 size와 무관하게 모든 문서를 처리
    → 집계만 필요하면 size: 0 설정 (hits 반환 생략)
    → size: 10000은 10,000개 문서 본문을 가져오는 Fetch 비용만 추가

실수 2: Pipeline 집계를 Bucket 집계와 혼동
  오해:
    "moving_avg는 날짜별 집계 안에서 바로 쓸 수 있다"

  실제:
    Pipeline 집계는 다른 집계의 결과를 입력으로 사용
    먼저 Date Histogram(Bucket)으로 월별 합계를 구한 후
    그 결과에 moving_avg(Pipeline)를 적용하는 2단계 구조

실수 3: 중첩 집계가 카테시안 곱이라고 오해
  요청:
    { "aggs": {
      "by_cat": { "terms": { "field": "category" } },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }}

  오해: "by_cat × avg_price = 모든 조합 계산"
  실제: 각 category 버킷 내에서 avg_price 계산
        → by_cat이 10개 카테고리 버킷을 생성하면
           avg_price는 각 버킷 내에서 독립적으로 계산
        → 카테시안 곱이 아닌 "각 버킷 내 집계"
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
집계 설계 원칙:

  집계만 필요할 때 → size: 0
    { "size": 0, "aggs": { ... } }
    → Fetch Phase 생략 → 집계 결과만 빠르게 반환

  집계 계층 설계:
    Bucket  → "무엇으로 그룹핑할 것인가"
    Metric  → "각 그룹 내에서 무엇을 계산할 것인가"
    Pipeline → "집계 결과를 다시 가공할 것인가"

  분산 집계의 특성 이해:
    각 샤드에서 부분 결과 → Coordinator에서 병합
    → Terms 집계: 샤드별 top-N 반환 → 병합 시 오차 가능
    → Avg/Sum: 부분 합산 후 결합 → 오차 없음
    → Cardinality: HyperLogLog++ 확률적 추정 → 오차 있음

  캐시 활용:
    동일 집계 조건 반복 → Request Cache 활용
    filter 집계 → Filter Cache(Query Cache) 활용
```

---

## 🔬 내부 동작 원리

### 1. 집계 3계층 구조

```
┌──────────────────────────────────────────────────────────┐
│                   Pipeline Aggregation                   │
│  (다른 집계의 결과를 입력으로 받아 추가 계산)                      │
│                                                          │
│  moving_avg, derivative, cumulative_sum,                 │
│  bucket_selector, bucket_sort                            │
│                                                          │
│  입력: 다른 Bucket/Metric 집계의 결과값                        │
│  출력: 가공된 통계값                                          │
└──────────────────────────────┬───────────────────────────┘
                               │ 결과를 입력으로
┌──────────────────────────────┴───────────────────────────┐
│                    Bucket Aggregation                    │
│  (문서를 기준에 따라 그룹(버킷)으로 분류)                         │
│                                                          │
│  terms      → 필드 값별 그룹                                │
│  date_histogram → 시간 구간별 그룹                           │
│  histogram  → 숫자 구간별 그룹                               │
│  range      → 명시적 범위별 그룹                             │
│  filter     → 특정 조건 문서 그룹                            │
│  geo_grid   → 지리 격자 그룹                                │
│                                                          │
│  각 버킷: doc_count + 하위 집계 결과                          │
└──────────────────────────────┬───────────────────────────┘
                               │ 버킷 내 계산
┌──────────────────────────────┴───────────────────────────┐
│                    Metric Aggregation                    │
│  (문서 집합에서 수치를 계산)                                   │
│                                                          │
│  avg, sum, min, max, count    → 단일 수치                  │
│  stats, extended_stats         → 여러 수치 한번에            │
│  percentiles, percentile_ranks → 분위수                    │
│  cardinality                   → 고유값 수 (HyperLogLog)   │
│  top_hits                      → 상위 문서 샘플             │
│                                                          │
│  입력: 버킷 내 문서 집합 또는 전체 문서 집합                       │
│  출력: 단일 숫자 또는 수치 집합                                 │
└──────────────────────────────────────────────────────────┘
```

### 2. 분산 집계 실행 흐름

```
요청:
  GET /orders/_search
  {
    "size": 0,
    "aggs": {
      "by_category": {
        "terms": { "field": "category" },
        "aggs": {
          "total_revenue": { "sum": { "field": "amount" } },
          "avg_price":    { "avg": { "field": "price" } }
        }
      }
    }
  }

Phase 1: 각 샤드에서 부분 집계 실행

  Shard 0 (100개 문서):
    category별 그룹 + sum(amount) + avg(price) 계산
    결과:
      keyboard: {doc_count: 30, sum_amount: 4,500,000, avg_price: 150,000}
      mouse:    {doc_count: 20, sum_amount: 1,600,000, avg_price:  80,000}
      monitor:  {doc_count: 50, sum_amount: 17,500,000, avg_price: 350,000}

  Shard 1 (100개 문서):
    keyboard: {doc_count: 25, sum_amount: 3,750,000, avg_price: 150,000}
    mouse:    {doc_count: 35, sum_amount: 2,800,000, avg_price:  80,000}
    headset:  {doc_count: 40, sum_amount: 3,200,000, avg_price:  80,000}

  Shard 2 (100개 문서):
    keyboard: {doc_count: 45, sum_amount: 6,750,000, avg_price: 150,000}
    monitor:  {doc_count: 30, sum_amount: 10,500,000, avg_price: 350,000}
    headset:  {doc_count: 25, sum_amount: 2,000,000, avg_price:  80,000}

Phase 2: Coordinator에서 병합

  keyboard:
    doc_count = 30 + 25 + 45 = 100
    sum_amount = 4,500,000 + 3,750,000 + 6,750,000 = 15,000,000
    avg_price: 단순 합산 불가 → 가중 평균 필요
      = (30×150,000 + 25×150,000 + 45×150,000) / (30+25+45) = 150,000

  mouse:
    doc_count = 20 + 35 = 55 (Shard 2에는 없음)
    sum_amount = 1,600,000 + 2,800,000 = 4,400,000

  → 샤드별 부분 결과를 Coordinator가 카테고리별로 합산

병합 방식 차이:
  sum, count: 단순 합산 → 오차 없음
  avg:        가중 평균 (문서 수 × 평균값 합산 후 총 문서 수 나눔) → 오차 없음
  terms:      각 샤드의 상위 N개만 반환 → 병합 시 누락 가능 (Ch5-02 상세)
  cardinality: HyperLogLog++ 스케치 병합 → 확률적 오차
```

### 3. 쿼리 + 집계 동시 실행 처리 순서

```
요청:
  {
    "size": 10,
    "query": { "bool": { "filter": [
      { "term": { "category": "keyboard" } },
      { "range": { "price": { "gte": 100000 } } }
    ]}},
    "aggs": {
      "avg_price": { "avg": { "field": "price" } }
    }
  }

처리 순서:
  ① 쿼리 실행: filter 조건으로 문서 필터링
     → keyboard 카테고리 + price >= 100000 문서 집합 선별

  ② 집계 실행: 필터링된 문서 집합에 대해 avg(price) 계산
     → 검색 결과에 포함된 문서만 집계 대상
     (global 집계 제외)

  ③ Query Phase: 상위 10개 문서 ID + 점수 수집
  ④ Merge Phase: 전체 상위 10개 선정
  ⑤ Fetch Phase: 10개 문서 본문 가져오기
  ⑥ 집계 결과 병합

  핵심: 집계는 쿼리의 필터링 결과를 기준으로 실행
        (쿼리 조건을 만족하는 문서에서만 집계)

global 집계:
  쿼리 필터를 무시하고 전체 문서에 집계
  {
    "query": { "term": { "category": "keyboard" } },
    "aggs": {
      "all_avg_price": {
        "global": {},
        "aggs": { "avg": { "avg": { "field": "price" } } }
      }
    }
  }
  → keyboard만 검색 결과로 반환
  → 하지만 avg는 전체 인덱스의 평균가격

size: 0의 의미:
  hits를 반환하지 않음 (빈 배열)
  집계 결과만 필요할 때 사용
  → Query Phase 비용 최소화 (점수 계산 생략)
  → Fetch Phase 완전 생략
  → 집계 처리량 향상
```

### 4. 중첩 집계(Sub-Aggregation) 실행 원리

```
중첩 집계:
  {
    "aggs": {
      "by_category": {         ← Level 1 (Bucket)
        "terms": { "field": "category", "size": 5 },
        "aggs": {
          "monthly": {          ← Level 2 (Bucket, 각 category 버킷 내)
            "date_histogram": { "field": "created_at", "calendar_interval": "month" },
            "aggs": {
              "revenue": {      ← Level 3 (Metric, 각 month 버킷 내)
                "sum": { "field": "amount" }
              }
            }
          }
        }
      }
    }
  }

처리 순서:
  1. terms(category) → keyboard 버킷, mouse 버킷, monitor 버킷, ...
  2. 각 category 버킷 내에서 date_histogram(month) 실행
     keyboard 버킷 → 2024-01, 2024-02, 2024-03 버킷
     mouse 버킷    → 2024-01, 2024-02 버킷
  3. 각 month 버킷 내에서 sum(amount) 계산

결과 구조:
  aggregations:
    by_category:
      buckets:
        - key: "keyboard"
          doc_count: 100
          monthly:
            buckets:
              - key_as_string: "2024-01"
                doc_count: 35
                revenue: { value: 5,250,000 }
              - key_as_string: "2024-02"
                ...

메모리 비용:
  Level 1: 5 버킷
  Level 2: 5 카테고리 × 3 월 = 15 버킷
  Level 3: 15 버킷에서 각각 sum 계산
  → 중첩이 깊어질수록 버킷 수 폭발적 증가
  → 메모리 주의: 각 버킷은 힙에 상주
```

### 5. Aggregation Request Cache

```
Request Cache (집계 캐시):

  대상:
    size: 0 요청 (hits 없는 순수 집계)
    샤드 레벨 집계 결과를 캐시

  캐시 키:
    요청 본문(JSON) 전체 + 샤드 ID
    → 동일 요청 → 동일 캐시 키 → 캐시 HIT

  캐시 무효화:
    세그먼트가 refresh 될 때 해당 샤드의 캐시 무효화
    → 새 문서 인덱싱 후 refresh 시 캐시 재계산

  효과:
    반복적인 대시보드 쿼리 (매 분/시간 동일 집계)
    → 데이터 변경 없으면 캐시 재사용 → 빠른 응답

  설정:
    indices.requests.cache.size: 1% (힙의 1%, 기본값)
    GET /index/_settings → index.requests.cache.enable: true

  확인:
    GET _nodes/stats/indices/request_cache?pretty
    → hit_count, miss_count, memory_size_in_bytes

  캐시 비활성화:
    { "request_cache": false, "aggs": { ... } }
    → 실시간성이 중요한 집계에 사용
```

---

## 💻 실전 실험

```bash
# 집계 기본 구조 실험
PUT /orders
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "category": { "type": "keyword" },
      "amount":   { "type": "double" },
      "price":    { "type": "double" },
      "quantity": { "type": "integer" },
      "created_at": { "type": "date" }
    }
  }
}

POST /orders/_bulk
{ "index": {} }
{ "category": "keyboard", "amount": 150000, "price": 150000, "quantity": 1, "created_at": "2024-01-15" }
{ "index": {} }
{ "category": "mouse",    "amount":  80000, "price":  80000, "quantity": 1, "created_at": "2024-01-20" }
{ "index": {} }
{ "category": "keyboard", "amount": 300000, "price": 150000, "quantity": 2, "created_at": "2024-02-10" }
{ "index": {} }
{ "category": "monitor",  "amount": 350000, "price": 350000, "quantity": 1, "created_at": "2024-02-15" }
POST /orders/_refresh

# Bucket + Metric 조합
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "total_amount": { "sum":  { "field": "amount" } },
        "avg_price":    { "avg":  { "field": "price" } },
        "max_amount":   { "max":  { "field": "amount" } }
      }
    }
  }
}

# Pipeline 집계 (Date Histogram + 누적합)
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "monthly": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_revenue": { "sum": { "field": "amount" } },
        "cumulative_revenue": {
          "cumulative_sum": {
            "buckets_path": "monthly_revenue"
          }
        }
      }
    }
  }
}

# global 집계 (쿼리 필터 무시)
GET /orders/_search
{
  "size": 0,
  "query": { "term": { "category": "keyboard" } },
  "aggs": {
    "keyboard_avg": { "avg": { "field": "price" } },
    "overall_avg": {
      "global": {},
      "aggs": { "all_avg": { "avg": { "field": "price" } } }
    }
  }
}
# keyboard_avg: keyboard만의 평균
# overall_avg.all_avg: 전체 평균

# Request Cache 통계 확인
GET _nodes/stats/indices/request_cache?pretty
# hit_count, miss_count 비율로 캐시 효과 확인
```

---

## 📊 집계 유형별 분산 병합 방식

| 집계 유형 | 샤드에서 반환 | Coordinator 병합 | 오차 여부 |
|---------|------------|----------------|---------|
| `sum` | 부분 합계 | 단순 합산 | 없음 |
| `avg` | (합계, 건수) | 가중 평균 | 없음 |
| `min`, `max` | 부분 최솟/최댓값 | 전체 최솟/최댓값 | 없음 |
| `terms` | 상위 N개 | 병합 후 상위 K개 | **있음** |
| `cardinality` | HLL++ 스케치 | 스케치 병합 | 있음 (약 ±5%) |
| `date_histogram` | 버킷별 카운트 | 버킷별 합산 | 없음 |
| `percentiles` | TDigest 스케치 | 스케치 병합 | 있음 (약 ±1%) |

---

## ⚖️ 트레이드오프

```
중첩 집계 깊이:
  깊을수록: 세밀한 분석 가능
  깊을수록: 버킷 수 폭발 → 메모리 증가 → 힙 압박 → OOM 위험
  → 3단계 이상 중첩은 신중하게 (버킷 수 사전 계산)

집계 버킷 수:
  terms size 크게 → 더 많은 카테고리 반환 → 메모리 증가
  terms size 작게 → 빠르고 작은 메모리 → 상위 N개만

Request Cache:
  refresh_interval 길게 → 캐시 지속 시간 증가 → 캐시 효과↑
  refresh_interval 짧게 → 캐시 잦은 무효화 → 캐시 효과↓

분산 집계 정확도:
  정확도 높은 집계(sum, avg) → 문제없음
  정확도 낮은 집계(terms, cardinality) → 오차 고려
  → 서비스에 허용 가능한 오차 수준 판단 필요
```

---

## 📌 핵심 정리

```
집계 3계층:
  Bucket  → 문서를 기준으로 그룹 분류 (terms, date_histogram, range)
  Metric  → 각 그룹 내 수치 계산 (avg, sum, min, max, cardinality)
  Pipeline → 다른 집계 결과를 입력으로 가공 (moving_avg, cumulative_sum)

분산 집계 2페이즈:
  Phase 1: 각 샤드에서 부분 집계 계산
  Phase 2: Coordinator에서 부분 결과 병합
  → sum/avg: 오차 없음 / terms/cardinality: 오차 가능

size: 0:
  hits 반환 생략 → 집계만 빠르게
  Query Phase 최소화, Fetch Phase 생략

Request Cache:
  size: 0 요청 + 데이터 변경 없음 → 샤드 레벨 캐시 재사용
  refresh 시 무효화 → 실시간성 필요 시 비활성화
```

---

## 🤔 생각해볼 문제

**Q1.** 집계 요청에서 `size: 0`을 설정하지 않으면 어떤 추가 비용이 발생하는가?

<details>
<summary>해설 보기</summary>

`size: 0`이 없으면 기본값 `size: 10`이 적용되어 Query Phase에서 상위 10개 문서를 선정하고(점수 계산 포함) Fetch Phase에서 10개 문서의 `_source`를 가져온다. 집계 결과만 필요한 경우 이 두 단계는 완전히 낭비다. 특히 대규모 인덱스에서 Query Phase의 점수 계산과 Fetch Phase의 I/O 비용이 집계 비용보다 클 수 있다. 항상 집계 전용 요청에는 `size: 0`을 명시하는 것이 권장된다.

</details>

**Q2.** `cardinality` 집계가 정확한 고유값 수 대신 근사값을 반환하는 이유는?

<details>
<summary>해설 보기</summary>

정확한 카디널리티 계산은 모든 고유값을 메모리에 올려 비교해야 한다. 예를 들어 1억 개의 고유 사용자 ID를 저장하려면 수 GB의 메모리가 필요하다. HyperLogLog++ 알고리즘은 확률적 데이터 구조로, 1KB 미만의 고정 메모리로 오차율 약 5% 이내에서 카디널리티를 추정한다. `precision_threshold` 설정(기본 3000)을 높이면 정확도를 높일 수 있지만 메모리 사용이 증가한다. 대용량 데이터에서 정확한 카디널리티보다 근사값이 실용적인 이유가 여기 있다.

</details>

**Q3.** 동일한 집계 요청을 1분마다 반복 실행할 때 Request Cache가 효과를 내려면 어떤 조건이 충족되어야 하는가?

<details>
<summary>해설 보기</summary>

Request Cache는 요청 JSON이 동일하고, 해당 요청 이후 해당 샤드에서 refresh가 발생하지 않은 경우에만 캐시 HIT이 된다. 따라서 1분마다 새 데이터가 인덱싱되고 refresh_interval이 1초이면 매 요청마다 캐시가 무효화되어 효과가 없다. 반면 `refresh_interval: 5m`으로 설정하거나, 과거 데이터(변경 없는 인덱스)에 대한 집계라면 캐시가 계속 살아있어 효과적이다. 대시보드에서 실시간성이 1~5분 정도 허용된다면 refresh_interval을 그에 맞게 설정해 캐시 효율을 높이는 것이 권장된다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Terms Aggregation 내부 ➡️](./02-terms-aggregation.md)**

</div>
