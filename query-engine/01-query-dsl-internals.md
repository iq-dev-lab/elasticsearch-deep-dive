# Query DSL 내부 구조 — Query Context vs Filter Context

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Query Context와 Filter Context는 내부적으로 어떻게 다르게 처리되는가?
- Filter Cache가 비트셋(bitset)으로 저장되는 이유와 재사용 원리는?
- `bool` 쿼리의 `must`, `filter`, `should`, `must_not`은 각각 점수에 어떻게 기여하는가?
- 같은 조건을 `must`와 `filter`에 넣었을 때 결과와 성능이 어떻게 달라지는가?
- 어떤 조건을 `filter`로, 어떤 조건을 `must`로 넣어야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

검색 쿼리 최적화의 가장 쉽고 효과적인 방법이 Query Context와 Filter Context의 올바른 분리다. 점수가 필요 없는 조건(카테고리 필터, 날짜 범위, 재고 여부)을 `must` 대신 `filter`로 옮기기만 해도 캐시 효과로 응답 시간이 수십 % 개선될 수 있다. 이 차이를 모르면 모든 조건을 `must`에 넣고 성능 문제를 인덱스 탓으로 돌리게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 필터 조건을 모두 must에 넣음
  쿼리:
    {
      "query": {
        "bool": {
          "must": [
            { "match":  { "title": "keyboard" } },
            { "term":   { "category": "electronics" } },
            { "term":   { "in_stock": true } },
            { "range":  { "price": { "lte": 200000 } } }
          ]
        }
      }
    }

  문제:
    category, in_stock, price 조건은 점수와 무관
    must에 넣으면 매번 점수 계산 실행
    캐시도 적용 안 됨 → 반복 요청에도 매번 재계산
    → 불필요한 CPU 낭비

  올바른 접근:
    {
      "query": {
        "bool": {
          "must":   [ { "match": { "title": "keyboard" } } ],
          "filter": [
            { "term":  { "category": "electronics" } },
            { "term":  { "in_stock": true } },
            { "range": { "price": { "lte": 200000 } } }
          ]
        }
      }
    }
    → title 검색만 점수 계산
    → 나머지 3개 조건은 Filter Cache에서 비트셋으로 재사용

실수 2: should를 "AND 조건"으로 오해
  쿼리:
    { "bool": { "should": [
      { "term": { "category": "keyboard" } },
      { "term": { "category": "mouse" } }
    ]}}

  기대: category가 keyboard이면서 mouse인 문서
  실제: category가 keyboard이거나 mouse인 문서 (OR)
  → should = 점수 부스팅을 위한 선택적 조건, "OR 논리"

  올바른 AND: must나 filter에 여러 조건 추가

실수 3: must_not이 점수에 영향준다고 오해
  실제:
    must_not → Filter Context로 처리
    → 점수 계산 없음
    → 캐시 가능
    → 단순히 해당 문서를 결과에서 제외
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Query Context vs Filter Context 분류 기준:

  Query Context (점수 계산 필요):
    사용자 검색어와 관련성을 측정해야 하는 조건
    → 제목, 설명, 본문 등 텍스트 검색
    → "얼마나 잘 맞는가"를 숫자로 표현

  Filter Context (점수 불필요):
    조건을 만족하는지 여부만 판단하는 조건
    → 카테고리, 상태값, 날짜 범위, 가격 범위
    → "해당하는가 / 해당 안 하는가" 이진 판단

  bool 쿼리 절 선택:
    must        → Query Context (점수 기여 + 반드시 포함)
    filter      → Filter Context (점수 없음 + 반드시 포함 + 캐시)
    should      → Query Context (점수 부스팅, 없어도 됨)
    must_not    → Filter Context (반드시 제외 + 캐시)

  황금률:
    전문 검색(match, multi_match) → must
    이진 필터(term, range, exists) → filter
    점수 부스팅(선택적 조건) → should
    제외 조건 → must_not
```

---

## 🔬 내부 동작 원리

### 1. Query Context — 스코어링 포함 실행

```
Query Context에서 처리되는 쿼리:

  { "query": { "match": { "title": "keyboard" } } }

  내부 흐름:
    ① Analyzer로 "keyboard" 처리 → ["keyboard"]
    ② Term Dictionary(FST)에서 "keyboard" 탐색
    ③ Posting List 추출: [doc_1, doc_4, doc_7, ...]
    ④ 각 문서에 대해 BM25 점수 계산 (Ch4-02 상세):
         score(doc_1) = TF × IDF × norm
         score(doc_4) = ...
    ⑤ 점수 순 정렬 후 상위 K개 반환

  Query Context 쿼리의 특징:
    ① 점수(score) 계산 항상 실행
    ② 캐시 적용 안 됨 (결과가 매번 다를 수 있으므로)
    ③ CPU 집중적 (점수 계산 = 부동소수점 연산)

  bool.must에 넣으면 Query Context:
    { "bool": { "must": [ { "match": ... } ] } }
    → 이 조건이 점수에 기여
    → 조건 만족 못하면 문서 제외
```

### 2. Filter Context — 비트셋 캐시

```
Filter Context에서 처리되는 쿼리:

  { "bool": { "filter": [ { "term": { "category": "electronics" } } ] } }

  내부 흐름:
    ① "electronics" Term Dictionary에서 탐색
    ② Posting List: [doc_1, doc_2, doc_5, doc_7, ...]
    ③ 결과를 비트셋(Roaring Bitset)으로 변환:
       [0, 0, 1, 0, 1, 1, 0, 1, ...] (doc_id 인덱스 → 1 = 포함)
    ④ 비트셋을 노드 메모리(segment 레벨)에 캐시
    ⑤ 점수 계산 없음 (0.0으로 고정)

  Roaring Bitset의 효율성:
    일반 비트셋: 문서 수만큼 비트 배열 → 희소할 때 낭비
    Roaring Bitset: 압축 알고리즘으로 메모리 효율화
      밀집 구간: 비트맵으로 표현
      희소 구간: 정수 배열로 표현
    → 대용량 문서 집합도 수 MB 이하로 캐시

  Filter Cache 재사용:
    같은 조건의 filter 쿼리 → 캐시 HIT → 즉시 비트셋 반환
    비트셋 AND 연산으로 여러 filter 조건 결합:
      category="electronics" 비트셋: [1,0,1,0,1,1,0,1]
      in_stock=true 비트셋:          [1,1,1,0,1,0,0,1]
      AND 결합:                      [1,0,1,0,1,0,0,1]
    → 점수 계산 없이 비트 연산만으로 필터링

  캐시 저장 단위:
    세그먼트 레벨에서 캐시
    segment별로 독립 비트셋 유지
    segment 병합 시 캐시 무효화 → 재구성
    자주 쓰이는 filter는 빠르게 warm-up
```

### 3. bool 쿼리의 4가지 절

```
bool 쿼리 전체 구조:

  {
    "bool": {
      "must":     [...],   // Query Context, AND, 점수 기여
      "filter":   [...],   // Filter Context, AND, 점수 없음, 캐시
      "should":   [...],   // Query Context, OR, 점수 부스팅
      "must_not": [...]    // Filter Context, NOT, 점수 없음, 캐시
    }
  }

must 절:
  ┌──────────────────────────────────────────────────────┐
  │  must: [{ "match": { "title": "keyboard" } }]        │
  │                                                      │
  │  - 반드시 매칭되어야 함 (AND 논리)                         │
  │  - Query Context → 점수 계산 실행                       │
  │  - 이 절의 점수가 최종 _score에 합산                       │
  │  - 캐시 없음                                           │
  └──────────────────────────────────────────────────────┘

filter 절:
  ┌──────────────────────────────────────────────────────┐
  │  filter: [{ "term": { "category": "electronics" } }] │
  │                                                      │
  │  - 반드시 매칭되어야 함 (AND 논리)                         │
  │  - Filter Context → 점수 계산 없음 (0.0)                │
  │  - Roaring Bitset으로 캐시 → 재사용                      │
  │  - 점수에 기여하지 않음                                   │
  └──────────────────────────────────────────────────────┘

should 절:
  ┌──────────────────────────────────────────────────────┐
  │  should: [                                           │
  │    { "match": { "title": "wireless" } },             │
  │    { "match": { "description": "bluetooth" } }       │
  │  ]                                                   │
  │                                                      │
  │  - 매칭 안 해도 됨 (OR 논리, 선택적)                       │
  │  - 매칭되면 점수 증가 (부스팅)                             │
  │  - must/filter 없이 단독 사용 시: minimum_should_match   │
  │    기본 1 → 적어도 1개 should 조건 만족 필수                │
  │  - must/filter와 함께 사용 시: 0개 매칭도 허용              │
  │    (매칭되면 점수만 올라감)                                │
  └──────────────────────────────────────────────────────┘

must_not 절:
  ┌──────────────────────────────────────────────────────┐
  │  must_not: [{ "term": { "status": "out_of_stock" } }]│
  │                                                      │
  │  - 매칭되면 결과에서 제외 (NOT 논리)                        │
  │  - Filter Context → 점수 계산 없음                      │
  │  - Roaring Bitset으로 캐시 → 재사용                      │
  └──────────────────────────────────────────────────────┘

점수 합산 공식:
  _score = sum(must 절 점수) + sum(매칭된 should 절 점수)
  filter, must_not → 점수 기여 없음
```

### 4. constant_score와 boosting 쿼리

```
constant_score:
  Filter Context를 Query Context처럼 고정 점수로 반환

  { "constant_score": {
      "filter": { "term": { "category": "electronics" } },
      "boost": 1.5
  }}

  동작:
    filter 조건으로 문서 필터링 (캐시 활용)
    매칭된 모든 문서에 boost 값을 고정 점수로 부여
    → 필터링 성능 + 고정 점수 부여 조합

  용도:
    특정 카테고리 문서에 고정 점수 부여
    랭킹 로직에서 카테고리별 가중치

boosting 쿼리:
  positive 조건 점수에서 negative 조건 점수를 빼는 방식

  { "boosting": {
      "positive": { "match": { "title": "keyboard" } },
      "negative": { "match": { "title": "gaming" } },
      "negative_boost": 0.5
  }}

  동작:
    positive: 일반 점수 계산
    negative: 매칭되면 점수에 negative_boost 계수 곱함
    → "gaming keyboard" 문서는 점수가 50% 감소
    → 제외는 아니고 점수만 낮춤

  용도:
    특정 키워드 포함 문서를 낮은 순위로 밀어내기
    must_not과 달리 완전 제외가 아닌 점수 하향
```

### 5. Filter Cache 효과 측정

```
캐시 효과를 극대화하는 패턴:

  효과 있음:
    반복적으로 사용되는 filter 조건
    → "in_stock: true"는 모든 상품 검색에 포함
    → 한 번 캐시되면 모든 요청에서 재사용

  효과 없음:
    per-request 고유값을 filter에 사용
    { "filter": { "term": { "user_id": "12345" } } }
    → 사용자마다 다른 값 → 캐시 재사용 불가
    → filter가 아닌 must에 두거나, 캐시 효과 기대 안 함

  캐시 무효화 조건:
    세그먼트 병합 시
    인덱스 설정 변경 시
    노드 재시작 시

  캐시 메모리 확인:
    GET _nodes/stats/indices/query_cache?pretty
    → query_cache.memory_size_in_bytes: 현재 캐시 크기
    → query_cache.hit_count: 캐시 히트 횟수
    → query_cache.miss_count: 캐시 미스 횟수
    → 히트율 = hit / (hit + miss)

  캐시 크기 설정:
    indices.queries.cache.size: 10% (힙의 10%, 기본값)
    → 너무 작으면 캐시 eviction 빈번 → 효과 감소
    → 너무 크면 힙 압박
```

---

## 💻 실전 실험

```bash
# 실험용 인덱스
PUT /query-test
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "title":    { "type": "text" },
      "category": { "type": "keyword" },
      "price":    { "type": "integer" },
      "in_stock": { "type": "boolean" }
    }
  }
}

POST /query-test/_bulk
{ "index": { "_id": "1" } }
{ "title": "Mechanical Keyboard Cherry MX", "category": "keyboard", "price": 150000, "in_stock": true }
{ "index": { "_id": "2" } }
{ "title": "Wireless Gaming Keyboard", "category": "keyboard", "price": 120000, "in_stock": true }
{ "index": { "_id": "3" } }
{ "title": "Gaming Mouse RGB", "category": "mouse", "price": 80000, "in_stock": false }
{ "index": { "_id": "4" } }
{ "title": "Mechanical Switch Tester", "category": "accessory", "price": 30000, "in_stock": true }
POST /query-test/_refresh

# must vs filter 차이 확인 (점수 비교)
GET /query-test/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "keyboard" } },
        { "term": { "in_stock": true } }
      ]
    }
  }
}
# in_stock 조건도 점수에 영향 (비효율)

GET /query-test/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "title": "keyboard" } } ],
      "filter": [ { "term": { "in_stock": true } } ]
    }
  }
}
# in_stock은 Filter Context → 점수 영향 없음, 캐시 활용

# 두 쿼리의 _score 비교 (filter 사용 시 조금 낮을 수 있음)

# should 부스팅 효과
GET /query-test/_search
{
  "query": {
    "bool": {
      "must": [ { "match": { "title": "keyboard" } } ],
      "should": [
        { "match": { "title": "mechanical" } },
        { "match": { "title": "wireless" } }
      ]
    }
  }
}
# mechanical 또는 wireless 포함 문서 점수 상승

# constant_score 사용
GET /query-test/_search
{
  "query": {
    "constant_score": {
      "filter": { "term": { "category": "keyboard" } },
      "boost": 2.0
    }
  }
}
# keyboard 카테고리 모든 문서에 2.0 고정 점수

# Filter Cache 통계 확인
GET _nodes/stats/indices/query_cache?pretty

# profile API로 Query vs Filter 실행 시간 비교
GET /query-test/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must":   [ { "match": { "title": "keyboard" } } ],
      "filter": [ { "term": { "in_stock": true } } ]
    }
  }
}
# profile.shards[0].searches[0].query 확인
# TermQuery(must)와 TermQuery(filter) 실행 시간 비교
```

---

## 📊 Query Context vs Filter Context 비교

| 항목 | Query Context | Filter Context |
|------|-------------|---------------|
| 점수 계산 | O (BM25 실행) | X (0.0 고정) |
| 캐시 | X | O (Roaring Bitset) |
| bool 절 | `must`, `should` | `filter`, `must_not` |
| CPU 비용 | 높음 | 낮음 (비트 연산) |
| 사용 기준 | "얼마나 관련있나" | "해당하나/아니나" |
| 예시 | match, multi_match | term, range, exists |

---

## ⚖️ 트레이드오프

```
filter 남용:
  모든 조건을 filter에 → 점수 없음 → 정렬 기준 없음
  → 검색 결과가 의미 없는 순서로 나열됨
  → 전문 검색의 핵심인 관련성 순위 사라짐

must 남용:
  필터링 조건까지 must에 → 불필요한 점수 계산
  → 캐시 효과 없음 → 성능 저하
  → 동일 조건 반복 요청 시 매번 재계산

should의 minimum_should_match:
  must/filter와 함께 should → 기본 0개 must (선택적)
  should만 단독 → 기본 1개 must (적어도 하나 매칭)
  명시적 설정 권장:
    "minimum_should_match": 1  (적어도 1개 should 매칭 필요)
    "minimum_should_match": "75%" (75% 이상 매칭 필요)
```

---

## 📌 핵심 정리

```
Query Context:
  목적: "얼마나 관련있나" → 점수 계산 필요
  절:   must, should
  캐시: X
  비용: CPU 높음

Filter Context:
  목적: "해당하나/아니나" → 이진 판단
  절:   filter, must_not
  캐시: O (Roaring Bitset, 세그먼트 레벨)
  비용: 비트 연산, 빠름

황금률:
  전문 검색(match, multi_match) → must
  이진 조건(term, range, exists, bool) → filter
  점수 부스팅 → should (선택적)
  제외 조건 → must_not

bool 쿼리 최적화:
  filter 조건은 자주 재사용되는 것이어야 캐시 효과
  per-request 고유값은 캐시 효과 없음
  Filter Cache 히트율 모니터링으로 효과 확인
```

---

## 🤔 생각해볼 문제

**Q1.** `must`에 `term` 쿼리를 넣었을 때와 `filter`에 넣었을 때 결과(문서 목록)는 같은가?

<details>
<summary>해설 보기</summary>

반환되는 문서 목록은 동일하다. 두 경우 모두 "해당 조건을 만족하는 문서만 반환"한다. 차이는 점수다. `must`에 넣으면 BM25 점수 계산이 실행되어 `_score` 값에 영향을 미친다. `filter`에 넣으면 점수 계산이 없어 `_score`에 영향을 주지 않는다. 점수에 기여하지 않는 `term` 조건은 `filter`가 항상 올바른 선택이다.

</details>

**Q2.** Filter Cache에서 per-user 필터(예: user_id)는 왜 캐시 효과가 없는가?

<details>
<summary>해설 보기</summary>

Filter Cache는 동일한 filter 조건이 반복 요청될 때 비트셋을 재사용한다. `user_id: 12345`와 `user_id: 67890`는 서로 다른 filter 조건이므로 각각 별도 캐시 항목이 된다. 사용자가 수백만 명이면 수백만 개의 캐시 항목이 생성되지만 각각 1회만 사용되므로 캐시 재사용이 없다. 오히려 캐시 메모리만 낭비하고 유용한 공통 filter(in_stock, category 등)를 eviction할 수 있다. per-user 조건은 `must` 또는 `filter`에 두되 캐시 효과를 기대하지 않는 것이 맞다.

</details>

**Q3.** `should` 절에 여러 조건을 넣고 `minimum_should_match: 2`로 설정하면 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

`minimum_should_match: 2`는 `should` 절 중 최소 2개를 만족해야 결과에 포함됨을 의미한다. 예를 들어 should에 5개 조건이 있으면 그 중 2개 이상을 만족하는 문서만 반환된다. 더 많은 should를 만족할수록 점수가 높아진다. 이 설정으로 "OR" 기반 should를 "최소 N개 AND" 방식으로 활용할 수 있다. 주의할 점은 `must`나 `filter`가 있을 때는 should의 minimum_should_match 기본값이 0이므로, should 조건을 필수로 만들려면 명시적으로 설정해야 한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: BM25 스코어링 완전 분해 ➡️](./02-bm25-scoring.md)**

</div>
