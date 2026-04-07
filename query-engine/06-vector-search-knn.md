# 벡터 검색(kNN) — Dense Vector·HNSW·하이브리드 검색

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Dense Vector 필드는 디스크에 어떻게 저장되고, 역색인과 어떻게 다른가?
- HNSW(Hierarchical Navigable Small World) 그래프가 근사 최근접 이웃을 탐색하는 원리는?
- 완전 탐색(Exact kNN)과 근사 탐색(HNSW ANN)의 트레이드오프는?
- BM25 텍스트 검색과 kNN 벡터 검색을 결합하는 하이브리드 검색은 어떻게 동작하는가?
- `num_candidates`와 `k` 파라미터는 각각 무엇을 조정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

LLM 기반 서비스가 확산되면서 벡터 검색은 선택이 아닌 필수가 됐다. "Elasticsearch 배터리 소모"로 검색했을 때 "iPhone 배터리 설정"을 찾아주는 의미론적 검색(Semantic Search)은 BM25만으로는 불가능하다. 반면 벡터 검색만 사용하면 "정확한 키워드"가 있는 문서를 찾는 데 약하다. HNSW 구조와 하이브리드 검색의 원리를 이해하면 두 방식을 결합해 최고의 검색 품질을 낼 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 완전 탐색(exact_search)으로 대규모 벡터 검색
  설정:
    "knn": {
      "field": "embedding",
      "query_vector": [0.1, 0.2, ...],
      "k": 10,
      "num_candidates": 1000000  // 전체 문서 수
    }

  문제:
    num_candidates가 클수록 정확도 증가 but 비용 폭발
    100만 문서 × 1536차원 벡터 = 약 6GB
    각 쿼리마다 100만 개 벡터와 코사인 유사도 계산
    → 쿼리 당 수 초 소요 → 실시간 서비스 불가

  올바른 접근:
    num_candidates: 100~200 (HNSW 근사 탐색)
    → 0.01초 내 응답, 정확도 95%+ 달성
    완전 탐색은 오프라인 배치, 작은 데이터셋에만

실수 2: 벡터와 텍스트 점수를 단순 합산
  시도:
    BM25 점수: 3.5 (0~무한대 범위)
    코사인 유사도: 0.87 (0~1 범위)

    단순 합산: 3.5 + 0.87 = 4.37
    → 점수 스케일이 달라 BM25가 벡터 점수를 압도
    → 사실상 BM25만 사용하는 것과 다름없음

  올바른 접근:
    Reciprocal Rank Fusion (RRF) 활용
    → 순위 기반 결합 (점수 스케일 무관)
    → ES 8.8+ 기본 제공: knn + query의 rrf 결합

실수 3: embedding 모델 변경 후 재인덱싱 없이 검색
  상황:
    text-embedding-ada-002 (1536차원)으로 인덱싱
    새 모델 text-embedding-3-large (3072차원)로 검색

  결과: 차원 불일치 오류 또는 의미없는 유사도
  이유:
    인덱싱 시점의 벡터와 검색 쿼리 벡터가 다른 공간
    → 코사인 유사도가 의미없음
  해결: 모델 변경 시 전체 재인덱싱 필수
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
벡터 검색 설계 원칙:

  임베딩 모델 선택:
    다국어 지원 필요: multilingual-e5, paraphrase-multilingual
    한국어 특화: KoSimCSE, KoBERT 기반 모델
    영문 고성능: text-embedding-ada-002, e5-large

  차원 설정:
    낮은 차원 (256~512): 빠른 검색, 낮은 정확도
    높은 차원 (1536~3072): 느린 검색, 높은 정확도
    → 서비스 요구사항에 맞게 선택, 대부분 768차원이 균형점

  HNSW 파라미터:
    m (연결 수): 기본 16, 높을수록 정확도↑ 메모리↑
    ef_construction: 기본 100, 높을수록 정확도↑ 인덱싱 속도↓
    num_candidates: 기본 k×5 권장, 높을수록 정확도↑ 검색 속도↓

  하이브리드 검색 (권장):
    BM25 (키워드) + kNN (의미) 결합
    → 두 방식의 강점 모두 활용
    → RRF로 점수 스케일 문제 해결
```

---

## 🔬 내부 동작 원리

### 1. Dense Vector 필드 저장 구조

```
dense_vector 매핑:
  "embedding": {
    "type": "dense_vector",
    "dims": 768,           // 벡터 차원수
    "index": true,         // HNSW 그래프 구축 (kNN 검색용)
    "similarity": "cosine" // 유사도 함수: cosine, dot_product, l2_norm
  }

저장 구조 (역색인과 비교):
  역색인: 단어 → 문서 ID 목록
  dense_vector: 문서 ID → 768차원 부동소수점 배열

  물리적 저장:
    .vec 파일: 벡터 데이터 (float32 × dims × 문서 수)
    .vex 파일: HNSW 그래프 구조 (인덱싱 시 구축)
    doc_values와 유사한 컬럼 지향 저장

  메모리 사용:
    문서 1개 × 768차원 × 4바이트(float32) = 3KB
    100만 문서 × 768차원 = 3GB
    → 대용량 벡터 인덱스는 메모리(Page Cache) 확보 필수

유사도 함수:
  cosine:      방향 유사도 (크기 무관, 정규화 필요)
               코사인 유사도 = (A·B) / (|A| × |B|)
  dot_product: 내적 (크기 포함, L2 정규화 벡터에서 cosine과 동일)
  l2_norm:     유클리드 거리 (절대 거리 기반, 값 낮을수록 유사)

index: false 설정:
  HNSW 그래프 미구축 → kNN 검색 불가
  script_score로 완전 탐색만 가능 (소규모에서 사용)
```

### 2. HNSW — 근사 최근접 이웃 알고리즘

```
HNSW (Hierarchical Navigable Small World):
  그래프 기반 근사 최근접 이웃(ANN) 알고리즘
  "Small World" = 몇 홉(hop)만으로 임의 두 노드 연결 가능

구조:
  ┌─────────────────────────────────────────────────────┐
  │  Layer 2 (최상위, 적은 노드):                           │
  │    [doc_A] ──── [doc_F]                             │
  │                                                     │
  │  Layer 1 (중간):                                     │
  │    [doc_A] ── [doc_C] ── [doc_F]                    │
  │       └────── [doc_D]                               │
  │                                                     │
  │  Layer 0 (최하위, 모든 노드):                           │
  │    [doc_A] ── [doc_B] ── [doc_C] ── [doc_D]         │
  │       └── [doc_E] ── [doc_F] ── [doc_G]             │
  └─────────────────────────────────────────────────────┘

  각 노드: 벡터를 가진 문서
  각 엣지: 유사한 벡터끼리 연결 (m개 이웃 연결)
  층(Layer): 위로 갈수록 노드 수 감소, 빠른 탐색용

탐색 알고리즘:
  ① 최상위 레이어에서 임의 진입 노드 선택
  ② 쿼리 벡터와 가장 가까운 방향으로 이동 (탐욕 탐색)
  ③ 개선이 없으면 한 층 아래로 이동
  ④ 최하위 레이어(0)에서 num_candidates 후보 수집
  ⑤ 수집된 후보 중 상위 k개 반환

  예시 (쿼리: "keyboard review" 임베딩):
    Layer 2: doc_F에서 시작 → doc_A가 더 유사 → 이동
    Layer 1: doc_A → doc_C → doc_D (더 유사한 방향)
    Layer 0: 주변 탐색 → 10개 후보 수집
    → 상위 k=5 반환

탐욕 탐색의 "근사" 이유:
  항상 현재 위치에서 가장 유사한 방향으로만 이동
  → 전역 최적이 아닌 지역 최적 탐색
  → 정확도 약간 감소 but 속도 O(log N)

완전 탐색 vs HNSW:
  완전:  O(N × 차원) → 100만 문서 = 10억 연산
  HNSW:  O(log N × 연결 수) → 100만 문서 = ~1000 연산
  → HNSW가 수천~수만 배 빠름, 정확도는 95%+ 달성
```

### 3. HNSW 파라미터 이해

```
m (이웃 연결 수, 기본 16):
  각 노드가 연결하는 이웃 수
  m↑ → 더 많은 연결 → 탐색 경로 다양 → 정확도↑
  m↑ → 그래프 크기 증가 → 메모리 사용 증가
  m↑ → 인덱싱 시간 증가
  권장: 4~64 범위, 대부분 16이 균형점

ef_construction (인덱싱 시 후보 수, 기본 100):
  인덱싱 시 각 노드를 삽입할 때 탐색하는 후보 수
  ef_construction↑ → 더 정교한 그래프 구축 → 정확도↑
  ef_construction↑ → 인덱싱 시간 증가
  ef_construction은 검색 속도에 영향 없음 (인덱싱 시에만)
  권장: 100~200

num_candidates (검색 시 후보 수):
  HNSW 탐색 시 수집하는 후보 문서 수
  num_candidates > k 필수 (k개 최종 반환을 위한 후보)
  num_candidates↑ → 정확도↑ but 검색 속도↓
  권장: k의 5~10배 (k=10이면 num_candidates=50~100)

  Recall vs Speed 트레이드오프:
    num_candidates=50:  빠름, recall ≈ 90%
    num_candidates=100: 균형, recall ≈ 95%
    num_candidates=500: 느림, recall ≈ 99%
```

### 4. 하이브리드 검색 — BM25 + kNN

```
왜 하이브리드 검색인가:

  BM25만 사용:
    ✅ 정확한 키워드 매칭 (제품 코드, 고유명사)
    ❌ 의미론적 유사성 없음 ("배터리 소모" ≠ "배터리 설정")

  kNN만 사용:
    ✅ 의미론적 유사성 (동의어, 맥락 이해)
    ❌ 정확한 키워드 무시 ("iPhone 15 Pro" → 다른 스마트폰도 반환)

  하이브리드: 두 방식의 강점 결합
    "iPhone 15 배터리 설정" 검색
    BM25: "iPhone 15" 정확 키워드 매칭
    kNN:  "배터리 설정 방법" 의미론적 유사 문서
    → 결합: "iPhone 15 배터리 관련" 문서 최상위

Reciprocal Rank Fusion (RRF) 공식:
  score_rrf(d) = Σ 1 / (rank_i(d) + k)

  rank_i(d): 각 검색 방식(BM25, kNN)에서 문서 d의 순위
  k: 상수 (기본 60, 낮은 순위 문서의 점수 감소 속도 조절)

  예시:
    BM25 순위: doc_A=1위, doc_B=3위, doc_C=50위
    kNN 순위:  doc_A=5위, doc_B=1위, doc_C=2위

    RRF 점수:
      doc_A: 1/(1+60) + 1/(5+60) = 0.0164 + 0.0154 = 0.0318
      doc_B: 1/(3+60) + 1/(1+60) = 0.0159 + 0.0164 = 0.0323
      doc_C: 1/(50+60) + 1/(2+60) = 0.0091 + 0.0161 = 0.0252

    최종 순위: doc_B > doc_A > doc_C
    → BM25와 kNN 모두에서 상위인 문서가 유리

ES 8.8+ 하이브리드 검색 API:
  GET /products/_search
  {
    "query": {
      "match": { "description": "배터리 설정" }
    },
    "knn": {
      "field": "embedding",
      "query_vector": [0.1, 0.3, ...],
      "k": 10,
      "num_candidates": 100
    },
    "rank": {
      "rrf": {
        "window_size": 100,   // 각 검색의 상위 N개 후보
        "rank_constant": 60   // k 상수
      }
    }
  }
```

### 5. 벡터 검색 실전 파이프라인

```
전형적인 RAG(Retrieval Augmented Generation) 파이프라인:

  문서 처리 (오프라인):
    원본 문서 → 청킹(Chunking) → 임베딩 모델 → 벡터
    벡터를 ES dense_vector 필드에 인덱싱

  검색 (온라인):
    사용자 질문 → 임베딩 모델 → 쿼리 벡터
    → kNN 검색 (HNSW) → 상위 K 문서 청크 검색
    → LLM에 검색 결과 + 사용자 질문 전달
    → LLM이 검색 결과 기반으로 답변 생성

  청킹 전략이 검색 품질에 미치는 영향:
    너무 작은 청크 (100자): 맥락 부족 → 낮은 검색 정확도
    너무 큰 청크 (5000자): 벡터 하나에 너무 많은 정보 → 평균화 문제
    권장: 256~512 토큰, 문단 단위 분할

  임베딩 모델 선택 포인트:
    도메인 특화: 법률, 의료, 기술 문서에는 해당 도메인 파인튜닝 모델
    언어: 한국어 포함 서비스는 다국어 모델 필수
    차원: 성능과 속도의 균형 (768차원이 범용적)
```

---

## 💻 실전 실험

```bash
# dense_vector 매핑 설정
PUT /vector-test
{
  "mappings": {
    "properties": {
      "title":     { "type": "text" },
      "content":   { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 4,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}

# 문서 인덱싱 (실제는 임베딩 모델 결과를 사용)
POST /vector-test/_bulk
{ "index": { "_id": "1" } }
{ "title": "키보드 추천", "content": "기계식 키보드 리뷰", "embedding": [0.8, 0.2, 0.1, 0.3] }
{ "index": { "_id": "2" } }
{ "title": "마우스 추천", "content": "무선 마우스 비교", "embedding": [0.1, 0.9, 0.2, 0.1] }
{ "index": { "_id": "3" } }
{ "title": "게이밍 장비", "content": "게임용 키보드 마우스 세트", "embedding": [0.5, 0.5, 0.2, 0.4] }
POST /vector-test/_refresh

# kNN 벡터 검색
GET /vector-test/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.7, 0.3, 0.1, 0.2],
    "k": 2,
    "num_candidates": 10
  }
}

# 하이브리드 검색 (BM25 + kNN + RRF)
GET /vector-test/_search
{
  "query": {
    "match": { "content": "키보드" }
  },
  "knn": {
    "field": "embedding",
    "query_vector": [0.7, 0.3, 0.1, 0.2],
    "k": 2,
    "num_candidates": 10
  },
  "rank": {
    "rrf": {
      "window_size": 10,
      "rank_constant": 60
    }
  }
}

# filter와 kNN 결합 (특정 조건 문서만 벡터 검색)
GET /vector-test/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.7, 0.3, 0.1, 0.2],
    "k": 2,
    "num_candidates": 10,
    "filter": { "match": { "title": "키보드" } }
  }
}

# HNSW 파라미터 커스텀
PUT /vector-custom
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 32,
          "ef_construction": 200
        }
      }
    }
  }
}

# 벡터 인덱스 통계
GET /vector-test/_stats?pretty
# knn_vectors 섹션 확인: 벡터 수, 크기
```

---

## 📊 BM25 vs kNN vs 하이브리드 비교

| 항목 | BM25 (텍스트) | kNN (벡터) | 하이브리드 (RRF) |
|------|-------------|----------|----------------|
| 정확한 키워드 매칭 | 매우 강함 | 약함 | 강함 (BM25 기여) |
| 의미론적 유사성 | 없음 | 매우 강함 | 강함 (kNN 기여) |
| 동의어/맥락 이해 | 동의어 설정 필요 | 자동 | 자동 |
| 속도 | 빠름 | HNSW로 빠름 | 중간 |
| 인덱스 크기 | 작음 | 큼 (벡터 차원 × 문서 수) | 매우 큼 |
| 설명 가능성 | 높음 (BM25 수식) | 낮음 (블랙박스) | 중간 |

---

## ⚖️ 트레이드오프

```
kNN 정확도 vs 속도 (num_candidates):
  높은 num_candidates → recall↑, 응답시간↑
  낮은 num_candidates → recall↓, 응답시간↓
  → SLA(응답 시간) 요구사항에 맞는 num_candidates 결정

HNSW 인덱스 크기:
  m↑, ef_construction↑ → 정확도↑, 메모리↑
  768차원 100만 문서 = ~3GB 벡터 데이터
  + HNSW 그래프 (m에 비례) = 추가 수백 MB
  → Page Cache 확보 중요 (힙 50% 제한의 이유)

임베딩 모델 업그레이드:
  더 좋은 모델 = 전체 재인덱싱 필요
  → 모델 버전 관리, 인덱스 별칭으로 무중단 전환
  → 임베딩 모델 변경 비용을 설계에 반영

하이브리드 vs 단일 방식:
  하이브리드: 최고 품질, 복잡도↑, 비용↑
  BM25만: 단순, 빠름, 의미 이해 없음
  kNN만: 의미 이해, 정확 키워드 약함
  → 서비스 검색 품질 요구와 비용 균형
```

---

## 📌 핵심 정리

```
Dense Vector 저장:
  문서 ID → N차원 부동소수점 배열 (역색인과 반대 방향)
  .vec, .vex 파일 (HNSW 그래프)
  메모리 사용: 차원 × 문서 수 × 4바이트

HNSW:
  계층적 그래프 기반 ANN 알고리즘
  탐욕 탐색으로 O(log N) 근사 탐색
  m: 이웃 연결 수, ef_construction: 인덱싱 정확도
  num_candidates: 검색 시 후보 수 (정확도 vs 속도)

하이브리드 검색 (RRF):
  BM25 순위 + kNN 순위를 역수 기반으로 결합
  점수 스케일 무관 → 두 방식의 강점 균형 있게 반영
  rank_constant(60): 낮은 순위 감소 속도 조절

선택 기준:
  정확 키워드 중심 → BM25
  의미 이해 중심  → kNN
  최고 품질       → 하이브리드 (BM25 + kNN + RRF)
```

---

## 🤔 생각해볼 문제

**Q1.** HNSW가 "근사" 탐색인데, 정확도 100%가 필요한 경우는 어떻게 하는가?

<details>
<summary>해설 보기</summary>

데이터 규모가 작고 응답 지연이 허용된다면 완전 탐색(Exact kNN)을 사용한다. `index: false`로 HNSW 그래프 미구축 후 `script_score`로 모든 문서에 대한 코사인 유사도를 계산하거나, `knn` 쿼리에서 `num_candidates`를 전체 문서 수로 설정하면 된다. 그러나 대규모 데이터에서는 비현실적이다. 현실적 대안은 `num_candidates`를 높여(예: k의 50배) recall을 99%+까지 높이거나, Quantization(양자화)으로 벡터를 압축해 메모리를 줄이고 더 많은 후보를 탐색하는 방법이 있다.

</details>

**Q2.** `similarity: "cosine"`과 `similarity: "dot_product"`를 선택하는 기준은?

<details>
<summary>해설 보기</summary>

코사인 유사도는 벡터의 크기(magnitude)를 무시하고 방향만 비교한다. 임베딩 벡터가 L2 정규화(크기=1)되어 있으면 코사인 유사도와 내적(dot product)이 수학적으로 동일한 결과를 낸다. 따라서 대부분의 임베딩 모델이 정규화된 벡터를 출력한다면 `dot_product`가 cosine보다 계산이 빠르다(정규화 과정 생략). 벡터 크기가 의미를 가지는 경우(예: 크기가 확신도를 나타내는 경우)에는 `dot_product`가 더 적합하다. 일반적인 의미론적 검색에는 `cosine`이 안전한 선택이다.

</details>

**Q3.** 하이브리드 검색에서 `rank_constant`를 낮추면 검색 결과가 어떻게 달라지는가?

<details>
<summary>해설 보기</summary>

RRF 공식에서 `rank_constant(k)` 값이 작을수록 상위 순위 문서와 하위 순위 문서의 점수 차이가 커진다. 예를 들어 k=60이면 1위와 2위의 점수 차이는 1/61 - 1/62 = 0.00026이지만, k=10이면 1/11 - 1/12 = 0.0076으로 커진다. 즉, k를 낮추면 순위 1위 문서에 강한 가중치를 주어 BM25나 kNN에서 명확히 1위인 문서가 최종 결과에서 더 유리해진다. k=60은 균형 있는 결합, k 낮추면 강한 신호에 집중하는 결합이다.

</details>

---

<div align="center">

**[⬅️ 이전: 주요 쿼리 유형 비교](./05-query-types-comparison.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — 집계 아키텍처 ➡️](../aggregation/01-aggregation-architecture.md)**

</div>
