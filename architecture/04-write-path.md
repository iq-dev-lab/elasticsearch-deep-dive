# 쓰기 경로 — Primary에서 Replica로 복제되는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 문서 하나를 인덱싱하면 내부적으로 어떤 순서로 처리되는가?
- `wait_for_active_shards` 설정은 내구성과 성능에 어떤 영향을 미치는가?
- Primary Shard에서 Replica로 복제가 실패하면 어떻게 처리되는가?
- 동일한 문서 ID로 두 번 인덱싱하면 무슨 일이 일어나는가?
- `_version`과 `_seq_no`는 어떤 역할을 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

쓰기 경로를 이해하지 못하면 "왜 인덱싱 후 바로 검색이 안 되지?", "왜 일부 문서가 사라졌지?", "왜 같은 문서가 두 번 들어갔지?" 같은 문제의 원인을 찾지 못한다. 특히 대규모 Bulk 인덱싱 시 `wait_for_active_shards` 설정이 처리량과 내구성 사이 어디에 위치해야 하는지는 비즈니스 요구사항과 직결된 판단이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 인덱싱 직후 검색 결과를 기대
  코드:
    elasticsearchClient.index(request);
    SearchResponse result = elasticsearchClient.search(searchRequest);
    // result가 비어있음 → "버그다!" 판단

  실제:
    인덱싱 → 인메모리 버퍼에 저장
    refresh(기본 1초)가 실행되어야 세그먼트 생성 → 검색 가능
    1초 이내에는 검색 불가 → "Near Real-Time"의 "Near"

  해결:
    테스트에서만: POST /myindex/_refresh (즉시 세그먼트 생성)
    프로덕션: 1초 지연을 설계에 반영 (NRT 특성 수용)

실수 2: wait_for_active_shards를 "all"로 설정
  설정:
    PUT /myindex { "settings": { "write.wait_for_active_shards": "all" } }

  문제:
    Replica 노드 1개 장애 시 → 해당 Replica가 응답 못함
    쓰기 요청이 타임아웃될 때까지 대기
    → 레플리카 장애가 쓰기 중단으로 이어짐

  올바른 접근:
    wait_for_active_shards: 1 (기본값) — Primary 확인만으로 응답
    고내구성 필요 시: quorum (과반수 Active Shard 확인)

실수 3: 같은 ID로 덮어쓰기를 DELETE 후 PUT으로 구현
  코드:
    elasticsearchClient.delete("products", "1");
    elasticsearchClient.index("products", "1", newDocument);

  문제:
    DELETE는 삭제 마킹만 함 (세그먼트에서 실제 제거는 병합 시)
    불필요한 API 호출 2번
    삭제와 재인덱싱 사이 타이밍 이슈 가능

  올바른 접근:
    같은 ID로 PUT → ES가 자동으로 새 버전으로 덮어씀
    _version이 자동 증가, 이전 문서는 삭제 마킹
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
쓰기 내구성 설계:

  쓰기 속도 최대화 (Bulk 초기 적재용):
    number_of_replicas: 0     # 복제 없음 → 쓰기 비용 최소
    refresh_interval: -1       # 자동 refresh 비활성화
    wait_for_active_shards: 1  # Primary 확인만으로 응답
    → 완료 후 레플리카 복원 + 수동 refresh

  운영 중 균형 설정 (기본 권장):
    number_of_replicas: 1
    refresh_interval: 1s       # NRT 1초 허용
    wait_for_active_shards: 1  # 빠른 응답 + Primary 내구성

  고내구성 필요 시 (금융 데이터 등):
    number_of_replicas: 2
    wait_for_active_shards: 2  # Primary + Replica 1개 이상 확인
    → Replica 장애 1개까지 허용하면서 내구성 보장
```

---

## 🔬 내부 동작 원리

### 1. 쓰기 요청의 전체 경로

```
PUT /products/_doc/1
{ "title": "Mechanical Keyboard", "price": 150000 }

Step 1: 클라이언트 → 아무 노드 (Coordinating)
  HTTP 요청 수신

Step 2: Coordinating — 샤드 라우팅
  shard_num = hash("1") % 3 = 0
  "Shard 0 Primary는 node-1에 있다" (클러스터 상태 참조)
  → node-1로 요청 포워딩

Step 3: Primary Shard (node-1) — 로컬 인덱싱
  ① 유효성 검증 (매핑 타입 확인, 필수 필드 등)
  ② 로컬 Lucene 인덱싱
     - 인메모리 버퍼에 문서 추가
     - Translog에 기록 (내구성 보장 — flush 전 데이터 손실 방지)
  ③ _version, _seq_no 할당
     _version: 1 (신규 문서), 덮어쓸 때마다 증가
     _seq_no: 전역 단조 증가 (복제 순서 보장)

Step 4: Primary → Replica 병렬 복제
  Shard 0 Replica: node-2에 위치
  Primary가 동일 요청을 Replica에 전송 (병렬)

  Replica (node-2):
    ① 동일한 인덱싱 작업 수행 (로컬 Lucene 인덱싱)
    ② "완료" 응답 → Primary로 전송

Step 5: Primary → Coordinating → 클라이언트
  wait_for_active_shards 조건 충족 확인 후 응답

  wait_for_active_shards: 1 (기본값)
    → Primary 로컬 인덱싱 완료만으로 즉시 응답
    → Replica 복제는 비동기로 계속 진행

  wait_for_active_shards: 2
    → Primary + Replica 1개 확인 후 응답
    → 응답 지연 증가 but 내구성 향상

응답:
  {
    "_index": "products",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": { "total": 2, "successful": 2, "failed": 0 },
    "_seq_no": 0
  }
```

### 2. Translog — flush 전 데이터 손실 방지

```
왜 Translog가 필요한가:

  인덱싱 흐름:
    문서 → 인메모리 버퍼 (검색 불가)
           ↓ refresh (1초마다)
         세그먼트 (검색 가능, OS Page Cache에만 존재)
           ↓ flush (translog 크기 초과 또는 명시적 호출)
         디스크 fsync (내구성 보장)

  세그먼트가 디스크에 fsync되기 전 노드 재시작 → 데이터 손실!

Translog의 역할:
  ┌─────────────────────────────────────────────────────┐
  │                    Translog                         │
  │                                                     │
  │  모든 인덱싱/삭제 작업을 순서대로 기록                       │
  │  디스크에 즉시 기록 (기본: fsync per request)             │
  │                                                     │
  │  노드 재시작 시:                                       │
  │    마지막 flush 이후의 translog 읽기                    │
  │    → 재생(replay)하여 데이터 복구                        │
  │                                                     │
  │  flush 후:                                          │
  │    translog 초기화 (이미 세그먼트에 반영됨)                │
  └─────────────────────────────────────────────────────┘

Translog 내구성 설정:
  index.translog.durability: request (기본값)
    → 매 요청마다 translog fsync
    → 노드 장애 시 데이터 손실 없음
    → 성능 비용 있음

  index.translog.durability: async
    → 주기적으로만 fsync (기본 5초)
    → 쓰기 성능 향상 (fsync 횟수 감소)
    → 노드 장애 시 최대 5초 데이터 손실 가능
    → 손실 허용 가능한 로그성 데이터에 적합
```

### 3. 버전 관리와 충돌 방지

```
_version (외부 버전 관리):
  신규 문서 인덱싱 → _version: 1
  같은 ID로 재인덱싱 → _version: 2 (자동 증가)
  DELETE 후 동일 ID 재인덱싱 → _version 계속 증가 (재사용 없음)

낙관적 동시성 제어:
  두 클라이언트가 동시에 같은 문서를 수정하는 문제:

  클라이언트 A: GET → { _seq_no: 5, _primary_term: 1, price: 100 }
  클라이언트 B: GET → { _seq_no: 5, _primary_term: 1, price: 100 }

  클라이언트 A: PUT /products/_doc/1?if_seq_no=5&if_primary_term=1
               { price: 200 }
               → 성공! _seq_no: 6

  클라이언트 B: PUT /products/_doc/1?if_seq_no=5&if_primary_term=1
               { price: 150 }
               → 실패! VersionConflictEngineException
                 (현재 _seq_no=6, 요청된 if_seq_no=5 불일치)

  → 클라이언트 B는 최신 문서를 다시 GET하고 재시도

_seq_no vs _version:
  _version → 문서 단위 단조 증가, 사용자 노출
  _seq_no  → 샤드 단위 전역 단조 증가, 복제 순서 보장에 사용
  if_seq_no + if_primary_term → 분산 환경에서 안전한 낙관적 잠금
```

### 4. Replica 복제 실패 시 처리

```
복제 실패 시나리오:

  Primary가 로컬 인덱싱 완료
  Replica 복제 시도 → Replica 노드 응답 없음 (장애)

  처리 흐름:
  ① Primary가 Master 노드에 "Replica가 응답하지 않음" 보고
  ② Master가 해당 Replica를 In-sync Replica Set(IRS)에서 제거
  ③ 클러스터 상태 업데이트
  ④ 해당 Replica가 복구되면 Primary로부터 변경 사항 재동기화

  wait_for_active_shards 관점:
    total shards = Primary + Replica = 2
    Replica 1개 장애 = active shards = 1 (Primary만)

    wait_for_active_shards: 1 → Primary만 있으면 응답 OK
    wait_for_active_shards: 2 → Replica도 응답해야 함
                              → Replica 장애 시 쓰기 타임아웃!

  결론:
    wait_for_active_shards: all 또는 고값 설정은
    Replica 장애가 쓰기 중단으로 이어지는 위험 있음
    운영 환경 기본값 1이 권장되는 이유
```

---

## 💻 실전 실험

```bash
# 문서 인덱싱 (기본)
PUT /products/_doc/1
{
  "title": "Mechanical Keyboard",
  "price": 150000,
  "category": "electronics"
}

# 인덱싱 직후 바로 검색 (refresh 전이라 결과 없을 수 있음)
GET /products/_search
{ "query": { "match": { "title": "keyboard" } } }

# 즉시 refresh 후 검색 (테스트 용도)
POST /products/_refresh
GET /products/_search
{ "query": { "match": { "title": "keyboard" } } }

# 낙관적 동시성 제어 실험
GET /products/_doc/1
# 응답에서 _seq_no, _primary_term 확인

PUT /products/_doc/1?if_seq_no=0&if_primary_term=1
{ "title": "Updated Keyboard", "price": 200000 }
# _seq_no가 다르면 VersionConflictEngineException 발생

# wait_for_active_shards 동작 확인
PUT /products/_doc/2?wait_for_active_shards=2
{ "title": "Mouse", "price": 50000 }

# Translog 설정 확인 및 변경
GET /products/_settings?include_defaults=true
# index.translog.durability 확인

PUT /products/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "5s"
}

# 쓰기 통계 확인 (인덱싱 횟수, 시간)
GET /products/_stats/indexing?pretty

# Bulk 인덱싱 (쓰기 성능 최적화)
POST /products/_bulk
{ "index": { "_id": "10" } }
{ "title": "Product A", "price": 10000 }
{ "index": { "_id": "11" } }
{ "title": "Product B", "price": 20000 }
{ "delete": { "_id": "1" } }
```

---

## 📊 wait_for_active_shards 설정별 트레이드오프

| 설정값 | 의미 | 내구성 | 쓰기 속도 | Replica 장애 영향 |
|--------|------|--------|----------|-----------------|
| `1` (기본) | Primary만 확인 | 낮음 | 빠름 | 없음 |
| `2` | Primary + Replica 1개 | 중간 | 중간 | Replica 장애 시 타임아웃 |
| `all` | 모든 Active Shard | 높음 | 느림 | 임의 Replica 장애 = 쓰기 중단 |
| `quorum` | 과반수 확인 | 중간~높음 | 중간 | 소수 장애 허용 |

---

## ⚖️ 트레이드오프

```
내구성 vs 쓰기 성능:
  wait_for_active_shards 높임 → 내구성 ↑, 쓰기 속도 ↓, 장애 내성 ↓
  translog.durability: async  → 성능 ↑, 노드 재시작 시 최대 sync_interval 손실

NRT vs 실시간:
  refresh_interval: 1s  → 최대 1초 지연, 쓰기 성능 영향 최소
  refresh_interval: -1  → 수동 refresh만, 쓰기 성능 최대 (초기 적재용)
  refresh_interval: 100ms → 더 짧은 지연, 세그먼트 병합 부하 증가

Translog 크기 vs flush 빈도:
  translog 크기 크게 → flush 빈도 감소 → 쓰기 성능 향상
                     → 재시작 시 replay 시간 증가
  translog 크기 작게 → flush 빈도 증가 → IO 부하 증가
```

---

## 📌 핵심 정리

```
쓰기 경로 5단계:
  클라이언트 → Coordinating (라우팅)
             → Primary Shard (로컬 인덱싱 + Translog)
             → Replica 병렬 복제
             → wait_for_active_shards 조건 충족 확인
             → 클라이언트 응답

Translog 역할:
  세그먼트가 디스크에 fsync되기 전 장애 대비
  재시작 시 translog 재생으로 데이터 복구
  flush 후 초기화

버전 관리:
  _version → 문서 단위 충돌 감지
  _seq_no + _primary_term → 분산 환경 낙관적 잠금
  if_seq_no + if_primary_term → 안전한 동시성 제어

NRT의 "Near" 이유:
  인덱싱 → 인메모리 버퍼 → refresh → 세그먼트 (검색 가능)
  refresh 기본 주기: 1초
  → 인덱싱 후 최대 1초 지연
```

---

## 🤔 생각해볼 문제

**Q1.** `refresh_interval: -1`로 설정하면 인덱싱한 문서가 왜 검색이 안 되는가?

<details>
<summary>해설 보기</summary>

`refresh_interval: -1`은 자동 refresh를 비활성화한다. 인덱싱된 문서는 인메모리 버퍼에만 존재하고, 세그먼트로 변환되지 않아 Lucene의 역색인으로 탐색할 수 없다. `POST /index/_refresh`를 명시적으로 호출하거나 flush가 발생해야 세그먼트가 생성되고 검색이 가능해진다. 초기 대량 적재(Bulk Indexing) 시 refresh를 비활성화해 쓰기 성능을 높이고, 완료 후 한 번 refresh하는 패턴이 권장되는 이유다.

</details>

**Q2.** Primary가 Replica에 복제하기 전에 클라이언트에 응답하는 것이 가능한가?

<details>
<summary>해설 보기</summary>

`wait_for_active_shards: 1`(기본값)이면 Primary 로컬 인덱싱 완료만으로 클라이언트에 응답한다. Replica 복제는 비동기로 계속 진행된다. 응답 직후 Primary 노드가 죽고 Replica에 복제가 완료되지 않았다면 해당 문서는 손실될 수 있다. `wait_for_active_shards: 2`로 설정하면 Replica 복제 확인 후 응답하므로 이 손실을 방지할 수 있다.

</details>

**Q3.** 동일 문서 ID로 UPDATE할 때 기존 문서는 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

Lucene 세그먼트는 불변(Immutable)이므로 기존 문서를 직접 수정할 수 없다. 대신 기존 문서에 삭제 마킹을 하고, 새 문서를 새 세그먼트(또는 인메모리 버퍼)에 추가한다. 따라서 UPDATE는 내부적으로 DELETE(마킹) + INSERT로 처리된다. 삭제 마킹된 문서는 세그먼트 병합(Merge) 시 실제로 제거되고, 그때 비로소 디스크 공간이 반환된다. 빈번한 업데이트가 많은 경우 삭제 마킹 문서가 누적되어 세그먼트 크기와 병합 부하가 증가한다.

</details>

---

<div align="center">

**[⬅️ 이전: 샤드와 레플리카](./03-shard-replica.md)** | **[홈으로 🏠](../README.md)** | **[다음: 읽기 경로 — Scatter-Gather ➡️](./05-read-path-scatter-gather.md)**

</div>
