# Standard·Nori Analyzer — 영문과 한국어 토큰화 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Standard Analyzer가 Unicode Text Segmentation 기반으로 단어를 분리하는 구체적인 규칙은?
- Nori 형태소 분석기는 "전자제품"을 어떻게 ["전자", "제품"]으로 분리하는가?
- `decompound_mode`의 `none`·`discard`·`mixed` 차이는 무엇인가?
- 한국어에서 Standard Analyzer를 쓰면 어떤 문제가 발생하는가?
- Nori의 품사 필터로 명사만 추출하려면 어떻게 설정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

한국어 서비스에서 가장 흔한 검색 품질 문제의 원인은 Analyzer 선택이다. Standard Analyzer로 한국어를 처리하면 "안녕하세요"가 글자 단위로 쪼개지거나 아예 하나의 토큰이 되어 형태소 기반 검색이 불가능해진다. 반대로 Nori를 제대로 설정하지 않으면 복합어 분해가 되지 않아 "스마트폰"으로 검색했을 때 "스마트" 관련 문서를 찾지 못한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 한국어 인덱스에 Standard Analyzer 사용
  설정:
    "title": { "type": "text" }  // 기본 = standard analyzer

  결과:
    "전자제품 검색엔진" 인덱싱
    Standard Analyzer 토큰: ["전자제품", "검색엔진"]
    (한국어는 공백 기준으로만 분리됨)

    검색: "전자"
    역색인에는 "전자"가 없고 "전자제품"만 있음 → 검색 실패

    검색: "제품"
    역색인에는 "제품"이 없고 "전자제품"만 있음 → 검색 실패

    결론: 한국어 복합어를 Standard Analyzer로 처리하면
          복합어를 분해해서 검색하는 것이 불가능

실수 2: Nori 설치 없이 사용 시도
  오류: Unknown analyzer type [nori]
  이유: Nori는 Elasticsearch에 기본 포함이지만
        일부 환경에서는 analysis-nori 플러그인 설치 필요

  설치:
    bin/elasticsearch-plugin install analysis-nori
    Docker: 이미지에 플러그인 포함 여부 확인

실수 3: decompound_mode를 none으로 설정
  설정:
    "nori_tokenizer": { "decompound_mode": "none" }

  결과:
    "전자제품" → ["전자제품"] (분해 없음)
    "스마트폰" → ["스마트폰"] (분해 없음)

    검색: "스마트" → 결과 없음 (역색인에 "스마트폰"만 있음)
    → 복합어 부분 검색 불가

  올바른 설정:
    decompound_mode: "discard" → 복합어를 분해하고 원형 제거
    decompound_mode: "mixed"   → 복합어와 분해 결과 모두 보존
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
언어별 Analyzer 선택 기준:

  영문 일반:
    standard → 대부분 적합
    english  → 어간 추출 + 영문 불용어 포함, 더 정교한 영문 검색

  한국어:
    nori     → 형태소 분석, 복합어 분해
    설치 필수: analysis-nori 플러그인 (또는 이미 포함된 이미지)

  한영 혼합 텍스트:
    nori + 영문 Token Filter 조합
    또는 copy_to로 별도 필드에 다른 Analyzer 적용

  Nori 권장 설정 (일반 상품/콘텐츠 검색):
    {
      "nori_tokenizer": {
        "type": "nori_tokenizer",
        "decompound_mode": "mixed",
        "discard_punctuation": true
      },
      "filter": ["nori_part_of_speech"]  // 조사, 어미 제거
    }

  decompound_mode 선택:
    discard → 검색 정확도 중심 (복합어 원형 제거)
    mixed   → recall 중심 (복합어 + 분해 모두 검색 가능)
    none    → 분해 없음 (특수 목적)
```

---

## 🔬 내부 동작 원리

### 1. Standard Analyzer — Unicode Text Segmentation

```
Standard Analyzer 내부 구성:
  char_filter: []
  tokenizer:   standard (Unicode Text Segmentation, UAX #29)
  filter:      [lowercase]
  (stop은 기본 비활성화)

Unicode Text Segmentation (UAX #29):
  국제 유니코드 표준에 따른 단어 분리 규칙
  언어에 맞는 "단어 경계(Word Boundary)"를 정의

  분리 규칙 핵심:
  ① 공백, 구두점 → 분리 기준 (구두점은 토큰에서 제거)
  ② 알파벳+숫자 연속 → 하나의 토큰
  ③ 어포스트로피: "it's" → 하나의 토큰 (소유격 처리)
  ④ 하이픈: "well-known" → ["well", "known"] 분리
  ⑤ 이메일, URL → 단일 토큰으로 유지

예시:
  "Dr. Smith's email: smith@example.com, cost: $99.99"

  Tokenize 결과:
    ["Dr", "Smith's", "email", "smith@example.com", "cost", "99.99"]

  lowercase 후:
    ["dr", "smith's", "email", "smith@example.com", "cost", "99.99"]

  특징:
    "Dr." → "Dr" (마침표 제거)
    "Smith's" → "smith's" (어포스트로피 유지, 소문자화)
    "smith@example.com" → 단일 토큰 (이메일 인식)
    "$99.99" → "99.99" (달러 기호 제거, 소수점 유지)

한국어에서 Standard Analyzer의 한계:
  한국어는 UAX #29 기준 "Letter" 카테고리
  → 공백이 없으면 하나의 토큰으로 처리

  "전자제품이좋아요" → ["전자제품이좋아요"] (단 하나의 토큰!)
  "전자 제품 좋아요" → ["전자", "제품", "좋아요"] (공백 있으면 분리)

  문제:
    공백이 있어도 "전자제품"은 형태소 분해 없이 하나의 토큰
    "전자" 검색 → 역색인에 "전자"가 없으므로 실패
    → 한국어는 형태소 분석기 필수
```

### 2. Nori 형태소 분석기 — 한국어 분리 원리

```
Nori(노리) = Lucene Korean Analyzer
  세종 말뭉치 기반 한국어 형태소 사전 내장
  품사 태깅 + 복합어 분해 + 어근 추출

형태소(Morpheme):
  언어에서 의미를 갖는 최소 단위
  "전자제품" = "전자(명사)" + "제품(명사)"
  "달리다"   = "달리(동사 어간)" + "다(어미)"
  "빠른"     = "빠르(형용사 어간)" + "ㄴ(관형사형 어미)"

Nori 처리 예시: "전자제품을 검색합니다"

  형태소 분석:
    전자  [NNG: 일반명사]
    제품  [NNG: 일반명사]
    을    [JX:  보조사]
    검색  [NNG: 일반명사]
    하    [XSV: 동사 파생 접미사]
    ㅂ니다 [EF:  종결 어미]

  조사, 어미 제거 후 토큰:
    ["전자제품", "검색"] 또는 decompound에 따라 다름

decompound_mode 비교:

  none (분해 없음):
    "전자제품" → ["전자제품"]
    복합어 원형만 역색인에 저장
    검색: "전자" → 실패

  discard (분해 후 복합어 제거):
    "전자제품" → ["전자", "제품"]
    복합어 원형 제거, 분해 결과만 역색인
    검색: "전자" → 성공
    검색: "전자제품" → 실패! (역색인에 없음)
    → match 쿼리는 "전자", "제품"으로 분해 후 각각 탐색 → 성공

  mixed (분해 + 복합어 모두 보존):
    "전자제품" → ["전자제품", "전자", "제품"]
    복합어와 분해 결과 모두 역색인
    검색: "전자" → 성공 (분해 결과)
    검색: "전자제품" → 성공 (복합어 원형)
    비용: 역색인 크기 증가 (토큰 수 증가)
```

### 3. Nori 품사 필터 — nori_part_of_speech

```
한국어 품사 태그 (세종 품사 체계):
  NNG: 일반명사   (전자, 제품, 검색)
  NNP: 고유명사   (서울, 삼성)
  VV:  동사       (달리다, 먹다)
  VA:  형용사     (빠르다, 예쁘다)
  JX:  보조사     (은, 는, 이, 가)
  JC:  접속조사   (와, 과)
  EF:  종결어미   (다, 니다, 어요)
  XSN: 명사 파생 접미사 (~적, ~성)

nori_part_of_speech Token Filter:
  특정 품사를 역색인에서 제외
  검색에 의미없는 품사 제거

  기본 제거 품사 (stoptags):
    E  - 어미 전체
    IC - 감탄사
    J  - 조사 전체
    MAG, MAJ - 부사
    MM - 관형사
    SP, SSC, SSO, SC, SE - 특수 기호
    XPN, XSA, XSN, XSV - 접사

설정 예시:
  "filter": {
    "nori_pos_filter": {
      "type": "nori_part_of_speech",
      "stoptags": ["E", "J", "SC", "SP", "SSB", "SSC", "SY", "XPN", "XSA", "XSN", "XSV"]
    }
  }

실제 토큰화 비교:
  입력: "빠른 전자제품을 검색합니다"

  stoptags 없이:
    ["빠르", "ㄴ", "전자", "제품", "을", "검색", "하", "ㅂ니다"]
    (어미, 조사 모두 포함)

  stoptags 적용 후:
    ["빠르", "전자", "제품", "검색"]
    (어미 E, 조사 J 제거)

  nori_readingform (독음 변환):
    한자를 한글 독음으로 변환
    "大韓民國" → "대한민국"
    한자가 포함된 문서 검색 시 유용
```

### 4. 한영 혼합 텍스트 처리

```
문제: "iPhone 15 Pro 배터리 성능"

  Nori만 사용 시:
    한국어 형태소 분석 적용
    영문 "iPhone", "Pro" → 하나의 영문 토큰 (형태소 분석 없음)
    숫자 "15" → 별도 토큰

  처리 결과:
    ["iPhone", "15", "Pro", "배터리", "성능"]
    (영문은 그대로, 한국어만 형태소 분석)

  Nori의 영문 처리:
    영문 단어는 Nori가 그대로 통과
    영문 소문자화는 별도 lowercase Token Filter 필요

  권장 한영 혼합 설정:
    tokenizer: nori_tokenizer (한국어 형태소 분석)
    filter:
      - nori_part_of_speech (한국어 조사/어미 제거)
      - lowercase           (영문 소문자화)
      - (선택) english_possessive_stemmer (영문 소유격)
      - (선택) stop (영문 불용어)

  결과:
    ["iphone", "15", "pro", "배터리", "성능"]
```

### 5. 사용자 사전 — 신조어, 고유명사 처리

```
문제: Nori 기본 사전에 없는 단어
  "갤럭시" → 단어 경계 인식 못할 수 있음
  신조어, 브랜드명, 기술 용어

사용자 사전(User Dictionary):
  커스텀 단어와 분해 규칙 정의
  파일 형식: 단어 [분해결과] [품사]

  user_dictionary.txt:
    갤럭시
    삼성전자 삼성 전자
    아이폰 아이 폰
    엘라스틱서치

  설정:
    "nori_tokenizer": {
      "type": "nori_tokenizer",
      "user_dictionary": "analysis/user_dictionary.txt"
    }

  효과:
    "삼성전자" → ["삼성", "전자"] (사전 정의대로 분해)
    "엘라스틱서치" → ["엘라스틱서치"] (단어 통째로 인식)

사용자 사전 관리:
  파일 변경 후 인덱스 재시작 필요 (동적 갱신 불가)
  → 변경 잦은 사전은 synonym Token Filter로 대체 가능
  → synonym filter는 인덱스 reload_search_analyzers API로 갱신 가능
```

---

## 💻 실전 실험

```bash
# Nori 플러그인 설치 확인
GET _nodes/plugins?pretty
# nori가 목록에 있는지 확인

# Standard Analyzer로 한국어 처리 (한계 확인)
GET _analyze
{
  "analyzer": "standard",
  "text": "전자제품을 검색합니다"
}
# 결과: ["전자제품을", "검색합니다"] (형태소 분해 없음)

# Nori Tokenizer 직접 테스트
GET _analyze
{
  "tokenizer": "nori_tokenizer",
  "text": "전자제품을 검색합니다"
}

# decompound_mode 비교 실험
PUT /nori-test
{
  "settings": {
    "analysis": {
      "analyzer": {
        "nori_none":    { "tokenizer": "nori_none_tok" },
        "nori_discard": { "tokenizer": "nori_discard_tok" },
        "nori_mixed":   { "tokenizer": "nori_mixed_tok" }
      },
      "tokenizer": {
        "nori_none_tok":    { "type": "nori_tokenizer", "decompound_mode": "none" },
        "nori_discard_tok": { "type": "nori_tokenizer", "decompound_mode": "discard" },
        "nori_mixed_tok":   { "type": "nori_tokenizer", "decompound_mode": "mixed" }
      }
    }
  }
}

GET /nori-test/_analyze
{ "analyzer": "nori_none",    "text": "전자제품 스마트폰" }

GET /nori-test/_analyze
{ "analyzer": "nori_discard", "text": "전자제품 스마트폰" }

GET /nori-test/_analyze
{ "analyzer": "nori_mixed",   "text": "전자제품 스마트폰" }

# 품사 필터 적용
GET _analyze
{
  "tokenizer": "nori_tokenizer",
  "filter": [
    {
      "type": "nori_part_of_speech",
      "stoptags": ["E", "J", "SC", "SP", "XSV", "XSA"]
    },
    "lowercase"
  ],
  "text": "빠른 전자제품을 검색합니다 iPhone 15"
}

# 한영 혼합 권장 설정
PUT /korean-search
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["nori_pos_filter", "nori_readingform", "lowercase"]
        }
      },
      "filter": {
        "nori_pos_filter": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "J", "SC", "SP", "SSB", "SSC", "SY", "XPN", "XSA", "XSN", "XSV"]
        }
      }
    }
  }
}

GET /korean-search/_analyze
{
  "analyzer": "korean",
  "text": "빠른 스마트폰 추천 iPhone 15 Pro"
}
```

---

## 📊 Standard vs Nori Analyzer 비교

| 항목 | Standard | Nori |
|------|---------|------|
| 분리 기준 | 공백·구두점 (UAX #29) | 형태소 분석 (세종 말뭉치) |
| 한국어 복합어 | 분해 불가 | decompound_mode로 분해 |
| 조사·어미 제거 | 불가 | nori_part_of_speech로 제거 |
| 영문 처리 | 자연스러움 | 영문은 그대로 통과 |
| 품사 태깅 | 없음 | 있음 (명사만 추출 가능) |
| 플러그인 필요 | 없음 | analysis-nori 필요 |
| 설치·설정 복잡도 | 낮음 | 높음 |

---

## ⚖️ 트레이드오프

```
Nori decompound_mode:
  none    → 인덱스 작음, 복합어 부분 검색 불가
  discard → 인덱스 중간, 부분 검색 가능, 복합어 원형 검색은 match 쿼리 분해로 해결
  mixed   → 인덱스 큼 (토큰 수 많음), 복합어·부분 모두 검색 가능

사용자 사전:
  이득: 브랜드명, 신조어 정확한 분해
  비용: 사전 관리 부담, 변경 시 재시작 필요

품사 필터 범위:
  넓게 (많은 품사 제거) → 역색인 작음, 의미 있는 명사만 검색
  좁게 (적은 품사 제거) → 역색인 큼, 동사·형용사로도 검색 가능
  → 서비스 특성에 따라 조정 (상품 검색: 명사 중심, 문서 검색: 넓게)
```

---

## 📌 핵심 정리

```
Standard Analyzer:
  UAX #29 기준 단어 분리
  공백·구두점이 분리 기준
  한국어: 공백 기준만 분리 → 복합어 분해 불가 → 한국어 서비스 부적합

Nori Analyzer:
  세종 말뭉치 기반 형태소 분석
  복합어 분해: decompound_mode (none/discard/mixed)
  조사·어미 제거: nori_part_of_speech Token Filter
  영문은 그대로 통과 → lowercase 필터 추가 필요

decompound_mode 선택:
  검색 정확도 중심 → discard
  recall 중심 (부분·전체 모두 검색) → mixed
  분해 불필요 → none

한국어 서비스 필수 구성:
  nori_tokenizer (decompound_mode: discard 또는 mixed)
  + nori_part_of_speech (조사, 어미 제거)
  + lowercase (영문 소문자화)
  + (선택) 사용자 사전 (브랜드명, 신조어)
```

---

## 🤔 생각해볼 문제

**Q1.** "삼성전자" 검색 시 decompound_mode가 `discard`이면 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

`discard` 모드에서 "삼성전자"는 인덱싱 시 ["삼성", "전자"]로 분해되고 복합어 원형은 역색인에 저장되지 않는다. 검색어 "삼성전자"도 동일하게 분석되어 ["삼성", "전자"]로 변환된다. match 쿼리는 기본적으로 OR 연산이므로 "삼성" 또는 "전자"를 포함한 모든 문서가 반환된다. 정확히 "삼성전자"라는 회사만 찾으려면 `match_phrase`를 사용하거나, `mixed` 모드로 복합어 원형도 역색인에 포함시켜야 한다.

</details>

**Q2.** 영문 제품명 "MacBook Pro"를 Nori Analyzer로 처리하면 어떤 토큰이 생성되는가?

<details>
<summary>해설 보기</summary>

Nori는 영문 단어를 형태소 분석 없이 그대로 통과시킨다. "MacBook Pro"는 공백 기준으로 분리되어 ["MacBook", "Pro"]가 된다. lowercase 필터를 추가하면 ["macbook", "pro"]가 된다. "MacBook"이 내부적으로 분해되지 않아 "Mac"이나 "Book"으로는 검색할 수 없다. 영문 브랜드명의 부분 검색이 필요하다면 edge_ngram 필터 추가 또는 별도 필드에 ngram Analyzer를 적용하는 방법을 사용한다.

</details>

**Q3.** Nori의 사용자 사전 파일을 수정했을 때 변경 내용을 반영하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

사용자 사전(`user_dictionary` 파일)은 정적 파일이므로 변경 후 ES 노드 재시작이 필요하다. 재시작 없이 사전을 갱신하는 방법은 없다. 빈번하게 변경이 필요한 단어 목록(예: 신조어, 동의어)은 `user_dictionary` 대신 synonym Token Filter의 동의어 파일로 관리하는 것이 좋다. synonym 파일은 `POST /index/_reload_search_analyzers` API로 노드 재시작 없이 갱신할 수 있다. 단, search_analyzer에만 적용되며 인덱싱 Analyzer 변경은 재인덱싱이 필요하다.

</details>

---

<div align="center">

**[⬅️ 이전: 분석 파이프라인](./01-analysis-pipeline.md)** | **[홈으로 🏠](../README.md)** | **[다음: 커스텀 Analyzer 설계 ➡️](./03-custom-analyzer.md)**

</div>
