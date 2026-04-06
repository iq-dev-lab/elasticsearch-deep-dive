# 클러스터 상태와 Master 노드 — 메타데이터와 Split-Brain

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 클러스터 상태(Cluster State)에는 무엇이 담겨 있고, 누가 관리하는가?
- Master 노드는 어떻게 선출되고, 선출 과정에서 무슨 일이 발생하는가?
- Split-Brain이란 무엇이고, Elasticsearch는 어떻게 이를 방지하는가?
- ES 7.x 이전과 이후의 Master Election 방식은 어떻게 달라졌는가?
- `cluster.initial_master_nodes` 설정은 왜 최초 1회만 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

클러스터가 갑자기 Red 상태가 되거나, 두 클러스터가 서로 다른 매핑을 갖게 되는 Split-Brain 상황은 운영에서 가장 위험한 장애 중 하나다. 원인은 대부분 Master 노드의 역할과 선출 메커니즘을 이해하지 못한 설정에서 비롯된다. Master 노드가 클러스터의 "두뇌"라면, 이 두뇌가 어떻게 동작하고 어떻게 보호하는지 아는 것은 안정적인 ES 운영의 전제 조건이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Master-eligible 노드를 2개만 구성
  설정:
    node-1: master-eligible, data
    node-2: master-eligible, data

  문제:
    node-1 장애 → node-2 혼자 남음
    Quorum(과반수) = 2/2 = node-2 혼자로는 Quorum 불충족
    → 클러스터 전체 마비 (새 Master 선출 불가)

    반대 시나리오 (네트워크 단절):
    node-1과 node-2가 서로를 죽었다고 판단
    각자 자신을 Master로 선출 시도
    → Split-Brain: 두 개의 독립 클러스터가 생성
    → 서로 다른 문서가 각각 인덱싱됨
    → 네트워크 복구 후 데이터 충돌 (어느 것이 진실인가?)

실수 2: ES 7.x 이후에도 minimum_master_nodes 설정
  설정:
    discovery.zen.minimum_master_nodes: 2  ← 구버전 설정

  문제:
    ES 7.x부터 이 설정은 무시됨
    새 설정: cluster.initial_master_nodes 로 대체
    → 운영자가 보안감을 느끼지만 실제로는 아무 효과 없음

실수 3: 모든 노드를 Master-eligible로 구성
  설정:
    node.roles: [master, data]  (10개 노드 전체)

  문제:
    Master Election 시 10개 노드 모두 투표에 참여
    클러스터 상태를 10개 노드에 동기화해야 함
    → 클러스터 규모가 커질수록 상태 배포 부하 증가
    → Master 선출 속도 저하, 클러스터 반응성 나빠짐
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
권장 Master 노드 구성:

  전용 Master-eligible 노드 3대 (홀수!)
    node-master-1: node.roles: [master]
    node-master-2: node.roles: [master]
    node-master-3: node.roles: [master]

  Data 노드 N대
    node-data-*: node.roles: [data]

  이유:
    3대 → Quorum = 2 (과반수)
    1대 장애 → 남은 2대로 Quorum 충족 → 새 Master 선출 가능
    Data 노드 GC pause → Master에 영향 없음
    Master 전용 → 낮은 힙, 낮은 CPU → 안정적인 클러스터 관리

  ES 7.x+ 초기 설정:
    cluster.initial_master_nodes: ["node-master-1", "node-master-2", "node-master-3"]
    → 이 설정은 클러스터 최초 구성 시 단 한 번만 사용
    → 클러스터 재시작 시에는 기존 클러스터 상태를 사용하므로 무시됨
```

---

## 🔬 내부 동작 원리

### 1. 클러스터 상태(Cluster State)의 구조

```
클러스터 상태 = 클러스터의 모든 메타데이터

┌──────────────────────────────────────────────────────────┐
│                      Cluster State                       │
│                                                          │
│  ① 노드 정보                                              │
│     - 클러스터에 속한 노드 목록                              │
│     - 각 노드의 역할, IP, 포트                              │
│                                                          │
│  ② 인덱스 메타데이터                                        │
│     - 인덱스 이름 목록                                      │
│     - 각 인덱스의 매핑(Mapping) 정의                         │
│     - 각 인덱스의 설정(Settings)                            │
│                                                          │
│  ③ 샤드 라우팅 테이블                                       │
│     - "Shard 0 Primary → node-1"                        │
│     - "Shard 0 Replica → node-2"                        │
│     - "Shard 1 Primary → node-2" ...                    │
│     → 모든 문서 라우팅의 기준이 되는 지도                      │
│                                                          │
│  ④ 클러스터 메타데이터                                       │
│     - 클러스터 UUID                                       │
│     - 상태 버전 번호 (단조 증가)                              │
│                                                          │
└──────────────────────────────────────────────────────────┘

클러스터 상태의 특징:
  - Master 노드만 수정 가능 (단일 진실의 원천)
  - 변경 시 모든 노드에 동기적으로 배포
  - 매우 빈번한 읽기(라우팅 결정 시마다) + 드문 쓰기(인덱스 생성, 샤드 재배치)
  - 크기: 수십 MB ~ 수백 MB (인덱스/필드 수에 비례)
```

### 2. Master Election — ES 7.x 이전 (Zen Discovery)

```
ES 7.x 이전: Zen Discovery

  문제가 있었던 방식:
  ┌──────────────────────────────────────────────────┐
  │  각 노드가 독립적으로 "누가 Master인가?" 판단         │
  │                                                  │
  │  1. 현재 Master에게 Ping 전송                      │
  │  2. 응답 없으면 → "Master 죽었다" 판단               │
  │  3. Master-eligible 노드들이 서로 Ping              │
  │  4. 가장 오래된 노드(clusterStateVersion 높은 것)가   │
  │     스스로 Master 선언                              │
  │                                                  │
  │  문제: 이 판단이 각 노드에서 독립적으로 발생            │
  │    → 네트워크 단절 시 양쪽이 동시에 Master 선언 가능    │
  │    → Split-Brain                                 │
  └──────────────────────────────────────────────────┘

  방어책: minimum_master_nodes
    투표 참여 노드 수 ≥ minimum_master_nodes 이어야 Master 선출
    3대 구성 → minimum_master_nodes: 2 설정
    → 양쪽 분리 시 각각 2개 Quorum을 충족하지 못함 → 중단

  한계:
    이 설정을 올바르게 유지하는 것이 운영자 책임
    노드 추가 시 minimum_master_nodes 값도 수동으로 업데이트해야 함
    → 설정 실수 = Split-Brain 위험
```

### 3. Master Election — ES 7.x 이후 (Raft 기반)

```
ES 7.x+: Raft 기반 합의 알고리즘

핵심 변경:
  minimum_master_nodes 수동 관리 → 자동 Quorum 계산
  cluster.initial_master_nodes 으로 초기 멤버만 지정
  이후 노드 추가/제거 시 ES가 자동으로 투표 구성 업데이트

선출 과정:
  ┌──────────────────────────────────────────────────────┐
  │                  Master Election                     │
  │                                                      │
  │  1. 현재 Master 감지 실패 (heartbeat timeout)          │
  │                                                      │
  │  2. 후보 노드가 StartJoinRequest 브로드캐스트           │
  │     "나에게 투표해줘, 내 term은 N+1이야"                 │
  │                                                      │
  │  3. 각 Master-eligible 노드가 투표 (JoinRequest)      │
  │     조건: 요청자의 term이 자신보다 크거나 같음             │
  │     조건: 아직 이 term에서 다른 후보에게 투표 안 함         │
  │                                                      │
  │  4. 과반수(Quorum) 투표 확보 → Master 선출              │
  │     3노드 구성 → Quorum = 2                           │
  │     5노드 구성 → Quorum = 3                           │
  │                                                      │
  │  5. 새 Master가 클러스터 상태 배포                       │
  │     모든 노드가 새 Master 인식                           │
  └──────────────────────────────────────────────────────┘

Raft의 핵심 보장:
  동일 term에서 최대 1개의 Master만 선출
  → Split-Brain 수학적으로 불가능
  (양쪽이 각각 과반수를 얻으려면 공통 노드가 양쪽에 투표해야 하는데 불가)
```

### 4. Split-Brain 시나리오와 방지

```
Split-Brain 발생 시나리오:

  초기 구성: node-1(Master), node-2, node-3, node-4, node-5

  네트워크 파티션 발생:
  ┌────────────────────┐    X    ┌────────────────────┐
  │  Zone A            │ 단절    │  Zone B            │
  │  node-1 (Master)   │◄──────►│  node-3            │
  │  node-2            │        │  node-4            │
  │                    │        │  node-5            │
  └────────────────────┘        └────────────────────┘

  ES 7.x 이전 (minimum_master_nodes 잘못 설정 시):
    Zone A: node-1이 계속 Master (혼자 결정)
    Zone B: node-3이 새 Master 선출 (3대 중 3대 투표 = 과반수)
    → 두 Master가 동시에 존재
    → 각자 독립적으로 인덱싱 → 데이터 분기
    → 네트워크 복구 시 어느 쪽이 진실인지 판단 불가

  ES 7.x+ Raft 기반 (3 Master-eligible 노드 가정):
    Zone A: node-1 혼자 → Quorum(2) 미충족 → Master 지위 포기
    Zone B: node-2, node-3 → Quorum(2) 충족 → 새 Master 선출
    → 항상 한쪽만 Master
    → 데이터 분기 없음

올바른 Master-eligible 노드 수:
  최소 3개 (홀수) → 1개 장애 시 2개로 Quorum 충족
  5개         → 2개 장애 시 3개로 Quorum 충족
  짝수는 피할 것 → 50:50 분리 시 어느 쪽도 Quorum 못 가질 수 있음
```

### 5. 클러스터 상태 변경 흐름

```
새 인덱스 생성 요청이 클러스터 상태를 어떻게 변경하는가:

  클라이언트 → PUT /new-index

  1. 요청이 아무 노드에나 도착 (Coordinating)

  2. Coordinating → Master 노드로 포워딩
     (클러스터 상태 변경은 항상 Master가 처리)

  3. Master가 새 클러스터 상태 구성
     - 새 인덱스 메타데이터 추가
     - 샤드 배치 계산 (어느 노드에 샤드를 둘지)
     - 상태 버전 번호 +1

  4. Master → 모든 노드에 새 클러스터 상태 배포
     각 노드가 "OK" 응답

  5. Master → 해당 노드들에 샤드 초기화 명령

  6. 클라이언트에 응답 반환

  클러스터 상태 크기가 클수록:
    배포에 걸리는 시간 증가
    → 인덱스/필드 수를 적절히 제한해야 하는 이유
    → Mapping Explosion이 위험한 이유 (Ch3-04 참고)
```

---

## 💻 실전 실험

```bash
# 현재 Master 노드 확인
GET _cat/master?v

# 클러스터 상태 전체 조회 (크기 주의 — 운영 환경에서 조심)
GET _cluster/state?pretty

# 클러스터 상태 특정 파트만 조회
GET _cluster/state/metadata?pretty       # 인덱스 매핑·설정
GET _cluster/state/routing_table?pretty  # 샤드 배치 테이블
GET _cluster/state/nodes?pretty          # 노드 목록

# 클러스터 상태 버전 확인
GET _cluster/state/version?pretty

# Master-eligible 노드 목록
GET _cat/nodes?v&h=name,master,node.role

# 클러스터 설정 확인 (ES 7.x+ 방식)
GET _cluster/settings?pretty&include_defaults=true
# cluster.initial_master_nodes 는 초기 구성 후 제거하는 것을 권장

# 클러스터 상태 변경 감지 모니터링
GET _cluster/health?wait_for_status=green&timeout=30s

# 현재 투표 구성 확인 (ES 7.x+)
GET _cluster/voting_config_exclusions
```

---

## 📊 ES 버전별 Master Election 비교

| 항목 | ES 6.x 이하 (Zen) | ES 7.x+ (Raft 기반) |
|------|-----------------|-------------------|
| Split-Brain 방지 | `minimum_master_nodes` 수동 설정 | 자동 Quorum 계산 |
| 노드 추가 시 | `minimum_master_nodes` 수동 업데이트 필요 | 자동으로 투표 구성 업데이트 |
| 초기 설정 | `discovery.zen.ping.unicast.hosts` | `cluster.initial_master_nodes` |
| 선출 방식 | Bully 알고리즘 변형 | Raft 합의 알고리즘 |
| 동시 Master 가능성 | 설정 실수 시 가능 | 수학적으로 불가능 |

---

## ⚖️ 트레이드오프

```
Master 전용 노드의 이득:
  Data 노드 GC pause → Master 영향 없음
  클러스터 상태 관리에 전념 → 빠른 샤드 재배치
  대규모 클러스터에서 안정성 향상

Master 전용 노드의 비용:
  추가 서버 비용 (Master 전용 3대)
  소규모 클러스터에서는 불필요한 오버헤드

클러스터 상태 크기의 트레이드오프:
  인덱스·필드 수 많음 → 클러스터 상태 커짐
    → 상태 배포 지연 → 클러스터 반응성 저하
    → Mapping Explosion이 단순 저장 문제가 아닌 이유

Quorum 설정의 트레이드오프:
  Quorum 높임 → Split-Brain 안전 ↑, 장애 내성 ↓
  Quorum 낮춤 → 장애 내성 ↑, Split-Brain 위험 ↑
  → 홀수 노드로 자동 최적화 (3노드 → Quorum=2, 5노드 → Quorum=3)
```

---

## 📌 핵심 정리

```
클러스터 상태(Cluster State):
  모든 메타데이터의 단일 진실 원천
  Master만 수정 가능 → 변경 시 전 노드 동기 배포
  노드 정보 + 인덱스 메타데이터 + 샤드 라우팅 테이블 포함

Master Election (ES 7.x+):
  Raft 기반 합의 → 동일 term에서 최대 1개 Master 보장
  cluster.initial_master_nodes → 최초 구성 시 1회만 사용
  Quorum = (Master-eligible 수 / 2) + 1
  홀수 구성 필수 → 3대 권장 (Quorum=2)

Split-Brain 방지:
  ES 7.x+ → Raft로 수학적 불가능
  ES 6.x  → minimum_master_nodes 올바른 설정 필수
  네트워크 파티션 시 Quorum 있는 쪽만 서비스 계속

운영 핵심:
  Master 전용 노드 3대 (Data 노드와 역할 분리)
  Master 노드는 힙 최소화 (클러스터 상태 관리에만 집중)
  인덱스/필드 수 제한 → 클러스터 상태 크기 관리
```

---

## 🤔 생각해볼 문제

**Q1.** Master-eligible 노드를 4대로 구성하면 왜 좋지 않은가?

<details>
<summary>해설 보기</summary>

짝수 구성의 문제는 50:50 네트워크 파티션 시 어느 쪽도 Quorum(3/4)을 충족하지 못한다는 것이다. 4노드 → Quorum=3인데, 2:2로 분리되면 양쪽 모두 새 Master를 선출하지 못하고 클러스터 전체가 마비된다. 홀수 구성(3대)이라면 2:1 분리 시 2대 쪽이 Quorum(2/3)을 충족해 서비스를 계속할 수 있다.

</details>

**Q2.** `cluster.initial_master_nodes` 설정을 클러스터 재시작 후에도 남겨두면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

이 설정은 클러스터가 처음 구성될 때 "이 노드들로 새 클러스터를 시작하라"는 의미다. 클러스터가 이미 형성된 상태에서는 무시된다. 하지만 모든 노드가 동시에 재시작되는 경우 이 설정이 있으면 기존 클러스터 상태를 잃고 새 클러스터를 만들 위험이 있다. Elastic은 클러스터가 안정적으로 형성된 후 이 설정을 제거할 것을 권장한다.

</details>

**Q3.** 클러스터 상태가 매우 커지면 어떤 현상이 발생하는가?

<details>
<summary>해설 보기</summary>

클러스터 상태는 변경 시 Master에서 모든 노드로 동기적으로 배포된다. 상태 크기가 수백 MB에 달하면 배포 시간이 늘어나고, 그 동안 새로운 클러스터 상태 변경(인덱스 생성, 샤드 재배치 등)이 지연된다. 극단적인 경우 Master Election 타임아웃이 발생해 불필요한 재선출이 일어난다. Mapping Explosion(의도치 않은 필드 폭증)이 단순한 저장 문제를 넘어 클러스터 안정성 문제로 이어지는 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: 전체 아키텍처 개요](./01-architecture-overview.md)** | **[홈으로 🏠](../README.md)** | **[다음: 샤드와 레플리카 ➡️](./03-shard-replica.md)**

</div>
