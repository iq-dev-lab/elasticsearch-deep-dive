# BM25 스코어링 완전 분해 — TF·IDF·필드 길이 정규화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- BM25에서 TF(단어 빈도)가 선형이 아닌 포화 곡선으로 증가하는 이유는?
- IDF(역문서 빈도)가 희귀한 단어에 높은 점수를 부여하는 수식의 의미는?
- 필드 길이 정규화(`b` 파라미터)가 왜 "긴 문서 불이익"이 되는가?
- `k1`과 `b` 파라미터를 조정하면 점수에 어떤 영향이 있는가?
- `_explain` API로 실제 BM25 계산 결과를 어떻게 확인하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"왜 이 문서가 저 문서보다 점수가 높은가"를 설명할 수 없으면 검색 품질 개선이 불가능하다. BM25 수식을 이해하면 특정 단어가 점수에 미치는 영향을 예측하고, 파라미터를 조정해 서비스 특성에 맞는 점수 체계를 구성할 수 있다. `_explain` API가 분석 도구가 아닌 블랙박스로만 보이는 이유가 BM25를 모르기 때문이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: "단어가 많을수록 점수가 무한히 높아진다"고 오해
  오해:
    "keyboard"가 100번 나오는 문서가 5번 나오는 문서보다
    20배 높은 점수를 받는다

  실제 BM25 TF 컴포넌트:
    TF_score = (tf × (k1 + 1)) / (tf + k1 × (1 - b + b × fieldLen/avgFieldLen))

    tf=5:  5 × 2.2 / (5 + 1.2 × ...) ≈ 일정값
    tf=100: 포화(saturation) → 5번과 큰 차이 없음

    k1 기본값 1.2:
    → tf가 높아져도 점수 증가가 급격히 줄어듦
    → "이 단어를 스팸처럼 반복하면 점수가 무한히 오른다" → X

실수 2: IDF를 고려하지 않고 단어 선택
  설계:
    검색: "the keyboard" → "the"와 "keyboard"에 동일 가중치 기대

  실제:
    "the": 거의 모든 문서에 등장 → IDF ≈ 0 (거의 0점)
    "keyboard": 일부 문서만 → IDF 높음 (높은 점수)
    → 실제로 "the"는 점수에 거의 기여 안 함

실수 3: 긴 설명 필드에 b=0 설정 (필드 길이 정규화 비활성화)
  설정:
    "similarity": { "default": { "type": "BM25", "b": 0 } }

  의도: "긴 문서가 불이익받지 않게"
  부작용:
    짧은 제목 문서 vs 긴 설명 문서
    b=0이면 긴 문서의 TF 조정 없음
    → 단순히 단어가 많이 반복된 긴 문서가 유리
    → 짧고 정확한 문서가 오히려 불이익
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
BM25 파라미터 튜닝 기준:

  k1 (TF 포화 조절, 기본값 1.2):
    낮은 k1 (예: 0.5):
      TF 포화가 빠름 → 단어 반복이 점수에 거의 영향 없음
      → 단순 키워드 필터링에 가까움 (단어 존재 여부만 중요)

    높은 k1 (예: 2.0):
      TF 포화가 느림 → 단어 반복이 점수에 더 많은 영향
      → 특정 단어를 자주 언급하는 문서를 더 높이 평가
      → 기술 문서, 전문 용어 검색에 적합

  b (필드 길이 정규화, 기본값 0.75):
    b=0: 필드 길이 무시 → 긴 문서가 단어 반복으로 유리
    b=1: 완전 정규화 → 필드 길이에 완전히 비례해 TF 조정
    b=0.75: 중간값 (ES 기본)

    짧은 필드 (title, keyword):
      b를 낮게 (0.0~0.3) → 길이 차이가 거의 없으므로 정규화 불필요

    긴 필드 (body, description):
      b를 높게 (0.75~1.0) → 긴 문서의 TF 과대평가 방지

  서비스별 권장:
    뉴스 기사 검색: k1=1.2, b=0.75 (기본값, 대부분 적합)
    상품 제목 검색: k1=1.5, b=0.3 (제목은 짧음, 반복 강조)
    코드 검색:      k1=0.5, b=0.0 (존재 여부 중심)
```

---

## 🔬 내부 동작 원리

### 1. BM25 수식 완전 분해

```
BM25 최종 점수 수식:

  score(d, q) = Σ IDF(t) × TF_norm(t, d)

  각 검색 토큰 t에 대해 IDF와 TF_norm을 곱하고 합산

─────────────────────────────────────────────────────

IDF (Inverse Document Frequency):

  IDF(t) = ln(1 + (N - n(t) + 0.5) / (n(t) + 0.5))

  N    = 전체 문서 수
  n(t) = 단어 t가 등장하는 문서 수

  해석:
    n(t) 작음 (희귀 단어):
      분자 커짐, 분모 작음 → ln 값 커짐 → IDF 높음 → 점수 기여 큼
    n(t) 큼 (흔한 단어):
      분자 작아짐, 분모 커짐 → ln 값 작아짐 → IDF 낮음 → 점수 기여 작음
    n(t) = N (모든 문서에 등장):
      ln(1 + 0.5/N+0.5) ≈ ln(1) = 0 → IDF ≈ 0 → 점수 기여 없음

  예시 (N=1000):
    "elasticsearch": n=50  → IDF = ln(1 + 950.5/50.5) ≈ 2.95
    "the":           n=990 → IDF = ln(1 + 10.5/990.5) ≈ 0.01
    "aardvark":      n=2   → IDF = ln(1 + 998.5/2.5)  ≈ 5.99

  → "aardvark" 검색 = "elasticsearch" 검색보다 희귀 → IDF 2배

─────────────────────────────────────────────────────

TF_norm (Term Frequency with Normalization):

  TF_norm(t, d) = (tf × (k1 + 1)) / (tf + k1 × (1 - b + b × |d|/avgdl))

  tf     = 문서 d에서 단어 t의 등장 횟수
  k1     = TF 포화 파라미터 (기본 1.2)
  b      = 필드 길이 정규화 파라미터 (기본 0.75)
  |d|    = 문서 d의 필드 토큰 수
  avgdl  = 인덱스 내 평균 필드 토큰 수

  해석:
    분자: tf × (k1 + 1) = tf × 2.2 (k1=1.2)
    분모: tf + k1 × (1 - b + b × |d|/avgdl)

    tf 증가 → 분자·분모 모두 증가 → 비율이 k1+1에 수렴(포화)
    → tf가 아무리 높아도 (k1+1) = 2.2를 초과 불가

  포화 비교:
    tf=1:  (1 × 2.2) / (1 + 1.2 × norm) = ...
    tf=5:  (5 × 2.2) / (5 + 1.2 × norm) ≈ 1.8 (큰 증가)
    tf=10: (10 × 2.2) / (10 + 1.2 × norm) ≈ 1.95 (작은 증가)
    tf=50: ≈ 2.1 (거의 포화)
    → tf 증가에 따른 점수 증가율이 점점 감소

─────────────────────────────────────────────────────

필드 길이 정규화 (b 파라미터):

  |d|/avgdl 비율:
    짧은 문서: |d| < avgdl → 비율 < 1 → TF_norm 상승
    긴 문서:   |d| > avgdl → 비율 > 1 → TF_norm 하강

  직관:
    "keyboard"가 짧은 제목(5단어)에서 1번 등장
    vs 긴 설명(100단어)에서 1번 등장
    → 짧은 문서의 1번은 "밀도가 높음" → 더 관련있을 가능성
    → b=0.75: 긴 문서의 tf를 약간 하향 조정

  b=0일 때: 1 - 0 + 0 = 1 → |d|/avgdl 항 사라짐 → 필드 길이 무시
  b=1일 때: 완전 정규화 → |d|/avgdl에 완전 비례 조정
```

### 2. 실제 계산 예시

```
인덱스 설정:
  전체 문서: 1000개
  avgdl (title 필드 평균 단어 수): 5
  "keyboard" 등장 문서 수: 50

문서 1: title = "Mechanical Keyboard" (2단어, tf=1)
문서 2: title = "Best Keyboard for Keyboard Lovers who love Keyboard" (9단어, tf=3)

IDF("keyboard"):
  = ln(1 + (1000 - 50 + 0.5) / (50 + 0.5))
  = ln(1 + 950.5 / 50.5)
  = ln(1 + 18.82)
  = ln(19.82)
  ≈ 2.99

TF_norm(문서1, "keyboard"):
  tf=1, |d|=2, avgdl=5, k1=1.2, b=0.75
  = (1 × 2.2) / (1 + 1.2 × (1 - 0.75 + 0.75 × 2/5))
  = 2.2 / (1 + 1.2 × (0.25 + 0.30))
  = 2.2 / (1 + 1.2 × 0.55)
  = 2.2 / (1 + 0.66)
  = 2.2 / 1.66
  ≈ 1.325

TF_norm(문서2, "keyboard"):
  tf=3, |d|=9, avgdl=5, k1=1.2, b=0.75
  = (3 × 2.2) / (3 + 1.2 × (0.25 + 0.75 × 9/5))
  = 6.6 / (3 + 1.2 × (0.25 + 1.35))
  = 6.6 / (3 + 1.2 × 1.60)
  = 6.6 / (3 + 1.92)
  = 6.6 / 4.92
  ≈ 1.341

score(문서1) = IDF × TF_norm1 = 2.99 × 1.325 ≈ 3.96
score(문서2) = IDF × TF_norm2 = 2.99 × 1.341 ≈ 4.01

결론:
  tf=3인 문서2가 tf=1인 문서1보다 조금 높지만
  문서2가 9단어로 긴 탓에 TF_norm이 거의 같음
  → BM25가 단순 tf 반복보다 "단어 밀도"를 고려한다는 의미
```

### 3. 유사도 함수 설정 변경

```
인덱스 레벨 BM25 파라미터 변경:

  PUT /products
  {
    "settings": {
      "similarity": {
        "my_bm25": {
          "type": "BM25",
          "k1": 1.5,
          "b": 0.3
        }
      }
    },
    "mappings": {
      "properties": {
        "title": {
          "type": "text",
          "similarity": "my_bm25"
        }
      }
    }
  }

필드별 다른 similarity 설정:
  title(짧음):       b=0.3 (길이 정규화 약하게)
  description(김):   b=0.75 (길이 정규화 강하게)

  PUT /products
  {
    "settings": {
      "similarity": {
        "title_sim": { "type": "BM25", "k1": 1.2, "b": 0.3 },
        "desc_sim":  { "type": "BM25", "k1": 1.2, "b": 0.75 }
      }
    },
    "mappings": {
      "properties": {
        "title":       { "type": "text", "similarity": "title_sim" },
        "description": { "type": "text", "similarity": "desc_sim" }
      }
    }
  }

다른 유사도 함수:
  boolean:  tf/idf 없이 단순 매칭 여부만 → 1.0 고정
  classic:  TF-IDF (BM25 이전 방식)
  scripted: 완전 커스텀 수식 (Painless 스크립트)
```

---

## 💻 실전 실험

```bash
# BM25 점수 계산 과정 _explain으로 확인
GET /query-test/_explain/1
{
  "query": { "match": { "title": "keyboard" } }
}

# 응답 구조:
# {
#   "explanation": {
#     "value": 3.9xx,               ← 최종 점수
#     "description": "weight(title:keyboard in 0)",
#     "details": [
#       {
#         "value": 2.9xx,           ← IDF
#         "description": "idf, computed as..."
#       },
#       {
#         "value": 1.3xx,           ← TF_norm
#         "description": "tf, computed as..."
#       }
#     ]
#   }
# }

# IDF 비교: 흔한 단어 vs 희귀 단어
POST /query-test/_bulk
{ "index": { "_id": "10" } }
{ "title": "unique xyzabc term document" }
POST /query-test/_refresh

GET /query-test/_explain/1
{ "query": { "match": { "title": "keyboard" } } }
# keyboard: 여러 문서에 있음 → IDF 낮음

GET /query-test/_explain/10
{ "query": { "match": { "title": "xyzabc" } } }
# xyzabc: 1개 문서만 → IDF 높음

# BM25 파라미터 커스텀 인덱스
PUT /bm25-custom
{
  "settings": {
    "number_of_shards": 1,
    "similarity": {
      "custom_bm25": { "type": "BM25", "k1": 0.5, "b": 0.0 }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "similarity": "custom_bm25"
      }
    }
  }
}

POST /bm25-custom/_bulk
{ "index": { "_id": "1" } }
{ "title": "keyboard keyboard keyboard keyboard keyboard" }
{ "index": { "_id": "2" } }
{ "title": "keyboard review" }
POST /bm25-custom/_refresh

# k1=0.5: tf 포화 빠름 → 반복이 점수에 별 영향 없음
GET /bm25-custom/_search
{ "query": { "match": { "title": "keyboard" } } }
GET /bm25-custom/_explain/1 { "query": { "match": { "title": "keyboard" } } }
GET /bm25-custom/_explain/2 { "query": { "match": { "title": "keyword" } } }
# 두 문서 점수 차이가 작음 (tf 포화 때문)

# field_value_factor로 외부 신호 결합
GET /query-test/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "keyboard" } },
      "field_value_factor": {
        "field": "price",
        "factor": 0.000001,
        "modifier": "log1p"
      },
      "boost_mode": "sum"
    }
  }
}
# BM25 점수 + 가격 기반 점수 합산
```

---

## 📊 BM25 파라미터 효과 비교

| 파라미터 | 낮은 값 | 기본값 | 높은 값 |
|---------|--------|--------|--------|
| `k1` | TF 포화 빠름 (존재 여부 중심) | 1.2 | TF 반복이 점수에 더 영향 |
| `b` | 필드 길이 무시 | 0.75 | 완전 길이 정규화 |
| `b=0` 효과 | 긴 문서 유리 가능 | — | — |
| `b=1` 효과 | — | — | 짧은 문서 유리 |

---

## ⚖️ 트레이드오프

```
높은 k1:
  이득: 단어 반복이 중요한 도메인 (기술 문서, 특허)에 적합
  비용: 스팸성 반복 단어가 점수를 부당하게 높일 수 있음

높은 b:
  이득: 긴 문서의 단어 반복 과대평가 방지
  비용: 긴 문서 전체가 불이익 (짧은 글에 유리한 검색)

낮은 b:
  이득: 문서 길이 무관하게 단어 빈도로만 판단
  비용: 단어를 무작정 반복한 긴 문서가 유리해질 수 있음

BM25 외 유사도 함수:
  boolean: 점수 계산 없음 → 랭킹 불필요할 때 (필터 전용)
  scripted: 완전 커스텀 → 유연하지만 복잡하고 느림
  → 대부분 BM25 기본값이 최선의 출발점
```

---

## 📌 핵심 정리

```
BM25 = IDF × TF_norm의 합산

IDF (역문서 빈도):
  희귀 단어 → IDF 높음 → 점수 기여 큼
  흔한 단어 → IDF 낮음 → 점수 기여 작음
  수식: ln(1 + (N - n + 0.5) / (n + 0.5))

TF_norm (정규화된 단어 빈도):
  tf 증가 → 점수 증가 but 포화 (k1+1에 수렴)
  긴 문서 → avgdl 초과 → TF_norm 하향 조정 (b 파라미터)
  k1=1.2, b=0.75 기본값 대부분 적합

파라미터 튜닝:
  k1: TF 포화 속도 조절 (단어 반복 중요도)
  b:  필드 길이 정규화 강도 (짧은 필드: 낮게, 긴 필드: 높게)

_explain API:
  실제 IDF, TF_norm 수치 확인 가능
  검색 품질 문제의 원인 분석에 필수
```

---

## 🤔 생각해볼 문제

**Q1.** 동일 문서에 "keyboard"가 1번 등장할 때와 100번 등장할 때 BM25 점수 비율은 대략 얼마인가?

<details>
<summary>해설 보기</summary>

BM25의 TF 포화 특성으로 인해 100배 차이가 나는 tf가 점수에서는 훨씬 작은 차이를 만든다. k1=1.2, b=0.75, 평균 문서 길이 등 기본값에서 tf=1 대비 tf=100의 TF_norm 비율은 약 1.5~2배 수준이다. 선형 TF라면 100배 차이가 나야 하지만 BM25는 포화 곡선으로 이를 제한한다. `_explain` API로 직접 두 문서를 비교하면 실제 값을 확인할 수 있다.

</details>

**Q2.** 모든 문서에 "the"가 등장하면 BM25에서 "the"의 IDF는 얼마가 되는가?

<details>
<summary>해설 보기</summary>

N=n(the)이면 IDF = ln(1 + (N - N + 0.5) / (N + 0.5)) = ln(1 + 0.5/N+0.5)이다. N이 충분히 크면 이 값은 ln(1 + 0) ≈ 0에 가까워진다. 실질적으로 0에 가까운 IDF는 "the"가 점수에 거의 기여하지 않는다는 의미다. 이것이 불용어(stopwords)를 역색인에서 제거하지 않아도 점수 계산상 자연스럽게 무의미해지는 이유다. BM25의 IDF 항이 불용어 제거 효과를 수학적으로 내포한다.

</details>

**Q3.** 상품 제목 검색 서비스에서 `b` 파라미터를 0으로 설정하는 것이 적절한가?

<details>
<summary>해설 보기</summary>

상품 제목은 대부분 짧고(3~10단어) 평균 길이가 비슷하므로 `b=0`도 합리적인 선택이다. 필드 길이 차이가 거의 없으면 정규화의 효과도 작고, `b=0`으로 설정하면 계산이 단순해진다. 단, "Wireless Gaming Mechanical RGB Keyboard with Programmable Keys for PC Mac"처럼 긴 제목도 있다면 b를 완전히 0으로 하기보다 0.3~0.5 정도의 약한 정규화가 더 적절하다. 실제로는 실험적으로 검색 품질을 측정해가며 조정하는 것이 최선이다.

</details>

---

<div align="center">

**[⬅️ 이전: Query DSL 내부 구조](./01-query-dsl-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분산 스코어링 문제 ➡️](./03-distributed-scoring-problem.md)**

</div>
