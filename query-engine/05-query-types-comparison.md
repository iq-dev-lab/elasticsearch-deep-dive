# 주요 쿼리 유형 비교 — match·term·range·bool·nested 내부 동작

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `match`와 `term` 쿼리의 결정적 차이는 무엇이고, 언제 각각을 써야 하는가?
- `range` 쿼리가 역색인이 아닌 BKD-Tree를 사용하는 이유는?
- `nested` 쿼리가 별도 Lucene 문서를 "조인"하는 방식은?
- `multi_match`의 `best_fields`, `most_fields`, `cross_fields` 차이는?
- 각 쿼리의 상대적 비용 순위는 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"검색이 안 된다"는 문제의 원인 중 가장 흔한 것이 `match`와 `term`의 혼동이다. keyword 필드에 `match`를 쓰거나 text 필드에 `term`을 쓰면 예상치 못한 결과가 나온다. `nested` 쿼리의 원리를 모르면 배열 내 객체 필터링에서 잘못된 결과를 얻는다. 각 쿼리가 내부적으로 어떤 자료구조를 사용하는지 알면 쿼리 선택이 근거 있는 결정이 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: keyword 필드에 match 쿼리
  매핑:
    "category": { "type": "keyword" }
  인덱싱:
    { "category": "Electronics & Computers" }

  시도:
    { "query": { "match": { "category": "electronics" } } }

  결과: 없음
  이유:
    keyword 필드는 분석 없이 저장 → 역색인에 "Electronics & Computers" 전체
    match 쿼리가 keyword 필드에 적용되면 내부적으로 term 쿼리로 변환
    → "electronics" ≠ "Electronics & Computers" → 매칭 실패

  올바른 쿼리:
    { "query": { "term": { "category": "Electronics & Computers" } } }
    또는 대소문자 무관하려면 keyword를 normalize로 소문자화하거나
    text 필드에서 검색

실수 2: object 타입 배열에서 잘못된 nested 쿼리 없이 필터링
  매핑:
    "tags": { "type": "object" }  (기본 object, nested 아님)
  데이터:
    { "tags": [
      { "name": "red",  "count": 10 },
      { "name": "blue", "count": 5 }
    ]}

  쿼리:
    { "bool": { "filter": [
      { "term": { "tags.name": "red" } },
      { "term": { "tags.count": 5 } }
    ]}}

  결과: 매칭됨 (잘못된 결과!)
  이유:
    object 타입은 flat 변환:
      tags.name  = ["red", "blue"]
      tags.count = [10, 5]
    "red"와 5가 독립 배열에서 각각 존재 → AND 조건 통과
    하지만 "red"와 5는 같은 태그가 아님!

  올바른 설계:
    "tags": { "type": "nested" }
    { "nested": { "path": "tags", "query": { "bool": {
      "filter": [
        { "term": { "tags.name": "red" } },
        { "term": { "tags.count": 5 } }
      ]
    }}}}
    → 같은 nested 객체 내에서 두 조건을 동시에 만족해야 함

실수 3: range 쿼리 성능을 역색인 탐색으로 오해
  오해:
    "price: 1~200000 범위 쿼리는 역색인에서 각 숫자 토큰을 탐색한다"

  실제:
    숫자 필드는 BKD-Tree (Block KD-Tree) 사용
    range: { price: { gte: 100, lte: 200000 } }
    → BKD-Tree에서 해당 범위를 O(log N) 탐색
    → 역색인 탐색이 아님
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
쿼리 선택 가이드:

  사용자 입력 텍스트 검색:
    → match (Analyzer 적용, 전문 검색)

  정확한 값 필터링 (카테고리, 상태):
    → term (keyword 필드, 정확 일치, Filter Context)

  여러 값 중 하나:
    → terms (term의 OR 버전)

  숫자/날짜 범위:
    → range (BKD-Tree, Filter Context)

  여러 필드 동시 검색:
    → multi_match (best_fields: 점수 최대 필드, most_fields: 모든 필드 합산)

  null/존재 여부:
    → exists

  배열 내 객체 관계 보존 필터:
    → nested

  ID 기반 조회:
    → ids (단일 또는 배열)

  복합 조건:
    → bool (must/filter/should/must_not 조합)
```

---

## 🔬 내부 동작 원리

### 1. match vs term — Analyzer 유무

```
match 쿼리:

  GET /index/_search
  { "query": { "match": { "title": "Mechanical Keyboard" } } }

  내부 처리:
    ① search_analyzer 실행:
       "Mechanical Keyboard" → ["mechanical", "keyboard"]
    ② 각 토큰에 대해 TermQuery 생성:
       TermQuery("title", "mechanical")
       TermQuery("title", "keyboard")
    ③ OR로 결합 (기본) → BooleanQuery(should, minimumMatch=1)
    ④ 각 TermQuery: Posting List 탐색 → 점수 계산

  operator: "and" 설정 시:
    → BooleanQuery(must) → 두 단어 모두 있어야 함

  text 필드에 적합: Analyzer로 분리한 토큰이 역색인과 일치
  keyword 필드에 적용하면: 내부적으로 term 쿼리 변환

─────────────────────────────────────────────────────────

term 쿼리:

  GET /index/_search
  { "query": { "term": { "category": "electronics" } } }

  내부 처리:
    ① Analyzer 없음 → "electronics" 그대로 사용
    ② TermQuery("category", "electronics")
    ③ Term Dictionary(FST)에서 "electronics" 탐색
    ④ Posting List 반환

  keyword 필드에 적합: 분석 없이 원본 값과 정확 일치
  text 필드에 term 사용 시:
    "Mechanical Keyboard" → 역색인에 "mechanical", "keyboard"로 저장됨
    term("title", "Mechanical Keyboard") → 역색인에 없음 → 결과 없음!

결론:
  text 필드 + 전문 검색 → match
  keyword 필드 + 정확 일치 → term
```

### 2. range 쿼리 — BKD-Tree 내부

```
BKD-Tree (Block KD-Tree):
  다차원 공간을 분할하는 트리 자료구조
  1차원(가격, 날짜): 정렬된 범위 탐색에 최적
  다차원(위경도): geo_point 범위 탐색

range 쿼리 처리:
  { "range": { "price": { "gte": 100000, "lte": 200000 } } }

  내부:
    ① price 필드의 BKD-Tree 접근
    ② 100000 이상 200000 이하 범위 탐색
    ③ O(log N + K) 비용 (N=전체, K=결과 수)
    ④ 해당 범위 문서 ID 목록 반환

  vs 역색인 탐색의 차이:
    역색인: 정확한 값 탐색 (term 쿼리)
    BKD-Tree: 범위 탐색 (range 쿼리)
    → 숫자/날짜는 역색인이 없거나 비효율적
    → BKD-Tree가 범위 탐색에 훨씬 최적

날짜 range:
  { "range": { "created_at": {
    "gte": "2024-01-01",
    "lte": "2024-12-31",
    "format": "yyyy-MM-dd"
  }}}
  → 날짜를 epoch ms로 변환 후 BKD-Tree 탐색

Filter Context에서 BKD-Tree:
  range를 filter에 배치 → 결과를 Roaring Bitset으로 캐시
  반복 요청에서 캐시 재사용 → 매우 빠름
```

### 3. multi_match — 여러 필드 동시 검색

```
multi_match 쿼리 유형:

  best_fields (기본):
    각 필드에서 가장 높은 점수를 가진 필드 선택
    { "multi_match": {
      "query": "keyboard review",
      "fields": ["title", "description"],
      "type": "best_fields"
    }}

    내부:
      title에서 "keyboard review" 점수 계산
      description에서 "keyboard review" 점수 계산
      두 점수 중 최댓값 선택 + tie_breaker × 나머지 합산
    
    tie_breaker (기본 0.0):
      0.0: 최고 점수만 사용
      0.3: 최고 점수 + 다른 필드 점수 × 0.3
      1.0: 모든 필드 점수 합산 (most_fields와 동일)

    적합: 검색어가 주로 한 필드에 있는 경우
          (제목에 있으면 본문보다 더 관련성 높음)

─────────────────────────────────────────────────────────

  most_fields:
    모든 필드의 점수를 합산
    { "multi_match": {
      "query": "keyboard",
      "fields": ["title^3", "description", "tags"],
      "type": "most_fields"
    }}

    내부:
      title 점수 × 3 (boost)
      description 점수
      tags 점수
      → 모두 합산

    적합: 여러 필드가 함께 검색어를 포함할 때 강화
          (title과 description 모두 포함하면 더 관련성 높음)

─────────────────────────────────────────────────────────

  cross_fields:
    여러 필드를 하나의 큰 필드처럼 처리
    { "multi_match": {
      "query": "John Smith",
      "fields": ["first_name", "last_name"],
      "type": "cross_fields"
    }}

    문제 해결:
      best_fields/most_fields:
        "John" → first_name 점수
        "Smith" → last_name 점수
        → 각 필드에서 독립 계산 → "John Smith"가 한 사람인지 고려 못함

      cross_fields:
        모든 필드를 하나의 통합 필드로 처리
        "John"을 first_name 또는 last_name 어디서든 찾음
        "Smith"도 동일
        → 사람 이름, 주소 등 여러 필드에 분산된 정보에 적합
```

### 4. nested 쿼리 — 객체 관계 보존

```
nested 타입과 일반 object 타입의 차이:

  object (기본) → flat 변환:
    {"tags": [{"name": "red", "count": 10}, {"name": "blue", "count": 5}]}

    역색인 저장:
      tags.name  = ["red", "blue"]   (배열 합산)
      tags.count = [10, 5]           (배열 합산)

    → 배열 내 각 객체의 필드들이 섞임
    → "name=red AND count=5" 쿼리 → 각 배열에서 독립 확인 → 틀린 매칭

  nested → 별도 Lucene 문서:
    원본 문서: doc_42
    nested 문서:
      hidden_doc_42_0: { tags.name: "red",  tags.count: 10 }
      hidden_doc_42_1: { tags.name: "blue", tags.count: 5 }

    → 각 nested 객체가 독립적인 Lucene 문서로 저장 (검색 불가, 숨김)
    → nested 쿼리가 해당 숨겨진 문서를 탐색

  nested 쿼리 처리:
    { "nested": {
      "path": "tags",
      "query": { "bool": { "filter": [
        { "term": { "tags.name": "red" } },
        { "term": { "tags.count": 10 } }
      ]}}
    }}

    내부:
      hidden_doc_42_0 탐색: name=red ✅, count=10 ✅ → 매칭
      → 부모 doc_42를 결과에 포함

      hidden_doc_42_1 탐색: name=blue ✗ → 미매칭
      → doc_42를 이 nested 조건으로 포함 안 함

  nested 비용:
    각 nested 객체 = 별도 Lucene 문서 → 인덱스 크기 증가
    nested 쿼리 = "조인" 연산 → 일반 쿼리보다 비쌈
    → 정말 관계 보존 쿼리가 필요할 때만 사용
```

### 5. 쿼리 비용 계층 요약

```
같은 결과를 낼 수 있다면 비용 낮은 쿼리를 선택:

  Tier 1 (가장 빠름):
    ids           → ID 직접 조회
    term + filter → Roaring Bitset 캐시 활용
    exists        → doc_values 유무 확인

  Tier 2 (빠름):
    match, multi_match → Analyzer + TermQuery
    terms              → 여러 term OR
    range (숫자/날짜)  → BKD-Tree

  Tier 3 (중간):
    match_phrase → position 검증 추가
    bool         → 복합 절 비용 합산
    prefix       → FST 접두사 열거

  Tier 4 (느림):
    wildcard (*앞에 있음), fuzzy, regexp → FST 전체 열거
    span 쿼리 → 복잡 position 탐색

  Tier 5 (가장 느림):
    script       → 문서마다 런타임 실행
    percolate    → 역방향 쿼리
    nested       → hidden 문서 조인 비용
    more_like_this → 유사 문서 탐색
```

---

## 💻 실전 실험

```bash
# match vs term 차이 실험
PUT /type-test
{
  "mappings": {
    "properties": {
      "title":    { "type": "text" },
      "category": { "type": "keyword" }
    }
  }
}

POST /type-test/_doc/1
{ "title": "Mechanical Keyboard", "category": "Electronics & Computers" }
POST /type-test/_refresh

# text 필드에 match (Analyzer 적용)
GET /type-test/_search
{ "query": { "match": { "title": "mechanical" } } }
# → 결과 있음 (역색인에 "mechanical" 토큰 존재)

# text 필드에 term (Analyzer 없음)
GET /type-test/_search
{ "query": { "term": { "title": "mechanical" } } }
# → 결과 있음 (소문자 "mechanical"이 역색인에 있음)

GET /type-test/_search
{ "query": { "term": { "title": "Mechanical" } } }
# → 결과 없음! 역색인에 "Mechanical"(대문자) 없음

# keyword 필드에 term (정확 일치)
GET /type-test/_search
{ "query": { "term": { "category": "Electronics & Computers" } } }
# → 결과 있음

GET /type-test/_search
{ "query": { "term": { "category": "electronics" } } }
# → 결과 없음 (대소문자, 특수문자 포함 정확 일치 필요)

# nested 객체 관계 실험
PUT /nested-test
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "tags": {
        "type": "nested",
        "properties": {
          "name":  { "type": "keyword" },
          "count": { "type": "integer" }
        }
      }
    }
  }
}

POST /nested-test/_doc/1
{
  "name": "Product A",
  "tags": [
    { "name": "red",  "count": 10 },
    { "name": "blue", "count": 5 }
  ]
}
POST /nested-test/_refresh

# 잘못된 쿼리 (object처럼): red이면서 count=5 → 잘못 매칭됨
# (nested 타입에서는 이 쿼리가 올바르게 동작)

# 올바른 nested 쿼리: red이면서 count=10
GET /nested-test/_search
{
  "query": {
    "nested": {
      "path": "tags",
      "query": { "bool": { "filter": [
        { "term": { "tags.name": "red" } },
        { "term": { "tags.count": 10 } }
      ]}}
    }
  }
}
# → Product A 반환 (red 태그의 count가 10이므로)

# red이면서 count=5 (같은 객체 내)
GET /nested-test/_search
{
  "query": {
    "nested": {
      "path": "tags",
      "query": { "bool": { "filter": [
        { "term": { "tags.name": "red" } },
        { "term": { "tags.count": 5 } }
      ]}}
    }
  }
}
# → 결과 없음 (red의 count는 10, 5는 blue의 count)

# multi_match 유형 비교
GET /type-test/_search
{
  "query": {
    "multi_match": {
      "query": "keyboard",
      "fields": ["title^3", "category"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}
```

---

## 📊 주요 쿼리 비교

| 쿼리 | Analyzer | 자료구조 | 적합 필드 | 비용 |
|------|---------|---------|---------|------|
| `match` | O | 역색인 | text | 중간 |
| `term` | X | 역색인 | keyword | 낮음 |
| `terms` | X | 역색인 | keyword | 낮음 |
| `range` | X | BKD-Tree | 숫자, 날짜 | 낮음~중간 |
| `match_phrase` | O | 역색인 + position | text | 중간~높음 |
| `prefix` | X | FST 접두사 | keyword | 중간 |
| `wildcard` (*앞) | X | FST 전체 열거 | keyword | 높음 |
| `nested` | - | hidden 문서 조인 | nested 타입 | 높음 |
| `script` | - | 런타임 실행 | 모든 필드 | 매우 높음 |

---

## ⚖️ 트레이드오프

```
nested vs object:
  nested: 관계 보존 쿼리 가능, 인덱스 크기·비용 증가
  object: 빠름, 배열 내 객체 관계 혼재 (flat 변환)
  → 관계 쿼리가 반드시 필요할 때만 nested 사용

match vs term on keyword:
  match on keyword → 내부적으로 term으로 변환 (동일 결과)
  하지만 의도를 명확히 하려면 keyword에는 term 사용 권장
  → 코드 가독성과 의도 명시

multi_match type 선택:
  best_fields: 단일 필드에서 집중 매칭 (제목 검색)
  most_fields: 여러 필드 포괄 강화 (종합 검색)
  cross_fields: 분산 정보 통합 (이름, 주소)
  → 서비스 특성에 맞는 점수 전략 선택
```

---

## 📌 핵심 정리

```
match vs term:
  match: Analyzer 적용 → text 필드 전문 검색
  term:  Analyzer 없음 → keyword 필드 정확 일치
  혼동 시 검색 실패 (역색인 토큰 불일치)

range:
  숫자/날짜 → BKD-Tree (역색인 아님)
  filter에 배치 → Roaring Bitset 캐시 활용
  → 자주 사용되는 날짜 범위 필터링에 매우 효과적

nested:
  배열 내 객체를 별도 Lucene 문서로 저장
  nested 쿼리로 객체 내 필드 관계 정확히 필터링
  비용: object 대비 높음 → 필요할 때만 사용

multi_match 유형:
  best_fields: 최고 점수 필드 우선 (tie_breaker로 나머지 반영)
  most_fields: 모든 필드 점수 합산
  cross_fields: 여러 필드를 하나로 처리 (분산 정보 통합)

비용 순위:
  term/filter < match < range < match_phrase < prefix < wildcard(*) < nested < script
```

---

## 🤔 생각해볼 문제

**Q1.** `match` 쿼리를 keyword 필드에 적용하면 내부적으로 어떤 일이 일어나는가?

<details>
<summary>해설 보기</summary>

match 쿼리는 keyword 필드에서는 Analyzer 적용 없이 그대로 term 쿼리로 변환된다. keyword 필드의 search_analyzer가 없으면 keyword tokenizer(전체 문자열을 단일 토큰으로)를 사용한다. 따라서 `match { "category": "electronics" }`는 `term { "category": "electronics" }`와 동일하게 동작한다. 단, keyword 필드에 normalizer가 설정되어 있으면(예: lowercase normalizer) match 시 normalizer가 적용된다. 이 경우 대소문자 무관 정확 일치가 가능해진다.

</details>

**Q2.** nested 쿼리에서 `inner_hits`를 사용하면 어떤 추가 정보를 얻을 수 있는가?

<details>
<summary>해설 보기</summary>

`inner_hits`를 nested 쿼리에 추가하면 부모 문서뿐 아니라 매칭된 nested 객체(숨겨진 문서)의 내용도 응답에 포함된다. 예를 들어 "tags 중 name=red인 태그" 쿼리에서 inner_hits를 사용하면 매칭된 태그 객체 `{"name": "red", "count": 10}`도 응답에 나타난다. 여러 nested 객체 중 어떤 객체가 쿼리를 만족했는지 확인할 수 있어, 특정 조건을 만족한 배열 요소를 추출하는 용도로 활용된다.

</details>

**Q3.** `terms` 쿼리(여러 값 OR)와 `bool.should` + 여러 `term`의 성능 차이는?

<details>
<summary>해설 보기</summary>

`terms` 쿼리는 내부적으로 여러 TermQuery를 OR로 결합하는 것과 동일하지만, 단일 쿼리 객체로 표현되어 최적화가 적용된다. 반면 `bool.should`에 여러 `term`을 나열하면 각각 별도 쿼리 객체가 생성되고 조합 비용이 추가된다. 결과는 동일하지만 `terms`가 대부분 더 효율적이다. 또한 `terms` 쿼리는 filter에 두면 Roaring Bitset 캐시도 활용된다. 여러 값의 OR 조건은 `terms`로 표현하는 것이 권장된다.

</details>

---

<div align="center">

**[⬅️ 이전: 쿼리 실행 계획](./04-query-execution-plan.md)** | **[홈으로 🏠](../README.md)** | **[다음: 벡터 검색(kNN) ➡️](./06-vector-search-knn.md)**

</div>
