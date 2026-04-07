# 분산 스코어링 문제 — 로컬 IDF와 DFS_QUERY_THEN_FETCH

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 같은 문서가 어느 샤드에 있느냐에 따라 BM25 점수가 달라지는 이유는?
- `DFS_QUERY_THEN_FETCH`는 어떻게 글로벌 IDF로 점수를 통일하는가?
- 왜 대규모 클러스터에서는 로컬 IDF 문제가 자연스럽게 수렴하는가?
- 로컬 IDF 문제가 실무에서 문제가 되는 구체적인 시나리오는 무엇인가?
- 점수 왜곡 없이 성능을 유지하는 실용적인 대응 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

검색 결과 순위가 샤드 구성에 따라 달라진다는 사실을 모르면 "왜 이 문서가 저 문서보다 점수가 낮은지" 원인을 찾지 못한다. 특히 데이터가 적은 초기 단계나 샤드 수가 많고 데이터 분포가 불균등한 경우에 뚜렷하게 나타난다. `DFS_QUERY_THEN_FETCH`라는 해결책의 비용과 대규모 클러스터에서 수렴하는 원리를 이해하면 언제 개입해야 하는지 판단할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 점수 불일치를 버그로 오해
  상황:
    같은 검색어에서 날마다 결과 순위가 바뀜
    문서 A가 어떤 날은 1위, 다른 날은 3위

  실제 원인:
    새 문서 인덱싱 → 샤드별 IDF 통계 변경
    → 점수 재계산 → 순위 변동
    → "버그"가 아닌 IDF의 정상 동작

  해결:
    데이터가 적고 순위 일관성이 중요하면 DFS_QUERY_THEN_FETCH
    또는 단일 샤드로 운영 (테스트 환경)

실수 2: DFS_QUERY_THEN_FETCH를 항상 켜야 한다고 생각
  설정:
    모든 검색 요청에 search_type=dfs_query_then_fetch 적용

  문제:
    모든 샤드에 사전 통계 요청 1회 추가
    → 전체 검색 요청이 2배 네트워크 왕복
    → 샤드 수 × 2배 부하
    → 처리량 감소, 지연 증가

  실제:
    대규모 클러스터(샤드당 충분한 문서)에서는 로컬 IDF가 글로벌 IDF에
    자연스럽게 수렴 → DFS_QUERY_THEN_FETCH 불필요

실수 3: 샤드 수를 늘려도 점수 문제가 해결되지 않는다고 오해
  실제:
    샤드 수 증가 → 각 샤드의 문서 수 감소 → 통계 오차 증가
    → 오히려 로컬 IDF 문제가 더 심해짐
    올바른 방향: 샤드당 충분한 문서 + DFS 또는 단일 샤드
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
분산 스코어링 전략:

  개발/테스트 환경:
    number_of_shards: 1 → 로컬 IDF = 글로벌 IDF → 점수 일관성 보장
    → 스코어링 디버깅이 단순해짐

  소규모 프로덕션 (샤드당 문서 < 10만):
    DFS_QUERY_THEN_FETCH 적용 검토
    → 성능 비용 vs 점수 정확도 트레이드오프 판단

  대규모 프로덕션 (샤드당 문서 > 100만):
    로컬 IDF가 글로벌 IDF에 수렴 → 기본값(QUERY_THEN_FETCH) 유지
    → 점수 차이는 미미, 성능 우선

  순위 점수가 중요하지 않은 경우:
    filter 위주 검색 → Query Context 최소화 → 점수 문제 무관

  대안: 단일 샤드 인덱스 + 다수 레플리카
    Primary 1개 → IDF 통계 일관성 보장
    Replica 다수 → 읽기 처리량 확장
    → 쓰기 확장성은 포기, 검색 순위 일관성 확보
```

---

## 🔬 내부 동작 원리

### 1. 로컬 IDF 발생 원리

```
분산 환경에서 IDF 계산:

  인덱스: products (Primary Shard 3개)
  전체 문서: 300개
  "keyboard"가 등장하는 문서: 30개

  이상적 상황 (균등 분포):
    각 샤드: 100개 문서, "keyboard" 10개 문서
    샤드별 IDF = ln(1 + (100-10+0.5)/(10+0.5)) ≈ 2.27
    → 모든 샤드의 IDF 동일 → 점수 일관성 OK

  실제 상황 (불균등 분포):
    Shard 0: 100개 문서, "keyboard" 2개  → IDF = ln(1 + 98.5/2.5)  ≈ 4.0
    Shard 1: 100개 문서, "keyboard" 25개 → IDF = ln(1 + 75.5/25.5) ≈ 1.38
    Shard 2: 100개 문서, "keyboard" 3개  → IDF = ln(1 + 97.5/3.5)  ≈ 3.35

  결과:
    doc_A (Shard 0의 "keyboard" 포함 문서):
      IDF = 4.0 → 높은 점수

    doc_B (Shard 1의 "keyboard" 포함 문서):
      IDF = 1.38 → 낮은 점수

    동일한 tf, 동일한 내용의 문서라도:
      어느 샤드에 있느냐에 따라 점수가 거의 3배 차이!

  문서 라우팅:
    문서 ID의 해시로 샤드 결정 → 랜덤 분포
    → 특정 단어가 어느 샤드에 많이 배치될지 예측 불가
    → IDF 불균등은 구조적으로 발생할 수 있음
```

### 2. QUERY_THEN_FETCH (기본) vs DFS_QUERY_THEN_FETCH

```
기본: QUERY_THEN_FETCH (2페이즈)

  Phase 1 (Query Phase):
    Coordinator → Shard 0, 1, 2에 쿼리 전송
    각 샤드: 자신의 로컬 IDF 통계로 점수 계산
             상위 K개 문서 ID + 점수 반환

    Shard 0: doc_A = 3.96 (로컬 IDF 4.0 기준)
    Shard 1: doc_B = 1.38 (로컬 IDF 1.38 기준)
    Shard 2: doc_C = 3.35 (로컬 IDF 3.35 기준)

  Phase 2 (Fetch Phase):
    Coordinator: 병합 → doc_A > doc_C > doc_B 순서
    상위 문서 본문 가져오기

  문제:
    doc_B가 내용상 doc_A보다 더 관련있어도 낮은 점수
    → Shard 1에 "keyboard"가 많아 IDF가 낮기 때문

─────────────────────────────────────────────────────────

DFS_QUERY_THEN_FETCH (3페이즈, 선택적)

  Phase 0 (DFS = Distributed Frequency Search):
    Coordinator → 모든 샤드에 "통계 수집 요청"
    각 샤드: 자신의 단어 빈도, 문서 수 반환
      Shard 0: "keyboard" df=2,  전체=100
      Shard 1: "keyboard" df=25, 전체=100
      Shard 2: "keyboard" df=3,  전체=100

    Coordinator: 글로벌 통계 계산
      전체 문서 N = 300
      "keyboard" 등장 문서 n = 2+25+3 = 30
      글로벌 IDF = ln(1 + (300-30+0.5)/(30+0.5)) ≈ 2.27

  Phase 1 (Query Phase):
    Coordinator → 모든 샤드에 쿼리 전송 + 글로벌 IDF 전달
    각 샤드: 글로벌 IDF 2.27로 점수 계산
      doc_A: 2.27 × TF_norm = ...
      doc_B: 2.27 × TF_norm = ...
      (IDF 동일 → tf와 필드 길이만 점수 차이 만듦)

  Phase 2 (Fetch Phase):
    정확한 점수로 병합 → 관련성에 충실한 순위

  비용:
    Phase 0 추가 → 총 2회 왕복 → 성능 2배 비용
    모든 샤드에 DFS 요청 → 샤드 수에 비례한 추가 부하
```

### 3. 대규모 클러스터에서 수렴하는 이유

```
중심 극한 정리 관점:

  각 샤드의 문서가 전체 인덱스의 무작위 표본이라고 가정
  (해시 기반 라우팅 → 랜덤 분포에 가까움)

  샤드당 문서 수 N_s, "keyboard" 빈도 비율 p:
    예상 df_s = N_s × p
    표준 편차 = sqrt(N_s × p × (1-p))
    상대 편차 = sqrt(p(1-p)/N_s) → N_s 커질수록 감소

  샤드당 100만 문서:
    p=0.03 (3% 문서에 "keyboard"):
    상대 편차 ≈ sqrt(0.03 × 0.97 / 1,000,000) ≈ 0.00017
    → IDF 오차 극히 미미

  샤드당 1,000문서:
    상대 편차 ≈ sqrt(0.03 × 0.97 / 1000) ≈ 0.0054
    → IDF 오차 있음

  실무 임계점:
    샤드당 약 100만 문서 이상 → 로컬/글로벌 IDF 차이 1% 미만
    샤드당 10만 문서 미만 → 점수 왜곡 발생 가능

  결론:
    대용량 인덱스 (샤드당 수백만 문서)
    → 로컬 IDF가 글로벌 IDF에 통계적으로 수렴
    → DFS_QUERY_THEN_FETCH 불필요
    → 성능 우선

    소용량 인덱스 (테스트, 소규모 프로덕션)
    → 로컬 IDF 왜곡 가능
    → DFS_QUERY_THEN_FETCH 또는 단일 샤드 검토
```

### 4. 실무에서 점수 왜곡이 문제가 되는 시나리오

```
시나리오 1: 출시 초기 (데이터 적음)
  인덱스: 1,000개 문서, 샤드 5개 → 샤드당 200개
  "keyboard" 샤드 0에 2개, 샤드 1에 50개
  → IDF 차이 수 배 → 순위 왜곡 뚜렷

  대응: 출시 초기에는 단일 샤드, 성장 후 리샤딩 또는 Reindex

시나리오 2: 특정 카테고리 편중 인덱싱
  특정 날에 "keyboard" 관련 문서 1,000개 대량 인덱싱
  해시 분포에 따라 특정 샤드에 집중될 수 있음
  → 해당 샤드의 IDF 급락 → 해당 문서 점수 낮아짐

시나리오 3: A/B 테스트 검색 품질 비교
  실험군/대조군에서 동일 쿼리 결과 비교
  샤드 차이로 인한 IDF 편차가 실험 결과를 오염
  → A/B 테스트 시 DFS 사용 또는 단일 샤드 권장

시나리오 4: 인기 검색어 vs 비인기 검색어
  인기 검색어: 많은 문서에 등장 → IDF 낮고 샤드간 수렴 빠름
  비인기 검색어: 적은 문서에 등장 → IDF 높고 샤드간 편차 큼
  → 롱테일 검색어에서 점수 왜곡이 더 뚜렷하게 나타남
```

---

## 💻 실전 실험

```bash
# 단일 샤드 vs 다중 샤드 IDF 차이 실험
PUT /single-shard
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 }
}

PUT /multi-shard
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 0 }
}

# 동일 문서를 두 인덱스에 삽입
POST _bulk
{ "index": { "_index": "single-shard", "_id": "1" } }
{ "title": "keyboard keyboard keyboard" }
{ "index": { "_index": "single-shard", "_id": "2" } }
{ "title": "keyboard review" }
{ "index": { "_index": "single-shard", "_id": "3" } }
{ "title": "mouse review" }
{ "index": { "_index": "multi-shard", "_id": "1" } }
{ "title": "keyboard keyboard keyboard" }
{ "index": { "_index": "multi-shard", "_id": "2" } }
{ "title": "keyboard review" }
{ "index": { "_index": "multi-shard", "_id": "3" } }
{ "title": "mouse review" }

POST single-shard/_refresh
POST multi-shard/_refresh

# 두 인덱스에서 동일 쿼리 실행, 점수 비교
GET /single-shard/_search
{ "query": { "match": { "title": "keyboard" } } }

GET /multi-shard/_search
{ "query": { "match": { "title": "keyboard" } } }
# 다중 샤드에서 점수 차이 가능성

# DFS_QUERY_THEN_FETCH로 글로벌 IDF 사용
GET /multi-shard/_search?search_type=dfs_query_then_fetch
{ "query": { "match": { "title": "keyboard" } } }
# 단일 샤드 결과와 비교

# _explain으로 IDF 값 직접 확인
GET /single-shard/_explain/1
{ "query": { "match": { "title": "keyboard" } } }
# idf description 확인

GET /multi-shard/_explain/1
{ "query": { "match": { "title": "keyboard" } } }
# 샤드 배치에 따라 다른 IDF

# 샤드별 문서 분포 확인
GET _cat/shards/multi-shard?v&h=shard,docs.count,store.size

# profile로 DFS vs 기본 실행 시간 비교
GET /multi-shard/_search
{
  "profile": true,
  "query": { "match": { "title": "keyboard" } }
}
```

---

## 📊 search_type 비교

| 항목 | QUERY_THEN_FETCH (기본) | DFS_QUERY_THEN_FETCH |
|------|----------------------|---------------------|
| 네트워크 왕복 | 2회 | 3회 (DFS 추가) |
| IDF 통계 | 로컬 (샤드별) | 글로벌 (전체 합산) |
| 점수 정확도 | 샤드 분포에 따라 다름 | 글로벌 IDF 기준으로 정확 |
| 성능 | 빠름 | 약 2배 느림 |
| 적합 상황 | 대용량 (샤드당 > 100만) | 소용량, A/B 테스트 |

---

## ⚖️ 트레이드오프

```
DFS_QUERY_THEN_FETCH:
  이득: 글로벌 IDF → 샤드 위치 무관 일관된 점수
  비용: Phase 0 추가 → 2배 네트워크 비용 → 처리량 감소

로컬 IDF 허용:
  이득: 성능 우선, 대규모에서 자연 수렴
  비용: 소규모·불균등 분포에서 점수 왜곡 가능

단일 샤드 운영:
  이득: IDF 일관성 완전 보장, 디버깅 단순
  비용: 쓰기 처리량 제한, 샤드 수 늘리기 어려움
  → 쓰기 처리량보다 검색 일관성이 중요한 경우

실용적 결론:
  개발/테스트: 단일 샤드 or DFS
  소규모 프로덕션: DFS 고려
  대규모 프로덕션: 기본값 (자연 수렴 활용)
```

---

## 📌 핵심 정리

```
로컬 IDF 문제:
  BM25 IDF = 샤드 내 문서 수와 단어 빈도로 계산
  샤드마다 다른 IDF → 같은 문서도 어느 샤드에 있느냐에 따라 다른 점수
  데이터 적거나 불균등 분포일수록 왜곡 심함

DFS_QUERY_THEN_FETCH:
  Phase 0: 모든 샤드의 통계를 수집 → 글로벌 IDF 계산
  Phase 1: 글로벌 IDF로 일관된 점수 계산
  비용: Phase 0 추가 → 약 2배 성능 비용

대규모 수렴:
  샤드당 100만+ 문서 → 로컬 IDF ≈ 글로벌 IDF (통계적 수렴)
  → DFS 없이도 점수 일관성 자연적으로 확보

실무 권장:
  개발: 단일 샤드 (디버깅 단순)
  소규모/A/B: DFS_QUERY_THEN_FETCH
  대규모: 기본값 유지 (성능 우선)
```

---

## 🤔 생각해볼 문제

**Q1.** 샤드가 1개인 인덱스에서 DFS_QUERY_THEN_FETCH를 사용하면 성능 차이가 있는가?

<details>
<summary>해설 보기</summary>

단일 샤드에서 DFS Phase 0은 해당 단일 샤드에만 통계를 요청한다. 얻어오는 통계가 그 샤드 자체의 것이므로 로컬 IDF와 글로벌 IDF가 동일하다. 즉, DFS를 사용해도 점수가 변하지 않는다. 단지 불필요한 통계 수집 요청이 한 번 추가되므로 약간의 성능 저하는 있지만 단일 샤드에서는 미미하다. 따라서 단일 샤드 환경에서 DFS를 켜두는 것은 의미 없다.

</details>

**Q2.** 로컬 IDF 문제를 해결하기 위해 샤드 수를 1로 줄이는 대신 레플리카를 늘리면 읽기 처리량을 확보할 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. Primary Shard 1개 + Replica N개 구성에서 읽기 요청은 Primary와 모든 Replica에 분산된다. IDF 계산은 Primary의 통계를 기준으로 하고 (Replica는 Primary와 동일한 데이터를 가짐) 검색 요청은 모든 복사본에서 처리될 수 있다. 레플리카를 5개로 늘리면 읽기 처리량이 약 6배(Primary + 5 Replica)가 된다. 단, 쓰기 처리량은 Primary가 1개이므로 확장되지 않는다. 검색 중심 서비스에서 점수 일관성이 중요하다면 이 방법이 효과적이다.

</details>

**Q3.** 동일 내용의 문서가 샤드 간 IDF 차이로 점수가 다를 때, 이를 `function_score`로 보정하는 방법은?

<details>
<summary>해설 보기</summary>

`function_score`에 `script_score`를 사용해 BM25 점수를 무시하고 커스텀 점수를 계산하거나, `weight` 함수로 특정 샤드의 문서에 가중치를 부여하는 방법이 있다. 더 실용적인 방법은 점수 정규화로, BM25 점수를 min-max 스케일링해 샤드 간 절대 점수 차이를 무의미하게 만드는 것이다. 그러나 가장 근본적인 해결책은 DFS를 사용하거나 단일 Primary를 유지하는 것이다. `function_score` 보정은 복잡도가 높고 완전한 해결이 어렵다.

</details>

---

<div align="center">

**[⬅️ 이전: BM25 스코어링 완전 분해](./02-bm25-scoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: 쿼리 실행 계획 ➡️](./04-query-execution-plan.md)**

</div>
