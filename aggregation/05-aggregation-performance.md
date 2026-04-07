# 집계 성능 최적화 — Filter Aggregation·global_ordinals·pre-aggregation

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Filter Aggregation이 Query Cache(Filter Cache)를 활용해 반복 집계를 가속하는 원리는?
- `eager_global_ordinals`는 무엇을 미리 준비하고 어떻게 Terms 집계를 빠르게 만드는가?
- Rollup과 Transform의 차이는 무엇이고 각각 어떤 상황에 적합한가?
- 대시보드 집계를 빠르게 하는 실용적인 캐시 전략은 무엇인가?
- 집계 성능 문제를 진단하는 절차는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"집계 쿼리가 5초 걸린다"는 문제를 인덱스 크기 탓으로만 돌리기 쉽다. 하지만 실제로는 Filter Aggregation으로 캐시를 활용하거나, `eager_global_ordinals`로 Terms 집계 초기화 비용을 제거하거나, 자주 보는 통계를 Transform으로 사전 집계해두면 50ms 이내로 줄일 수 있는 경우가 많다. 이 문서의 최적화 기법은 인덱스 재설계 없이 쿼리 레벨에서 적용 가능하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 동일 기간의 여러 집계를 따로 요청
  패턴:
    Request 1: "이번 달 카테고리별 매출"
    Request 2: "이번 달 평균 주문 금액"
    Request 3: "이번 달 신규 사용자 수"

  문제:
    3번의 독립적인 집계 요청
    각각 filter(이번 달) + 집계 실행
    filter 조건이 동일해도 각 요청마다 재계산

  올바른 접근:
    하나의 요청에 여러 agg를 묶어서 전송
    { "size": 0, "query": { "range": { "created_at": ... } },
      "aggs": {
        "by_category":  { "terms": ... },
        "avg_amount":   { "avg": ... },
        "new_users":    { "cardinality": { "field": "user_id" } }
      }}
    → 단일 패스로 3개 집계 모두 계산

실수 2: Filter Aggregation 대신 Query로 필터링
  패턴:
    Request 1: "카테고리=keyboard인 주문 집계"
    { "query": { "term": { "category": "keyboard" } }, "aggs": {...} }

    Request 2: "카테고리=mouse인 주문 집계"
    { "query": { "term": { "category": "mouse" } }, "aggs": {...} }

  문제:
    각 요청의 쿼리 부분이 Query Context → Filter Cache 미적용
    (term이지만 query에 있으면 Query Context)

  올바른 접근 (Filter Aggregation):
    { "size": 0, "aggs": {
      "keyboard": { "filter": { "term": { "category": "keyboard" } },
                   "aggs": { "revenue": { "sum": { "field": "amount" } } } },
      "mouse":    { "filter": { "term": { "category": "mouse" } },
                   "aggs": { "revenue": { "sum": { "field": "amount" } } } }
    }}
    → filter 비트셋이 Filter Cache에 캐시 → 반복 요청 시 재사용

실수 3: global_ordinals가 매번 재생성된다는 사실을 모름
  상황:
    Terms 집계를 매 분 실행
    처음 10초, 그 다음 1초, 이후 0.5초로 빨라지는 패턴 발견

  이유:
    Terms 집계 첫 실행: global_ordinals 구축 (느림)
    이후: 캐시된 global_ordinals 재사용 (빠름)
    문제: refresh 후 global_ordinals 무효화 → 재구축

  해결:
    eager_global_ordinals: true
    → refresh 시 global_ordinals 미리 재구축
    → 집계 요청 시 항상 준비된 상태
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
집계 성능 최적화 체크리스트:

  □ 여러 집계를 단일 요청으로 묶기 (단일 패스)

  □ Filter Aggregation 사용 (캐시 활용):
    반복적인 카테고리/상태 기반 집계
    → filter 조건을 Filter Cache에 캐시

  □ eager_global_ordinals 설정 (Terms 집계 warm-up):
    "category": { "type": "keyword",
                  "eager_global_ordinals": true }
    → refresh 시 자동 global_ordinals 구축

  □ Request Cache 활용:
    size: 0 요청 + 데이터 변경 없음
    → 샤드 레벨 집계 결과 캐시

  □ 인덱스 롤오버 + ILM:
    과거 인덱스는 read-only → 불변 → 캐시 효과 극대화

  □ Transform/Rollup으로 사전 집계:
    자주 보는 통계는 미리 집계해 별도 인덱스에 저장
    → 실시간 집계 비용 없음

  □ 인덱스 분리 전략:
    집계 대상 데이터를 별도 인덱스로 분리
    → 집계 전용 최적화 매핑 적용 가능
```

---

## 🔬 내부 동작 원리

### 1. Filter Aggregation과 Query Cache

```
Filter Aggregation:

  { "size": 0, "aggs": {
    "keyboard_sales": {
      "filter": { "term": { "category": "keyboard" } },
      "aggs": {
        "total":   { "sum":  { "field": "amount" } },
        "average": { "avg":  { "field": "amount" } }
      }
    },
    "mouse_sales": {
      "filter": { "term": { "category": "mouse" } },
      "aggs": {
        "total":   { "sum":  { "field": "amount" } },
        "average": { "avg":  { "field": "amount" } }
      }
    }
  }}

  내부 처리:
    ① "category=keyboard" filter
       → Filter Cache 조회
       → MISS: 비트셋 생성 + 캐시 저장
       → HIT: 비트셋 즉시 반환 (수 μs)

    ② keyboard 비트셋으로 마스킹 후 sum, avg 계산
    ③ "category=mouse" filter → Filter Cache 조회 → 동일
    ④ mouse 마스킹 후 sum, avg 계산

  캐시 효과:
    첫 번째 요청: 비트셋 생성 → 캐시 저장
    이후 요청:    캐시 HIT → 비트셋 즉시 재사용
    → 반복 집계에서 필터링 비용 ≈ 0

  vs Query Context(must)의 차이:
    { "query": { "term": { "category": "keyboard" } }, "aggs": {...} }
    → Query Context → 점수 계산 포함 → Filter Cache 미적용
    → 반복 요청에도 매번 재계산

  filters 집계 (여러 필터를 한 번에):
    { "aggs": {
      "multi_cat": {
        "filters": {
          "filters": {
            "keyboard": { "term": { "category": "keyboard" } },
            "mouse":    { "term": { "category": "mouse" } }
          }
        },
        "aggs": { "total": { "sum": { "field": "amount" } } }
      }
    }}
    → 각 filter가 독립적으로 캐시
    → 단일 패스로 여러 필터 집계 실행
```

### 2. Global Ordinals — Terms 집계 최적화

```
Global Ordinals란:
  keyword 필드의 모든 고유값(카테고리 등)을 정수 ID로 매핑한 사전
  Terms 집계의 핵심 자료구조

  예시:
    keyboard → 0
    monitor  → 1
    mouse    → 2

  역할:
    문서가 어느 카테고리에 속하는지 문자열 비교 대신 정수 비교로 처리
    → 비교 속도 대폭 향상
    → 메모리 효율 (정수가 문자열보다 작음)

Global Ordinals 구축 비용:
  세그먼트별 로컬 Ordinals → 전역 Global Ordinals로 변환
  → 전체 인덱스의 keyword 필드 모든 고유값 수집 + 정렬 + 번호 부여
  → 처음 Terms 집계 시 구축 (수십 ms ~ 수 초)
  → 세그먼트가 변경되면(refresh 시) 무효화 → 재구축 필요

eager_global_ordinals 설정:

  "category": {
    "type": "keyword",
    "eager_global_ordinals": true
  }

  효과:
    refresh 발생 시 → 새 세그먼트 생성 → global_ordinals 즉시 재구축
    다음 Terms 집계 요청 시 → 이미 준비된 global_ordinals 사용 → 빠름

  vs eager_global_ordinals: false (기본):
    refresh 발생 → global_ordinals 재구축 안 함
    첫 Terms 집계 요청 → global_ordinals 구축 (느림)
    이후 요청 → 캐시 재사용 (빠름)
    다음 refresh → 다시 첫 요청이 느림

  적합한 필드:
    Terms 집계 자주 사용 + refresh 빈번한 필드
    카테고리, 상태, 태그 등

  부적합한 필드:
    고유값이 매우 많은 필드 (user_id 등)
    → global_ordinals 크기 폭발
    refresh 드문 정적 인덱스
    → 이미 빌드된 global_ordinals 오래 유지 → 효과 없음

  모니터링:
    GET _nodes/stats/indices/segments?pretty
    → segments.ordinals_memory_in_bytes: 현재 ordinals 메모리

확인:
  GET /index/_stats/fielddata?pretty
  → "ordinals" 관련 통계 확인
```

### 3. Transform API — 사전 집계

```
Transform vs Rollup:

  Rollup (레거시, ES 8.x에서 deprecated):
    주기적으로 원본 데이터를 집계해 롤업 인덱스 저장
    별도 Rollup 쿼리 API 사용
    → 새 Transform API로 대체 권장

  Transform API (권장):
    원본 인덱스 → 집계 처리 → 대상 인덱스
    scheduled(주기 실행) 또는 continuous(실시간) 모드

  Transform 설정 예시:
    POST _transform/daily_revenue_transform
    {
      "source": {
        "index": "orders",
        "query": { "range": { "created_at": { "gte": "now-7d" } } }
      },
      "dest": { "index": "daily_revenue" },
      "sync": {
        "time": { "field": "created_at", "delay": "1m" }
      },
      "pivot": {
        "group_by": {
          "date":     { "date_histogram": { "field": "created_at", "calendar_interval": "day" } },
          "category": { "terms": { "field": "category" } }
        },
        "aggregations": {
          "total_revenue": { "sum": { "field": "amount" } },
          "order_count":   { "value_count": { "field": "_id" } },
          "avg_amount":    { "avg": { "field": "amount" } }
        }
      }
    }

  결과 (daily_revenue 인덱스):
    { "date": "2024-01-15", "category": "keyboard", "total_revenue": 4500000, "order_count": 30, "avg_amount": 150000 }
    { "date": "2024-01-15", "category": "mouse",    "total_revenue": 1600000, "order_count": 20, "avg_amount": 80000 }
    ...

  대시보드에서:
    daily_revenue 인덱스에서 단순 sum/avg 집계
    → 원본 orders 인덱스(수백만 건) 탐색 불필요
    → 사전 집계 인덱스(수천 건) 탐색 → 수십 ms

  적합한 상황:
    동일한 통계를 반복적으로 조회 (대시보드)
    집계가 느리지만 실시간성이 1~10분 지연 허용
    원본 데이터가 크고 집계 결과가 작을 때

  부적합한 상황:
    실시간(초 단위) 집계 필요
    집계 조건이 매번 다른 경우 (임시 분석)
```

### 4. 집계 성능 진단 절차

```
Step 1: _profile로 집계 병목 확인
  GET /orders/_search
  {
    "profile": true,
    "size": 0,
    "aggs": { "by_category": { "terms": { "field": "category" } } }
  }

  profile 결과:
    aggregations 섹션에서 각 집계의 실행 시간 확인
    집계 초기화, 수집, 병합 단계별 시간 분석

Step 2: Request Cache 히트율 확인
  GET _nodes/stats/indices/request_cache?pretty
  hit_count / (hit_count + miss_count) = 히트율

Step 3: fielddata 메모리 확인
  GET _nodes/stats/indices/fielddata?pretty
  → 과도한 fielddata → keyword 필드 대체 필요

Step 4: Global Ordinals 상태 확인
  GET _nodes/stats/indices/segments?pretty
  ordinals_memory_in_bytes → 높으면 많은 ordinals 구축

Step 5: slowlog로 느린 집계 식별
  PUT /orders/_settings
  {
    "index.search.slowlog.threshold.query.warn": "1s",
    "index.search.slowlog.threshold.query.info": "500ms"
  }
  → 1초 이상 집계 쿼리 자동 로그

Step 6: 최적화 적용 후 비교
  Profile 재실행 → 개선 폭 확인
  Request Cache 히트율 변화 확인
```

---

## 💻 실전 실험

```bash
# Filter Aggregation vs Query로 필터링 비교
# Filter Aggregation (캐시 활용)
GET /orders/_search
{
  "profile": true,
  "size": 0,
  "aggs": {
    "keyboard": {
      "filter": { "term": { "category": "keyboard" } },
      "aggs": { "total": { "sum": { "field": "amount" } } }
    },
    "mouse": {
      "filter": { "term": { "category": "mouse" } },
      "aggs": { "total": { "sum": { "field": "amount" } } }
    }
  }
}
# 두 번 실행 후 profile 시간 비교 (두 번째가 빠름 = 캐시 효과)

# eager_global_ordinals 설정
PUT /orders/_mapping
{
  "properties": {
    "category": {
      "type": "keyword",
      "eager_global_ordinals": true
    }
  }
}

# refresh 후 Terms 집계 속도 변화 확인
POST /orders/_refresh

# global_ordinals 미리 구축되어 있으므로 첫 요청도 빠름
GET /orders/_search
{
  "profile": true,
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category" } } }
}

# Transform 생성 (실제 적용 예시)
PUT _transform/category_daily_transform
{
  "source": { "index": "orders" },
  "dest":   { "index": "orders_daily_summary" },
  "pivot": {
    "group_by": {
      "date":     { "date_histogram": { "field": "created_at", "calendar_interval": "day" } },
      "category": { "terms": { "field": "category" } }
    },
    "aggregations": {
      "total":   { "sum":   { "field": "amount" } },
      "count":   { "value_count": { "field": "_id" } },
      "avg_amt": { "avg":   { "field": "amount" } }
    }
  },
  "frequency": "1m",
  "sync": { "time": { "field": "created_at", "delay": "1m" } }
}

# Transform 시작
POST _transform/category_daily_transform/_start

# Transform 상태 확인
GET _transform/category_daily_transform/_stats

# 사전 집계 인덱스에서 집계 (빠름)
GET /orders_daily_summary/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": { "revenue": { "sum": { "field": "total" } } }
    }
  }
}
```

---

## 📊 집계 성능 최적화 기법 비교

| 기법 | 적용 대상 | 효과 | 구현 복잡도 |
|------|---------|------|-----------|
| 여러 집계 단일 요청 | 동일 조건 여러 집계 | 단일 패스 처리 | 낮음 |
| Filter Aggregation | 반복 필터 집계 | Filter Cache 재사용 | 낮음 |
| `eager_global_ordinals` | Terms 집계 자주 사용 필드 | refresh 후 첫 집계 빠름 | 낮음 |
| Request Cache | size:0 반복 집계 | 완전 캐시 | 없음 (자동) |
| Transform API | 반복 통계 조회 | 집계 비용 제로 | 높음 |
| 인덱스 분리 | 집계 전용 인덱스 | 탐색 범위 축소 | 중간 |

---

## ⚖️ 트레이드오프

```
eager_global_ordinals:
  이득: refresh 후 첫 Terms 집계 빠름
  비용: refresh 시 global_ordinals 재구축 시간 추가
        refresh_interval이 매우 짧으면 refresh 자체가 느려짐
  → refresh 빈도가 1초 이상이면 효과적

Filter Aggregation:
  이득: Filter Cache 재사용
  비용: 필터 조건이 매번 다르면 캐시 효과 없음
  → 공통 필터(in_stock, category) 집계에 적합

Transform:
  이득: 실시간 집계 비용 없음
  비용: 원본-집계 간 지연(1분~), 추가 인덱스 관리
  → 지연 허용 + 반복 조회에 적합

Request Cache:
  이득: 완전 캐시 (가장 빠름)
  비용: refresh 후 무효화
  → refresh 주기 길게 = 캐시 효과 극대화
     실시간 데이터 = 캐시 효과 없음
```

---

## 📌 핵심 정리

```
Filter Aggregation + Query Cache:
  filter 비트셋을 노드 메모리에 캐시
  동일 filter 반복 → 비트셋 즉시 재사용 → 필터링 비용 ≈ 0
  vs query(must) → 캐시 없음

eager_global_ordinals:
  Terms 집계의 global_ordinals를 refresh 시 미리 구축
  → 집계 요청 시 항상 준비 완료 상태
  refresh 빈번 + Terms 집계 자주 → 적합

Transform API:
  원본 인덱스 → 집계 → 사전 집계 인덱스
  실시간 집계 비용 없이 통계 조회 가능
  1~10분 지연 허용 시 강력한 최적화

성능 진단 순서:
  _profile → 병목 집계 식별
  request_cache 히트율 → 캐시 효과
  fielddata 사용량 → OOM 위험 확인
  slowlog → 느린 집계 자동 탐지
```

---

## 🤔 생각해볼 문제

**Q1.** `eager_global_ordinals: true`로 설정했는데 Terms 집계 첫 요청이 여전히 느리다면 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

여러 가능성이 있다. 첫째, 세그먼트가 매우 많다면 global_ordinals 구축 자체는 refresh 시 실행됐지만 그 비용이 커서 refresh가 느려진 것이다. 이 경우 집계 요청은 빠르지만 refresh 지연이 발생한다. 둘째, 다른 필드의 global_ordinals 구축 비용이 더 크게 영향을 줄 수 있다. 셋째, 집계의 병목이 global_ordinals가 아닌 다른 부분(doc_values 읽기, 버킷 생성, 네트워크)일 수 있다. `_profile` API로 집계 각 단계의 실행 시간을 확인해 실제 병목을 찾아야 한다.

</details>

**Q2.** Transform의 `continuous` 모드와 수동 배치 Transform의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

`continuous` 모드(sync 설정)는 주기적으로 원본 데이터를 확인해 변경된 데이터를 집계 인덱스에 반영한다. `sync.time.field`로 지정한 날짜 필드를 기준으로 변경을 감지한다. `frequency` 설정(기본 1분)마다 새로운 원본 데이터를 처리한다. 반면 수동 배치 모드(`POST _transform/id/_start`로 시작, 한 번만 실행)는 현재 시점의 전체 데이터를 한 번 변환하고 완료된다. 대시보드처럼 지속적인 업데이트가 필요하면 `continuous`, 과거 데이터 일회성 집계라면 배치 모드가 적합하다.

</details>

**Q3.** 대시보드의 집계가 너무 느려 Transform을 적용했는데, 집계 조건이 사용자마다 다른 경우(예: 사용자별 필터링) Transform이 효과적인가?

<details>
<summary>해설 보기</summary>

사용자마다 다른 필터가 필요한 경우 Transform의 효과가 제한된다. Transform은 미리 정해진 집계 기준으로 데이터를 집계하므로, 사용자별로 다른 조건을 모두 사전에 집계하려면 조합 수만큼의 Transform이 필요해 비효율적이다. 이런 경우에는 Filter Aggregation + 캐시, 인덱스 분리(사용자별 샤드 배치), 또는 Elastic의 `search.max_buckets`와 메모리 튜닝으로 실시간 집계 성능을 개선하는 방향이 현실적이다. Transform은 "전체 매출 현황"처럼 공통 집계 기준이 있는 경우에 가장 효과적이다.

</details>

---

<div align="center">

**[⬅️ 이전: 집계와 메모리](./04-aggregation-memory.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — 인덱스 설계 전략 ➡️](../operations-tuning/01-index-design-strategy.md)**

</div>
