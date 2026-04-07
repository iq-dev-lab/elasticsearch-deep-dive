# 클러스터 성능 진단 — `_cat`·`_cluster`·`_nodes/stats` 핵심 지표

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `_cluster/health`의 Green/Yellow/Red는 각각 어떤 상황을 의미하는가?
- `_cat/nodes`에서 어떤 지표를 보고 어떤 노드가 문제인지 판단하는가?
- `_nodes/stats`에서 indexing, search, GC 지표를 어떻게 해석하는가?
- slowlog를 설정하고 분석하는 실전 절차는 무엇인가?
- 성능 문제 유형별로 어떤 API를 먼저 확인해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"ES가 느려졌어요"라는 신고를 받았을 때 어디서 시작해야 하는가를 아는 것이 운영의 핵심이다. 힙 부족인지, 디스크 I/O 병목인지, 특정 쿼리가 문제인지, 세그먼트 병합이 과부하인지 — 진단 없이 추측으로 해결하면 원인도 찾지 못하고 서비스를 더 불안정하게 만든다. `_cat`, `_cluster`, `_nodes` API는 ES 클러스터의 모든 상태를 실시간으로 보여주는 진단 도구다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 클러스터 상태를 확인하지 않고 인덱스 설정만 변경
  상황: "검색이 느리다" → 매핑 변경, 샤드 수 변경 시도
  실제: 노드 1개의 힙이 95% → GC pause → 모든 작업 지연
  → 인덱스 설정 문제가 아니라 인프라 문제

실수 2: Yellow 상태를 Red와 동일하게 취급해 패닉
  오해: "Yellow = 데이터 손실 위험"
  실제:
    Yellow = Replica 미할당 (Primary 정상, 데이터 접근 가능)
    Red    = Primary 미할당 (해당 데이터 접근 불가)
  → Yellow는 장애가 아닌 "가용성 저하 위험" 상태
    Primary가 살아있으므로 서비스는 계속 가능

실수 3: 모든 노드에 _nodes/stats를 매번 전체 조회
  조회:
    GET _nodes/stats  (전체 통계, 매우 큰 응답)

  문제:
    응답 크기 수 MB ~ 수십 MB
    매번 전체 조회 → 불필요한 부하

  올바른 접근:
    GET _nodes/stats/indices/search  // 검색 통계만
    GET _nodes/stats/jvm             // JVM 통계만
    GET _nodes/stats/os,process      // OS, 프로세스만
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
성능 문제 진단 워크플로:

  Step 1: 클러스터 전체 상태 확인 (30초)
    GET _cluster/health?pretty
    GET _cat/nodes?v&h=name,heap.percent,cpu,load_1m,master,node.role
    → 이상 노드 식별

  Step 2: 문제 유형 분류
    힙 80%+ → JVM/메모리 문제 → 03-heap-memory-tuning 참고
    CPU 90%+ → 쿼리/집계 부하 또는 GC storm
    load_1m 높음 → 디스크 I/O 또는 CPU 포화
    Yellow/Red → 샤드 미할당 문제 → 06-operational-problems 참고

  Step 3: 구체적 통계 수집
    GET _nodes/stats/indices/search,indexing?pretty
    → 인덱싱/검색 처리량, 응답 시간 추세

  Step 4: 느린 쿼리/집계 식별
    slowlog 확인 or GET _nodes/hot_threads?pretty

  Step 5: 원인 조치 후 지표 재확인
    조치 전후 _nodes/stats 비교
```

---

## 🔬 내부 동작 원리

### 1. `_cluster/health` — 클러스터 건강 상태

```
GET _cluster/health?pretty

응답:
  {
    "cluster_name": "my-cluster",
    "status": "yellow",                      ← Green/Yellow/Red
    "timed_out": false,
    "number_of_nodes": 3,
    "number_of_data_nodes": 3,
    "active_primary_shards": 15,             ← 활성 Primary 수
    "active_shards": 20,                     ← 활성 전체 샤드 수
    "relocating_shards": 0,                  ← 이동 중인 샤드
    "initializing_shards": 0,               ← 초기화 중인 샤드
    "unassigned_shards": 5,                  ← 미할당 샤드 수 ← 문제!
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 80.0
  }

상태별 의미:
  Green:  모든 Primary + Replica 할당 완료 → 완전 정상
  Yellow: Primary 할당 완료 / Replica 일부 미할당 → 서비스 가능, 장애 내성 감소
  Red:    일부 Primary 미할당 → 해당 샤드 데이터 접근 불가, 부분 서비스 불가

  주의: 전체 상태 = 가장 나쁜 인덱스 상태
        인덱스 하나가 Red → 클러스터 전체 Red
        → 어느 인덱스가 문제인지 별도 확인 필요

인덱스별 상태 확인:
  GET _cat/indices?v&health=red   // Red 인덱스만
  GET _cat/indices?v&h=health,index,pri,rep,docs.count,store.size
```

### 2. `_cat/nodes` — 노드별 리소스 현황

```
GET _cat/nodes?v&h=name,ip,heap.percent,heap.current,heap.max,ram.percent,cpu,load_1m,master,node.role

주요 컬럼:
  name:          노드 이름
  heap.percent:  JVM 힙 사용률 (%)
  heap.current:  현재 힙 사용량
  heap.max:      최대 힙 크기
  ram.percent:   전체 RAM 사용률 (%)
  cpu:           CPU 사용률 (%)
  load_1m:       1분 평균 CPU 부하 (코어 수 기준)
  master:        *이면 현재 Master 노드
  node.role:     m(master-eligible), d(data), i(ingest)

임계값 기준:
  heap.percent:
    < 75%: 정상
    75~85%: 주의 (GC 빈번해짐)
    > 85%: 위험 (GC pause, OOM 위험)
    → 지속적으로 높으면 힙 크기 조정 or 데이터 분산

  cpu:
    < 70%: 정상
    70~90%: 주의
    > 90%: 위험 (쿼리 처리 지연, GC storm 가능)

  load_1m:
    코어 수와 비교
    16코어 노드에서 load_1m = 20 → 과부하

  실시간 모니터링:
    GET _cat/nodes?v&h=name,heap.percent,cpu,load_1m&s=heap.percent:desc
    → 힙 사용률 높은 노드 상위 정렬
```

### 3. `_nodes/stats` — 상세 성능 통계

```
GET _nodes/stats/indices/search,indexing,merges,store?pretty

인덱싱 통계:
  indices.indexing.index_total:       총 인덱싱 건수
  indices.indexing.index_time_in_millis: 누적 인덱싱 시간
  indices.indexing.index_current:     현재 진행 중인 인덱싱 수
  → 처리량: index_total / 가동시간
  → 평균 지연: index_time / index_total

검색 통계:
  indices.search.query_total:         총 쿼리 건수
  indices.search.query_time_in_millis: 누적 쿼리 시간
  indices.search.query_current:       현재 진행 중인 쿼리 수
  indices.search.fetch_total:         총 Fetch 건수
  indices.search.fetch_time_in_millis: 누적 Fetch 시간
  → 평균 쿼리 시간: query_time / query_total
  → 평균 Fetch 시간: fetch_time / fetch_total

병합 통계:
  indices.merges.current:             현재 진행 중인 병합 수
  indices.merges.total_time_in_millis: 누적 병합 시간
  → 병합이 너무 오래 걸리면 인덱싱/검색 성능에 영향

JVM 통계:
  GET _nodes/stats/jvm?pretty
  jvm.mem.heap_used_percent:          힙 사용률
  jvm.gc.collectors.young.collection_count: Young GC 횟수
  jvm.gc.collectors.young.collection_time_in_millis: Young GC 시간
  jvm.gc.collectors.old.collection_count: Old GC 횟수
  jvm.gc.collectors.old.collection_time_in_millis: Old GC 시간

  주의 기준:
    Old GC 빈번 (분당 1회 이상) → 힙 부족 신호
    Old GC time 높음 (초 단위) → GC pause → 서비스 지연

OS/디스크 통계:
  GET _nodes/stats/os,fs?pretty
  os.cpu.percent:              CPU 사용률
  fs.data[*].available_in_bytes: 사용 가능한 디스크 공간
  → 디스크 90% 이상 → 워터마크 주의
```

### 4. slowlog 설정과 분석

```
slowlog 목적:
  임계값 이상 걸리는 쿼리/인덱싱을 자동으로 로그에 기록
  "어떤 쿼리가 느린가"를 운영 중 추적

slowlog 설정:
  인덱스별 설정:
  PUT /myindex/_settings
  {
    "index.search.slowlog.threshold.query.warn":  "2s",
    "index.search.slowlog.threshold.query.info":  "1s",
    "index.search.slowlog.threshold.query.debug": "500ms",
    "index.search.slowlog.threshold.fetch.warn":  "1s",
    "index.indexing.slowlog.threshold.index.warn": "500ms"
  }

  클러스터 전체:
  PUT _cluster/settings
  {
    "persistent": {
      "cluster.search.slowlog.threshold.query.warn": "5s"
    }
  }

slowlog 파일 위치:
  {ES_HOME}/logs/{cluster_name}_index_search_slowlog.log
  {ES_HOME}/logs/{cluster_name}_index_indexing_slowlog.log

slowlog 분석:
  [2024-01-15T10:30:01,234][WARN ][i.s.s.query] [node-1] [products][0]
  took[2.1s], took_millis[2100], total_hits[150],
  search_type[QUERY_THEN_FETCH], total_shards[3],
  source[{"query":{"wildcard":{"title":{"value":"*board*"}}}}]

  핵심 정보:
    took: 실제 소요 시간
    total_hits: 검색 결과 수
    source: 실제 쿼리 본문 → 어떤 쿼리가 느린지 확인

  패턴 분석:
    특정 쿼리 패턴이 반복 → 해당 쿼리 최적화
    많은 total_hits + 낮은 from → Deep Pagination
    wildcard(*) → prefix 또는 ngram으로 대체

_nodes/hot_threads 실시간 분석:
  GET _nodes/hot_threads?pretty
  → 현재 CPU를 많이 쓰는 스레드 스택 추적
  → "어떤 작업이 CPU를 잡고 있는가" 실시간 확인

  GC 스레드가 많으면 → 힙 튜닝 필요
  검색 스레드가 많으면 → 쿼리 최적화 필요
  병합 스레드가 많으면 → refresh_interval 조정 필요
```

---

## 💻 실전 실험

```bash
# 클러스터 전체 건강 확인
GET _cluster/health?pretty

# 노드별 리소스 현황 (힙 사용률 내림차순)
GET _cat/nodes?v&h=name,heap.percent,heap.current,heap.max,ram.percent,cpu,load_1m,master&s=heap.percent:desc

# 인덱스별 상태 확인 (문제 있는 인덱스 찾기)
GET _cat/indices?v&h=health,status,index,pri,rep,docs.count,store.size&s=health

# Red/Yellow 인덱스만 보기
GET _cat/indices?v&health=red
GET _cat/indices?v&health=yellow

# 노드 상세 통계 (검색/인덱싱/JVM)
GET _nodes/stats/indices/search,indexing?pretty
GET _nodes/stats/jvm?pretty
GET _nodes/stats/os,fs?pretty

# 현재 진행 중인 작업 확인
GET _tasks?detailed=true&pretty

# 실시간 CPU 사용 스레드 확인
GET _nodes/hot_threads?pretty

# slowlog 설정
PUT /search-test/_settings
{
  "index.search.slowlog.threshold.query.warn":  "100ms",
  "index.search.slowlog.threshold.query.info":  "50ms",
  "index.search.slowlog.threshold.fetch.warn":  "50ms"
}

# 느린 쿼리 의도적으로 실행 (wildcard 앞에 *)
GET /search-test/_search
{ "query": { "wildcard": { "title": { "value": "*keyboard*" } } } }
# slowlog 파일에서 기록 확인

# 클러스터 통계 요약
GET _cluster/stats?pretty

# 샤드 이동 중인 경우 확인
GET _cat/recovery?v&active_only=true
```

---

## 📊 진단 API 요약

| API | 용도 | 주요 지표 |
|-----|------|---------|
| `_cluster/health` | 클러스터 전체 상태 | status(Green/Yellow/Red), unassigned_shards |
| `_cat/nodes` | 노드 리소스 현황 | heap.percent, cpu, load_1m |
| `_cat/indices` | 인덱스별 상태 | health, docs.count, store.size |
| `_cat/shards` | 샤드별 상태 | state, docs, store, node |
| `_nodes/stats/jvm` | JVM/GC 상태 | heap_used_percent, GC count/time |
| `_nodes/stats/indices` | 인덱싱/검색 통계 | index_total, query_total, fetch_time |
| `_nodes/hot_threads` | 실시간 CPU 사용 | 스레드 스택 추적 |
| slowlog | 느린 쿼리 자동 로깅 | 쿼리 본문, 소요 시간 |

---

## ⚖️ 트레이드오프

```
slowlog 임계값 낮게 (예: 50ms):
  이득: 더 많은 느린 쿼리 탐지
  비용: 로그 파일 빠르게 증가 → 디스크 부담

slowlog 임계값 높게 (예: 5s):
  이득: 진짜 심각한 쿼리만 기록
  비용: 중간 정도 느린 쿼리 탐지 못함

_nodes/stats 전체 조회:
  이득: 모든 정보 한번에
  비용: 응답 크기 큼, 파싱 비용
  → 필요한 항목만 명시: _nodes/stats/jvm,indices/search

_nodes/hot_threads:
  이득: 실시간 CPU 사용 파악
  비용: 프로파일링 오버헤드 (짧게 사용)
```

---

## 📌 핵심 정리

```
클러스터 상태:
  Green:  완전 정상
  Yellow: Primary 정상, Replica 일부 없음 (서비스는 가능)
  Red:    Primary 없음 (해당 데이터 접근 불가)

진단 순서:
  1. _cluster/health → 전체 상태 + unassigned_shards
  2. _cat/nodes → 문제 노드 식별 (힙/CPU/load)
  3. _nodes/stats → 구체적 통계 (인덱싱/검색/GC)
  4. slowlog → 느린 쿼리 식별
  5. _nodes/hot_threads → 실시간 CPU 사용

핵심 임계값:
  heap.percent > 85% → 힙 튜닝 필요
  Old GC 빈번(분당 1회+) → 힙 부족
  unassigned_shards > 0 → 샤드 미할당 진단
  cpu > 90% → 쿼리/집계 최적화 또는 확장
```

---

## 🤔 생각해볼 문제

**Q1.** `_cluster/health`에서 `active_primary_shards`가 설정값보다 낮다면 어떤 의미인가?

<details>
<summary>해설 보기</summary>

설정된 Primary Shard 수보다 `active_primary_shards`가 낮다면 일부 Primary가 할당되지 않은 것이다. 이는 Red 상태의 원인이며 해당 Primary Shard에 있는 데이터에 접근할 수 없다. 주요 원인은 해당 Primary를 보유했던 노드의 장애, 디스크 공간 부족으로 인한 샤드 이동 불가, 노드 수 부족 등이다. `_cluster/allocation/explain`으로 왜 Primary가 할당되지 않는지 상세 이유를 확인할 수 있다.

</details>

**Q2.** `_nodes/hot_threads`에서 GC 스레드가 지속적으로 나타난다면 즉각 해야 할 조치는?

<details>
<summary>해설 보기</summary>

GC 스레드가 `hot_threads`에 자주 나타나는 것은 JVM이 메모리 회수에 너무 많은 시간을 쓴다는 신호다(GC Storm). 즉각 조치로는 fielddata 캐시 강제 제거(`POST _cache/clear?fielddata=true`)로 힙 압박을 즉시 줄이는 것이 효과적이다. 대용량 집계나 쿼리가 원인이라면 해당 요청을 차단하거나 종료(`DELETE _tasks/{task_id}`)한다. 중장기적으로는 힙 크기 조정, fielddata 사용 줄이기(keyword 필드로 전환), 노드 증설을 검토한다.

</details>

**Q3.** slowlog가 `warn` 레벨에서 기록되었는데 실제 사용자는 5초가 아닌 1초라고 느낀다면 어떤 이유가 있을 수 있는가?

<details>
<summary>해설 보기</summary>

ES slowlog는 ES 서버 내부 처리 시간만 기록한다. 사용자 체감 시간에는 네트워크 왕복 시간(RTT), 로드 밸런서 지연, 클라이언트 직렬화/역직렬화 시간, 여러 ES 요청의 직렬 실행이 포함된다. 예를 들어 ES 내부 처리는 200ms이지만 애플리케이션에서 5개의 ES 요청을 순서대로 실행하면 총 1초가 된다. 또한 slowlog는 샤드 레벨 시간이므로 여러 샤드의 Scatter-Gather 병합 시간이 제외될 수 있다. 전체 경로의 성능을 측정하려면 APM(Application Performance Monitoring) 도구를 함께 사용해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: 인덱스 설계 전략](./01-index-design-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 힙 메모리 튜닝 ➡️](./03-heap-memory-tuning.md)**

</div>
