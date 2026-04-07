# 운영 중 발생하는 문제 패턴 — Unassigned Shards·Yellow/Red 복구

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Yellow와 Red 상태의 정확한 차이는 무엇이고 각각 어떻게 접근해야 하는가?
- `_cluster/allocation/explain`으로 샤드 미할당 원인을 어떻게 찾는가?
- 디스크 워터마크 초과로 샤드 이동이 멈추는 상황은 어떻게 해결하는가?
- 노드 장애 후 데이터 손실 없이 클러스터를 복구하는 순서는?
- Split-Brain 발생 후 클러스터를 안전하게 복구하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

클러스터가 Red 상태가 되면 일부 데이터에 접근할 수 없어 서비스 장애로 직결된다. Yellow 상태가 지속되면 노드 1개 추가 장애 시 Red로 전환되는 위험이 있다. 이런 상황에서 원인을 빠르게 파악하고 안전하게 복구하는 절차를 알면 장애 시간을 수십 분에서 수 분으로 줄일 수 있다. 잘못된 조치(강제 할당, 설정 변경)는 데이터 손실을 초래할 수 있어 반드시 순서대로 진행해야 한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: Yellow 상태에서 패닉하여 데이터 삭제
  상황:
    Yellow 상태 지속
    "어딘가 문제가 있다" → 인덱스 재생성 시도

  위험:
    Yellow = Primary 정상, Replica만 미할당
    데이터는 모두 Primary에 존재
    인덱스 삭제 → 데이터 영구 손실!

  올바른 접근:
    Yellow 상태는 서비스에 영향 없음
    원인 파악 후 해결 (급할 필요 없음)
    _cluster/allocation/explain으로 원인 파악 먼저

실수 2: 디스크 부족 상황에서 샤드 강제 할당
  상황:
    디스크 95% → 샤드 이동 불가 → Yellow
    "샤드를 강제로 할당하면 되지 않나?"
    → cluster.routing.allocation.enable: all 설정

  결과:
    여전히 디스크 부족 → 할당해도 곧 다시 미할당
    더 심각하면 Red 상태
    디스크 정리 없이 해결 불가

실수 3: Red 상태에서 원인 파악 전 노드 재시작
  상황:
    노드 1개 장애 → Red 상태
    "노드 재시작하면 해결되지 않나?"

  위험:
    다른 노드에 Replica가 있으면 재시작 후 복구 가능
    하지만 이미 복구 중인 Replica가 있다면
    재시작으로 복구 중단 → 더 오래 걸림
    최악: 데이터 손실된 상태로 복구 완료

  올바른 접근:
    즉각 재시작 대신 _cluster/health, _cat/shards로 상태 파악 먼저
    Replica가 다른 노드에 있는지 확인 후 판단
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
장애 대응 원칙:

  1. 현재 상태 파악 먼저 (조치 전 진단)
     GET _cluster/health?pretty
     GET _cat/shards?v&s=state&h=index,shard,prirep,state,node,reason

  2. 원인 파악
     GET _cluster/allocation/explain?pretty
     → 가장 먼저 조치할 미할당 샤드의 원인 확인

  3. 원인별 대응
     디스크 부족 → 공간 확보 또는 임계값 조정
     노드 없음 → 노드 복구 또는 Replica 재할당
     설정 문제 → 할당 필터 확인 및 수정

  4. 조치 후 상태 확인
     GET _cluster/health?wait_for_status=green&timeout=60s

  절대 하지 말 것:
    원인 파악 전 강제 primary 재할당 (reroute allocate-empty-primary)
    → 데이터 손실 위험!
    인덱스 삭제로 Yellow/Red 해결 시도 → 데이터 손실!
```

---

## 🔬 내부 동작 원리

### 1. Yellow vs Red 상태 완전 분석

```
Green 상태:
  모든 Primary Shard 할당 완료
  모든 Replica Shard 할당 완료
  → 완전한 가용성 + 최대 내결함성

Yellow 상태:
  모든 Primary Shard 할당 완료   ← 데이터 접근 가능!
  일부 Replica Shard 미할당      ← 내결함성 감소
  → 서비스 정상, 하지만 노드 추가 장애 시 Red 위험

  Yellow 원인:
    ① 노드 수 < (1 + replica 수)
       2노드, replica=1 → Primary + Replica 같은 노드 불가
       → Replica 미할당
    ② 디스크 워터마크 초과로 새 샤드 할당 차단
    ③ 노드 장애로 Replica가 있던 노드 없어짐
    ④ 명시적 할당 필터로 특정 노드 제외됨
    ⑤ 인덱스 설정에 할당 필터 오류

Red 상태:
  일부 Primary Shard 미할당      ← 해당 데이터 접근 불가!
  → 해당 샤드의 데이터에 대한 검색/인덱싱 실패
  → 부분 서비스 불가

  Red 원인:
    ① Primary가 있던 노드 장애 + Replica도 없거나 미할당
    ② Snapshot 복원 중 실패
    ③ 인덱스 손상 (드문 경우)
    ④ 강제 할당 실수

  전체 상태 = 가장 나쁜 인덱스 상태
    → 테스트 인덱스 1개가 Red이면 클러스터 전체 Red
    → GET _cat/indices?v&health=red 로 문제 인덱스 식별
```

### 2. `_cluster/allocation/explain` — 미할당 원인 파악

```
특정 샤드 분석:
  GET _cluster/allocation/explain
  {
    "index": "products",
    "shard": 0,
    "primary": false    // false = Replica
  }

또는 첫 번째 미할당 샤드 자동 선택:
  GET _cluster/allocation/explain?pretty

응답 구조:
  {
    "index": "products",
    "shard": 0,
    "primary": false,
    "current_state": "unassigned",
    "unassigned_info": {
      "reason": "NODE_LEFT",          ← 원인 코드
      "at": "2024-01-15T10:30:00Z",
      "details": "node_left[node-2]"
    },
    "can_allocate": "no",
    "allocate_explanation": "...",
    "node_allocation_decisions": [
      {
        "node_id": "abc123",
        "node_name": "node-1",
        "can_allocate": "no",
        "deciders": [
          {
            "decider": "same_shard",
            "decision": "NO",
            "explanation": "the shard cannot be allocated to the same node the primary is on"
          }
        ]
      },
      {
        "node_id": "def456",
        "node_name": "node-3",
        "can_allocate": "no",
        "deciders": [
          {
            "decider": "disk_threshold",
            "decision": "NO",
            "explanation": "the node is above the high watermark cluster setting"
          }
        ]
      }
    ]
  }

decider 해석:
  same_shard:       Primary와 같은 노드에 Replica 불가
  disk_threshold:   디스크 워터마크 초과
  filter:           할당 필터로 제외됨 (node.attr 등)
  replica_after_primary_active: Primary 할당 후 Replica 할당
  max_retry:        재할당 최대 시도 횟수 초과

원인별 해결:
  NODE_LEFT + same_shard:  노드 추가 필요 또는 replica 수 감소
  disk_threshold:           디스크 공간 확보
  filter:                  필터 설정 확인 및 수정
```

### 3. 디스크 워터마크 — 샤드 이동 멈춤 원인

```
디스크 워터마크 3단계:

  Low Watermark (기본: 85%):
    새 샤드를 이 노드에 배치하지 않음
    기존 샤드는 이 노드에 유지 가능

  High Watermark (기본: 90%):
    이 노드의 기존 샤드를 다른 노드로 이동 시작
    (샤드 리밸런싱 트리거)

  Flood Stage (기본: 95%):
    이 노드가 속한 인덱스를 read-only로 잠금
    쓰기 완전 차단 (인덱스 보호)

  ┌─────────────────────────────────────────────────┐
  │  디스크 사용률                                     │
  │                                                 │
  │  0%──────────────85%────90%────95%──────100%    │
  │                  Low   High  Flood              │
  │                   ↑     ↑     ↑                 │
  │               새샤드  샤드   읽기                   │
  │               차단   이동   전용                   │
  └─────────────────────────────────────────────────┘

Flood Stage 복구 절차:
  ① 디스크 공간 확보 (오래된 인덱스 삭제 또는 데이터 이동)
  ② read-only 잠금 해제:
     PUT /myindex/_settings
     { "index.blocks.read_only_allow_delete": null }
  ③ 또는 전체 해제:
     PUT _cluster/settings
     { "persistent": { "cluster.blocks.read_only_allow_delete": null } }

임시 임계값 조정 (공간 확보 전 응급 조치):
  PUT _cluster/settings
  {
    "transient": {
      "cluster.routing.allocation.disk.watermark.low": "90%",
      "cluster.routing.allocation.disk.watermark.high": "95%",
      "cluster.routing.allocation.disk.watermark.flood_stage": "99%"
    }
  }
  주의: 이는 임시 조치, 반드시 디스크 공간 확보 후 원복
```

### 4. 노드 장애 복구 절차

```
노드 장애 시나리오:
  3노드 클러스터 (node-1, node-2, node-3)
  node-2 갑작스러운 장애

  즉각 반응:
    node-2의 Primary Shard → 다른 노드의 Replica 승격 시작
    node-2의 Replica Shard → 새 Replica 생성 시작 (다른 노드에)

  ES 기본 대기 시간 (delayed allocation):
    index.unassigned.node_left.delayed_timeout: 1m (기본)
    → 1분 동안 재할당 지연 (노드 재시작 기대)
    → 1분 후 재할당 시작

Step 1: 현재 상태 파악
  GET _cluster/health?pretty
  GET _cat/shards?v&s=state&h=index,shard,prirep,state,unassigned.reason,node

Step 2: 미할당 원인 확인
  GET _cluster/allocation/explain?pretty

Step 3: 노드 복구 가능성 판단
  복구 가능: node-2를 빠르게 재시작할 수 있음
    → 빠른 재시작 시 지연 없이 재합류, Replica 재구성 자동
    → 재할당 지연 시간을 더 늘려 재시작 기다리기:
      PUT _cluster/settings
      { "transient": { "index.unassigned.node_left.delayed_timeout": "10m" } }

  복구 불가: node-2 완전 손실
    → Replica를 다른 노드에 새로 생성 (자동 시작)
    → 재할당 지연 해제:
      PUT _cluster/settings
      { "transient": { "index.unassigned.node_left.delayed_timeout": "0s" } }
    → 즉시 재할당 시작

Step 4: Recovery 진행 모니터링
  GET _cat/recovery?v&active_only=true
  → source_host: Recovery 데이터 원본 노드
  → target_host: Recovery 대상 노드
  → bytes_percent: 진행률

Step 5: Green 상태 대기
  GET _cluster/health?wait_for_status=green&timeout=300s

절대 하지 말 것:
  Primary 손실 + Replica 없는 상황에서:
  POST _cluster/reroute
  { "commands": [{ "allocate_empty_primary": {
    "index": "products", "shard": 0, "node": "node-1", "accept_data_loss": true
  }}]}
  → accept_data_loss: true → 해당 샤드 데이터 모두 손실!
  → 데이터 복구 가능성 없을 때만 최후의 수단으로
```

### 5. Split-Brain 발생과 복구

```
Split-Brain 발생 상황 (ES 7.x 이전, 잘못된 설정):
  네트워크 파티션으로 클러스터가 두 개로 분리
  양쪽 모두 자신이 Master라고 주장
  → 양쪽 모두 독립적으로 인덱싱 → 데이터 분기

ES 7.x+에서 Split-Brain 예방:
  Raft 기반 합의 → 수학적으로 Split-Brain 불가
  → 현대 ES에서는 이 시나리오 자체가 발생하지 않음

Split-Brain 복구 (ES 6.x 이하 또는 레거시):
  ① 두 클러스터 중 "진짜" 클러스터 식별
     어느 쪽이 더 최신 데이터인가
     어느 쪽이 더 완전한 샤드를 가지는가
  ② 잘못된 쪽 클러스터 노드 모두 중지
  ③ 올바른 쪽 클러스터로 통합
  ④ 필요하면 Snapshot에서 복원

일반적인 Red 상태 복구 플로우:
  GET _cluster/health → Red 확인
  ↓
  GET _cat/shards?v&health=red → Red 인덱스/샤드 식별
  ↓
  GET _cluster/allocation/explain → 원인 파악
  ↓
  원인별 해결:
    디스크 → 공간 확보
    노드 없음 → 노드 복구 or 재할당
    설정 → 필터 수정
  ↓
  GET _cluster/health?wait_for_status=green → 복구 대기
```

---

## 💻 실전 실험

```bash
# 클러스터 상태 진단 전체 흐름
GET _cluster/health?pretty

# 문제 인덱스/샤드 식별
GET _cat/shards?v&s=state&h=index,shard,prirep,state,unassigned.reason,node

# Red/Yellow 인덱스만 확인
GET _cat/indices?v&health=red
GET _cat/indices?v&health=yellow

# 미할당 샤드 원인 분석 (자동 선택)
GET _cluster/allocation/explain?pretty

# 특정 샤드 지정 분석
GET _cluster/allocation/explain
{
  "index": "products",
  "shard": 0,
  "primary": false
}

# 디스크 사용량 확인
GET _cat/nodes?v&h=name,disk.used_percent,disk.used,disk.total

# 워터마크 임시 조정 (응급 시)
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "99%"
  }
}

# read-only 잠금 해제 (Flood Stage 복구)
PUT /myindex/_settings
{ "index.blocks.read_only_allow_delete": null }

# 재할당 지연 해제 (노드 복구 불가 시)
PUT _cluster/settings
{ "transient": { "index.unassigned.node_left.delayed_timeout": "0s" } }

# Recovery 진행 모니터링
GET _cat/recovery?v&active_only=true

# 할당 라우팅 재활성화 (비활성화된 경우)
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}

# Green 상태 대기 (최대 5분)
GET _cluster/health?wait_for_status=green&timeout=300s
```

---

## 📊 상태별 대응 요약

| 상태 | 의미 | 서비스 영향 | 즉각 조치 |
|------|------|----------|---------|
| Green | 완전 정상 | 없음 | 모니터링 유지 |
| Yellow | Replica 미할당 | 내결함성 감소 | 원인 파악, 여유있게 해결 |
| Red | Primary 미할당 | 데이터 일부 접근 불가 | 즉각 원인 파악 후 복구 |
| Disk Flood | 디스크 포화 | 쓰기 차단 | 디스크 공간 확보 최우선 |

---

## ⚖️ 트레이드오프

```
delayed_timeout 길게:
  이득: 노드 재시작 시 불필요한 Replica 재구성 방지
  비용: 실제 장애 시 Yellow/Red 상태 오래 지속
  → 인프라 안정성에 따라 1~10분 설정

allocate_empty_primary:
  이득: Red 상태 즉각 해소
  비용: 해당 샤드 데이터 완전 손실
  → 데이터 손실 명시 동의 필요, 최후 수단

디스크 워터마크 임시 상향:
  이득: 즉각 샤드 재할당 가능
  비용: 디스크 포화 위험 증가
  → 반드시 임시 적용 + 디스크 공간 확보 병행

replica 수 감소:
  이득: 노드 수 적을 때 Yellow 해소
  비용: 내결함성 감소 → 노드 장애 시 Red 위험 증가
  → 노드 추가가 근본 해결책
```

---

## 📌 핵심 정리

```
Yellow vs Red:
  Yellow: Primary 정상, Replica 미할당 → 서비스 가능, 내결함성 감소
  Red:    Primary 미할당 → 해당 샤드 데이터 접근 불가

진단 순서:
  _cluster/health → _cat/shards (미할당 식별)
  → _cluster/allocation/explain (원인 파악)
  → 원인별 해결

주요 원인과 해결:
  노드 부족 → 노드 추가 또는 replica 수 감소
  디스크 부족 → 공간 확보 + 워터마크 복원
  설정 오류 → 할당 필터 수정
  노드 장애 → 노드 복구 or 재할당 지연 해제

절대 금지:
  원인 파악 전 강제 할당 (accept_data_loss: true)
  Yellow 상태에서 인덱스 삭제
  Flood Stage에서 워터마크 조정 없이 쓰기 강행
```

---

## 🤔 생각해볼 문제

**Q1.** 단일 노드 ES 환경에서 `number_of_replicas: 1`로 설정했을 때 왜 항상 Yellow인가?

<details>
<summary>해설 보기</summary>

Primary Shard와 Replica Shard는 반드시 다른 노드에 위치해야 한다. 단일 노드에서는 Primary가 있는 노드에 Replica를 둘 수 없기 때문에 Replica가 항상 Unassigned 상태가 된다. 이것이 Yellow 상태의 원인이다. 해결 방법은 노드를 추가하거나 `number_of_replicas: 0`으로 설정하는 것이다. 개발 환경에서는 `number_of_replicas: 0`이 권장된다.

</details>

**Q2.** `_cluster/allocation/explain`에서 "max retry exceeded"가 나왔다면 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

ES는 샤드 할당에 반복적으로 실패하면 max retry(기본 5회)를 초과하고 자동 재시도를 중단한다. 이 상태에서는 수동으로 재시도를 초기화해야 한다. `POST _cluster/reroute?retry_failed=true`를 실행하면 max retry를 초과한 샤드들의 재시도 카운터를 초기화하고 다시 할당을 시도한다. 단, 근본 원인(디스크 부족, 노드 없음 등)이 해결되지 않으면 다시 실패하므로 원인 해결이 선행되어야 한다.

</details>

**Q3.** Snapshot을 복원하는 것이 Red 상태의 데이터 복구에서 언제 고려되어야 하는가?

<details>
<summary>해설 보기</summary>

Primary Shard가 미할당이고 해당 샤드의 데이터가 있는 노드가 모두 복구 불가능한 경우, 그리고 다른 노드에 Replica도 없는 경우에 Snapshot 복원이 유일한 방법이다. `allocate_empty_primary`로 강제 할당하면 해당 샤드 데이터가 완전히 손실되므로, Snapshot 복원이 데이터 손실을 최소화할 수 있다. Snapshot 복원은 마지막 스냅샷 시점 이후의 데이터는 복구할 수 없지만, 강제 할당보다 훨씬 적은 데이터 손실로 복구가 가능하다. 따라서 정기적인 Snapshot 정책이 DR(Disaster Recovery) 전략의 필수 요소다.

</details>

---

<div align="center">

**[⬅️ 이전: 검색 성능 최적화](./05-search-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Spring Data Elasticsearch ➡️](../spring-elasticsearch/01-spring-data-elasticsearch.md)**

</div>
