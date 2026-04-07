<div align="center">

# 🔍 Elasticsearch Deep Dive

**"match 쿼리를 쓰는 것과, Lucene이 역색인을 어떻게 탐색하고 BM25로 점수를 계산하는지 아는 것은 다르다"**

<br/>

> *"`database-internals`에서 B-Tree를 배웠다면, 이제 '왜 전문 검색에는 B-Tree가 아닌 역색인인가'라는 질문으로 시작하는 레포"*

Lucene 세그먼트가 왜 불변(Immutable)인지, FST(Finite State Transducer)가 Term Dictionary를 어떻게 메모리에 압축 저장하는지, 분산 샤드에서 BM25 점수를 병합할 때 로컬 IDF 문제가 왜 발생하는지, Aggregation이 왜 힙(Heap)을 이렇게 많이 쓰는지  
**왜 이렇게 설계됐는가** 라는 질문으로 Elasticsearch와 Lucene 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Elasticsearch](https://img.shields.io/badge/Elasticsearch-8.11-005571?style=flat-square&logo=elasticsearch&logoColor=white)](https://www.elastic.co/guide/en/elasticsearch/reference/current/)
[![Lucene](https://img.shields.io/badge/Lucene-9.x-E5212D?style=flat-square&logo=apachelucene&logoColor=white)](https://lucene.apache.org/)
[![Spring](https://img.shields.io/badge/Spring_Data_ES-5.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-37개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Elasticsearch에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`match` 쿼리로 검색하세요" | `match` → Analyzer → 토큰 분리 → FST로 Term Dictionary 탐색 → Posting List 교집합 → BM25 점수 계산 전 과정 |
| "인덱스를 만드세요" | 샤드 수가 왜 나중에 변경 불가인지 (라우팅 수식), 세그먼트가 왜 불변인지, refresh/flush/fsync 각각의 역할 |
| "Elasticsearch는 검색에 강합니다" | B-Tree로 `LIKE '%검색어%'`가 Full Scan인 이유, 역색인이 Term → 문서 ID 목록을 O(len)에 찾는 원리 |
| "Aggregation으로 분석하세요" | Terms Aggregation이 각 샤드에서 top-N을 구하고 병합할 때 발생하는 오차, fielddata vs doc_values의 메모리 위치 차이 |
| "샤드를 늘리면 빨라집니다" | Scatter-Gather 구조에서 샤드가 많을수록 Coordinator 부담이 늘어나는 이유, 로컬 IDF로 인한 분산 스코어링 왜곡 |
| "Near Real-Time 검색입니다" | refresh가 왜 1초 주기인지, 디스크가 아닌 OS Page Cache에만 있어도 검색 가능한 이유, translog가 내구성을 보장하는 방식 |
| 이론 나열 | Kibana Dev Tools에서 즉시 실행 가능한 `_analyze` / `_explain` / `_cat` / `_segments` 실험 + Docker Compose 환경 + Spring 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Architecture](https://img.shields.io/badge/🔹_Architecture-클러스터·노드·샤드·세그먼트_계층_구조-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./architecture/01-architecture-overview.md)
[![Lucene](https://img.shields.io/badge/🔹_Lucene-B--Tree_vs_역색인_완전_비교-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./lucene-inverted-index/01-btree-vs-inverted-index.md)
[![Analysis](https://img.shields.io/badge/🔹_Analysis-분석_파이프라인_토큰이_만들어지는_과정-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./text-analysis/01-analysis-pipeline.md)
[![Query](https://img.shields.io/badge/🔹_Query-Query_DSL_내부_구조-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./query-engine/01-query-dsl-internals.md)
[![Agg](https://img.shields.io/badge/🔹_Aggregation-집계_아키텍처-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./aggregation/01-aggregation-architecture.md)
[![Ops](https://img.shields.io/badge/🔹_Operations-인덱스_설계_전략-005571?style=for-the-badge&logo=elasticsearch&logoColor=white)](./operations-tuning/01-index-design-strategy.md)
[![Spring](https://img.shields.io/badge/🔹_Spring-Spring_Data_Elasticsearch-005571?style=for-the-badge&logo=spring&logoColor=white)](./spring-elasticsearch/01-spring-data-elasticsearch.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Elasticsearch 아키텍처 — 분산 시스템의 기초

> **핵심 질문:** Elasticsearch는 Lucene 위에 무엇을 추가했는가? 클러스터/노드/샤드/세그먼트는 어떤 계층 구조이며, 쓰기와 읽기는 각각 어떤 경로로 처리되는가?

<details>
<summary><b>전체 아키텍처 개요부터 Scatter-Gather 읽기 경로까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 전체 아키텍처 개요 — 클러스터·노드·인덱스·샤드·세그먼트](./architecture/01-architecture-overview.md) | 클러스터 → 노드 → 인덱스 → 샤드 → 세그먼트 5계층 구조, Elasticsearch가 Lucene을 분산화하는 방식, 각 노드 타입(Master/Data/Coordinating)의 역할 분리 |
| [02. 클러스터 상태와 Master 노드 — 메타데이터와 Split-Brain](./architecture/02-cluster-state-master.md) | 클러스터 메타데이터(인덱스 매핑, 샤드 배치 정보)가 어떻게 관리되는가, Master Election 변화(Bully → Zen → Raft 기반), `minimum_master_nodes` 설정이 Split-Brain을 막는 원리 |
| [03. 샤드와 레플리카 — Primary/Replica 역할과 라우팅 수식](./architecture/03-shard-replica.md) | Primary와 Replica 샤드의 역할 분리, `hash(routing) % number_of_primary_shards` 라우팅 수식이 샤드 수 변경을 불가능하게 만드는 이유, Reindex 없이 샤드를 늘리지 못하는 이유 |
| [04. 쓰기 경로 — Primary에서 Replica로 복제되는 과정](./architecture/04-write-path.md) | 문서가 Coordinator → Primary Shard → Replica Shard로 전파되는 단계별 과정, `wait_for_active_shards` 설정의 의미와 내구성 트레이드오프, 쓰기 실패와 재시도 처리 방식 |
| [05. 읽기 경로 — Scatter-Gather와 적응형 레플리카 선택](./architecture/05-read-path-scatter-gather.md) | Coordinator가 모든 샤드에 요청을 병렬 분산하고 결과를 병합하는 Scatter-Gather 구조, Query Phase와 Fetch Phase 분리 이유, 적응형 레플리카 선택(ARS)이 응답 속도를 기반으로 샤드를 선택하는 방식 |

</details>

<br/>

### 🔹 Chapter 2: Lucene 역색인 — 검색의 본질

> **핵심 질문:** 왜 전문 검색에는 B-Tree가 아닌 역색인인가? 세그먼트는 왜 불변이며, "Near Real-Time"에서 "Near"는 왜 붙는가?

<details>
<summary><b>B-Tree vs 역색인 비교부터 doc_values 내부까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. B-Tree vs 역색인 — 왜 검색에는 역색인인가](./lucene-inverted-index/01-btree-vs-inverted-index.md) | MySQL `LIKE '%word%'`가 Full Scan인 이유, 역색인이 Term → 문서 ID 목록을 즉시 반환하는 원리, B-Tree가 강한 영역과 역색인이 강한 영역의 명확한 경계, `database-internals` 연결 지점 |
| [02. 역색인 구조 완전 분해 — Term Dictionary·Posting List·Frequency·Position](./lucene-inverted-index/02-inverted-index-structure.md) | Term Dictionary가 정렬된 단어 목록을 저장하는 구조, Posting List에 문서 ID가 delta-encoding으로 압축 저장되는 방식, Term Frequency / Position / Offset 정보가 추가되는 이유와 비용 |
| [03. FST — Term Dictionary를 메모리에 압축 저장하는 방법](./lucene-inverted-index/03-fst-term-dictionary.md) | Finite State Transducer가 공통 접두사/접미사를 공유해 메모리를 절약하는 구조, O(len) 탐색이 가능한 이유, HashMap 대비 메모리 효율 비교, Lucene이 FST를 직접 구현한 배경 |
| [04. Lucene Segment — 불변 세그먼트와 병합 정책](./lucene-inverted-index/04-lucene-segment.md) | 세그먼트가 불변(Immutable)인 이유(역색인 재구성 비용), 삭제가 실제 삭제가 아닌 삭제 마크로 처리되는 방식, 세그먼트 병합(Merge)이 삭제 문서를 실제로 제거하는 타이밍, TieredMergePolicy 동작 원리 |
| [05. Near Real-Time 검색 — refresh·flush·translog 역할](./lucene-inverted-index/05-near-real-time-search.md) | 인메모리 버퍼 → refresh → 새 세그먼트(검색 가능) → flush → fsync(디스크 내구성) 3단계 흐름, OS Page Cache에만 있어도 검색 가능한 이유, translog가 flush 전 데이터 손실을 막는 방식, `linux-for-backend-deep-dive`의 Page Cache·mmap 연결 |
| [06. doc_values vs fielddata — 정렬·집계를 위한 컬럼 지향 저장](./lucene-inverted-index/06-doc-values-vs-fielddata.md) | 역색인이 역방향 조회(문서 ID → 값)를 지원하지 못하는 이유, doc_values가 컬럼 지향으로 디스크에 저장되어 mmap으로 접근하는 방식(off-heap), fielddata가 힙에 올라와 OOM을 일으키는 원인, text 필드에 doc_values가 기본 비활성화된 이유 |

</details>

<br/>

### 🔹 Chapter 3: 텍스트 분석 — 토큰이 만들어지는 과정

> **핵심 질문:** "Hello World!"는 어떻게 `["hello", "world"]`가 되는가? 인덱스 시점과 검색 시점 Analyzer가 달라지면 어떤 일이 발생하는가?

<details>
<summary><b>분석 파이프라인부터 Mapping 폭발 문제까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 분석 파이프라인 — Character Filter·Tokenizer·Token Filter 체인](./text-analysis/01-analysis-pipeline.md) | Character Filter(HTML 태그 제거, 특수문자 변환) → Tokenizer(단어 분리) → Token Filter(소문자화, 동의어, 불용어) 3단계 체인 구조, `_analyze` API로 각 단계 중간 결과를 확인하는 방법 |
| [02. Standard·Nori Analyzer — 영문과 한국어 토큰화 원리](./text-analysis/02-standard-nori-analyzer.md) | Standard Analyzer의 Unicode Text Segmentation 기반 분리 규칙, Nori(한국어) 형태소 분석기의 품사 태깅과 어근 분리 원리, "전자제품"이 ["전자", "제품"]으로 분리되는 과정, `decompound_mode` 설정 효과 |
| [03. 커스텀 Analyzer 설계 — 동의어·Stemming·n-gram](./text-analysis/03-custom-analyzer.md) | 동의어 처리(index-time vs search-time 동의어의 차이), 불용어(Stopwords) 제거, Porter/Snowball Stemmer가 "running"을 "run"으로 줄이는 원리, n-gram Token Filter로 부분 검색을 가능하게 하는 비용 |
| [04. 매핑(Mapping) 완전 가이드 — text·keyword·dynamic mapping](./text-analysis/04-mapping-guide.md) | text(분석됨, 전문 검색용)와 keyword(분석 안 됨, 정렬·집계·정확 일치용)의 차이, dynamic mapping의 편의성과 Mapping Explosion 위험(자동 생성 필드 폭증), `strict` 모드로 의도치 않은 필드를 막는 방법 |
| [05. 인덱스 시점 vs 검색 시점 분석 — Analyzer 불일치 디버깅](./text-analysis/05-index-vs-search-time-analysis.md) | 인덱스 시점과 검색 시점 Analyzer가 달라졌을 때 검색이 실패하는 메커니즘, `search_analyzer` 설정의 용도와 함정, `_analyze` API로 토큰 불일치를 디버깅하는 실전 절차, Nori와 동의어 필터를 조합할 때 발생하는 순서 문제 |

</details>

<br/>

### 🔹 Chapter 4: 쿼리 엔진 — 검색과 스코어링

> **핵심 질문:** BM25는 "관련성"을 어떻게 숫자로 계산하는가? 같은 문서가 어느 샤드에 있느냐에 따라 점수가 달라지는 이유는 무엇인가?

<details>
<summary><b>Query Context·Filter Context부터 벡터 검색(kNN)까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Query DSL 내부 구조 — Query Context vs Filter Context](./query-engine/01-query-dsl-internals.md) | Query Context(스코어 계산, 캐시 미적용)와 Filter Context(스코어 없음, 비트셋 캐시 적용)의 내부 동작 차이, `bool` 쿼리의 `must/filter/should/must_not` 각각의 스코어 기여 방식, Filter Cache가 비트셋(bitset)으로 저장되는 이유 |
| [02. BM25 스코어링 완전 분해 — TF·IDF·필드 길이 정규화](./query-engine/02-bm25-scoring.md) | TF(단어 빈도)가 제곱근 스케일로 포화되는 이유(`k1` 파라미터), IDF(역문서 빈도)가 희귀 단어에 높은 점수를 부여하는 로그 수식, `b` 파라미터가 필드 길이로 TF를 정규화하는 방식, `_explain` API로 각 항 수치를 직접 확인하는 실험 |
| [03. 분산 스코어링 문제 — 로컬 IDF와 DFS_QUERY_THEN_FETCH](./query-engine/03-distributed-scoring-problem.md) | 샤드마다 IDF 통계가 달라 같은 단어가 샤드에 따라 다른 점수를 받는 원인, `DFS_QUERY_THEN_FETCH`가 모든 샤드의 글로벌 통계를 먼저 수집해 IDF를 통일하는 과정(요청 2배 비용), 대규모 클러스터에서 로컬 IDF가 자연스럽게 수렴하는 이유 |
| [04. 쿼리 실행 계획 — `_explain` API와 Lucene Boolean Query 최적화](./query-engine/04-query-execution-plan.md) | `_explain` API로 쿼리가 어떤 단계로 점수를 계산하는지 분해하는 방법, Lucene의 MUST/SHOULD/FILTER 절 최적화(비용 낮은 절 먼저 평가), `profile` API로 샤드별 쿼리 실행 시간을 측정하는 방법 |
| [05. 주요 쿼리 유형 비교 — match·term·range·bool·nested 내부 동작](./query-engine/05-query-types-comparison.md) | `match`(분석됨)와 `term`(분석 안 됨)의 차이가 검색 결과에 미치는 영향, `range` 쿼리가 역색인이 아닌 doc_values를 사용하는 이유, `nested` 쿼리가 별도 Lucene 문서로 저장된 중첩 객체를 조인하는 방식, 각 쿼리의 상대적 비용 수준 |
| [06. 벡터 검색(kNN) — Dense Vector·HNSW·하이브리드 검색](./query-engine/06-vector-search-knn.md) | Dense Vector 필드가 디스크에 저장되는 방식, HNSW(Hierarchical Navigable Small World) 그래프가 근사 최근접 이웃(ANN)을 효율적으로 탐색하는 원리, BM25 텍스트 검색과 kNN 벡터 검색을 결합하는 하이브리드 검색 `rank_constant` 파라미터 |

</details>

<br/>

### 🔹 Chapter 5: 집계(Aggregation) — 분석의 원리

> **핵심 질문:** Terms Aggregation 결과가 왜 부정확할 수 있는가? Aggregation이 힙을 많이 쓰는 이유는 무엇이며 OOM은 어떻게 막는가?

<details>
<summary><b>집계 아키텍처부터 성능 최적화 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 집계 아키텍처 — Bucket·Metric·Pipeline 계층과 분산 실행](./aggregation/01-aggregation-architecture.md) | Bucket(그룹화), Metric(수치 계산), Pipeline(집계 위의 집계) 세 계층의 역할, 집계가 각 샤드에서 부분 결과를 계산하고 Coordinator에서 병합되는 분산 실행 방식, 쿼리와 집계가 동시에 실행될 때의 처리 순서 |
| [02. Terms Aggregation 내부 — 샤드별 top-N 병합과 오차](./aggregation/02-terms-aggregation.md) | 각 샤드가 독립적으로 상위 N개를 반환하고 Coordinator가 이를 병합할 때 실제 상위 K개가 누락될 수 있는 수학적 원인, `shard_size`를 높여 정확도를 높이는 원리와 비용 트레이드오프, `doc_count_error_upper_bound`로 오차 범위를 확인하는 방법 |
| [03. Date Histogram 집계 — 시간 버킷·타임존·캘린더 인식 간격](./aggregation/03-date-histogram-aggregation.md) | 에포크 타임스탬프를 기반으로 시간 버킷을 생성하는 내부 계산 방식, 타임존 변환이 버킷 경계에 미치는 영향, `calendar_interval`(month/quarter)이 고정 밀리초가 아닌 캘린더 인식 계산을 사용하는 이유, 서머타임(DST) 처리 방식 |
| [04. 집계와 메모리 — fielddata·Circuit Breaker·OOM 방지](./aggregation/04-aggregation-memory.md) | text 필드에 집계를 실행할 때 fielddata가 힙에 통째로 올라오는 구조, `indices.fielddata.cache.size`로 힙 사용량을 제한하는 방법, Circuit Breaker가 OOM 전에 요청을 거부하는 메커니즘, `indices.breaker.fielddata.limit` 설정 |
| [05. 집계 성능 최적화 — Filter Aggregation·global_ordinals·pre-aggregation](./aggregation/05-aggregation-performance.md) | Filter Aggregation이 Query Cache를 활용해 반복 집계를 빠르게 만드는 원리, `eager_global_ordinals`로 샤드 시작 시 Terms 집계 자료구조를 미리 준비하는 방법, 대용량 데이터 집계 시 pre-aggregation 패턴(Rollup/Transform)의 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 6: 운영과 성능 튜닝

> **핵심 질문:** 힙을 RAM의 50% 이하로 제한하는 이유는? Yellow/Red 상태의 원인은 어떻게 진단하고, 쓰기/검색 성능 병목은 어디서 오는가?

<details>
<summary><b>인덱스 설계 전략부터 운영 장애 복구까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 인덱스 설계 전략 — 샤드 크기·롤오버·ILM 핫-웜-콜드](./operations-tuning/01-index-design-strategy.md) | 샤드당 10~50GB 가이드라인의 근거, 시계열 인덱스에서 인덱스 롤오버(Rollover)로 샤드를 분리하는 패턴, ILM(Index Lifecycle Management)의 핫(최신, 쓰기) → 웜(조회) → 콜드(보관) 전환 정책 설계 |
| [02. 클러스터 성능 진단 — `_cat`·`_cluster`·`_nodes/stats` 핵심 지표](./operations-tuning/02-cluster-performance-diagnosis.md) | `_cat/nodes?v&h=heap.percent,cpu,load_1m`로 노드 상태 파악, `_cluster/health`의 Green/Yellow/Red 판단 기준, `_nodes/stats`에서 indexing/search/GC 지표 해석, 느린 쿼리 로그(`slowlog`) 설정과 분석 |
| [03. 힙 메모리 튜닝 — RAM 50% 제한과 OS Page Cache](./operations-tuning/03-heap-memory-tuning.md) | ES_JAVA_OPTS 힙을 RAM 50% 이하로 제한하는 이유(나머지 50%를 Lucene의 OS Page Cache로 활용), G1GC vs ZGC 선택 기준, GC 오버헤드 경보 조건, `_nodes/stats/jvm`으로 GC 현황을 모니터링하는 방법 |
| [04. 쓰기 성능 최적화 — bulk API·refresh_interval·translog 설정](./operations-tuning/04-write-performance.md) | `refresh_interval=-1`로 쓰기 집중 구간 성능을 높이는 방법과 검색 지연 트레이드오프, Bulk API 배치 크기 튜닝(5~15MB 권장), translog `durability=async` 설정의 내구성 위험, 대규모 초기 적재(initial load) 시 설정 조합 |
| [05. 검색 성능 최적화 — Filter Cache·Request Cache·shard preference](./operations-tuning/05-search-performance.md) | Filter Context 쿼리가 비트셋으로 노드 메모리에 캐시되는 방식, Aggregation 결과를 인덱스 레벨에서 캐시하는 Request Cache 동작 조건, `preference=_local`로 동일 노드 샤드 우선 라우팅해 캐시 히트율을 높이는 방법 |
| [06. 운영 중 발생하는 문제 패턴 — Unassigned Shards·Yellow/Red 복구](./operations-tuning/06-operational-problems.md) | Yellow(Replica 미할당)와 Red(Primary 미할당) 상태의 정확한 원인 차이, `_cluster/allocation/explain`으로 샤드 미할당 이유를 진단하는 절차, 디스크 워터마크 초과로 샤드 이동이 멈추는 상황 해결, Split-Brain 발생 후 복구 순서 |

</details>

<br/>

### 🔹 Chapter 7: Spring과 Elasticsearch 통합

> **핵심 질문:** `@Document`·`@Field` 어노테이션은 내부 매핑과 어떻게 연결되는가? MySQL → Elasticsearch 동기화에서 정합성은 어떻게 관리하는가?

<details>
<summary><b>Spring Data Elasticsearch부터 CDC 데이터 동기화까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Spring Data Elasticsearch — Repository·Operations·매핑 어노테이션](./spring-elasticsearch/01-spring-data-elasticsearch.md) | `@Document(indexName)` → 인덱스 자동 생성 매핑 연결, `@Field(type=FieldType.Text, analyzer="nori")` → Lucene 매핑 변환 과정, `ElasticsearchRepository`와 `ElasticsearchOperations`의 책임 분리, `@Mapping`으로 커스텀 매핑 JSON을 직접 주입하는 방법 |
| [02. 인덱싱 전략 — 동기·비동기 인덱싱과 Bulk 처리](./spring-elasticsearch/02-indexing-strategy.md) | 단건 인덱싱 vs Bulk 인덱싱의 성능 차이(네트워크 왕복 횟수), Spring에서 비동기 Bulk 인덱싱 구현 패턴, 인덱싱 실패 시 재처리 전략(dead-letter, retry with backoff), `ElasticsearchOperations.bulkIndex()` 활용 |
| [03. 검색 쿼리 작성 — NativeQuery·CriteriaQuery·StringQuery 비교](./spring-elasticsearch/03-search-query-building.md) | `NativeQuery`(Query DSL JSON 직접 사용)와 `CriteriaQuery`(타입 안전 빌더)의 표현력·유지보수 트레이드오프, `StringQuery`가 적합한 상황, `QueryBuilder`로 동적 쿼리를 조합할 때의 null 처리 패턴, `SearchHits`에서 스코어와 하이라이트 추출 방법 |
| [04. 데이터 동기화 패턴 — CDC·Debezium·검색-DB 정합성 관리](./spring-elasticsearch/04-data-sync-patterns.md) | MySQL → Elasticsearch 동기화 방식 비교(Application-level 이중 쓰기, 배치 동기화, CDC), Debezium이 MySQL binlog를 읽어 변경 이벤트를 Kafka로 전송하는 원리, 검색 인덱스와 DB 사이의 일시적 불일치 허용 범위 설계, 재인덱싱(Reindex API) 전략 |

</details>

---

## 🧪 실험 환경

```yaml
# docker-compose.yml
services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es-node
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"   # REST API
      - "9300:9300"   # 노드 간 Transport
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"   # Kibana Dev Tools — 모든 실험은 여기서
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:5601/api/status || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 10
      start_period: 60s

  elasticsearch-exporter:
    image: quay.io/prometheuscommunity/elasticsearch-exporter:latest
    container_name: es-exporter
    depends_on:
      elasticsearch:
        condition: service_healthy
    command:
      - '--es.uri=http://elasticsearch:9200'
      - '--es.all'
      - '--es.indices'
      - '--es.shards'
    ports:
      - "9114:9114"   # Prometheus scrape endpoint

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    depends_on:
      - elasticsearch-exporter
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheusdata:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=7d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafanadata:/var/lib/grafana
    ports:
      - "3000:3000"   # admin / admin 으로 접속

volumes:
  esdata:
  prometheusdata:
  grafanadata:
```

> `prometheus.yml`은 `docker-compose.yml`과 같은 디렉토리에 두면 자동 마운트됩니다. 실행: `docker compose up -d`

| 서비스 | 포트 | 용도 |
|--------|------|------|
| Elasticsearch | `9200` | REST API / `_analyze`, `_explain`, `_cat` 실험 |
| Kibana | `5601` | Dev Tools — 모든 쿼리 실험 |
| ES Exporter | `9114` | Prometheus scrape |
| Prometheus | `9090` | 메트릭 수집 |
| Grafana | `3000` | 대시보드 시각화 (admin/admin) |

```bash
# 실험용 공통 명령어 세트 (Kibana Dev Tools 또는 curl)

# 분석기 동작 단계별 확인
GET /myindex/_analyze
{
  "analyzer": "nori",
  "text": "전자제품 검색 엔진"
}

# 쿼리 점수 계산 과정 완전 분해
GET /myindex/_explain/1
{
  "query": { "match": { "title": "elasticsearch" } }
}

# 세그먼트 현황 확인
GET /myindex/_segments?pretty
GET _cat/segments/myindex?v&h=index,shard,segment,docs.count,size

# 클러스터 상태 진단
GET _cluster/health?pretty
GET _cat/shards?v&s=index,shard
GET _cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m

# 샤드 미할당 원인 분석
GET _cluster/allocation/explain?pretty

# fielddata 힙 사용량 확인
GET _nodes/stats/indices/fielddata?fields=*&pretty

# 느린 쿼리 로그 임계값 설정
PUT /myindex/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "1s"
}

# 쿼리 실행 계획 프로파일링
GET /myindex/_search
{
  "profile": true,
  "query": { "match": { "description": "fast search" } }
}

# 분산 스코어링 문제 확인 (글로벌 IDF 사용)
GET /myindex/_search?search_type=dfs_query_then_fetch
{
  "query": { "match": { "title": "elasticsearch" } }
}
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 원리를 모를 때의 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/운영 |
| 🔬 **내부 동작 원리** | Lucene/ES 소스 레벨 분석, 실제 인덱스 파일 구조, ASCII 구조도 |
| 💻 **실전 실험** | Kibana Dev Tools, `_explain`, `_analyze`, `_cat`, `_profile` API |
| 📊 **성능/비용 비교** | B-Tree vs 역색인, fielddata vs doc_values, 쿼리 비용 수준 비교 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "match 쿼리가 내부에서 어떻게 동작하는지 모른다" — 검색 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch2-01  B-Tree vs 역색인 → 왜 검색에는 역색인인가
       Ch2-02  역색인 구조 완전 분해 → Posting List 탐색 원리
Day 2  Ch3-01  분석 파이프라인 → 토큰이 만들어지는 과정
       Ch3-04  매핑 완전 가이드 → text vs keyword 차이
Day 3  Ch4-01  Query DSL 내부 → Query vs Filter Context
       Ch4-02  BM25 스코어링 → 점수 계산 원리
```

</details>

<details>
<summary><b>🟡 "Elasticsearch 내부 구조를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  전체 아키텍처 개요 → 클러스터·노드·샤드·세그먼트 계층
       Ch1-03  샤드와 레플리카 → 라우팅 수식, 왜 샤드 수를 못 바꾸는가
Day 2  Ch2-01  B-Tree vs 역색인 → 전문 검색의 본질
       Ch2-04  Lucene 세그먼트 → 불변 설계와 병합 정책
Day 3  Ch2-05  NRT 검색 → refresh·flush·translog 역할
       Ch2-06  doc_values vs fielddata → 집계 메모리의 핵심
Day 4  Ch3-01  분석 파이프라인 → Analyzer 체인
       Ch3-04  매핑 완전 가이드 → Mapping Explosion 방지
Day 5  Ch4-02  BM25 스코어링 → TF·IDF·필드 정규화 수식
       Ch4-03  분산 스코어링 문제 → 로컬 IDF vs 글로벌 IDF
Day 6  Ch5-02  Terms Aggregation → 샤드별 병합 오차 원리
       Ch5-04  집계와 메모리 → Circuit Breaker, OOM 방지
Day 7  Ch6-03  힙 메모리 튜닝 → RAM 50% 제한의 이유
       Ch6-06  운영 문제 패턴 → Yellow/Red 진단 절차
```

</details>

<details>
<summary><b>🔴 "Lucene 소스코드까지 파고들어 내부를 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Elasticsearch 분산 아키텍처
        → Docker로 3노드 클러스터 구성, Scatter-Gather 요청 흐름 직접 관찰

2주차  Chapter 2 전체 — Lucene 역색인 내부
        → _segments API로 세그먼트 생성/병합 관찰, doc_values 파일 크기 측정

3주차  Chapter 3 전체 — 텍스트 분석 파이프라인
        → _analyze API로 각 필터 단계별 토큰 확인, Nori 분해 모드 비교 실험

4주차  Chapter 4 전체 — 쿼리 엔진과 BM25 스코어링
        → _explain로 BM25 수식 항별 수치 확인, profile API로 샤드별 실행 시간 측정

5주차  Chapter 5 전체 — 집계 원리와 메모리
        → Terms Aggregation shard_size 변화에 따른 오차 측정, Circuit Breaker 실험

6주차  Chapter 6 전체 — 운영과 성능 튜닝
        → slowlog 설정 후 느린 쿼리 탐색, Unassigned Shard 의도적 발생 후 복구 연습

7주차  Chapter 7 전체 — Spring 통합
        → @Field 어노테이션 → 매핑 JSON 변환 과정 확인, Debezium CDC 파이프라인 구성
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 연결 지점 | 연관 챕터 |
|------|--------------|-----------|
| [database-internals](https://github.com/dev-book-lab/database-internals) | B-Tree 인덱스 구조 → 역색인과 근본적 비교의 출발점 | Ch2-01(B-Tree vs 역색인), Ch4-02(IDF와 MySQL 통계의 비교) |
| [mysql-deep-dive](https://github.com/dev-book-lab/mysql-deep-dive) | InnoDB 행 지향 저장 vs Lucene 문서 지향 저장, LIKE 검색의 한계 | Ch2-01(역색인 필요성), Ch7-04(MySQL → ES 동기화 CDC) |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | Page Cache, mmap이 Lucene 세그먼트 접근과 직결 | Ch2-05(NRT와 OS Page Cache), Ch2-06(doc_values의 off-heap mmap), Ch6-03(힙 50% 제한과 Page Cache) |
| [network-deep-dive](https://github.com/dev-book-lab/network-deep-dive) | REST API 기반 클러스터 간 통신, Gossip 프로토콜 | Ch1-02(Master Election 네트워크), Ch1-05(Scatter-Gather HTTP 통신) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, `ElasticsearchAutoConfiguration` 등록 원리 | Ch7-01(Spring Data ES 자동 구성) |
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | `@Transactional`과 ES 인덱싱의 원자성 범위 | Ch7-02(인덱싱 실패 재처리 패턴) |

> 💡 이 레포는 **Lucene과 Elasticsearch 내부 동작**에 집중합니다. Spring을 모르더라도 Chapter 1~6을 순수 검색 엔진 관점으로 학습할 수 있습니다. `database-internals`로 B-Tree를 먼저 이해하면 Chapter 2의 역색인 비교가 훨씬 깊이 연결됩니다.

---

## 🙏 Reference

- [Elasticsearch 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/)
- [Elasticsearch: The Definitive Guide — Clinton Gormley & Zachary Tong](https://www.elastic.co/guide/en/elasticsearch/guide/current/)
- [Lucene in Action, 2nd Edition — Michael McCandless et al.](https://www.manning.com/books/lucene-in-action-second-edition)
- [Elasticsearch 소스 코드 (GitHub)](https://github.com/elastic/elasticsearch)
- [Elastic Blog](https://www.elastic.co/blog/)
- [Adrien Grand's Blog — Lucene 코어 개발자](https://www.elastic.co/blog/author/adrien-grand)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — 분산 검색 시스템 설계 관련 챕터

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"match 쿼리를 쓰는 것과, Lucene이 역색인을 어떻게 탐색하고 BM25로 점수를 계산하는지 아는 것은 다르다"*

</div>
