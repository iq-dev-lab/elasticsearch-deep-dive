# Terms Aggregation 내부 — 샤드별 top-N 병합과 오차

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Terms Aggregation 결과가 부정확할 수 있는 수학적 원인은 무엇인가?
- `shard_size`를 높이면 왜 정확도가 높아지고 비용은 어떻게 변하는가?
- `doc_count_error_upper_bound`와 `sum_other_doc_count`는 무엇을 의미하는가?
- `order` 설정으로 정렬 기준을 바꾸면 오차에 어떤 영향이 있는가?
- 오차 없는 정확한 카테고리별 집계가 필요할 때 어떤 대안이 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"카테고리별 상위 10개를 보여달라"는 요구사항에서 Terms Aggregation을 쓰면 결과가 틀릴 수 있다. 실제 3위인 카테고리가 5위로 나오거나, 아예 상위 10위 밖으로 밀려날 수 있다. 이 사실을 모르면 비즈니스 지표를 잘못 보고하고, 오류 원인을 찾지 못한다. `shard_size`와 `doc_count_error_upper_bound`는 이 오차를 통제하는 핵심 파라미터다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Terms Aggregation 결과를 완전히 신뢰
  비즈니스 리포트:
    "카테고리별 매출 상위 5개" → Terms Aggregation 사용
    결과를 그대로 보고서에 사용

  실제:
    샤드 수가 많고 데이터 분포가 불균등한 경우
    실제 3위 카테고리가 집계에서 누락 가능
    → 비즈니스 결정에 틀린 데이터 사용

  올바른 접근:
    doc_count_error_upper_bound 확인
    오차 범위가 크면 shard_size 증가
    정확도 100% 필요 → 별도 정확 집계 방식 사용

실수 2: shard_size = size로 설정
  오해: "shard_size는 필요 없다, size만 설정하면 된다"
  결과: 기본 shard_size = size × 1.5 + 10
        이 기본값도 오차가 있을 수 있음

  샤드 수 많고 size 크게 설정한 경우:
    shard_size 자동 계산값이 충분하지 않을 수 있음
    → 명시적으로 shard_size 설정 권장

실수 3: order: _count 이외 정렬 시 오차 더 심함을 모름
  오해: "order: { avg_price: desc }로 평균 가격이 높은 카테고리 순서"
  실제:
    샤드별로 avg_price 기준 상위 N개를 반환
    각 샤드의 avg_price 계산은 그 샤드의 문서만 기준
    → Coordinator에서 글로벌 avg_price ≠ 샤드별 avg_price 합산
    → 정렬 기준이 부정확하면 오차 더 심함
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Terms Aggregation 설계 원칙:

  오차 허용 가능 (일반 대시보드):
    size: 10 (상위 10개)
    shard_size: 50~100 (샤드당 더 많이 가져와 병합)
    → 대부분의 경우 충분한 정확도

  오차 허용 불가 (정확한 비즈니스 지표):
    방법 1: shard_size를 매우 크게 (카테고리 수 × 10)
            → 비용 증가 but 정확도 향상
    방법 2: 단일 샤드 인덱스 사용
            → 샤드 병합 오차 없음
    방법 3: Composite Aggregation 사용
            → 모든 버킷을 순서대로 정확하게 반환 (페이지네이션 방식)
    방법 4: 사전 집계 (Rollup/Transform)
            → 미리 집계해 별도 인덱스에 저장

  오차 모니터링:
    doc_count_error_upper_bound: 각 버킷의 오차 상한
    sum_other_doc_count: top-N에 포함 안 된 문서 수
    → 이 값이 크면 shard_size 증가 필요
```

---

## 🔬 내부 동작 원리

### 1. Terms Aggregation 분산 처리와 오차 발생 원리

```
설정: size=3 (상위 3개 카테고리)
인덱스: 3개 샤드, 총 300개 문서

실제 전체 카운트:
  keyboard:  120개 (1위)
  mouse:      90개 (2위)
  monitor:    60개 (3위)
  headset:    20개 (4위)
  controller: 10개 (5위)

각 샤드의 문서 분포 (불균등 분포 가정):
  Shard 0 (100개):  keyboard:20, mouse:50, monitor:10, headset:15, controller:5
  Shard 1 (100개):  keyboard:60, mouse:20, monitor:30, headset:3, controller:3 (나머지)
  Shard 2 (100개):  keyboard:40, mouse:20, monitor:20, headset:2, controller:2 (나머지)

Phase 1: 각 샤드에서 상위 size=3개 반환

  Shard 0 top-3:
    mouse:50, headset:15, keyboard:20 → {mouse:50, headset:15, keyboard:20}
    (monitor:10은 4위라 제외!)

  Shard 1 top-3:
    keyboard:60, monitor:30, mouse:20 → {keyboard:60, monitor:30, mouse:20}

  Shard 2 top-3:
    keyboard:40, mouse:20, monitor:20 → {keyboard:40, mouse:20, monitor:20}

Phase 2: Coordinator 병합

  keyboard: 20 + 60 + 40 = 120  ✅
  mouse:    50 + 20 + 20 = 90   ✅
  monitor:  0 + 30 + 20 = 50    ❌ (Shard 0에서 제외됨 → 실제 60 아닌 50으로 계산)
  headset:  15 + 0 + 0 = 15     ❌ (Shard 1, 2에서 제외됨)

  집계 결과 top-3:
    1위: keyboard 120 ✅
    2위: mouse    90  ✅
    3위: monitor  50  ❌ (실제 60이지만 Shard 0 데이터 누락)
         headset  15  ❌ (실제 20이지만 Shard 1, 2 데이터 누락)

  monitor가 실제 3위(60)인데 집계 결과는 50
  → 오차 발생!

왜 오차가 발생하는가:
  각 샤드는 자신의 로컬 상위 N개만 반환
  샤드 0에서 monitor(10개)가 headset(15개)보다 적어 제외
  → Coordinator가 Shard 0의 monitor 데이터를 알 수 없음
  → 전역 집계에 monitor의 10개가 누락
```

### 2. shard_size — 오차 줄이기

```
shard_size: 각 샤드가 반환하는 최대 버킷 수 (size와 독립)

size=3, shard_size=10 설정:

Phase 1: 각 샤드에서 상위 10개 반환

  Shard 0 top-10 (전체 5개 카테고리만 있으므로 전부 반환):
    mouse:50, headset:15, keyboard:20, monitor:10, controller:5
    (이제 monitor:10도 포함!)

Phase 2: Coordinator 병합

  keyboard: 20 + 60 + 40 = 120 ✅
  mouse:    50 + 20 + 20 = 90  ✅
  monitor:  10 + 30 + 20 = 60  ✅ (이제 정확!)
  headset:  15 + 0 + 0 = 15    (일부 누락 가능성 있음)

  Coordinator top-3:
    keyboard: 120, mouse: 90, monitor: 60 → 정확한 결과!

shard_size 기본값:
  ES 자동 계산: max(10, size × 1.5)
  size=3: shard_size = max(10, 3×1.5) = 10

shard_size 권장값:
  정확도 중요: shard_size = 전체 카테고리 수 (모든 버킷 반환)
  성능 중요:   shard_size = size × (샤드 수 + 1)

shard_size 트레이드오프:
  shard_size↑ → 정확도↑, 각 샤드에서 반환 데이터 증가 → 네트워크, 메모리↑
  shard_size↓ → 빠름, 메모리 절약 → 오차 가능성↑
```

### 3. doc_count_error_upper_bound 해석

```
응답 구조:
  {
    "aggregations": {
      "by_category": {
        "doc_count_error_upper_bound": 15,  ← 전체 오차 상한
        "sum_other_doc_count": 30,          ← top-N 밖 문서 수
        "buckets": [
          {
            "key": "keyboard",
            "doc_count": 120,
            "doc_count_error_upper_bound": 0   ← 이 버킷의 오차 상한
          },
          {
            "key": "mouse",
            "doc_count": 90,
            "doc_count_error_upper_bound": 5   ← 실제값은 90~95 사이일 수 있음
          },
          {
            "key": "monitor",
            "doc_count": 50,
            "doc_count_error_upper_bound": 10  ← 실제값은 50~60 사이일 수 있음
          }
        ]
      }
    }
  }

doc_count_error_upper_bound:
  버킷별: 이 버킷의 실제 doc_count가 최대 이만큼 더 많을 수 있음
  전체:   모든 버킷의 오차 합산 상한

sum_other_doc_count:
  top-N에 포함되지 않은 나머지 문서의 수
  이 값이 크면 → 더 많은 카테고리가 있고 일부가 누락됐을 가능성

오차 해석:
  doc_count_error_upper_bound = 0:
    → 이 버킷은 모든 샤드에서 최상위에 있어 누락 없음 → 정확
  doc_count_error_upper_bound > 0:
    → 일부 샤드에서 이 버킷이 반환되지 않아 과소 계산 가능

show_term_doc_count_error:
  { "terms": { "field": "category", "show_term_doc_count_error": true } }
  → 각 버킷에 doc_count_error_upper_bound 포함 (기본 미포함)
```

### 4. Composite Aggregation — 오차 없는 정확한 집계

```
Composite Aggregation:
  페이지네이션 방식으로 모든 버킷을 순서대로 반환
  → 오차 없음 (모든 버킷 처리)

  { "aggs": {
    "all_categories": {
      "composite": {
        "size": 100,
        "sources": [
          { "category": { "terms": { "field": "category" } } }
        ]
      }
    }
  }}

  응답:
    { "aggregations": {
      "all_categories": {
        "after_key": { "category": "monitor" },   ← 다음 페이지 시작점
        "buckets": [
          { "key": { "category": "controller" }, "doc_count": 10 },
          { "key": { "category": "headset" },    "doc_count": 20 },
          { "key": { "category": "keyboard" },   "doc_count": 120 },
          { "key": { "category": "monitor" },    "doc_count": 60 },
          ...
        ]
      }
    }}

  다음 페이지:
    { "aggs": { "all_categories": { "composite": {
      "size": 100,
      "after": { "category": "monitor" },   ← 이전 after_key 사용
      "sources": [...]
    }}}}

  특징:
    오차 없음 (모든 버킷을 순서대로 처리)
    서브 집계(sub-agg) 지원 없음 (일부 메트릭만)
    주로 대규모 Export, Transform에 사용

  Terms vs Composite:
    Terms: 빠름, 상위 N개, 오차 가능
    Composite: 느림, 전체 버킷, 오차 없음
```

### 5. order 설정과 오차의 관계

```
default order (doc_count desc):
  doc_count가 많은 카테고리가 각 샤드에서 상위 반환될 가능성↑
  → 오차가 상대적으로 작음 (가장 흔한 것이 누락될 가능성 낮음)

custom order (avg_price desc):
  각 샤드에서 avg_price 높은 카테고리를 반환
  하지만 avg_price는 샤드 로컬 계산 → 글로벌 avg_price와 다를 수 있음

  Shard 0에서 A카테고리 avg_price=200,000 (10개 문서)
  Shard 1에서 A카테고리 avg_price=100,000 (90개 문서)
  글로벌 avg = (200,000×10 + 100,000×90) / 100 = 110,000

  Shard 0 기준 A는 상위 → 반환
  Shard 1 기준 A는 하위 → 제외 가능
  → 병합 시 A의 글로벌 avg_price 계산에 Shard 1 데이터 누락

  결론:
    order: _count → 오차 최소 (ES 권장)
    order: 다른 메트릭 → 더 큰 오차, 신중하게 사용
    정확한 정렬이 필요하면 → 전체 집계 후 애플리케이션 정렬
```

---

## 💻 실전 실험

```bash
# Terms Aggregation 오차 확인
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category_small": {
      "terms": {
        "field": "category",
        "size": 3,
        "shard_size": 3,          // shard_size = size → 오차 가능
        "show_term_doc_count_error": true
      }
    },
    "by_category_accurate": {
      "terms": {
        "field": "category",
        "size": 3,
        "shard_size": 100,        // shard_size 크게 → 정확도↑
        "show_term_doc_count_error": true
      }
    }
  }
}
# 두 집계의 doc_count_error_upper_bound 비교

# doc_count_error_upper_bound 확인
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category",
        "size": 5,
        "show_term_doc_count_error": true
      }
    }
  }
}
# aggregations.by_category.doc_count_error_upper_bound 확인
# 각 버킷의 doc_count_error_upper_bound 확인

# Composite Aggregation (오차 없는 전체 버킷)
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "all_categories": {
      "composite": {
        "size": 10,
        "sources": [
          { "category": { "terms": { "field": "category" } } }
        ]
      }
    }
  }
}
# after_key로 다음 페이지 요청

# 다음 페이지
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "all_categories": {
      "composite": {
        "size": 10,
        "after": { "category": "keyboard" },
        "sources": [
          { "category": { "terms": { "field": "category" } } }
        ]
      }
    }
  }
}

# order 비교: doc_count vs avg_price
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_count": {
      "terms": { "field": "category", "size": 5, "order": { "_count": "desc" } }
    },
    "by_avg_price": {
      "terms": {
        "field": "category", "size": 5,
        "order": { "avg_p": "desc" }
      },
      "aggs": { "avg_p": { "avg": { "field": "price" } } }
    }
  }
}
```

---

## 📊 shard_size에 따른 정확도 vs 비용

| shard_size | 정확도 | 네트워크 비용 | 메모리 비용 | 적합 상황 |
|-----------|-------|------------|----------|---------|
| size (최소) | 낮음 | 최소 | 최소 | 근사치 허용 |
| size × 1.5 + 10 (기본) | 중간 | 중간 | 중간 | 일반 대시보드 |
| 카테고리 수 전체 | 높음 | 높음 | 높음 | 정확한 집계 |
| Composite Agg | 완전 정확 | 페이지당 중간 | 낮음 | Export, 정밀 분석 |

---

## ⚖️ 트레이드오프

```
정확도 vs 성능:
  shard_size 증가 → 정확도↑, 샤드당 버킷 처리↑, 네트워크 전송↑
  shard_size 감소 → 성능↑, 오차 가능성↑
  → 대시보드: 성능 우선 (오차 허용)
  → 정산/정확 리포팅: 정확도 우선 (shard_size 크게 또는 Composite)

단일 샤드:
  이득: Terms 오차 없음
  비용: 쓰기 처리량 제한, 인덱스 크기 제한
  → 소규모 카탈로그 데이터에 적합

Composite Aggregation:
  이득: 오차 없음, 모든 버킷
  비용: 서브 집계 제한, 여러 요청 필요 (페이지네이션)
  → 오프라인 처리, 정확한 Export에 적합
```

---

## 📌 핵심 정리

```
Terms Aggregation 오차 원인:
  각 샤드가 로컬 상위 shard_size개만 반환
  → 글로벌로 상위이지만 특정 샤드에서 하위인 버킷 누락
  → Coordinator 병합 시 해당 데이터 없음 → 과소 계산

오차 지표:
  doc_count_error_upper_bound: 각 버킷/전체 오차 상한
  sum_other_doc_count: top-N 밖 누락 문서 수

오차 줄이기:
  shard_size↑ → 각 샤드에서 더 많이 반환 → 병합 시 누락 감소
  shard_size = 카테고리 전체 수 → 오차 거의 없음
  단일 샤드 → 오차 없음
  Composite Aggregation → 오차 없음 (페이지네이션)

order 설정:
  _count (기본) → 오차 최소
  다른 메트릭   → 추가 오차 가능 → 신중 사용
```

---

## 🤔 생각해볼 문제

**Q1.** 카테고리가 1,000개인 인덱스에서 상위 10개를 Terms Aggregation으로 구하면 `shard_size`를 얼마로 설정해야 오차를 최소화할 수 있는가?

<details>
<summary>해설 보기</summary>

오차를 최소화하려면 `shard_size`를 전체 카테고리 수(1,000)와 같거나 크게 설정하면 된다. 이렇게 하면 각 샤드가 모든 카테고리 버킷을 반환하므로 Coordinator가 완전한 정보를 바탕으로 병합할 수 있다. 단, 1,000개 버킷 × 샤드 수만큼의 데이터를 Coordinator로 전송하므로 네트워크와 메모리 비용이 크게 증가한다. 현실적인 타협점은 `shard_size: 200` 정도로 설정해 정확도를 상당히 높이되 비용을 제한하는 것이다. 정확도 100%가 필수라면 Composite Aggregation이 더 적합하다.

</details>

**Q2.** `sum_other_doc_count: 0`이면 Terms Aggregation 결과가 완전히 정확하다는 의미인가?

<details>
<summary>해설 보기</summary>

꼭 그렇지 않다. `sum_other_doc_count: 0`은 top-N(size) 안에 들어온 버킷들의 doc_count 합이 전체 문서 수와 같다는 의미다. 즉, 모든 문서가 반환된 버킷 중 하나에 속한다는 뜻이다. 하지만 각 버킷의 `doc_count_error_upper_bound`가 0이 아니라면 개별 버킷의 카운트는 여전히 과소 계산되었을 수 있다. 완전한 정확성을 보장하려면 `sum_other_doc_count: 0` + 모든 버킷의 `doc_count_error_upper_bound: 0` 두 조건을 모두 확인해야 한다.

</details>

**Q3.** Terms Aggregation에서 상위 N개가 아닌 모든 버킷의 정확한 카운트가 필요할 때 가장 효율적인 방법은?

<details>
<summary>해설 보기</summary>

Composite Aggregation을 사용하는 것이 가장 적합하다. 페이지네이션 방식으로 모든 버킷을 순서대로 반환하며 오차가 없다. 대안으로 `shard_size`를 전체 카테고리 수로 설정한 Terms Aggregation도 가능하지만, 카테고리 수가 매우 많으면 메모리와 네트워크 비용이 크다. Transform API를 사용해 사전 집계 인덱스를 만드는 방법도 있는데, 실시간성이 중요하지 않고 주기적으로 정확한 집계가 필요한 경우(일별 리포트 등)에 가장 효율적이다.

</details>

---

<div align="center">

**[⬅️ 이전: 집계 아키텍처](./01-aggregation-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: Date Histogram 집계 ➡️](./03-date-histogram-aggregation.md)**

</div>
