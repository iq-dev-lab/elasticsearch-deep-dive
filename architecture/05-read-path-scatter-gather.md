# 읽기 경로 — Scatter-Gather와 적응형 레플리카 선택

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Scatter-Gather 패턴에서 Query Phase와 Fetch Phase는 왜 분리되어 있는가?
- Coordinating 노드는 샤드별 결과를 어떻게 병합해서 정렬하는가?
- `from + size`(페이지네이션)가 깊어질수록 왜 성능이 나빠지는가?
- 적응형 레플리카 선택(ARS)이 응답 속도를 어떻게 개선하는가?
- `_count`, `_search` + aggregation은 Fetch Phase를 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

검색이 느린데 이유를 모르는 상황 대부분은 읽기 경로를 모르기 때문이다. "샤드를 늘리면 빨라진다"는 직관이 특정 상황에서 오히려 악화되는 이유, 페이지네이션이 100페이지 이후 갑자기 느려지는 이유, 레플리카를 늘려도 성능이 기대만큼 오르지 않는 이유 — 모두 읽기 경로의 Scatter-Gather 구조에서 답을 찾을 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: from + size 페이지네이션을 깊게 사용
  코드:
    GET /products/_search
    { "from": 10000, "size": 10 }  // 1,000번째 페이지

  내부에서 일어나는 일:
    각 샤드: 상위 10,010개 문서 ID + 점수 계산 후 반환
    (from=10000이면 앞의 10,000개를 건너뛰기 위해 먼저 다 가져와야 함)
    Coordinating: 샤드 수 × 10,010개 결과 병합
    → 3개 샤드 → 30,030개 결과를 힙에 올리고 정렬
    → from이 깊어질수록 메모리·CPU 부담 급증

  해결:
    search_after (커서 기반 페이지네이션) 사용
    또는 scroll API (대량 export 목적)

실수 2: 검색 결과 수가 많은데 _source를 전부 반환
  설정:
    GET /products/_search { "size": 1000 }
    → 1,000개 문서의 전체 _source 반환

  내부에서 일어나는 일:
    Fetch Phase: 1,000개 문서를 각 샤드에서 가져옴
    → 1,000개 × 문서 크기만큼 Coordinating 노드 힙 점유
    → 큰 문서 × 많은 결과 = OOM 위험

  해결:
    _source: false (문서 ID와 점수만 반환)
    _source: ["title", "price"] (필요한 필드만)
    hits.total만 필요하면 size: 0 사용

실수 3: 레플리카를 늘려도 검색이 빨라지지 않는다며 의아해함
  상황:
    number_of_replicas: 1 → 3으로 늘림
    단일 검색 요청의 응답 시간이 크게 개선되지 않음

  이유:
    단일 요청 응답 시간 = Query Phase 중 가장 느린 샤드 시간
    레플리카가 많아도 단일 요청은 각 샤드에서 하나만 선택
    → 레플리카 이점은 동시 요청 처리량(처리량 확장)에서 나타남
       단일 요청 지연(Latency)이 아닌 처리량(Throughput) 개선
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 페이지네이션 패턴 선택:

  from + size (기본, 소규모 페이지):
    적합: 10페이지 이내, 사용자가 앞 페이지로 돌아갈 수 있는 UI
    한계: from > 10,000은 기본 차단 (index.max_result_window 설정)

  search_after (커서 기반):
    적합: 무한 스크롤, 대량 페이지네이션
    원리: 마지막 결과의 sort 값을 다음 요청에 전달
    이점: 깊은 페이지도 항상 O(page_size)로 처리
    한계: 이전 페이지로 돌아갈 수 없음

  Point-in-Time (PIT) + search_after:
    적합: 일관된 스냅샷 상태에서 페이지네이션
    원리: PIT 생성으로 검색 시점 고정
    이점: 인덱싱이 계속되어도 페이지 결과 일관성 보장

  scroll API:
    적합: 대량 데이터 export (백업, 마이그레이션)
    한계: 실시간 페이지네이션에 부적합 (스냅샷 기반, 메모리 유지)
    권장: Reindex나 export 목적으로만 사용
```

---

## 🔬 내부 동작 원리

### 1. Query Phase — Scatter

```
GET /products/_search
{
  "query": { "match": { "title": "keyboard" } },
  "size": 10,
  "sort": [{ "_score": "desc" }, { "price": "asc" }]
}

Coordinating 노드가 하는 일:
  ① 클러스터 상태에서 "products" 인덱스의 샤드 목록 확인
  ② 각 샤드에서 Primary 또는 Replica 중 하나 선택 (ARS 적용)
  ③ 선택된 3개 샤드에 동일한 쿼리를 병렬 전송

각 샤드 (Lucene)가 하는 일:
  ① "keyboard" → Analyzer → ["keyboard"]
  ② Term Dictionary(FST)에서 "keyboard" 탐색
  ③ Posting List에서 문서 ID 목록 추출
  ④ BM25 점수 계산
  ⑤ sort 기준으로 상위 (from + size)개 선택
     → from=0, size=10이면 상위 10개 반환
     → from=100, size=10이면 상위 110개 반환 후 앞 100개 버림
  ⑥ 문서 ID + 점수 + sort 필드 값만 Coordinating으로 반환
     (_source는 아직 가져오지 않음)

반환 예시:
  Shard 0: [(doc_id:42, score:2.1, price:100), (doc_id:7, score:1.8, price:200), ...]
  Shard 1: [(doc_id:15, score:2.3, price:150), (doc_id:88, score:1.5, price:300), ...]
  Shard 2: [(doc_id:63, score:1.9, price:50),  (doc_id:31, score:1.7, price:400), ...]
```

### 2. Merge Phase — Coordinating의 병합

```
Coordinating 노드의 병합:

  3개 샤드에서 각 10개씩 → 총 30개 후보

  정렬 기준: _score DESC, price ASC
  30개 중 상위 10개 선택:

  최종 상위 10개 (문서 ID만 있음):
    1위: Shard 1, doc_id 15, score 2.3, price 150
    2위: Shard 0, doc_id 42, score 2.1, price 100
    3위: Shard 2, doc_id 63, score 1.9, price 50
    ...

  이 시점에서 Coordinating은:
    - 어느 문서가 상위 10개인지 알고 있음
    - 각 문서의 점수와 sort 값 알고 있음
    - 실제 문서 본문(_source)은 아직 모름

왜 Query Phase에서 _source를 안 가져오는가:
  30개 후보 중 최종 10개만 필요
  30개 전체의 _source를 가져오면 20개는 낭비
  → Query Phase: 최소 정보(ID + 점수 + sort)만 주고받아 병합 결정
  → Fetch Phase: 확정된 문서만 실제 본문 조회
  → 네트워크 전송량과 메모리 사용 최소화
```

### 3. Fetch Phase — Gather

```
Fetch Phase:
  Coordinating이 최종 선정된 10개 문서를 샤드별로 그룹핑

  Shard 0에 있는 문서: doc_id 42, 88, ...
  Shard 1에 있는 문서: doc_id 15, 31, ...
  Shard 2에 있는 문서: doc_id 63, ...

  각 샤드에 "이 문서들의 _source 줘" 요청 (병렬)
    → 각 샤드가 저장된 _source JSON 반환

  Coordinating이 최종 응답 조립:
    Query Phase 순서대로 정렬
    _source 결합
    → 클라이언트에 최종 응답

  총 네트워크 왕복:
    Query Phase: 1회 (Coordinating → 모든 샤드 → Coordinating)
    Fetch Phase: 1회 (Coordinating → 해당 샤드만 → Coordinating)
    합계: 2회

주의: Fetch Phase는 해당 문서를 가진 샤드에만 요청
  Query Phase는 모든 샤드에 요청 (Scatter)
  Fetch Phase는 일부 샤드에만 요청 (Targeted Gather)
  → 샤드 수가 많아도 Fetch Phase의 실제 요청 대상은 제한적
```

### 4. Deep Pagination 문제 — 왜 느려지는가

```
from=10000, size=10 요청 시:

  각 샤드가 해야 하는 일:
    상위 (10000 + 10) = 10,010개의 문서 ID + 점수 계산 및 정렬
    → 3개 샤드 × 10,010개 = 30,030개 결과 생성

  Coordinating이 해야 하는 일:
    30,030개 결과를 힙에 올리고 정렬
    앞 10,000개 버리고 10,001~10,010번째 반환
    → 힙 메모리: 30,030개 × (ID + 점수 + sort 값) 크기

  from=100,000이라면:
    각 샤드: 100,010개 계산
    Coordinating: 300,030개 병합
    → 메모리·CPU 부담이 선형으로 증가

기본 보호 장치:
  index.max_result_window: 10000 (기본값)
  from + size > 10000이면 오류 반환
  → 강제로 깊은 페이지네이션 차단

search_after로 해결:
  마지막 결과의 sort 값을 다음 요청에 전달

  1페이지:
  GET /products/_search
  { "size": 10, "sort": [{"_score": "desc"}, {"_id": "asc"}] }
  → 마지막 결과: score=1.5, _id="42"

  2페이지 (search_after 사용):
  GET /products/_search
  {
    "size": 10,
    "sort": [{"_score": "desc"}, {"_id": "asc"}],
    "search_after": [1.5, "42"]
  }
  → 각 샤드는 search_after 이후의 상위 10개만 계산
  → O(page_size)로 항상 동일한 비용
```

### 5. 적응형 레플리카 선택 (ARS)

```
기존 방식 (Round-Robin):
  Coordinating이 Primary/Replica 중 순서대로 선택
  문제: 느린 노드(GC pause, 디스크 I/O 지연)에도 균등하게 요청 전송
       → 느린 노드가 전체 검색 지연 유발

ARS (Adaptive Replica Selection):
  각 샤드 선택 시 과거 응답 시간 통계 기반으로 선택
  ┌──────────────────────────────────────────────────────┐
  │                      ARS 알고리즘                     │
  │                                                      │
  │  각 샤드 복사본(Primary/Replica)에 대해:                │
  │                                                      │
  │  EWMA(지수 가중 이동 평균)로 응답 시간 추적              │
  │  현재 큐에 쌓인 요청 수 추적                             │
  │                                                      │
  │  선택 점수 = EWMA_응답시간 × (1 + 대기_요청_수)          │
  │  → 점수 낮은 복사본 선택 (빠르고 한가한 노드)              │
  │                                                      │
  │  효과:                                                │
  │    GC pause 중인 노드 → 응답 시간 급증 → 점수 높아짐       │
  │    → 해당 노드로의 요청 자동 감소                         │
  │    GC 완료 후 → 응답 시간 정상 → 점수 낮아짐              │
  │    → 요청 자동 증가                                    │
  └──────────────────────────────────────────────────────┘

설정:
  cluster.routing.use_adaptive_replica_selection: true (기본값 ES 7.x+)

preference 파라미터로 직접 제어:
  GET /products/_search?preference=_local
  → 요청받은 노드의 로컬 샤드 우선 선택
  → 로컬 OS Page Cache 활용 → 네트워크 왕복 감소

  GET /products/_search?preference=_only_local
  → 로컬 샤드만 사용 (없으면 오류)

  GET /products/_search?preference=custom_value
  → 동일한 preference 값이면 항상 같은 샤드 선택
  → 샤드 캐시 활용 극대화
```

---

## 💻 실전 실험

```bash
# 기본 검색 + profile로 Query/Fetch Phase 분석
GET /products/_search
{
  "profile": true,
  "query": { "match": { "title": "keyboard" } }
}
# profile 결과에서 query, fetch 단계별 시간 확인

# _source 필드 제한으로 Fetch Phase 비용 줄이기
GET /products/_search
{
  "query": { "match_all": {} },
  "_source": ["title", "price"],
  "size": 100
}

# _source 완전 제거 (ID + 점수만 반환)
GET /products/_search
{
  "query": { "match": { "title": "keyboard" } },
  "_source": false
}

# search_after 페이지네이션 (1페이지)
GET /products/_search
{
  "size": 10,
  "sort": [{ "_score": "desc" }, { "_id": "asc" }],
  "query": { "match": { "title": "keyboard" } }
}

# search_after 페이지네이션 (다음 페이지)
GET /products/_search
{
  "size": 10,
  "sort": [{ "_score": "desc" }, { "_id": "asc" } ],
  "search_after": [2.3, "15"],
  "query": { "match": { "title": "keyboard" } }
}

# Point-in-Time + search_after (일관된 페이지네이션)
POST /products/_pit?keep_alive=1m
# 응답: { "id": "pit_id_here" }

GET /products/_search
{
  "size": 10,
  "sort": [{ "_score": "desc" }, { "_id": "asc" }],
  "pit": { "id": "pit_id_here", "keep_alive": "1m" },
  "query": { "match": { "title": "keyboard" } }
}

# preference로 로컬 샤드 우선
GET /products/_search?preference=_local
{ "query": { "match_all": {} } }

# 응답 시간 통계 확인 (ARS 효과 측정)
GET _nodes/stats/indices/search?pretty
# query_total, query_time_in_millis 비교
```

---

## 📊 페이지네이션 방식 비교

| 방식 | 비용 | 이전 페이지 이동 | 실시간성 | 적합 사용 사례 |
|------|------|----------------|---------|-------------|
| `from + size` | O(from + size) × 샤드 수 | 가능 | 실시간 | 소규모 페이지 (10페이지 이내) |
| `search_after` | O(size) | 불가 | 실시간 | 무한 스크롤, 대량 페이지 |
| `PIT + search_after` | O(size) | 불가 | PIT 시점 고정 | 일관된 스냅샷 필요 |
| `scroll` | O(size) | 불가 | 스냅샷 | 대량 export, 마이그레이션 |

---

## ⚖️ 트레이드오프

```
샤드 수와 검색 성능:
  샤드 수 증가 → 동시 처리량(Throughput) 향상
  샤드 수 증가 → 단일 요청 지연(Latency)에는 큰 영향 없음
               오히려 Coordinating 병합 부하 증가로 악화 가능
  → 샤드는 Throughput 확장 도구, Latency 개선은 쿼리/매핑 최적화로

레플리카와 읽기 성능:
  레플리카 증가 → 동시 검색 처리량 향상
  레플리카 증가 → 단일 요청 지연 개선 없음
               단, ARS로 느린 노드 우회는 가능

_source vs stored fields vs doc_values:
  _source: 전체 JSON 저장, Fetch Phase에서 파싱 비용
  stored fields: 특정 필드만 별도 저장, _source보다 빠른 필드 접근
  doc_values: 정렬·집계용 컬럼 지향 저장, _source와 무관
```

---

## 📌 핵심 정리

```
읽기 경로 3단계:

  Query Phase (Scatter):
    모든 샤드에 병렬 쿼리 전송
    각 샤드: 점수 계산 후 상위 (from + size)개 문서 ID + 점수 반환
    _source 없음, 최소 정보만 전달

  Merge Phase (Coordinating):
    샤드별 결과를 전역 정렬하여 최종 상위 size개 선정
    깊은 from → 많은 후보 정렬 → O(from × 샤드 수) 비용

  Fetch Phase (Gather):
    선정된 문서의 _source를 해당 샤드에서만 가져옴
    샤드 수와 무관하게 결과 수에 비례하는 비용

Deep Pagination 대안:
  search_after → 커서 기반, 항상 O(size) 비용
  PIT + search_after → 일관된 스냅샷 상태 보장

ARS (적응형 레플리카 선택):
  응답 시간 EWMA 기반으로 빠른 복사본 우선 선택
  느린 노드 자동 우회 → tail latency 개선
```

---

## 🤔 생각해볼 문제

**Q1.** size: 0으로 검색하면 Fetch Phase가 발생하는가?

<details>
<summary>해설 보기</summary>

발생하지 않는다. `size: 0`이면 반환할 문서가 없으므로 Fetch Phase 자체가 생략된다. Aggregation 결과나 `hits.total`만 필요할 때 `size: 0`을 사용하면 Fetch Phase 비용 없이 집계 결과를 얻을 수 있다. 많은 Aggregation 전용 요청이 `size: 0`을 사용하는 이유다.

</details>

**Q2.** 샤드 1개에만 데이터가 있는 인덱스에서 Scatter-Gather는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

샤드가 1개이면 Scatter 요청이 1개 샤드에만 전송된다. Merge Phase는 단일 샤드 결과이므로 병합 없이 그대로 사용한다. Fetch Phase도 해당 샤드에만 요청한다. 분산 오버헤드가 없어 가장 단순한 경로이지만, 샤드가 1개인 인덱스는 병렬 검색 불가능, 용량 확장 불가 등의 제약이 있다.

</details>

**Q3.** `preference=_local` 설정이 항상 유리한가?

<details>
<summary>해설 보기</summary>

단일 노드로 트래픽이 집중되면 오히려 불리할 수 있다. `_local`은 요청받은 노드의 로컬 샤드를 우선 사용하므로, 클라이언트가 특정 노드에 고정되어 있다면 해당 노드만 과부하가 걸린다. 반면 클라이언트가 여러 노드에 분산되어 있다면 각 노드가 자신의 로컬 샤드를 처리해 캐시 효율이 높아지고 전체 처리량이 향상된다. 캐시 히트율을 높이고 싶은 경우 `preference=custom_value`로 동일 요청 유형이 같은 샤드로 가도록 유도하는 것도 방법이다.

</details>

---

<div align="center">

**[⬅️ 이전: 쓰기 경로](./04-write-path.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — B-Tree vs 역색인 ➡️](../lucene-inverted-index/01-btree-vs-inverted-index.md)**

</div>
