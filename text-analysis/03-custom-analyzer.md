# 커스텀 Analyzer 설계 — 동의어·Stemming·n-gram

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 동의어를 인덱스 시점에 처리하는 것과 검색 시점에 처리하는 것의 차이는 무엇인가?
- Porter/Snowball Stemmer가 "running"을 "run"으로 줄이는 알고리즘 원리는?
- Stemmer 사용 시 검색 precision이 낮아지는 이유는 무엇인가?
- n-gram Token Filter는 부분 검색을 어떻게 가능하게 하고, 그 비용은 얼마나 큰가?
- edge_ngram과 ngram의 차이는 무엇이고 자동완성에 왜 edge_ngram을 쓰는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"노트북"으로 검색했는데 "랩톱"이 나오지 않는 문제, "running"으로 검색했는데 "run"이 나오지 않는 문제, "스마"까지만 입력해도 자동완성이 동작해야 하는 요구사항 — 이 모두가 커스텀 Analyzer 설계로 해결해야 하는 실무 문제다. 단순히 필터를 붙이는 것이 아니라 인덱스 크기, 검색 정확도, 성능 사이의 트레이드오프를 이해하고 설계해야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 동의어를 인덱스 시점에만 적용
  설정:
    "index_analyzer" filter: ["synonym"]
    "search_analyzer" filter: []

  동의어 설정: "notebook, laptop, 노트북"

  인덱싱 시:
    "노트북" → ["노트북", "notebook", "laptop"] 모두 역색인

  검색 시 (동의어 없음):
    "laptop" → Analyzer 결과: ["laptop"]
    역색인: "laptop" 있음 → 검색 성공

  문제:
    "노트북" → ["노트북"] 검색
    역색인에 "노트북" 있음 → 성공
    역색인에 "notebook"도 있음 (동의어로 인덱싱됨) → 검색 안 함

    인덱스 시점 동의어: 역색인이 3배 늘어남 (모든 동의어 저장)
    동의어 추가 시 → 기존 문서 재인덱싱 필요
    → 유지보수 어려움

실수 2: ngram을 너무 작게 설정 (min_gram: 1)
  설정:
    "ngram": { "type": "ngram", "min_gram": 1, "max_gram": 5 }

  "hello" → ["h", "e", "l", "l", "o", "he", "el", "ll", "lo", "hel", ...]
  1글자짜리 토큰 포함 → 단일 알파벳으로 수많은 문서가 검색됨
  역색인 크기 폭증 (원본 텍스트 대비 수십~수백 배)
  → 인덱스 크기 폭발, 검색 정확도 급락

  min_gram: 2 이상 설정 권장

실수 3: Stemmer 사용 후 precision 저하를 예상 못함
  상황:
    영문 서비스에 Porter Stemmer 적용

  문제:
    "university" → "univers"
    "universe"   → "univers"
    → 두 단어가 같은 어간으로 매핑

    "university 검색" → "universe" 관련 문서도 반환
    → 의도하지 않은 검색 결과

  해결:
    Stemmer 미적용 필드와 병행 사용 (multi-field)
    스코어링으로 정확 매칭 우선 순위 부여
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
동의어 처리 설계 원칙:

  권장: 검색 시점(search-time) 동의어
    이유:
      인덱스 크기 변화 없음
      동의어 추가/수정 → 재인덱싱 없이 즉시 적용 가능
      (_reload_search_analyzers API 사용)

  방법:
    index_analyzer:  동의어 필터 없음
    search_analyzer: 동의어 필터 포함

    "title": {
      "type": "text",
      "analyzer": "my_index_analyzer",
      "search_analyzer": "my_search_analyzer"
    }

n-gram 설계 원칙:
  용도 명확히:
    자동완성 (접두사만) → edge_ngram (min: 2, max: 15)
    부분 검색 (중간 포함) → ngram (min: 2, max: 5)
    → ngram은 인덱스 크기를 신중하게 고려

  별도 필드로 분리:
    "title":          text (일반 전문 검색)
    "title.ngram":    text (ngram Analyzer, 부분 검색용)
    "title.autocomplete": text (edge_ngram Analyzer, 자동완성용)
    → 용도별 필드 분리로 일반 검색 성능 유지
```

---

## 🔬 내부 동작 원리

### 1. 동의어 처리 — index-time vs search-time

```
동의어 사전 예시:
  laptop, notebook, 노트북 → 모두 동의어
  스마트폰, 휴대폰, 핸드폰

방법 1: 인덱스 시점 동의어 (index-time synonym)

  인덱싱: "노트북 추천"
    Analyzer (synonym 포함):
    "노트북" → ["노트북", "laptop", "notebook"]
    역색인:    노트북:[doc1], laptop:[doc1], notebook:[doc1]

  검색: "laptop"
    Analyzer (synonym 없음):
    "laptop" → ["laptop"]
    역색인에서 "laptop" 탐색 → doc1 반환 ✅

  장점: 검색어에 동의어 처리 불필요 (역색인에 모두 있음)
  단점:
    역색인 크기 증가 (동의어 수 × 문서 수)
    동의어 추가 → 전체 재인덱싱 필요

방법 2: 검색 시점 동의어 (search-time synonym) ← 권장

  인덱싱: "노트북 추천"
    Analyzer (synonym 없음):
    "노트북" → ["노트북"]
    역색인:    노트북:[doc1]

  검색: "laptop"
    search_analyzer (synonym 포함):
    "laptop" → ["laptop", "notebook", "노트북"]
    역색인에서 "laptop" 탐색 → 없음
    역색인에서 "notebook" 탐색 → 없음
    역색인에서 "노트북" 탐색 → doc1 반환 ✅

  장점:
    인덱스 크기 변화 없음
    동의어 파일 수정 → _reload_search_analyzers로 즉시 반영
    재인덱싱 불필요
  단점:
    검색어마다 동의어 확장 → 검색 쿼리 복잡도 증가

동의어 파일 형식:
  # 단방향 동의어 (검색어 확장)
  laptop => notebook, 노트북

  # 양방향 동의어 (모두 동등)
  laptop, notebook, 노트북

  주의:
    단방향(=>): "laptop" 검색 시만 확장, "노트북" 검색 시 확장 안 함
    양방향(,):  어느 방향으로 검색해도 동의어 그룹 전체 탐색

search-time 동의어 갱신:
  POST /myindex/_reload_search_analyzers
  → 파일 재읽기, 재인덱싱 없이 즉시 적용
  → index_analyzer에는 적용 안 됨 (search_analyzer만 갱신)
```

### 2. Stemming — 어간 추출 알고리즘

```
어간 추출(Stemming)의 목적:
  "run", "runs", "running", "runner"
  → 모두 "run"이라는 같은 어근에서 파생
  → 하나의 토큰 "run"으로 통합 → recall 향상

Porter Stemmer (영어, 규칙 기반):
  5단계 규칙을 순서대로 적용하여 어미 제거

  Step 1a: 복수형 처리
    "caresses" → "caress"
    "ponies"   → "poni"
    "cats"     → "cat"

  Step 1b: ed, ing 처리
    "agreed"   → "agree"
    "plastered"→ "plaster"
    "running"  → "run"

  Step 2: 접미사 처리
    "relational"  → "relate"
    "conditional" → "condition"
    "rational"    → "rational"

  Step 3: 추가 접미사
    "triplicate" → "triplic"
    "formative"  → "form"

  Step 4: 복합 접미사
    "hopeful"     → "hope"
    "electrical"  → "electr"

  Step 5: 최종 정리
    "probate"  → "probat"
    "rate"     → "rate"

Snowball Stemmer:
  Porter의 개선 버전, 더 많은 규칙, 여러 언어 지원
  영어: english, German: german, French: french 등

Stemmer 문제점:
  Over-stemming (과한 어간 추출):
    "general"    → "general"
    "generalize" → "general"
    "generously" → "general"
    → 의미가 다른 단어들이 같은 토큰으로 병합 → precision 저하

  Under-stemming (부족한 어간 추출):
    "run" vs "ran" → "ran"이 다르게 처리될 수 있음
    → recall 저하

Lemmatization vs Stemming:
  Stemming: 규칙 기반, 빠름, 언어학적으로 부정확할 수 있음
  Lemmatization: 사전 기반, 느림, 언어학적으로 정확한 기본형 반환
  → ES 기본: Stemmer (속도 우선)
  → 한국어: Nori가 형태소 분석(lemmatization에 가까움) 수행
```

### 3. n-gram — 부분 검색 활성화

```
n-gram이란:
  연속된 N개의 문자 또는 단어 조각을 토큰으로 생성

문자 단위 n-gram (ngram Token Filter):
  입력: "search"
  min_gram=2, max_gram=3

  생성 토큰:
    2-gram: ["se", "ea", "ar", "rc", "ch"]
    3-gram: ["sea", "ear", "arc", "rch"]
  모두 합산: ["se", "ea", "ar", "rc", "ch", "sea", "ear", "arc", "rch"]

부분 검색이 가능한 이유:
  "arc" 검색 → "arc"가 역색인에 존재 → "search" 포함 문서 반환
  "rch" 검색 → 역색인에 존재 → "search" 포함 문서 반환
  즉, 단어의 어느 위치든 검색 가능

인덱스 크기 비용:
  원본: "search" (6자)
  생성 토큰 수: (max_gram - min_gram + 1) × (단어 길이 - n + 1) 개
  min=2, max=3: 5 + 4 = 9개 토큰 vs 원본 1개
  → 약 9배 역색인 크기 증가

  실제 영향:
    10만 단어 인덱스에 ngram 적용
    평균 단어 7자, min=2, max=5
    토큰 수: 약 28개/단어 → 원본 대비 28배 역색인 증가!
    → 인덱스 크기, 인덱싱 속도, 검색 성능에 큰 영향

edge_ngram (접두사만):
  입력: "search"
  min_gram=2, max_gram=5

  생성 토큰: ["se", "sea", "sear", "searc"]
  → 시작 부분(edge)의 n-gram만 생성

  자동완성에 최적:
    사용자가 "sea" 입력 → "sea"가 역색인에 있음 → "search" 반환
    → 접두사 기반 자동완성 구현
  비용: ngram 대비 훨씬 적음 (단어 앞부분만)

ngram vs edge_ngram 선택:
  부분 검색 (어느 위치든) → ngram
  자동완성 (앞에서부터) → edge_ngram
  용도가 다른 만큼 필드도 분리
```

### 4. 커스텀 Analyzer 종합 설계 예시

```
요구사항:
  상품 검색 서비스 (한영 혼합)
  1. 전문 검색: 형태소 분석, 동의어
  2. 자동완성: 접두사 입력 시 상품명 추천
  3. 검색 정확도: 부정확한 검색어도 일부 허용

설계:

PUT /products-v2
{
  "settings": {
    "analysis": {
      "char_filter": {
        "clean_html": { "type": "html_strip" }
      },
      "tokenizer": {
        "nori_mixed": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed"
        },
        "edge_ngram_tok": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20,
          "token_chars": ["letter", "digit"]
        }
      },
      "filter": {
        "pos_filter": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "J", "SC", "SP", "XSV", "XSA"]
        },
        "search_synonym": {
          "type": "synonym_graph",
          "synonyms_path": "analysis/synonyms.txt",
          "updateable": true
        }
      },
      "analyzer": {
        "index_korean": {
          "type": "custom",
          "char_filter": ["clean_html"],
          "tokenizer": "nori_mixed",
          "filter": ["pos_filter", "lowercase"]
        },
        "search_korean": {
          "type": "custom",
          "char_filter": ["clean_html"],
          "tokenizer": "nori_mixed",
          "filter": ["pos_filter", "lowercase", "search_synonym"]
        },
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "edge_ngram_tok",
          "filter": ["lowercase"]
        },
        "autocomplete_search": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "index_korean",
        "search_analyzer": "search_korean",
        "fields": {
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete_index",
            "search_analyzer": "autocomplete_search"
          }
        }
      }
    }
  }
}
```

---

## 💻 실전 실험

```bash
# 동의어 search-time 적용 테스트
PUT /synonym-test
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym": {
          "type": "synonym_graph",
          "synonyms": ["laptop, notebook, 노트북", "phone, 스마트폰, 휴대폰"]
        }
      },
      "analyzer": {
        "index_anal": {
          "tokenizer": "standard",
          "filter": ["lowercase"]
        },
        "search_anal": {
          "tokenizer": "standard",
          "filter": ["lowercase", "my_synonym"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "index_anal",
        "search_analyzer": "search_anal"
      }
    }
  }
}

POST /synonym-test/_doc/1 { "title": "노트북 추천" }
POST /synonym-test/_doc/2 { "title": "laptop 구매 가이드" }
POST /synonym-test/_refresh

# "laptop" 검색 시 "노트북 추천" 문서도 나와야 함
GET /synonym-test/_search
{ "query": { "match": { "title": "laptop" } } }

# Stemmer 비교 실험
GET _analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", { "type": "stemmer", "language": "english" }],
  "text": "running runs runner university generalization"
}

# edge_ngram 자동완성 실험
PUT /autocomplete-test
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "edge_ngram": {
          "type": "edge_ngram",
          "min_gram": 2, "max_gram": 10,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "ac_index": { "tokenizer": "edge_ngram", "filter": ["lowercase"] },
        "ac_search": { "tokenizer": "keyword",   "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ac_index",
        "search_analyzer": "ac_search"
      }
    }
  }
}

POST /autocomplete-test/_bulk
{ "index": {} }
{ "name": "elasticsearch" }
{ "index": {} }
{ "name": "elastic stack" }
{ "index": {} }
{ "name": "kibana" }
POST /autocomplete-test/_refresh

# "ela" 입력 시 자동완성
GET /autocomplete-test/_search
{ "query": { "match": { "name": "ela" } } }

# ngram 인덱스 크기 측정
GET /autocomplete-test/_stats/store?pretty
```

---

## 📊 동의어 처리 방식 비교

| 항목 | 인덱스 시점 | 검색 시점 (권장) |
|------|-----------|----------------|
| 역색인 크기 | 증가 (동의어 수 × 문서) | 변화 없음 |
| 동의어 추가 | 재인덱싱 필요 | reload_search_analyzers |
| 검색 속도 | 빠름 (이미 역색인에 있음) | 약간 느림 (쿼리 확장) |
| 유지보수 | 어려움 | 쉬움 |
| 추천 | 변경 없는 동의어 | 대부분의 경우 |

---

## ⚖️ 트레이드오프

```
Stemmer:
  이득: recall 향상 (어간이 같으면 모두 검색)
  비용: precision 저하 (의미 다른 단어가 같은 토큰)
  → 정보 검색(문서/뉴스)에 적합, 상품명 검색엔 주의

n-gram:
  이득: 부분 검색 가능
  비용: 역색인 크기 폭증 (최소 수 배 ~ 수십 배)
        인덱싱 속도 저하, 검색 속도 저하, 메모리 증가
  → 특수 목적 필드에만, 전체 필드에 적용 지양

edge_ngram:
  이득: 자동완성, 접두사 검색
  비용: ngram보다 훨씬 작음 (단어 앞부분만)
  → 자동완성 필드에 별도 적용 권장

동의어 그룹 크기:
  너무 많은 동의어 그룹 → 검색 쿼리 폭발 (쿼리 재작성)
  → 관련성 높은 핵심 동의어만 등록
```

---

## 📌 핵심 정리

```
동의어 처리:
  검색 시점(search-time) 권장
  인덱스 크기 유지 + 재인덱싱 없이 갱신 가능
  synonym_graph + updateable: true + _reload_search_analyzers

Stemmer:
  규칙 기반 어간 추출 → recall 향상, precision 저하 가능
  영어: Porter/Snowball
  한국어: Nori 형태소 분석이 대체
  multi-field로 Stemmer 적용 필드와 미적용 필드 병행

n-gram vs edge_ngram:
  ngram: 부분 검색 (어느 위치든), 비용 큼
  edge_ngram: 자동완성 (접두사), 비용 적음
  → 용도별 별도 필드로 분리 설계

커스텀 Analyzer 설계:
  인덱스 Analyzer와 검색 Analyzer를 명시적으로 분리
  동의어는 검색 Analyzer에만 포함
  자동완성은 별도 서브필드로 격리
```

---

## 🤔 생각해볼 문제

**Q1.** 동의어 파일을 `updateable: true`로 설정하고 `_reload_search_analyzers`로 갱신했을 때 기존 인덱싱된 문서는 새 동의어로 검색되는가?

<details>
<summary>해설 보기</summary>

검색 시점 동의어는 검색 쿼리 분석 시 적용되므로, 기존 인덱싱된 문서 역색인은 변하지 않는다. 예를 들어 "노트북"이 인덱싱된 상태에서 "laptop"을 새 동의어로 추가하면, 이후 "laptop" 검색 시 search_analyzer가 "laptop"을 "노트북"으로 확장하고 역색인에서 "노트북"을 탐색해 기존 문서를 찾는다. 반대로 "노트북"으로 검색하면 "laptop"으로도 확장되어 탐색한다. 기존 문서 재인덱싱 없이 새 동의어가 즉시 작동하는 것이 search-time 동의어의 핵심 이점이다.

</details>

**Q2.** edge_ngram 인덱싱 Analyzer와 검색 Analyzer를 다르게 설정하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

인덱싱 시에는 edge_ngram으로 "elast", "elas", "ela", "el" 같은 조각들을 역색인에 저장한다. 검색 시 "ela"를 입력하면 동일하게 edge_ngram을 적용하면 "el", "ela"가 생성되어 "el"로 시작하는 모든 단어가 함께 반환될 수 있다. 대신 검색 시에는 keyword tokenizer로 "ela" 그대로 하나의 토큰으로 처리하면, 역색인에서 "ela"로 정확히 조각이 저장된 단어만 반환된다. 즉, 인덱스 시 edge_ngram + 검색 시 keyword 조합이 자동완성의 정확도를 보장한다.

</details>

**Q3.** Porter Stemmer에서 "university"와 "universe"가 같은 어간으로 처리되는 경우, 실무에서 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

가장 일반적인 대응은 Stemmer 적용 필드와 미적용 필드를 함께 사용하는 multi-field 방식이다. `title` 필드는 Stemmer 없이 정확 매칭용으로, `title.stemmed` 필드는 Stemmer 적용 recall 향상용으로 분리한다. 검색 시 `multi_match`로 두 필드를 동시에 쿼리하되, Stemmer 없는 필드에 더 높은 가중치(boost)를 부여한다. 이렇게 하면 "university"로 검색 시 정확히 "university"가 포함된 문서가 상위에 오고, "universe" 관련 문서는 더 낮은 점수로 뒤에 위치한다.

</details>

---

<div align="center">

**[⬅️ 이전: Standard·Nori Analyzer](./02-standard-nori-analyzer.md)** | **[홈으로 🏠](../README.md)** | **[다음: 매핑(Mapping) 완전 가이드 ➡️](./04-mapping-guide.md)**

</div>
