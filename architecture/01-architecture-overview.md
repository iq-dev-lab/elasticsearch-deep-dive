# 전체 아키텍처 개요 — 클러스터·노드·인덱스·샤드·세그먼트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Elasticsearch는 Lucene 위에 무엇을 추가했는가? Lucene만으로는 왜 부족한가?
- 클러스터 → 노드 → 인덱스 → 샤드 → 세그먼트 5계층은 각각 무엇을 책임지는가?
- Master 노드, Data 노드, Coordinating 노드는 왜 역할이 분리되어 있는가?
- 하나의 검색 요청은 이 계층을 어떤 순서로 통과하는가?
- MySQL 테이블과 ES 인덱스·샤드는 어떤 관계인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Elasticsearch 트러블슈팅의 90%는 계층 구조를 이해하지 못해서 발생한다. "샤드가 미할당됐다"는 에러를 보고 인덱스 설정을 바꾸는 사람과, 노드의 디스크 워터마크 초과 → 샤드 이동 불가 → 미할당 상태 전환이라는 인과 관계를 아는 사람의 진단 속도는 완전히 다르다. 계층 구조는 Elasticsearch의 모든 설계 결정이 시작되는 지점이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: Elasticsearch를 처음 도입할 때의 흔한 오해

오해 1: "Elasticsearch = MySQL + 검색 기능"
  실제:
    MySQL → 행 지향, B-Tree 인덱스, 트랜잭션, 정규화
    ES     → 문서 지향, 역색인, 최종 일관성, 비정규화
    같은 데이터를 다루지만 설계 철학이 완전히 다름

오해 2: "인덱스 = 테이블, 그냥 하나 만들면 되지"
  실제:
    인덱스는 1개 이상의 Primary Shard로 쪼개짐
    각 샤드는 독립적인 Lucene 인스턴스
    샤드 수는 인덱스 생성 시 결정되며 이후 변경 불가
    → 처음 설계가 운영 전체에 영향

오해 3: "노드 한 대에 다 올리면 되지"
  실제:
    Master/Data/Coordinating 역할이 섞이면
    클러스터 상태 관리 부하 + 검색 부하가 한 노드에 집중
    → 대규모에서 Master 노드 GC pause → 클러스터 전체 불안정

오해 4: "Elasticsearch가 곧 Lucene이지"
  실제:
    Lucene = 단일 머신 로컬 라이브러리 (검색 코어)
    ES = Lucene을 분산 클러스터로 감싼 레이어
    분산 통신, 샤드 라우팅, 복제, REST API → 모두 ES가 추가한 것
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
계층별 책임을 명확히 이해한 접근:

  클러스터 수준: 노드 구성 설계
    Master 전용 노드 3대 (홀수, Quorum 보장)
    Data 노드 N대 (실제 데이터 저장 + 검색 처리)
    Coordinating 전용 노드 (클라이언트 요청 분산)
    → 역할 분리로 Master GC pause가 검색에 영향 없음

  인덱스 수준: 샤드 수 설계
    샤드 크기 목표: 10~50GB
    예상 데이터 100GB → Primary Shard 5개 (각 20GB)
    → 나중에 변경 불가이므로 처음부터 여유 있게 설계

  Lucene 수준: 세그먼트 이해
    각 샤드 내부는 여러 Lucene 세그먼트로 구성
    세그먼트는 불변 → 삭제는 삭제 마킹, 병합 시 실제 제거
    → 빈번한 삭제/수정 패턴은 ES에 비싸다
```

---

## 🔬 내부 동작 원리

### 1. Lucene vs Elasticsearch — 무엇이 추가됐는가

```
Lucene (로컬 라이브러리):
  ┌─────────────────────────────┐
  │          Lucene             │
  │  - 역색인 구축               │
  │  - 세그먼트 관리              │
  │  - BM25 스코어링             │
  │  - 단일 프로세스, 단일 머신     │
  │  - Java API로만 접근         │
  │  - 분산 기능 없음             │
  └─────────────────────────────┘

Elasticsearch가 추가한 것:
  ┌──────────────────────────────────────────────────────┐
  │                  Elasticsearch                       │
  │                                                      │
  │  분산 레이어:                                           │
  │    - 샤드 라우팅 (어느 샤드에 문서를 저장할지)               │
  │    - Primary → Replica 복제                           │
  │    - Scatter-Gather 검색 (모든 샤드에 병렬 요청 후 병합)     │
  │    - 클러스터 상태 관리 (Master 노드)                      │
  │                                                      │
  │  접근성:                                               │
  │    - REST API (HTTP/JSON)                            │
  │    - 다양한 클라이언트 SDK                               │
  │                                                      │
  │  운영 도구:                                             │
  │    - _cluster, _cat, _nodes API                      │
  │    - ILM (Index Lifecycle Management)                │
  │    - Snapshot/Restore                                │
  │                                                      │
  │  ┌───────────────────────────────────────────────┐   │
  │  │                Lucene (샤드 내부)               │   │
  │  │  역색인 | 세그먼트 | BM25 | FST Term Dictionary  │   │
  │  └───────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
```

### 2. 5계층 구조 완전 분해

```
계층 1: 클러스터 (Cluster)
  ┌──────────────────────────────────────────────────────┐
  │                   my-cluster                         │
  │  노드들의 집합 + 공유 클러스터 상태                          │
  │  클러스터 이름으로 노드가 자동 조인                           │
  │  클러스터 상태: 인덱스 목록, 샤드 배치, 매핑 정보 등            │
  │  → Master 노드가 클러스터 상태를 관리·배포                    │
  └────────────┬─────────────────────┬────────────────────┘
               │                     │
               ▼                     ▼

계층 2: 노드 (Node)
  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
  │  node-1        │   │  node-2        │   │  node-3        │
  │  [Master]      │   │  [Data]        │   │  [Data]        │
  │  [Data]        │   │                │   │                │
  │  역할:          │   │  역할:          │   │  역할:          │
  │  클러스터 상태   │   │  샤드 저장      │   │  샤드 저장      │
  │  샤드 배치 결정  │   │  검색 처리      │   │  검색 처리      │
  └────────┬───────┘   └───────┬────────┘   └───────┬────────┘

계층 3: 인덱스 (Index)  ←  MySQL의 "테이블"에 해당
  ┌──────────────────────────────────────────────────────┐
  │                    products (인덱스)                   │
  │  - 매핑(Mapping): 필드 타입 정의                         │
  │  - 설정(Settings): 샤드 수, 레플리카 수, 분석기 등           │
  │  - 실제 데이터는 샤드에 분산 저장                           │
  └────────────────────────┬─────────────────────────────┘

계층 4: 샤드 (Shard)  ←  Lucene 인스턴스 1개
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  Shard 0     │   │  Shard 1     │   │  Shard 2     │
  │  [Primary]   │   │  [Primary]   │   │  [Primary]   │
  │  node-1      │   │  node-2      │   │  node-3      │
  ├──────────────┤   ├──────────────┤   ├──────────────┤
  │  Shard 0     │   │  Shard 1     │   │  Shard 2     │
  │  [Replica]   │   │  [Replica]   │   │  [Replica]   │
  │  node-2      │   │  node-3      │   │  node-1      │
  └──────────────┘   └──────────────┘   └──────────────┘

  Primary: 쓰기 진입점, 복제 오케스트레이터
  Replica: 읽기 분산 + 장애 시 Primary 승격

계층 5: 세그먼트 (Segment)  ←  Lucene 내부
  ┌──────────────────────────────────────────────────────┐
  │               Shard 0 (Lucene 인스턴스)                │
  │                                                      │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
  │  │Segment 0 │  │Segment 1 │  │ In-memory Buffer │   │
  │  │(디스크)   │  │(디스크)   │  │ (refresh 전)     │   │
  │  │불변       │  │불변       │  │ 검색 불가          │   │
  │  │10,000문서 │  │ 5,000문서 │  │                  │   │
  │  └──────────┘  └──────────┘  └──────────────────┘   │
  │                                                      │
  │  refresh(기본 1초) → 새 세그먼트 생성 → 검색 가능         │
  │  flush → 디스크 fsync → 내구성 보장                    │
  └──────────────────────────────────────────────────────┘
```

### 3. 노드 타입과 역할 분리

```
Master-eligible 노드:
  역할: 클러스터 상태 관리
    - 인덱스 생성/삭제
    - 샤드 배치 결정 (어느 샤드를 어느 노드에 배치할지)
    - 노드 장애 감지 및 샤드 재배치
  권장 구성: 전용 Master 노드 3대 (홀수 = Quorum 보장)
  주의: Data 노드의 GC pause가 Master에 영향 없어야 함
        → 대규모에서 전용 분리 필수

Data 노드:
  역할: 실제 데이터 저장 + 검색·집계 처리
    - 샤드를 로컬 디스크에 저장
    - 검색 요청 시 로컬 샤드 탐색 후 결과 반환
    - 집계 연산 수행
  특징: 메모리/디스크/CPU 모두 많이 사용
        힙 크기와 OS Page Cache 설정이 핵심

Coordinating 노드 (= Client 노드):
  역할: 클라이언트 요청의 라우팅·병합
    - 검색 요청 수신 → 관련 샤드에 병렬 전달
    - 각 샤드 결과 수집 → 병합 → 클라이언트 응답
  특징: 데이터 저장 안 함, 순수 라우팅 전담
        대규모 클러스터에서 전용 Coordinating 노드 권장
        (검색 결과 병합이 힙을 많이 사용하므로 Data 노드와 분리)

  모든 노드는 기본적으로 Coordinating 역할 수행
  → 별도 Coordinating 노드 없이도 어느 노드에 요청해도 됨
  → 대규모에서는 전용 분리 권장
```

### 4. 검색 요청의 전체 경로

```
GET /products/_search
{ "query": { "match": { "title": "elasticsearch" } } }

Step 1: 클라이언트 → Coordinating 노드
  HTTP 요청 수신
  "products" 인덱스의 샤드 목록 확인 (클러스터 상태 참조)
  Primary/Replica 중 하나 선택 (부하 분산)

Step 2: Coordinating → 각 샤드 [Query Phase / Scatter]
  Shard 0, 1, 2 모두에게 병렬로 쿼리 전송
  각 샤드 (Lucene 인스턴스) 내에서:
    "elasticsearch" → Analyzer → 토큰화
    Term Dictionary(FST) 탐색 → Posting List 추출
    BM25 점수 계산
    상위 10개 문서 ID + 점수만 반환 (본문 아직 없음)

Step 3: Coordinating 병합 [Merge Phase]
  3개 샤드에서 각 상위 10개 수신 → 총 30개에서 전체 상위 10개 선정

Step 4: Coordinating → 대상 샤드 [Fetch Phase / Gather]
  선정된 문서 ID를 가진 샤드에 _source 데이터 요청
  실제 문서 본문 반환

Step 5: Coordinating → 클라이언트
  최종 응답 조립 후 반환

총 네트워크 왕복: Scatter(1회) + Gather(1회) = 최소 2회
→ 샤드가 많을수록 Scatter 병렬 요청 수 증가, 코디네이터 메모리 부담 증가
```

---

## 💻 실전 실험

```bash
# 클러스터 기본 정보 확인
GET _cluster/health?pretty

# 노드 목록과 역할 확인
GET _cat/nodes?v&h=name,ip,heap.percent,ram.percent,cpu,master,node.role
# node.role 컬럼 해석:
#   m = master-eligible
#   d = data
#   i = ingest
#   - = coordinating only

# 인덱스 생성 (샤드 3개, 레플리카 1개)
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "title":      { "type": "text" },
      "price":      { "type": "double" },
      "category":   { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}

# 샤드 배치 확인 (어느 노드에 어느 샤드가 있는지)
GET _cat/shards/products?v&s=shard

# 세그먼트 현황 확인
GET /products/_segments?pretty
# 핵심 필드:
#   num_docs     → 살아있는 문서 수
#   deleted_docs → 삭제 마킹된 문서 수 (병합 시 실제 제거)
#   size_in_bytes → 세그먼트 크기
#   committed    → 디스크 fsync 완료 여부

# 클러스터 전체 통계
GET _cluster/stats?pretty

# 인덱스 상세 통계
GET /products/_stats?pretty
```

---

## 📊 MySQL vs Elasticsearch 개념 대응표

| MySQL | Elasticsearch | 차이점 |
|-------|--------------|--------|
| 인스턴스 클러스터 | 클러스터 | ES는 분산이 기본 설계 |
| 서버 | 노드 | 역할 분리 (Master/Data/Coordinating) |
| 데이터베이스 | (없음) | ES는 인덱스가 최상위 데이터 단위 |
| 테이블 | 인덱스 | ES 인덱스는 내부적으로 샤드로 분산 |
| 파티션 | 샤드 | ES 샤드는 완전한 Lucene 인스턴스 |
| 행(Row) | 문서(Document) | ES 문서는 JSON, 중첩 구조 가능 |
| 컬럼 | 필드(Field) | ES 필드는 타입별 저장 방식이 완전히 다름 |
| 인덱스(B-Tree) | 역색인 | 용도 완전히 다름 — 전문 검색 vs 정확 조회 |
| 트랜잭션(ACID) | (없음) | ES는 문서 단위 원자성만 보장 |
| EXPLAIN | _explain + _profile | ES는 샤드별 실행 계획 확인 가능 |

---

## ⚖️ 트레이드오프

```
분산 아키텍처의 이득:
  수평 확장  → 노드 추가할수록 처리량·저장 용량 선형 증가
  가용성    → 레플리카가 있으면 노드 1개 장애 시 검색 계속 가능
  병렬 검색  → 10개 샤드에 동시 탐색 → 단일 샤드 대비 최대 10배 처리량

분산 아키텍처의 비용:
  네트워크 왕복       → 모든 검색 최소 2회 왕복 (단일 노드 대비)
  코디네이터 부담      → 샤드 수 × 결과 크기만큼 힙 사용
  분산 스코어링 왜곡   → 샤드별 IDF 통계가 달라 점수가 달라질 수 있음
                        (Ch4-03 상세 설명)
  운영 복잡도         → 클러스터 상태, 샤드 배치, 레플리카 동기화 관리 필요

샤드 수 결정의 딜레마:
  적으면 → 샤드 하나가 너무 커져서 병합·검색 부담 집중
  많으면 → 코디네이터 병합 오버헤드 증가, 세그먼트 효율 저하
  Elastic 권장: 샤드 하나당 10~50GB 목표
```

---

## 📌 핵심 정리

```
Elasticsearch = Lucene(검색 코어) + 분산 레이어

5계층 구조:
  클러스터 → 노드 → 인덱스 → 샤드 → 세그먼트
  (전체)    (머신)  (논리단위) (Lucene) (불변 데이터)

노드 역할 분리:
  Master       → 클러스터 상태 관리, 샤드 배치 결정
  Data         → 샤드 저장 + 검색·집계 처리
  Coordinating → 요청 라우팅, 결과 병합

검색 요청 흐름:
  Coordinating
    → Scatter (모든 샤드 병렬 쿼리, 문서 ID + 점수 수집)
    → Merge   (상위 K개 문서 ID 선정)
    → Gather  (대상 샤드에서 _source 가져오기)
    → 응답 반환

핵심 설계 결정:
  샤드 수는 인덱스 생성 시 결정 → 이후 변경 불가
  세그먼트는 불변 → 삭제는 마킹, 병합 시 실제 제거
  레플리카는 읽기 분산 + 장애 복구 두 역할 동시 수행
```

---

## 🤔 생각해볼 문제

**Q1.** 샤드 수를 `number_of_shards: 100`으로 설정하면 검색이 더 빨라지는가?

<details>
<summary>해설 보기</summary>

반드시 그렇지 않다. 샤드가 100개이면 Coordinating 노드가 100개 샤드에 병렬 요청을 보내고, 각 샤드에서 상위 10개씩 최대 1,000개의 결과를 받아 병합해야 한다. 데이터가 적을 때 샤드가 너무 많으면 병합 오버헤드가 검색 이득을 초과한다. 각 샤드에 저장된 문서 수가 너무 적으면 Lucene 세그먼트도 효율적으로 최적화되지 않는다. Elastic이 샤드당 10~50GB를 권장하는 이유다.

</details>

**Q2.** 노드 1개 장애 시 Yellow 상태가 되는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Primary Shard는 살아있는 노드에서 계속 서비스되므로 검색·쓰기는 정상 동작한다. 단, 장애 노드에 있던 Replica Shard가 할당 해제된다. ES는 다른 노드에 Replica를 재할당하려 하지만, Primary가 있는 동일 노드에는 Replica를 배치할 수 없다. 남은 노드 수가 부족하면 Replica를 배치할 곳이 없어 Unassigned 상태가 되고, 이것이 Yellow다. Red는 Primary 자체가 할당 안 된 경우다.

</details>

**Q3.** Coordinating 노드를 전용으로 분리하면 어떤 이점이 있는가?

<details>
<summary>해설 보기</summary>

Coordinating 노드는 검색 결과 병합 시 힙 메모리를 많이 사용한다 (샤드 수 × 결과 크기). 이를 Data 노드에서 함께 처리하면 Data 노드의 힙이 검색 병합과 Lucene 역색인 캐시를 놓고 경쟁한다. 전용 Coordinating 노드를 두면 Data 노드는 OS Page Cache(Lucene 세그먼트 접근 최적화)에 집중할 수 있다. 트래픽이 많은 대규모 클러스터에서 효과적이다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 클러스터 상태와 Master 노드 ➡️](./02-cluster-state-master.md)**

</div>
