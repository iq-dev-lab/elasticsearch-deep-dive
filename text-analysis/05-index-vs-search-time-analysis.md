# 인덱스 시점 vs 검색 시점 분석 — Analyzer 불일치 디버깅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 인덱스 시점 Analyzer와 검색 시점 Analyzer가 다르면 정확히 어떤 일이 발생하는가?
- `search_analyzer`는 언제 설정하고, 잘못 설정하면 어떤 함정이 있는가?
- `_analyze` API로 Analyzer 불일치를 단계별로 디버깅하는 방법은?
- Nori와 동의어 필터를 조합할 때 순서가 왜 중요한가?
- edge_ngram 자동완성에서 인덱스/검색 Analyzer를 다르게 설정하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"이 단어로 검색이 안 된다"는 신고가 들어왔을 때 가장 먼저 의심해야 할 것이 Analyzer 불일치다. 인덱스 시점에 "스마트폰"으로 색인됐는데 검색 시점에 "스마트"+"폰"으로 분리되면, 역색인에는 없는 토큰을 찾는 상황이 된다. 이 문제는 `_analyze` API와 `_explain` API를 모르면 찾기 매우 어렵다. 특히 동의어, n-gram, 형태소 분석기를 조합할 때 불일치가 잘 발생한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 검색 시점 Analyzer만 변경 후 불일치 발생
  초기 설정:
    "title" → analyzer: standard
    검색: "Elasticsearch" → 소문자화 → "elasticsearch" 탐색
    역색인: "elasticsearch" 존재 → 검색 성공

  변경 후 (Analyzer 업그레이드):
    "title" → analyzer: my_english (stemmer 추가)
    단, 기존 인덱싱된 문서는 standard로 색인된 상태
    새 search_analyzer: my_english

  결과:
    "running" 검색 → my_english: "run" (어간 추출)
    역색인에는 standard로 색인된 "running" 존재
    "run"으로 탐색 → 없음 → 검색 실패!

  해결:
    analyzer 변경 시 → 반드시 전체 재인덱싱 필요
    search_analyzer만 변경은 불일치 초래

실수 2: Nori + synonym 순서 잘못 배치
  설정:
    filter: ["nori_part_of_speech", "synonym_filter", "nori_readingform"]

  문제:
    "스마트폰" → nori 분해 → ["스마트", "폰"]
    → synonym: "스마트폰" 동의어 탐색 → 이미 분해됨 → 동의어 미매칭
    "스마트폰"이 역색인에 없고 분해된 토큰만 있음
    → "핸드폰" 동의어로 "스마트폰" 검색 실패

  올바른 순서:
    synonym_filter 먼저 → 분해 전 원형 단어에 동의어 적용
    filter: ["synonym_graph", "nori_part_of_speech"]

실수 3: edge_ngram 검색 시 동일 Analyzer 사용
  설정 (잘못):
    index analyzer:  edge_ngram (min:2, max:10)
    search analyzer: edge_ngram (동일)

  검색어: "elas"
  edge_ngram 적용: ["el", "ela", "elas"]
  → "el"로 시작하는 모든 단어가 검색됨!
  → "elasticsearch", "elastic", "elephant" 모두 반환

  올바른 설정:
    index analyzer:  edge_ngram (토큰 조각 저장)
    search analyzer: keyword (검색어 그대로 단일 토큰)
    → "elas" 단일 토큰으로 탐색 → edge_ngram으로 생성된 "elas" 조각과 매칭
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Analyzer 불일치 예방 원칙:

  1. 인덱스 Analyzer = 검색 Analyzer (기본 원칙)
     search_analyzer를 별도로 설정하지 않으면 analyzer와 동일
     → 대부분의 경우 analyzer 하나만 설정으로 충분

  2. search_analyzer를 별도 설정하는 경우:
     동의어 (search-time 동의어): analyzer에 없고 search_analyzer에 있음
     edge_ngram 자동완성: index는 ngram, search는 keyword
     → 의도적인 비대칭이며, 이유가 명확해야 함

  3. 설계 변경 시 체크리스트:
     ① _analyze API로 index 시점 토큰 확인
     ② _analyze API로 search 시점 토큰 확인
     ③ 두 결과가 역색인에서 교차할 수 있는지 확인
     ④ _explain API로 실제 탐색 토큰 확인

  4. 동의어 + 형태소 분석 조합 순서:
     search-time: synonym_graph → nori_part_of_speech → lowercase
     (동의어를 먼저 적용해야 복합어 원형에 동의어 매핑)
```

---

## 🔬 내부 동작 원리

### 1. 인덱스 시점 vs 검색 시점 흐름 완전 분해

```
인덱스 시점 (문서 저장):

  문서: {"title": "Elasticsearch Deep Dive"}
        │
        ▼
  analyzer: my_analyzer
        │
  Character Filter → Tokenizer → Token Filter 체인
        │
  토큰: ["elasticsearch", "deep", "dive"]
        │
        ▼
  역색인:
    elasticsearch → [doc_1]
    deep          → [doc_1]
    dive          → [doc_1]

검색 시점 (쿼리 처리):

  match { "title": "Deep Dive" }
        │
        ▼
  search_analyzer: my_analyzer (명시 없으면 analyzer와 동일)
        │
  Character Filter → Tokenizer → Token Filter 체인
        │
  검색 토큰: ["deep", "dive"]
        │
        ▼
  역색인에서 "deep" 탐색 → [doc_1]
  역색인에서 "dive" 탐색 → [doc_1]
  OR 결합 → doc_1 반환 ✅

불일치 시나리오:

  인덱스 시점: analyzer = standard
    토큰: ["elasticsearch", "deep", "dive"]

  검색 시점: search_analyzer = english (stemmer 포함)
    "Deep Dive" → ["deep", "dive"] → stemmer → ["deep", "dive"] (변화 없음)
    "Diving"    → ["diving"] → stemmer → ["dive"]
    역색인에 "dive" 있음 → 성공 (이 경우는 OK)

  문제가 되는 불일치:
  인덱스 시점: english (stemmer)
    "computing" → "comput"
  검색 시점: standard (stemmer 없음)
    "computing" → "computing"
  역색인에 "computing" 없고 "comput"만 있음 → 검색 실패!

핵심 원칙:
  인덱스 토큰 ∩ 검색 토큰 ≠ ∅ 이어야 검색 성공
  두 Analyzer의 결과가 교차점이 있어야 함
```

### 2. `_analyze` API — 단계별 디버깅 절차

```
디버깅 시나리오: "스마트폰" 검색 시 결과 없음

Step 1: 인덱스 시점 토큰 확인
  GET /myindex/_analyze
  {
    "field": "title",       ← 인덱스에 정의된 analyzer 사용
    "text": "최신 스마트폰 추천"
  }
  응답:
    { "tokens": [
      { "token": "최신", "position": 0 },
      { "token": "스마트", "position": 1 },
      { "token": "폰", "position": 2 },
      { "token": "추천", "position": 3 }
    ]}
  → "스마트폰"이 "스마트" + "폰"으로 분해됨

Step 2: 검색 시점 토큰 확인
  GET /myindex/_analyze
  {
    "field": "title",       ← search_analyzer가 있으면 그것 사용
    "text": "스마트폰"
  }
  응답 (만약 search_analyzer가 다른 경우):
    { "tokens": [
      { "token": "스마트폰", "position": 0 }
    ]}
  → 검색 시점에는 "스마트폰" 전체가 하나의 토큰

불일치 발견:
  인덱스: "스마트폰" → ["스마트", "폰"]
  검색:   "스마트폰" → ["스마트폰"]
  역색인에 "스마트폰" 없음 → 검색 실패!

Step 3: 특정 Analyzer 지정해서 비교
  GET _analyze
  {
    "analyzer": "nori_discard",
    "text": "스마트폰"
  }

  GET _analyze
  {
    "analyzer": "nori_mixed",
    "text": "스마트폰"
  }

Step 4: _explain으로 실제 검색 탐색 과정 확인
  GET /myindex/_explain/1
  {
    "query": { "match": { "title": "스마트폰" } }
  }
  → explanation.description 에서 실제 탐색 토큰 확인
```

### 3. Nori + 동의어 필터 순서 문제

```
시나리오: "스마트폰" = "핸드폰" = "휴대폰" 동의어 처리

문제가 되는 순서 (nori 먼저):
  filter: ["nori_part_of_speech", "synonym_graph"]

  인덱싱: "스마트폰 추천"
    nori: "스마트폰" → ["스마트", "폰"]
    synonym: ["스마트", "폰"] → 동의어 매칭 안 됨 (이미 분해됨)
    역색인: 스마트:[doc1], 폰:[doc1]

  검색: "핸드폰"
    nori: "핸드폰" → ["핸드", "폰"]
    synonym: 동의어 없음
    역색인에서 "핸드" 탐색 → 없음
    역색인에서 "폰" 탐색 → doc1
    → "폰"으로만 검색됨 (불완전한 매칭)

올바른 순서 (synonym 먼저):
  filter: ["synonym_graph", "nori_part_of_speech"]

  인덱싱: "스마트폰 추천"
    synonym: "스마트폰" → ["스마트폰", "핸드폰", "휴대폰"] (동의어 확장)
    nori: ["스마트폰" → ["스마트", "폰"],
           "핸드폰"  → ["핸드", "폰"],
           "휴대폰"  → ["휴대", "폰"]]
    역색인: 스마트:[doc1], 폰:[doc1], 핸드:[doc1], 휴대:[doc1]

  검색: "핸드폰"
    synonym: "핸드폰" → ["핸드폰", "스마트폰", "휴대폰"]
    nori: 분해 → ["핸드", "폰", "스마트", "폰", "휴대", "폰"]
    역색인에서 탐색 → doc1 반환 ✅

실무 권장 순서 (search-time 동의어):
  인덱스 filter: [nori_part_of_speech, lowercase]
  검색 filter:  [synonym_graph, nori_part_of_speech, lowercase]

  이유:
    인덱싱: 동의어 없이 순수 형태소만 저장
    검색:   동의어 먼저 확장 → 확장된 각 동의어를 형태소 분해
    → 역색인에 있는 토큰과 일치 보장
```

### 4. edge_ngram 자동완성 Analyzer 불일치의 필요성

```
요구사항: "elas" 입력 시 "elasticsearch" 자동완성

올바른 설계 (의도적 불일치):

  인덱싱 Analyzer (edge_ngram):
    "elasticsearch"
    → edge_ngram (min:2, max:20)
    → ["el", "ela", "elas", "elast", ..., "elasticsearch"]
    역색인: el:[doc1], ela:[doc1], elas:[doc1], ..., elasticsearch:[doc1]

  검색 Analyzer (keyword):
    "elas"
    → keyword (분리 없음)
    → ["elas"]
    역색인에서 "elas" 탐색 → doc1 반환 ✅

의도적 불일치의 이유:
  만약 검색 시에도 edge_ngram 적용:
    "elas" → ["el", "ela", "elas"]
    → "el"로 시작하는 모든 단어가 매칭
    → "electric", "element", "elevator" 모두 반환
    → 자동완성의 의미 없어짐

  keyword tokenizer로 검색:
    "elas" → ["elas"] 단일 토큰
    → 역색인에 "elas"가 정확히 있는 문서만 반환
    → 자동완성 정확도 보장

다른 의도적 불일치 사례:

  케이스: 대소문자 무관 자동완성
    인덱싱:  edge_ngram → lowercase → ["el", "ela", "elas", ...]
    검색:    lowercase → ["elas"] (대소문자 정규화 후 단일 토큰)

  케이스: search-time 동의어
    인덱싱:  동의어 없음 (인덱스 작게 유지)
    검색:    동의어 확장 → 재인덱싱 없이 동의어 갱신
```

---

## 💻 실전 실험

```bash
# Analyzer 불일치 디버깅 전체 절차

# 1. 테스트 인덱스 설정
PUT /debug-test
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_index_anal": {
          "tokenizer": "nori_tokenizer",
          "filter": ["nori_part_of_speech", "lowercase"]
        },
        "my_search_anal": {
          "tokenizer": "nori_tokenizer",
          "filter": [
            { "type": "synonym_graph",
              "synonyms": ["스마트폰, 핸드폰, 휴대폰"] },
            "nori_part_of_speech",
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_index_anal",
        "search_analyzer": "my_search_anal"
      }
    }
  }
}

# 2. 문서 인덱싱
POST /debug-test/_doc/1 { "title": "최신 스마트폰 추천" }
POST /debug-test/_doc/2 { "title": "갤럭시 핸드폰 리뷰" }
POST /debug-test/_refresh

# 3. 인덱스 시점 토큰 확인
GET /debug-test/_analyze
{
  "field": "title",
  "text": "최신 스마트폰 추천"
}

# 4. 검색 시점 토큰 확인 (search_analyzer 직접 지정)
GET /debug-test/_analyze
{
  "analyzer": "my_search_anal",
  "text": "핸드폰"
}

# 5. 실제 검색 테스트
GET /debug-test/_search
{ "query": { "match": { "title": "핸드폰" } } }
# doc1("스마트폰")과 doc2("핸드폰") 모두 반환되어야 함

# 6. _explain으로 실제 탐색 확인
GET /debug-test/_explain/1
{ "query": { "match": { "title": "핸드폰" } } }
# explanation에서 탐색된 토큰과 점수 계산 과정 확인

# edge_ngram 자동완성 Analyzer 불일치 실험
PUT /autocomplete-debug
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "edge_ngram_tok": {
          "type": "edge_ngram", "min_gram": 2, "max_gram": 15,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "ac_index":  { "tokenizer": "edge_ngram_tok", "filter": ["lowercase"] },
        "ac_search": { "tokenizer": "keyword", "filter": ["lowercase"] },
        "ac_wrong":  { "tokenizer": "edge_ngram_tok", "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "name_correct": {
        "type": "text",
        "analyzer": "ac_index",
        "search_analyzer": "ac_search"
      },
      "name_wrong": {
        "type": "text",
        "analyzer": "ac_index",
        "search_analyzer": "ac_wrong"
      }
    }
  }
}

POST /autocomplete-debug/_bulk
{ "index": {} }
{ "name_correct": "elasticsearch", "name_wrong": "elasticsearch" }
{ "index": {} }
{ "name_correct": "elastic stack", "name_wrong": "elastic stack" }
{ "index": {} }
{ "name_correct": "element ui", "name_wrong": "element ui" }
POST /autocomplete-debug/_refresh

# 올바른 자동완성: "elas" → elasticsearch, elastic stack만 반환
GET /autocomplete-debug/_search
{ "query": { "match": { "name_correct": "elas" } } }

# 잘못된 자동완성: "elas" → el, ela, elas 모두 탐색 → element ui도 반환
GET /autocomplete-debug/_search
{ "query": { "match": { "name_wrong": "elas" } } }

# 인덱스에 정의된 analyzer/search_analyzer 확인
GET /debug-test/_mapping?pretty
# properties.title.analyzer, properties.title.search_analyzer 확인
```

---

## 📊 Analyzer 조합 유형별 정리

| 유형 | 인덱스 Analyzer | 검색 Analyzer | 이유 |
|------|--------------|-------------|------|
| 일반 전문 검색 | my_analyzer | (동일) | 대칭 → 일치 보장 |
| 동의어 검색 | 동의어 없음 | 동의어 포함 | 재인덱싱 없이 갱신 |
| 자동완성 | edge_ngram | keyword | 정확한 접두사 탐색 |
| 대소문자 무관 | lowercase | lowercase | 대칭 |
| 형태소+동의어 | nori | synonym → nori | 복합어 원형에 동의어 적용 |

---

## ⚖️ 트레이드오프

```
대칭 Analyzer (인덱스 = 검색):
  이득: 항상 일치 보장, 디버깅 단순
  비용: 동의어 변경 시 재인덱싱 필요

비대칭 Analyzer (의도적 불일치):
  이득: 동의어 갱신 용이, 자동완성 정확도
  비용: 설계 복잡도 증가, 불일치 실수 위험

Token Filter 순서:
  올바른 순서 → 의도한 토큰 생성
  잘못된 순서 → 동의어·불용어 미적용, 찾기 어려운 버그

_analyze API 활용:
  설계 단계에서 필수 검증 → 운영 중 검색 불일치 사전 차단
  변경 시마다 인덱스·검색 시점 양쪽 확인
```

---

## 📌 핵심 정리

```
인덱스 vs 검색 Analyzer:
  기본: 동일한 Analyzer 사용 (search_analyzer 미설정 시 analyzer 사용)
  의도적 비대칭: 동의어(검색만), edge_ngram 자동완성(인덱스만)

불일치 검색 실패 원인:
  인덱스 토큰 ∩ 검색 토큰 = ∅ → 검색 결과 없음
  Analyzer 변경 후 재인덱싱 누락, 순서 실수가 주요 원인

디버깅 절차:
  ① GET /index/_analyze (field 지정) → 인덱스 시점 토큰
  ② GET /index/_analyze (analyzer 지정) → 검색 시점 토큰
  ③ 두 결과 비교 → 교차점 확인
  ④ GET /index/_explain/doc_id → 실제 탐색 과정 확인

Nori + 동의어 올바른 순서:
  검색: synonym_graph → nori_part_of_speech → lowercase
  이유: 복합어 원형 상태에서 동의어 매핑 먼저

edge_ngram 자동완성 비대칭:
  인덱스: edge_ngram (조각 저장)
  검색: keyword (정확한 조각 탐색)
  이유: 검색 시 ngram 적용하면 너무 많은 결과
```

---

## 🤔 생각해볼 문제

**Q1.** 인덱스와 검색 Analyzer가 모두 동일하게 Stemmer를 포함한다면, "running" 인덱싱 후 "run"으로 검색했을 때 결과가 나오는가?

<details>
<summary>해설 보기</summary>

나온다. 인덱싱 시 "running" → Stemmer → "run"이 역색인에 저장된다. 검색 시 "run" → Stemmer → "run" (이미 어간이라 변화 없음)이 검색 토큰이 된다. 역색인에 "run"이 있으므로 검색 성공한다. 반대로 "running"으로 검색하면 Stemmer → "run"으로 변환 후 역색인의 "run"과 일치하므로 마찬가지로 결과가 나온다. 대칭 Stemmer 설정이 recall을 높이는 원리다.

</details>

**Q2.** `_explain` API에서 어떤 정보를 보면 Analyzer 불일치를 확인할 수 있는가?

<details>
<summary>해설 보기</summary>

`_explain` 응답의 `explanation.description` 필드에 실제 탐색에 사용된 토큰이 표시된다. 예를 들어 `"weight(title:스마트 in 0)"`이면 검색어가 "스마트"로 분해되어 탐색했다는 것을 알 수 있다. 예상한 토큰과 다른 토큰이 탐색되고 있다면 search_analyzer 설정이 기대와 다른 것이다. `match` 설명 아래 각 토큰별 `TermQuery` 항목을 보면 정확히 어떤 토큰으로 역색인을 탐색했는지 알 수 있다.

</details>

**Q3.** search-time 동의어를 적용했는데 동의어 파일을 수정하고 `_reload_search_analyzers`를 호출한 후 즉시 검색해보면 새 동의어가 적용되는가?

<details>
<summary>해설 보기</summary>

그렇다. `_reload_search_analyzers`는 search_analyzer의 필터 설정(동의어 파일 포함)을 노드 재시작 없이 즉시 갱신한다. 갱신 후 새로운 검색 요청부터 새 동의어가 적용된다. 단, 이 API는 search_analyzer에만 적용되며 index_analyzer(인덱싱 시점)에는 영향이 없다. 따라서 기존 인덱싱된 문서의 역색인은 변하지 않으며, 새 동의어로 인덱싱된 문서는 재인덱싱해야 새 토큰으로 저장된다. search-time 동의어의 핵심 이점은 역색인은 그대로 두고 검색 쿼리만 확장하는 것이기 때문에 대부분의 경우 재인덱싱 없이 즉각 효과가 나타난다.

</details>

---

<div align="center">

**[⬅️ 이전: 매핑(Mapping) 완전 가이드](./04-mapping-guide.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Query DSL 내부 구조 ➡️](../query-engine/01-query-dsl-internals.md)**

</div>
