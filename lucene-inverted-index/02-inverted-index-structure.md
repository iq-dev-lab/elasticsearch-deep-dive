# 역색인 구조 완전 분해 — Term Dictionary·Posting List·Frequency·Position

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Term Dictionary, Posting List, Term Frequency, Position, Offset은 각각 무엇인가?
- Posting List의 문서 ID가 delta-encoding으로 압축 저장되는 원리는?
- Position 정보가 있으면 어떤 검색이 추가로 가능해지는가?
- `index_options` 설정으로 저장 정보를 줄이면 무엇을 잃는가?
- 실제 Lucene 파일(.tim, .doc, .pos, .pay)은 각각 무엇을 저장하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

역색인의 내부 구조를 모르면 매핑 설정이 그냥 "옵션"으로 보인다. `index_options: docs`와 `index_options: positions`의 차이가 phrase 쿼리 가능 여부를 결정한다는 사실, delta-encoding이 Posting List 크기를 얼마나 줄이는지 알면 왜 역색인이 이렇게 빠른지 이해할 수 있다. 매핑 설계와 스토리지 비용, 쿼리 기능 사이의 트레이드오프는 이 구조에서 시작된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: phrase 쿼리가 안 되는 이유를 모름
  설정:
    PUT /myindex
    {
      "mappings": {
        "properties": {
          "title": { "type": "text", "index_options": "docs" }
        }
      }
    }

  시도:
    GET /myindex/_search
    { "query": { "match_phrase": { "title": "mechanical keyboard" } } }

  결과: 검색은 되지만 순서 무관 — "keyboard mechanical"도 반환
  이유: index_options: docs → Position 정보 저장 안 함
       "mechanical"과 "keyboard"가 인접한지(순서) 알 방법 없음

실수 2: 모든 필드에 term_vector 활성화
  설정:
    "title": { "type": "text", "term_vector": "with_positions_offsets" }
    (전체 인덱스에 적용)

  문제:
    Term Vector는 문서마다 역색인 정보를 별도로 저장
    → 인덱스 크기 대폭 증가 (일부 경우 2~3배)
    → 하이라이팅 등 특수 목적 외에는 불필요한 비용

실수 3: Posting List가 단순 배열이라고 가정
  오해: [1, 4, 7, 100, 10000, 10001] 그대로 저장
  실제: delta-encoding으로 압축
        [1, 3, 3, 93, 9900, 1] 로 변환 후 VByte 압축
  → 실제 디스크 크기는 원본 ID 목록보다 훨씬 작음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
index_options 설계 기준:

  docs (가장 작음):
    저장: 문서 ID만
    가능: 해당 단어 포함 여부
    불가: 빈도 기반 스코어링 정확도 저하, phrase 쿼리 불가
    적합: 존재 여부만 확인하는 필드 (예: 태그 필드)

  freqs (기본 text 설정):
    저장: 문서 ID + 단어 빈도(TF)
    가능: BM25 스코어링
    불가: phrase 쿼리, span 쿼리
    적합: 일반 전문 검색

  positions (기본 text 설정의 실제 기본값):
    저장: 문서 ID + TF + 위치(Position)
    가능: match_phrase, span 쿼리
    적합: "단어 순서가 중요한" 검색

  offsets (가장 큼):
    저장: 문서 ID + TF + Position + 문자 오프셋
    가능: 위 모두 + 고속 하이라이팅(Unified Highlighter)
    적합: 하이라이팅이 필요한 검색 결과 표시
```

---

## 🔬 내부 동작 원리

### 1. 역색인의 4개 계층 구조

```
역색인 전체 구조:

┌──────────────────────────────────────────────────────────┐
│                    Term Dictionary                       │
│  (정렬된 단어 목록 — 디스크, FST로 메모리에 압축 적재)              │
│                                                          │
│  "cherry"     → Posting List 위치 포인터                    │
│  "gaming"     → Posting List 위치 포인터                    │
│  "keyboard"   → Posting List 위치 포인터 ────────────┐      │
│  "mechanical" → Posting List 위치 포인터             │      │
│  ...                                              │      │
└───────────────────────────────────────────────────┼──────┘
                                                    │
                                                    ▼
┌──────────────────────────────────────────────────────────┐
│                     Posting List                         │
│              ("keyboard"에 대한 정보)                       │
│                                                          │
│  ┌──────┬──────────────┬──────────────┬───────────────┐  │
│  │doc_id│ Term Freq(TF)│  Positions   │    Offsets    │  │
│  ├──────┼──────────────┼──────────────┼───────────────┤  │
│  │  1   │      1       │    [1]       │  [11-18]      │  │
│  │  4   │      1       │    [1]       │  [7-14]       │  │
│  └──────┴──────────────┴──────────────┴───────────────┘  │
│                                                          │
│  doc_id 1: "Mechanical [Keyboard] Cherry MX"             │
│            position=1 (두 번째 단어), offset=11~18          │
│                                                          │
│  doc_id 4: "Gaming [Keyboard] RGB"                       │
│            position=1 (두 번째 단어), offset=7~14           │
└──────────────────────────────────────────────────────────┘
```

### 2. Delta-Encoding — Posting List 압축

```
문제: 문서 ID가 크거나 많으면 저장 공간이 너무 큼

예시: "the"가 등장하는 문서들 (매우 흔한 단어)
  실제 문서 ID: [1, 4, 7, 100, 10000, 10001, 10003, 50000]

  원본 저장 시:
    각 ID가 4바이트 정수라면 = 8개 × 4바이트 = 32바이트

Delta-Encoding (차이값만 저장):
  첫 번째: 1         (기준값)
  이후는 이전 값과의 차이(delta):
    4 - 1     = 3
    7 - 4     = 3
    100 - 7   = 93
    10000-100 = 9900
    10001-10000 = 1
    10003-10001 = 2
    50000-10003 = 39997

  Delta 시퀀스: [1, 3, 3, 93, 9900, 1, 2, 39997]

  특성: 문서 ID가 연속적으로 증가하면 delta가 작음
        작은 숫자는 VByte(가변 길이 인코딩)로 더 적은 바이트 사용

VByte (Variable Byte Encoding):
  1~127      → 1바이트
  128~16383  → 2바이트
  ...

  delta = 3  → 1바이트  (원본 ID 4바이트 대비 75% 절약)
  delta = 1  → 1바이트
  delta = 93 → 1바이트

  결과: [1, 3, 3, 93, ...] → 대부분 1~2바이트로 표현
        원본 32바이트 → 압축 후 12~16바이트 수준
        실제로 수십 % ~ 수 배 압축

복원:
  읽을 때 누적 합산:
    1 → 1
    1+3=4 → 4
    4+3=7 → 7
    7+93=100 → 100
    100+9900=10000 → 10000
    ...

  → 원본 ID 목록 완전 복원, 추가 CPU 비용은 덧셈만
```

### 3. Term Frequency — BM25 스코어링의 재료

```
Term Frequency(TF): 특정 문서 내 해당 단어 등장 횟수

예시:
  doc_5: "keyboard keyboard mechanical keyboard gaming"
  "keyboard" TF = 3

  doc_6: "keyboard review"
  "keyboard" TF = 1

Posting List ("keyboard"):
  doc_id=5, TF=3
  doc_id=6, TF=1

BM25에서 TF 활용 (Ch4-02 상세):
  TF가 높을수록 점수 기여 증가
  하지만 제곱근 스케일 → 무한정 증가하지 않음 (k1 파라미터)

  IDF(Inverse Document Frequency):
    "keyboard"가 전체 N개 문서 중 M개에 등장
    → IDF = log(N/M + 1)
    → 희귀한 단어일수록 IDF 높음 → 더 중요하다고 판단

BM25 최종 점수:
  score = TF_component × IDF
        ↑ Posting List에서 가져옴
```

### 4. Position과 Phrase 쿼리

```
Position: 해당 단어가 문서 내 몇 번째 토큰인가 (0부터 시작)

doc_1: "Mechanical Keyboard Cherry MX"
  Tokenize: ["mechanical"(0), "keyboard"(1), "cherry"(2), "mx"(3)]

Posting List ("mechanical"):
  doc_id=1, TF=1, positions=[0]

Posting List ("keyboard"):
  doc_id=1, TF=1, positions=[1]

Phrase 쿼리: match_phrase { "mechanical keyboard" }
  필요 조건: "mechanical" 다음(position+1)에 "keyboard"가 있어야 함

  알고리즘:
    "mechanical" → doc_1의 position=0
    "keyboard"   → doc_1의 position=1
    position(keyboard) - position(mechanical) = 1 → 인접! ✅
    → doc_1 반환

  반례: doc_7: "keyboard mechanical review"
    "mechanical" → position=1
    "keyboard"   → position=0
    position(keyboard) - position(mechanical) = -1 → 순서 반대! ❌
    → phrase 쿼리에서 제외

slop 파라미터 (근접 쿼리):
  match_phrase { "mechanical keyboard", slop: 1 }
  → 두 단어 사이에 1개 단어가 끼어 있어도 허용
  → "mechanical gaming keyboard" → slop=1로 매칭
  → position 차이: 2 이내면 허용
```

### 5. Offset과 하이라이팅

```
Offset: 원본 텍스트에서 해당 단어의 시작/끝 문자 위치

doc_1: "Mechanical Keyboard Cherry MX"
         0         10        19     26

Posting List ("keyboard"):
  doc_id=1, TF=1, positions=[1], offsets=[start=11, end=18]
  (실제 문자: "Keyboard", 11번째~18번째 문자)

하이라이팅에서 활용:
  검색어: "keyboard"
  결과: "Mechanical <em>Keyboard</em> Cherry MX"
  → offset 정보 없이는 원본 텍스트에서 위치를 재계산해야 함
  → offset 저장 시: 즉시 <em> 태그 삽입 가능 (빠른 하이라이팅)

Unified Highlighter:
  index_options: offsets → offset 직접 사용 (가장 빠름)
  index_options: positions → position으로 재계산
  index_options: docs → _source 텍스트 재분석 (가장 느림)
```

### 6. Lucene 실제 파일 구조

```
세그먼트별 파일 목록:

  _0.tim   → Term Dictionary (용어 사전)
               정렬된 단어 목록 + Posting List 포인터
               FST로 압축 → 메모리 적재

  _0.tip   → Term Index
               .tim 파일 탐색을 위한 2차 인덱스 (블록 오프셋)

  _0.doc   → Posting List (문서 ID + TF)
               delta-encoding + VByte 압축

  _0.pos   → Position 정보
               index_options >= positions 일 때만 생성

  _0.pay   → Payload + Offset 정보
               index_options = offsets 일 때만 생성

  _0.dvd   → doc_values 데이터 (정렬·집계용)
  _0.dvm   → doc_values 메타데이터

  _0.fdt   → _source 저장 (Field Data / Stored Fields)
  _0.fdx   → _source 인덱스

  _0.si    → Segment Info (세그먼트 메타데이터)
  _0.liv   → Live Documents (삭제 마킹 비트셋)

실제 확인:
  curl http://localhost:9200/myindex/_segments?pretty
  → 각 세그먼트의 크기, 문서 수, 삭제 수 확인
```

---

## 💻 실전 실험

```bash
# Term Vector로 역색인 내부 직접 확인
GET /search-test/_termvectors/1
{
  "fields": ["title"],
  "offsets": true,
  "positions": true,
  "term_statistics": true,
  "field_statistics": true
}
# 응답에서:
#   term_freq: 해당 문서의 단어 빈도
#   doc_freq: 전체 문서 중 이 단어가 등장하는 문서 수
#   positions: 단어 위치
#   start_offset, end_offset: 문자 오프셋

# index_options 비교 실험
PUT /test-positions
{
  "mappings": {
    "properties": {
      "title_docs":      { "type": "text", "index_options": "docs" },
      "title_freqs":     { "type": "text", "index_options": "freqs" },
      "title_positions": { "type": "text", "index_options": "positions" },
      "title_offsets":   { "type": "text", "index_options": "offsets" }
    }
  }
}

POST /test-positions/_doc/1
{
  "title_docs":      "mechanical keyboard review",
  "title_freqs":     "mechanical keyboard review",
  "title_positions": "mechanical keyboard review",
  "title_offsets":   "mechanical keyboard review"
}

# phrase 쿼리: positions 이상만 정확히 동작
GET /test-positions/_search
{
  "query": {
    "match_phrase": { "title_positions": "mechanical keyboard" }
  }
}

# docs 필드로 phrase 쿼리 (순서 무시됨)
GET /test-positions/_search
{
  "query": {
    "match_phrase": { "title_docs": "mechanical keyboard" }
  }
}

# 세그먼트 파일 정보 확인
GET /search-test/_segments?pretty
# _0.tim, _0.doc 등 파일 크기 확인

# 인덱스 스토리지 통계
GET /search-test/_stats/store?pretty
```

---

## 📊 index_options 저장 정보와 기능 비교

| index_options | 저장 정보 | 가능한 쿼리 | 스토리지 비용 |
|--------------|---------|-----------|------------|
| `docs` | 문서 ID만 | 단어 존재 여부 | 가장 작음 |
| `freqs` | 문서 ID + TF | BM25 스코어링 | 작음 |
| `positions` | + Position | phrase, span 쿼리 | 중간 |
| `offsets` | + Offset | 고속 하이라이팅 | 가장 큼 |

**text 필드 기본값**: `positions`

---

## ⚖️ 트레이드오프

```
저장 정보 많을수록:
  이득: 더 정교한 쿼리 (phrase, span, 고속 하이라이팅)
  비용: 인덱스 크기 증가, 인덱싱 속도 감소

delta-encoding의 가정:
  문서 ID가 순차적으로 증가한다는 가정 (실제로 그러함)
  → delta가 작을수록 압축률 높음
  → 문서 ID 순서가 임의적이면 압축 효율 감소

TF 저장의 트레이드오프:
  TF 없음 (docs) → 단어 존재/부재만 판단, BM25 점수 부정확
  TF 있음 (freqs) → 정확한 BM25 가능, 저장 비용 증가
  → 검색 품질 vs 스토리지 비용

Position 저장의 트레이드오프:
  Position 없음 → phrase 쿼리 불가
  Position 있음 → phrase, slop, span 쿼리 가능
  → 단순 키워드 검색만 하는 필드는 docs/freqs로 줄일 수 있음
```

---

## 📌 핵심 정리

```
역색인 4계층:
  Term Dictionary → 정렬된 단어 목록, 포인터 제공
  Posting List   → 단어별 문서 ID 목록 (delta + VByte 압축)
  Term Frequency → 문서 내 단어 빈도 (BM25 재료)
  Position       → 단어 위치 (phrase 쿼리 가능)
  Offset         → 문자 오프셋 (고속 하이라이팅)

Delta-Encoding 핵심:
  [1, 4, 7, 100] → [1, 3, 3, 93] 차이값 저장
  VByte로 추가 압축 → 작은 숫자 = 적은 바이트
  대용량 Posting List를 수십 % 압축

index_options 선택 기준:
  단순 필터링     → docs
  일반 전문 검색  → freqs (또는 positions)
  phrase 쿼리    → positions 필수
  하이라이팅     → offsets

Lucene 파일:
  .tim(Term Dictionary), .doc(Posting), .pos(Position), .pay(Offset)
```

---

## 🤔 생각해볼 문제

**Q1.** delta-encoding이 역색인에 효과적인 이유는 Lucene이 문서 ID를 어떻게 관리하기 때문인가?

<details>
<summary>해설 보기</summary>

Lucene은 세그먼트 내에서 문서 ID를 0부터 순차 증가하는 정수로 관리한다. 새 문서는 항상 더 큰 ID를 받고, Posting List는 오름차순으로 정렬되어 있다. 따라서 인접한 ID 간의 차이(delta)는 대부분 1이나 작은 수다. 특히 특정 단어가 연속된 문서에 자주 등장하면(예: 일반적인 단어 "the") delta가 매우 작아 VByte로 1바이트씩만 차지한다. 이것이 delta-encoding이 역색인에 특히 잘 맞는 이유다.

</details>

**Q2.** `match_phrase` 쿼리에서 slop을 높이면 성능에 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

slop 값이 높아지면 Position 비교 시 허용 범위가 넓어져 더 많은 후보 문서를 검토해야 한다. 내부적으로는 두 단어의 Position 차이가 slop 이내인지 확인하는 비용이 증가한다. 또한 slop이 높을수록 더 많은 문서가 매칭되어 Posting List 교집합 연산 비용도 증가한다. 반면 정확한 phrase 쿼리(slop=0)는 Position 비교가 엄격해서 후보를 빠르게 걸러낼 수 있다.

</details>

**Q3.** 같은 인덱스에서 text 필드에 `term_vector: with_positions_offsets`를 추가로 설정하면 `.pos`와 어떻게 다른가?

<details>
<summary>해설 보기</summary>

`.pos` 파일(index_options=positions)은 세그먼트 레벨 역색인에 Position 정보를 포함하는 것으로, 모든 문서에 대한 전역 역색인에 저장된다. `term_vector`는 문서별 개별 역색인으로, 각 문서마다 해당 문서의 단어·빈도·위치 정보를 별도로 저장하는 구조다. term_vector는 특정 문서의 역색인 정보를 빠르게 꺼낼 수 있어 `_termvectors` API나 "더보기 추천"(`more_like_this` 쿼리)에 활용되지만, 저장 공간을 많이 차지한다. 일반 검색을 위한 phrase 쿼리는 `.pos` 파일만으로 충분하며 term_vector는 불필요하다.

</details>

---

<div align="center">

**[⬅️ 이전: B-Tree vs 역색인](./01-btree-vs-inverted-index.md)** | **[홈으로 🏠](../README.md)** | **[다음: FST — Term Dictionary 압축 저장 ➡️](./03-fst-term-dictionary.md)**

</div>
