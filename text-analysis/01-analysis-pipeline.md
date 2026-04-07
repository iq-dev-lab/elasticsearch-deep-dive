# 분석 파이프라인 — Character Filter·Tokenizer·Token Filter 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "Hello World!"가 역색인에 `["hello", "world"]`로 저장되기까지 무슨 일이 일어나는가?
- Character Filter, Tokenizer, Token Filter는 각각 어떤 역할을 하는가?
- 파이프라인의 순서를 잘못 배치하면 어떤 결과가 나오는가?
- `_analyze` API로 각 단계의 중간 결과를 어떻게 확인하는가?
- Analyzer는 인덱싱 시점과 검색 시점 모두에서 실행되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"분명히 데이터를 저장했는데 검색이 안 된다"는 문제의 절반은 분석 파이프라인에서 발생한다. "Elasticsearch"를 인덱싱했는데 "elasticsearch"로 검색하면 안 되는 경우, HTML이 포함된 문서에서 태그가 토큰으로 색인되는 경우, 동의어 처리가 기대대로 동작하지 않는 경우 — 모두 파이프라인 구조를 모르면 원인조차 찾지 못한다. 분석 파이프라인은 검색 품질의 출발점이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Analyzer를 지정하지 않고 HTML 내용을 인덱싱
  데이터:
    { "content": "<h1>Elasticsearch Guide</h1><p>Fast search engine</p>" }

  기본 Standard Analyzer 결과:
    ["h1", "elasticsearch", "guide", "h1", "p", "fast", "search", "engine", "p"]
    → HTML 태그가 토큰으로 색인됨
    → "h1", "p" 같은 의미없는 토큰이 역색인에 쌓임
    → 검색 품질 저하 + 역색인 낭비

  해결: html_strip Character Filter 추가
    → HTML 태그 제거 후 토큰화
    → "elasticsearch", "guide", "fast", "search", "engine" 만 색인

실수 2: Token Filter 순서를 잘못 배치
  설정:
    token_filters: [lowercase, stop]  // 올바른 순서
    token_filters: [stop, lowercase]  // 잘못된 순서

  문제:
    불용어(stopwords) 기본 목록: "a", "an", "the", "is" (소문자)
    stop 필터를 lowercase 전에 적용하면:
      "The" → stop 필터 통과 (대문자라 불용어 불일치) → "the" → lowercase
      → "The"가 불용어로 제거되지 않음
    올바른 순서: lowercase 먼저 → 소문자화된 후 stop 필터 적용
    → Token Filter 순서가 결과에 직접 영향

실수 3: _analyze API를 몰라서 디버깅에 수십 분 소비
  상황:
    검색이 안 됨 → 인덱스 삭제 후 재생성 반복
    원인 파악에 실패

  올바른 접근:
    GET /myindex/_analyze { "text": "검색어", "analyzer": "my_analyzer" }
    → 어떤 토큰이 생성되는지 즉시 확인
    → 문제 있는 필터 단계를 pinpoint
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
분석 파이프라인 설계 원칙:

  순서: Character Filter → Tokenizer → Token Filter (고정)
  각 단계의 역할을 명확히 분리

  Character Filter (전처리):
    원본 텍스트를 수정 (토큰화 전)
    용도: HTML 태그 제거, 특수문자 변환, 불필요한 패턴 제거
    예: &amp; → and, <br> → 공백

  Tokenizer (핵심 분리):
    전처리된 텍스트를 토큰 목록으로 분리
    단 하나만 지정 가능 (파이프라인당)
    용도: 공백 기준 분리, 언어별 단어 분리

  Token Filter (후처리, 여러 개 순서대로):
    개별 토큰을 변환/추가/제거
    용도: 소문자화, 불용어 제거, 동의어 추가, 어간 추출, n-gram

  _analyze API로 각 단계 검증:
    파이프라인 설계 후 반드시 실제 토큰 결과 확인
    문제 발생 시 단계별로 좁혀가며 디버깅
```

---

## 🔬 내부 동작 원리

### 1. 파이프라인 3단계 전체 흐름

```
입력 텍스트: "<p>Hello, World! It's a beautiful day.</p>"

┌──────────────────────────────────────────────────────────────┐
│                  Stage 1: Character Filter                   │
│                                                              │
│  html_strip:                                                 │
│    "<p>Hello, World! It's a beautiful day.</p>"              │
│    → "Hello, World! It's a beautiful day."                   │
│                                                              │
│  (HTML 태그 제거, 기타 텍스트는 그대로)                             │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                    Stage 2: Tokenizer                        │
│                                                              │
│  standard tokenizer:                                         │
│    "Hello, World! It's a beautiful day."                     │
│    → ["Hello", "World", "It's", "a", "beautiful", "day"]     │
│                                                              │
│  규칙:                                                        │
│    쉼표, 느낌표, 마침표 → 분리 기준 (제거)                           │
│    어포스트로피(') → 단어 내부 유지 → "It's" 하나의 토큰               │
│    대소문자 그대로 유지 (lowercase는 Token Filter에서)              │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                  Stage 3: Token Filter (순서대로)              │
│                                                              │
│  ① lowercase:                                               │
│    ["Hello", "World", "It's", "a", "beautiful", "day"]       │
│    → ["hello", "world", "it's", "a", "beautiful", "day"]     │
│                                                              │
│  ② stop (영문 불용어):                                         │
│    ["hello", "world", "it's", "a", "beautiful", "day"]       │
│    → ["hello", "world", "beautiful", "day"]                  │
│       ("it's", "a" 제거)                                      │
│                                                              │
│  ③ (선택) stemmer:                                            │
│    ["hello", "world", "beautiful", "day"]                    │
│    → ["hello", "world", "beauti", "day"]                     │
│       ("beautiful" → "beauti" 어간 추출)                       │
└──────────────────────────────────────────────────────────────┘

최종 역색인 토큰: ["hello", "world", "beautiful", "day"]
```

### 2. Character Filter 종류와 동작

```
① html_strip:
  입력: "<b>Fast</b> &amp; <i>powerful</i> search"
  출력: "Fast & powerful search"
  동작:
    HTML 태그(<b>, <i>) 제거
    HTML 엔티티(&amp;) → 실제 문자(&)로 변환
    태그 제거 후 공백 처리 (연속 공백 정리)

② mapping (문자 치환):
  설정:
    "mappings": ["ph => f", "ß => ss"]
  입력: "Elephant" / "Straße"
  출력: "Elefant" / "Strasse"
  용도:
    특수 언어 문자 정규화
    특정 패턴을 다른 문자로 변환 전처리

③ pattern_replace (정규식 치환):
  설정:
    "pattern": "(\\d+)-?(\\d+)",
    "replacement": "$1$2"
  입력: "123-456"
  출력: "123456"
  용도:
    전화번호, 주민등록번호 등 특수 패턴 정규화
    인덱싱 전 데이터 형식 통일

Character Filter 적용 순서:
  여러 Character Filter를 지정하면 순서대로 적용
  char_filter: ["html_strip", "my_mapping"]
  → html_strip 먼저, 그 결과에 my_mapping 적용
```

### 3. Tokenizer 종류와 선택 기준

```
① standard (기본값):
  Unicode Text Segmentation(UAX #29) 기반
  입력: "Hello, World! It's a test."
  출력: ["Hello", "World", "It's", "test"]
  특징:
    대부분 언어에서 기본으로 사용 가능
    구두점을 분리 기준으로 사용하되 제거
    어포스트로피: "It's" → 단어 내부로 유지

② whitespace:
  공백만 기준으로 분리 (구두점 유지)
  입력: "Hello, World!"
  출력: ["Hello,", "World!"]
  특징:
    구두점을 제거하지 않음 (token filter로 처리해야)
    코드, URL 등 구두점이 의미있는 경우 사용

③ keyword:
  전체 텍스트를 하나의 토큰으로 반환
  입력: "Hello World"
  출력: ["Hello World"]
  용도:
    keyword 타입 필드의 내부 tokenizer
    전체 문자열이 하나의 단위인 경우

④ ngram / edge_ngram:
  문자 단위 n-gram 생성
  edge_ngram (자동완성용):
    입력: "hello", min_gram=2, max_gram=5
    출력: ["he", "hel", "hell", "hello"]
  용도: 자동완성, 접두사 검색

⑤ pattern:
  정규식으로 분리 기준 지정
  입력: "foo,bar,baz"
  pattern: ","
  출력: ["foo", "bar", "baz"]
  용도: CSV, 특수 구분자 형식 데이터

언어별 전용 Tokenizer:
  nori_tokenizer   → 한국어 형태소 분석 (Ch3-02 상세)
  kuromoji_tokenizer → 일본어
  smartcn_tokenizer  → 중국어
```

### 4. Token Filter 종류와 조합

```
자주 쓰이는 Token Filter:

① lowercase:
  ["Hello", "WORLD"] → ["hello", "world"]
  거의 모든 파이프라인에 포함 (대소문자 무관 검색)

② stop:
  불용어 목록에 있는 토큰 제거
  기본 영문 불용어: "a", "an", "and", "are", "as", "at", "be", ...
  ["the", "quick", "fox"] → ["quick", "fox"]
  주의: 검색 시에도 같은 stop 필터 필요
        ("the quick" 검색 → "quick"만 검색됨)

③ synonym / synonym_graph:
  동의어 추가
  설정: "laptop => notebook, computer"
  ["laptop"] → ["laptop", "notebook", "computer"]
  search-time에 적용하는 것이 일반적 (Ch3-03 상세)

④ stemmer / snowball:
  어간 추출 (영어)
  ["running", "runs", "ran"] → ["run", "run", "ran"]
  주의: 한국어는 stemmer 대신 Nori 형태소 분석기 사용

⑤ ascii_folding:
  유니코드 문자 → ASCII 변환
  ["résumé", "naïve"] → ["resume", "naive"]
  용도: 악센트 기호 무시 검색

⑥ length:
  최소/최대 길이 조건으로 토큰 필터
  min: 2, max: 100 → 1자 토큰 제거, 100자 초과 제거

⑦ unique:
  중복 토큰 제거
  ["elasticsearch", "elastic", "elasticsearch"] → ["elasticsearch", "elastic"]

Token Filter 순서의 중요성:
  항상 lowercase → stop → stemmer 순서

  이유:
    stop 필터: 소문자 불용어 목록과 비교
    → lowercase 먼저 해야 대소문자 무관하게 불용어 제거
    stemmer: 이미 소문자화된 토큰을 어간 추출
    → lowercase 먼저 해야 일관된 어간 추출
```

### 5. Analyzer 내장 조합 — 기본 제공 Analyzer

```
Elasticsearch 내장 Analyzer:

standard (기본):
  char_filter: []
  tokenizer:   standard
  filter:      [lowercase, stop(비활성)]
  대부분의 영문 텍스트에 적합

simple:
  tokenizer: lowercase (공백+구두점 기준 분리 + 소문자화)
  filter:    []
  숫자를 토큰에서 제외

whitespace:
  tokenizer: whitespace
  filter:    []
  공백만 기준, 소문자화 없음

english:
  char_filter: []
  tokenizer:   standard
  filter:      [english_possessive_stemmer, lowercase, stop(영문), porter_stem]
  영어 전문 검색에 최적화

fingerprint:
  tokenizer: standard
  filter:    [lowercase, ascii_folding, stop, fingerprint, unique]
  중복 감지, 클러스터링 용도

언어별 내장 Analyzer:
  french, german, spanish, italian, ...
  각 언어의 불용어 + 어간 추출기 내장
```

---

## 💻 실전 실험

```bash
# 기본 _analyze API
GET _analyze
{
  "analyzer": "standard",
  "text": "Hello, World! It's a beautiful day."
}

# 단계별 분해 — Tokenizer만 테스트
GET _analyze
{
  "tokenizer": "standard",
  "text": "<p>Hello World</p>"
}

# Character Filter + Tokenizer + Token Filter 조합 테스트
GET _analyze
{
  "char_filter": ["html_strip"],
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "<h1>Hello World</h1> <p>It's a test!</p>"
}

# 커스텀 Analyzer 정의 + 테스트
PUT /analyzer-test
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_html": {
          "type": "html_strip"
        }
      },
      "tokenizer": {
        "my_standard": {
          "type": "standard"
        }
      },
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "my_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      },
      "analyzer": {
        "my_english": {
          "type": "custom",
          "char_filter": ["my_html"],
          "tokenizer": "my_standard",
          "filter": ["lowercase", "my_stop", "my_stemmer"]
        }
      }
    }
  }
}

# 커스텀 Analyzer 테스트
GET /analyzer-test/_analyze
{
  "analyzer": "my_english",
  "text": "<p>Running fast search engines is beautiful</p>"
}

# 인덱스에 정의된 필드 Analyzer 테스트
GET /analyzer-test/_analyze
{
  "field": "title",
  "text": "Elasticsearch Guide"
}

# Token Filter 순서 실험: stop → lowercase vs lowercase → stop
GET _analyze
{
  "tokenizer": "standard",
  "filter": ["stop", "lowercase"],
  "text": "The quick brown fox"
}
# "The"가 stop에서 제거되는가? (소문자 "the"로 먼저 변환 안 했으므로 통과될 수 있음)

GET _analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "The quick brown fox"
}
# lowercase 먼저 → "the" → stop에서 제거됨
```

---

## 📊 Character Filter · Tokenizer · Token Filter 역할 비교

| 단계 | 처리 대상 | 입출력 | 개수 제한 | 주요 용도 |
|------|---------|-------|---------|---------|
| Character Filter | 원본 문자열 전체 | 문자열 → 문자열 | 0개 이상 | HTML 제거, 문자 치환 |
| Tokenizer | Character Filter 결과 문자열 | 문자열 → 토큰 목록 | 정확히 1개 | 단어 분리 |
| Token Filter | 개별 토큰 | 토큰 → 토큰(변환/추가/제거) | 0개 이상, 순서 중요 | 소문자화, 불용어, 동의어 |

---

## ⚖️ 트레이드오프

```
파이프라인 복잡도 vs 검색 품질:
  필터 많을수록:
    이득: 더 정교한 토큰 → 검색 품질 향상
    비용: 인덱싱 속도 감소, 역색인 크기 변화, 디버깅 복잡도 증가

  stop 필터:
    이득: 불용어 제거 → 역색인 크기 감소 → 검색 속도 향상
    비용: "to be or not to be" 같은 문구 검색 불가

  stemmer:
    이득: "run", "running", "runs" 모두 같은 토큰 → recall 향상
    비용: "university"와 "universe"가 같은 어간이 될 수 있음 → precision 저하

  인덱스 시점 vs 검색 시점 Analyzer 일치 필요:
    두 Analyzer가 다르면 → 역색인과 검색어 불일치 → 검색 실패
    → 항상 같은 Analyzer 사용하거나 명시적으로 search_analyzer 설정
    (Ch3-05 상세 설명)
```

---

## 📌 핵심 정리

```
분석 파이프라인 3단계 (순서 고정):
  1. Character Filter: 원본 텍스트 전처리 (HTML 제거 등)
  2. Tokenizer: 텍스트 → 토큰 목록 (정확히 1개)
  3. Token Filter: 토큰 변환/추가/제거 (0개 이상, 순서 중요)

Token Filter 권장 순서:
  lowercase → stop → stemmer
  lowercase가 먼저 → 대소문자 무관 불용어 제거 + 일관된 어간 추출

_analyze API = 파이프라인 디버깅의 핵심:
  전체 Analyzer 지정 또는 단계별 직접 지정 가능
  중간 결과 확인으로 문제 단계 pinpoint

Analyzer는 인덱싱 + 검색 시점 모두 실행:
  인덱싱 시: 문서 텍스트 → 역색인 토큰
  검색 시: 검색어 → 검색 토큰
  두 시점의 Analyzer가 달라지면 검색 불일치 발생
```

---

## 🤔 생각해볼 문제

**Q1.** `html_strip` Character Filter 없이 HTML 태그가 포함된 텍스트를 인덱싱하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

Standard Tokenizer는 HTML 태그를 일반 텍스트로 처리한다. `<h1>Title</h1>`에서 `<`와 `>`는 구두점으로 처리되어 제거되고 `h1`, `Title`, `h1`이 토큰으로 생성된다. 결과적으로 의미 없는 `h1`, `p`, `br` 같은 태그 이름들이 역색인에 쌓이고, 이런 문자열로 문서 검색이 가능해진다. 역색인 낭비 외에도 검색 품질이 저하되고 BM25 IDF 계산이 왜곡된다.

</details>

**Q2.** Tokenizer를 2개 이상 지정할 수 없는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Tokenizer는 하나의 문자열을 토큰 스트림으로 변환하는 역할을 한다. 두 Tokenizer를 순서대로 적용하면 첫 번째 Tokenizer가 이미 토큰 목록을 만들었는데 두 번째 Tokenizer는 문자열을 기대하는 구조적 불일치가 발생한다. 두 개의 토큰화 방식을 조합하려면 Token Filter 단계에서 처리해야 한다(예: `ngram` Token Filter는 이미 분리된 토큰 각각에 n-gram을 적용). Analyzer 자체를 두 개 만들고 `copy_to` 필드로 다른 Analyzer 결과를 별도 필드에 저장하는 방식도 가능하다.

</details>

**Q3.** 검색 성능을 위해 stop 필터로 불용어를 제거했는데, 사용자가 "to be or not to be"를 검색하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

stop 필터가 인덱싱 시점과 검색 시점 모두에 적용되면 "to", "be", "or", "not"이 모두 불용어로 제거되어 검색 쿼리가 비어버린다. 이 경우 match 쿼리는 아무 문서도 반환하지 않는다. 해결 방법은 `match` 쿼리의 `zero_terms_query` 옵션을 `all`로 설정해 쿼리가 비었을 때 전체 문서를 반환하거나, phrase 전용 필드에 stop 필터를 적용하지 않는 별도 Analyzer를 사용하는 것이다. "the quick brown fox" 같은 유명 문구 검색이 중요한 서비스에서는 stop 필터를 신중하게 적용해야 한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Standard·Nori Analyzer ➡️](./02-standard-nori-analyzer.md)**

</div>
