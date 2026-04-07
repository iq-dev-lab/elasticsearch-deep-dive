# 쿼리 실행 계획 — `_explain` API와 Lucene Boolean Query 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `_explain` API의 출력에서 어떤 정보를 어떻게 읽는가?
- Lucene이 MUST/SHOULD/FILTER 절을 처리 순서를 어떻게 최적화하는가?
- `_profile` API는 `_explain`과 어떻게 다르고, 언제 사용하는가?
- 비용이 높은 절을 먼저 평가하면 왜 성능이 나빠지는가?
- 쿼리 최적화 포인트를 `_profile` 결과에서 찾는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

검색이 느린 이유를 찾는 가장 정확한 방법이 `_profile` API다. "왜 이 문서의 점수가 이렇게 나왔는가"를 추적하는 도구가 `_explain`이다. 이 두 API를 모르면 쿼리 최적화가 근거 없는 추측이 된다. Lucene의 절 최적화 원리를 알면 쿼리 작성 방식만으로 성능을 20~50% 개선하는 것도 가능하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 비용 높은 쿼리를 filter 맨 앞에 배치
  쿼리:
    { "bool": { "filter": [
      { "script": { "script": "doc['price'].value * 2 > 100000" } },
      { "term": { "in_stock": true } },
      { "term": { "category": "keyboard" } }
    ]}}

  문제:
    script 쿼리는 모든 문서를 스캔하며 스크립트 실행 (가장 비쌈)
    맨 앞에 배치 → 전체 문서에 스크립트 실행
    이후 term 쿼리가 이미 스크립트를 통과한 결과만 필터링

  올바른 배치:
    { "bool": { "filter": [
      { "term": { "in_stock": true } },   // 가장 선택적
      { "term": { "category": "keyboard" } },
      { "script": { ... } }              // 남은 문서만 처리
    ]}}

  Lucene 자동 최적화:
    실제로 Lucene은 cost(예상 문서 수)를 추정해 자동 순서 조정
    하지만 script처럼 예측 어려운 경우 힌트가 없으면 최적 순서 보장 못함

실수 2: _explain 없이 점수 문제 디버깅
  상황:
    "왜 doc_A가 doc_B보다 점수가 높지?"
    코드 추측 → 매핑 변경 → 재인덱싱 → 확인 반복

  올바른 접근:
    GET /index/_explain/doc_A { "query": ... }
    → IDF, TF_norm, 각 절별 점수 기여 즉시 확인
    → 원인 파악 후 핀포인트 수정

실수 3: _profile 없이 "느린 쿼리" 원인 추측
  상황:
    특정 쿼리가 500ms 이상 걸림
    "인덱스 크기 문제", "샤드 수 문제" 추측

  올바른 접근:
    GET /index/_search { "profile": true, "query": ... }
    → 어느 절이 몇 ms를 사용하는지 정확히 확인
    → "wildcard 쿼리가 300ms 사용" → wildcard 제거 또는 대안 탐색
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
쿼리 성능 분석 워크플로:

  Step 1: 느린 쿼리 발견
    slowlog 설정으로 임계값 초과 쿼리 로그
    PUT /index/_settings {
      "index.search.slowlog.threshold.query.warn": "1s",
      "index.search.slowlog.threshold.query.info": "500ms"
    }

  Step 2: profile로 병목 절 찾기
    GET /index/_search { "profile": true, "query": {...} }
    → query 섹션에서 각 절별 time_in_nanos 확인
    → 가장 오래 걸리는 절 식별

  Step 3: _explain으로 점수 문제 분석
    GET /index/_explain/doc_id { "query": {...} }
    → BM25 세부 계산 확인
    → IDF, TF_norm, 각 절 기여도 추적

  Step 4: 최적화 적용
    비용 높은 쿼리 → filter로 이동 또는 제거
    wildcard → prefix 또는 ngram으로 대체
    script → 사전 계산 필드로 대체
    많은 should → function_score로 통합
```

---

## 🔬 내부 동작 원리

### 1. `_explain` API 출력 구조 완전 분해

```
쿼리:
  GET /query-test/_explain/1
  { "query": {
    "bool": {
      "must": [
        { "match": { "title": "mechanical keyboard" } }
      ],
      "filter": [
        { "term": { "in_stock": true } }
      ]
    }
  }}

출력 구조:
  {
    "_index": "query-test",
    "_id": "1",
    "matched": true,
    "explanation": {
      "value": 5.12,          ← 최종 점수
      "description": "sum of:",
      "details": [

        {                      ← must 절 (BM25)
          "value": 5.12,
          "description": "sum of:",
          "details": [

            {                  ← "mechanical" 단어 점수
              "value": 2.47,
              "description": "weight(title:mechanical in 0) [PerFieldSimilarity]",
              "details": [
                {
                  "value": 1.84,
                  "description": "score(freq=1.0), computed as boost * idf * tf from:",
                  "details": [
                    { "value": 2.2,  "description": "boost" },
                    { "value": 1.84, "description": "idf, computed as ln(1 + (N - n + 0.5) / (n + 0.5)) from:",
                      "details": [
                        { "value": 4,  "description": "n, number of documents containing term" },
                        { "value": 10, "description": "N, total number of documents with field" }
                      ]
                    },
                    { "value": 0.67, "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from:",
                      "details": [
                        { "value": 1.0, "description": "freq, occurrences of term within document" },
                        { "value": 1.2, "description": "k1, term saturation parameter" },
                        { "value": 0.75,"description": "b, length normalization parameter" },
                        { "value": 4,   "description": "dl, length of field" },
                        { "value": 5.0, "description": "avgdl, average length of field" }
                      ]
                    }
                  ]
                }
              ]
            },

            {                  ← "keyboard" 단어 점수
              "value": 2.65,
              "description": "weight(title:keyboard in 0) [PerFieldSimilarity]",
              "details": [ ... ]
            }
          ]
        },

        {                      ← filter 절 (점수 없음)
          "value": 0.0,
          "description": "match on required clause, product of:",
          "details": [
            { "value": 0.0,  "description": "# clause" },
            { "value": 1.0,  "description": "_name:in_stock:true [...]" }
          ]
        }

      ]
    }
  }

읽는 방법:
  최상위 value: 최종 점수
  "sum of:" 하위 → 각 절의 기여도 합산
  "weight(field:term)": 해당 단어의 BM25 점수
    idf → N(전체), n(단어 등장 문서 수)
    tf  → freq(등장 횟수), dl(문서 길이), avgdl(평균 길이)
  filter 절: value=0.0 (점수 없음), 1.0=매칭
```

### 2. Lucene Boolean Query 최적화 — 절 평가 순서

```
Lucene이 bool 쿼리 절을 최적화하는 방식:

최적화 원칙: Cost 기반 순서 재배열
  Cost(절) = 예상 매칭 문서 수
  Cost 낮은 절(소수 문서) → 먼저 평가
  → 빠르게 후보 집합을 줄임
  → 이후 절은 더 적은 문서에 적용

예시:
  쿼리:
    must: [match("title", "keyboard")]
    filter: [
      term("category", "electronics"),    // cost: 500 문서
      term("in_stock", true),             // cost: 8000 문서
      range("price", {lte: 200000})       // cost: 6000 문서
    ]

  Lucene이 추정하는 cost:
    "category=electronics": 500개 문서
    "price <= 200000": 6000개 문서
    "in_stock=true": 8000개 문서
    "title:keyboard": 200개 문서 (Posting List 크기)

  최적화된 실행 순서:
    1. title:keyboard (200) → 200개 후보
    2. category=electronics (500) → 교집합 → 더 작은 집합
    3. price <= 200000 (6000) → 더 줄어듦
    4. in_stock=true (8000) → 최종 필터

  vs 최악의 순서:
    1. in_stock=true (8000) → 8000개 후보
    2. price <= 200000 (6000) → 교집합
    3. category=electronics (500) → 줄어듦
    4. title:keyboard (200) → 최종
    → 각 단계에서 더 많은 문서를 처리

  Lucene 자동 최적화:
    filter 절은 대부분 자동으로 cost 순 정렬
    하지만 script 쿼리 같은 런타임 비용은 사전 예측 어려움
    → 비용 높은 쿼리는 filter 가장 뒤에 수동 배치

DISI (DocIdSetIterator) 기반 교집합:
  각 절이 DocIdSetIterator 반환
  가장 cost 낮은 DISI가 "lead" 역할
  나머지 DISI를 advance()로 따라가게 함
  → 단조 증가하는 doc_id 탐색으로 교집합 계산
  → 선형 탐색보다 훨씬 효율적
```

### 3. `_profile` API — 병목 절 찾기

```
profile API 출력 구조:

GET /query-test/_search
{
  "profile": true,
  "query": { "bool": {
    "must": [{ "match": { "title": "keyboard" } }],
    "filter": [
      { "term": { "category": "keyboard" } },
      { "range": { "price": { "lte": 200000 } } }
    ]
  }}
}

응답:
{
  "profile": {
    "shards": [{
      "id": "[node_id][index][0]",    ← 샤드 식별자
      "searches": [{
        "query": [
          {
            "type": "BooleanQuery",
            "description": "+title:keyboard #category:keyboard #price:[* TO 200000]",
            "time_in_nanos": 543210,   ← 이 절이 사용한 나노초
            "breakdown": {
              "set_min_competitive_score": 0,
              "match": 1234,
              "match_count": 3,
              "build_scorer_count": 1,
              "create_weight": 12000,
              "next_doc": 45678,
              "next_doc_count": 10,
              "score": 23456,
              "score_count": 3,
              "advance": 7890,
              "advance_count": 0,
              "compute_max_score": 0,
              "shallow_advance": 0,
              "build_scorer": 890123
            },
            "children": [
              {
                "type": "TermQuery",
                "description": "title:keyboard",
                "time_in_nanos": 123456,
                "breakdown": { ... }
              },
              {
                "type": "TermQuery",
                "description": "category:keyboard",
                "time_in_nanos": 45678,
                "breakdown": { ... }
              },
              {
                "type": "IndexOrDocValuesQuery",
                "description": "price:[* TO 200000]",
                "time_in_nanos": 34567,
                "breakdown": { ... }
              }
            ]
          }
        ],
        "rewrite_time": 5678,      ← 쿼리 재작성 시간
        "collector": [...]
      }]
    }]
  }
}

핵심 분석 포인트:
  time_in_nanos가 가장 큰 절 → 병목
  breakdown.next_doc_count → 반복 횟수 (많을수록 비용 큼)
  breakdown.score_count → 점수 계산 횟수
  children 순서 → Lucene이 실제 적용한 최적화 순서
```

### 4. 쿼리 유형별 비용 계층

```
비용 계층 (낮음 → 높음):

  가장 빠름 (상수 시간):
    Filter Cache HIT → 비트셋 조회 O(1)

  매우 빠름:
    term 쿼리 → Posting List 탐색 O(log V + K)
    terms 쿼리 → 여러 term의 OR
    exists 쿼리 → doc_values 유무 확인

  빠름:
    match 쿼리 → Analyzer + term 탐색
    prefix 쿼리 → FST 접두사 탐색 + Posting List 병합
    range 쿼리 (숫자/날짜) → BKD-Tree 범위 탐색

  중간:
    match_phrase 쿼리 → term + position 검증
    bool 쿼리 → 하위 절 조합 비용
    wildcard (앞에 * 없음) → 접두사 FST 탐색 + 후처리

  느림:
    wildcard (앞에 *) → FST 전체 열거
    fuzzy 쿼리 → Levenshtein 오토마톤 + FST
    regexp 쿼리 → 정규식 오토마톤 + FST
    span 쿼리 → position 기반 복잡 탐색

  가장 느림:
    script 쿼리 → 모든 후보 문서에 스크립트 실행
    percolate 쿼리 → 역방향 쿼리 매칭
    geo 복잡 쿼리 → 지리 계산

최적화 전략:
  비용 높은 쿼리는 filter의 맨 뒤에 배치
  또는 사전 계산된 필드로 대체
  script → runtime field 또는 인덱스 시 계산
```

---

## 💻 실전 실험

```bash
# _explain으로 점수 분해
GET /query-test/_explain/1
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "mechanical keyboard" } }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 200000 } } }
      ]
    }
  }
}
# explanation.details 에서 절별 점수 확인

# 두 문서의 점수 차이 원인 분석
GET /query-test/_explain/1
{ "query": { "match": { "title": "keyboard" } } }

GET /query-test/_explain/2
{ "query": { "match": { "title": "keyboard" } } }
# IDF는 동일, TF_norm(dl, avgdl)의 차이 확인

# _profile로 병목 분석
GET /query-test/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [{ "match": { "title": "keyboard" } }],
      "filter": [
        { "term": { "category": "keyboard" } },
        { "range": { "price": { "lte": 200000 } } }
      ],
      "should": [
        { "match": { "title": "mechanical" } },
        { "match": { "title": "wireless" } }
      ]
    }
  }
}
# profile.shards[0].searches[0].query[0].time_in_nanos
# children 각 절의 time_in_nanos 비교

# 비용 높은 wildcard vs prefix 비교
GET /query-test/_search
{
  "profile": true,
  "query": { "wildcard": { "title": "*board*" } }
}

GET /query-test/_search
{
  "profile": true,
  "query": { "prefix": { "title": "key" } }
}
# time_in_nanos 비교: wildcard가 훨씬 높음

# slowlog 설정으로 느린 쿼리 자동 로깅
PUT /query-test/_settings
{
  "index.search.slowlog.threshold.query.warn": "100ms",
  "index.search.slowlog.threshold.query.info": "50ms",
  "index.search.slowlog.threshold.fetch.warn": "100ms"
}

# 클러스터 레벨 slowlog 확인
GET _nodes/stats?pretty
# indices.search.query_time_in_millis, query_total 확인
```

---

## 📊 _explain vs _profile 비교

| 항목 | `_explain` | `_profile` |
|------|-----------|-----------|
| 목적 | 특정 문서의 점수 추적 | 쿼리 전체 성능 분석 |
| 대상 | 단일 문서 | 쿼리 전체 (모든 후보) |
| 출력 | BM25 수식 분해 | 절별 실행 시간 |
| 문서 매칭 여부 | 알 수 있음 | 알 수 없음 |
| 사용 시점 | "왜 이 점수?" | "왜 느려?" |
| 성능 영향 | 낮음 (단일 문서) | 있음 (전체 실행 추적) |

---

## ⚖️ 트레이드오프

```
_profile 사용 비용:
  profile=true → 쿼리 실행 시간 2~5배 증가
  → 운영 환경에서 일반 요청에 항상 켜두면 안 됨
  → 느린 쿼리 디버깅 시에만 일시적으로 사용

쿼리 단순화 vs 기능 완전성:
  복잡한 bool 쿼리 → 정교한 점수 but 느림
  단순한 filter + match → 빠름 but 점수 단순
  → function_score로 복잡 점수 + 성능 균형

Lucene 자동 최적화 신뢰:
  대부분 Lucene이 cost 기반으로 자동 최적화
  하지만 script, fuzzy, regexp → 예측 불가
  → 이런 쿼리는 수동으로 뒤에 배치하거나 대안 탐색
```

---

## 📌 핵심 정리

```
_explain API:
  특정 문서가 왜 이 점수를 받았는지 BM25 수식 분해
  IDF (N, n), TF_norm (freq, dl, avgdl, k1, b) 실제 값 확인
  매칭 여부 + 각 절 기여도 파악

_profile API:
  쿼리 전체 절별 실행 시간 측정
  time_in_nanos로 병목 절 식별
  Lucene이 적용한 최적화 순서 확인
  운영 환경에서는 일시적으로만 사용 (성능 영향)

Lucene Boolean Query 최적화:
  Cost 기반 절 순서 재배열 (cost 낮은 절 먼저)
  DISI 기반 단조 증가 doc_id 교집합
  script/fuzzy/regexp는 cost 예측 어려움 → 수동 뒤 배치

쿼리 비용 계층:
  최저: Filter Cache HIT
  낮음: term, range, match
  중간: prefix, match_phrase, wildcard(접미사X)
  높음: wildcard(앞*), fuzzy, regexp
  최고: script
```

---

## 🤔 생각해볼 문제

**Q1.** `_explain` API에서 `matched: false`가 나오면 어떤 정보를 확인해야 하는가?

<details>
<summary>해설 보기</summary>

`matched: false`는 해당 문서가 쿼리 조건을 만족하지 못한다는 뜻이다. `explanation.description`에 왜 매칭되지 않는지 나타나는데, 주로 두 가지 이유다. 첫째, `must` 또는 `filter` 조건을 만족하지 못한 경우다. 이때 어떤 절이 실패했는지 `details` 하위에서 확인한다. 둘째, Analyzer 결과가 역색인과 불일치하는 경우다. `_analyze` API로 인덱스·검색 시점 토큰을 모두 확인해 교차점이 없는지 점검한다. `matched: false`인 문서를 `_explain`으로 조회하는 것이 Analyzer 불일치 디버깅의 핵심 방법이다.

</details>

**Q2.** `_profile`에서 `build_scorer_count`가 높다는 것은 무엇을 의미하는가?

<details>
<summary>해설 보기</summary>

`build_scorer`는 Lucene이 각 절에 대한 Scorer 객체를 생성하는 과정이다. 세그먼트 수가 많으면 세그먼트마다 Scorer를 생성하므로 `build_scorer_count`가 높아진다. 또한 쿼리가 복잡하고 하위 절이 많은 경우에도 증가한다. 이 값이 높으면 세그먼트 병합(`_forcemerge`)이나 쿼리 단순화로 개선할 수 있다. `build_scorer` 시간이 전체 실행 시간의 큰 비중을 차지한다면 세그먼트 수가 너무 많다는 신호다.

</details>

**Q3.** bool 쿼리에서 filter 절의 실행 순서를 직접 제어할 수 있는가?

<details>
<summary>해설 보기</summary>

Elasticsearch DSL 레벨에서는 절의 배열 순서가 실행 순서를 보장하지 않는다. Lucene이 Cost 기반으로 자동 재배열한다. 다만 `script` 쿼리나 `geo_shape` 같이 Lucene이 cost를 예측하지 못하는 쿼리는 항상 cost가 최대로 간주되어 뒤로 밀린다. 이를 활용해 비용 높은 쿼리는 filter 배열 마지막에 두면 된다. 정확한 실행 순서는 `_profile` API의 `children` 배열로 확인할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: 분산 스코어링 문제](./03-distributed-scoring-problem.md)** | **[홈으로 🏠](../README.md)** | **[다음: 주요 쿼리 유형 비교 ➡️](./05-query-types-comparison.md)**

</div>
