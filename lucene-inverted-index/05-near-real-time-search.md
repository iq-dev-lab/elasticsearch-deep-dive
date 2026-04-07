# Near Real-Time 검색 — refresh·flush·translog 역할

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 인덱싱한 문서가 즉시 검색되지 않는 이유는 무엇인가?
- refresh, flush, fsync는 각각 무엇을 하고 어떤 순서로 일어나는가?
- OS Page Cache에만 있어도 검색이 가능한 이유는 무엇인가?
- translog는 어떤 장애로부터 데이터를 보호하는가?
- `linux-for-backend-deep-dive`에서 배운 Page Cache·mmap과 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"방금 저장했는데 검색이 안 돼요", "데이터 손실이 걱정돼요", "refresh를 자주 하면 성능이 나빠지나요?" — 이 질문들의 답이 모두 refresh/flush/translog 흐름에 있다. NRT의 "Near"가 의미하는 1초의 지연이 설계 결정인지 버그인지 구분하지 못하면 실무에서 불필요한 트러블슈팅을 반복하게 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 인덱싱 후 즉시 검색 결과를 기대
  코드:
    POST /products/_doc { "title": "New Product" }
    GET /products/_search { "query": { "match": { "title": "New" } } }
    // 결과 없음 → "버그?" → refresh_interval을 100ms로 줄임

  문제:
    refresh_interval 줄일수록 세그먼트 생성 빈도 증가
    작은 세그먼트 많아짐 → 병합 부하 증가 → 오히려 느려짐

  올바른 접근:
    NRT 설계를 수용: 1초 지연을 프로덕트 설계에 반영
    테스트에서만 POST /index/_refresh 명시적 호출

실수 2: refresh와 flush를 같은 것으로 혼동
  오해:
    "refresh_interval을 높이면 내구성이 낮아지나요?"

  실제:
    refresh: 검색 가능 여부와 관련 (OS Page Cache → 세그먼트 공개)
    flush:   내구성과 관련 (translog → 디스크 fsync)
    두 개는 독립적으로 동작
    refresh_interval을 1시간으로 설정해도 flush는 30초마다 발생 가능

실수 3: translog 없이 내구성을 논함
  상황:
    "ES는 쓰기 후 즉시 디스크에 저장하지 않으니 데이터 손실 위험이 있다"

  실제:
    세그먼트는 flush 전까지 OS Page Cache에만 있음 → 맞음
    하지만 translog가 모든 쓰기를 즉시 디스크에 기록
    노드 재시작 시 translog replay → 데이터 복구
    → 세그먼트가 디스크에 없어도 translog로 복구 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
NRT 특성을 활용한 설계:

  검색 지연 허용 (일반 서비스):
    refresh_interval: 1s (기본값)
    → 인덱싱 후 최대 1초 뒤 검색 가능
    → 대부분의 서비스에서 충분

  검색 지연 최소화 (실시간에 가깝게):
    refresh_interval: 500ms (또는 더 작게)
    주의: 세그먼트 생성 빈도 증가 → 병합 부하 증가
    → 검색 지연 vs 쓰기 성능 트레이드오프

  쓰기 성능 최대화 (대량 적재):
    refresh_interval: -1 (비활성화)
    → 수동 refresh만 발생
    → 큰 세그먼트로 누적 후 한 번에 세그먼트 생성
    → 적재 완료 후 refresh_interval: 1s 복원

  내구성 설정:
    index.translog.durability: request (기본)
    → 매 쓰기마다 translog fsync → 손실 없음

    index.translog.durability: async
    → 주기적 fsync → 최대 sync_interval 손실 가능
    → 손실 허용 가능한 로그성 데이터에 적합
```

---

## 🔬 내부 동작 원리

### 1. 인덱싱부터 검색 가능까지 3단계 흐름

```
문서 인덱싱 전체 흐름:

  POST /products/_doc { "title": "Mechanical Keyboard" }

  ┌─────────────────────────────────────────────────────────┐
  │                 Stage 1: 인메모리 버퍼                     │
  │                                                         │
  │  인덱싱 요청 수신                                           │
  │  → 인메모리 버퍼에 문서 추가 (즉시)                            │
  │  → translog에 기록 (즉시, 내구성 보장)                       │
  │                                                         │
  │  이 상태: 검색 불가 (역색인 없음)                              │
  │           데이터 손실 보호됨 (translog 있음)                  │
  └────────────────────────┬────────────────────────────────┘
                           │
                  refresh (기본 1초마다)
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────┐
  │                Stage 2: OS Page Cache                   │
  │                                                         │
  │  인메모리 버퍼 → 새 Lucene 세그먼트 생성                       │
  │  세그먼트 파일을 OS 파일 시스템에 쓰기                          │
  │  (디스크 fsync 없음 → OS Page Cache에만 존재)                │
  │                                                         │
  │  이 상태: 검색 가능! (역색인 완성, 파일 시스템에 존재)             │
  │           OS 재시작 시 데이터 손실 가능                       │
  │           (Page Cache는 휘발성)                           │
  │           translog가 여전히 복구 수단                       │
  └────────────────────────┬────────────────────────────────┘
                           │
                  flush (translog 크기 초과 또는 명시적 호출)
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────┐
  │                  Stage 3: 디스크 (fsync)                  │
  │                                                         │
  │  세그먼트 파일을 디스크에 fsync (물리 쓰기 완료)                  │
  │  translog checkpoint 갱신 (여기까지 flush됨 기록)            │
  │  flush된 세그먼트에 해당하는 translog 항목 삭제                 │
  │                                                         │
  │  이 상태: 완전한 내구성 보장                                  │
  │           노드 재시작해도 데이터 손실 없음                      │
  └─────────────────────────────────────────────────────────┘
```

### 2. OS Page Cache와 검색 — "Near"가 붙는 이유

```
OS Page Cache (linux-for-backend-deep-dive 연결):

  파일 시스템에 쓰기 = OS 버퍼(Page Cache)에 먼저 저장
  실제 디스크 쓰기는 OS가 비동기로 처리 (writeback)
  fsync() 시스템 콜 = "지금 당장 디스크에 써라" 명령

  Lucene 세그먼트가 Page Cache에만 있을 때 검색이 가능한 이유:

    mmap (memory-mapped file):
      파일을 마치 메모리처럼 접근하는 기법
      Lucene은 세그먼트 파일을 mmap으로 접근
      OS Page Cache에 있는 파일 → 메모리 접근과 동일한 속도

    ┌─────────────────────────────────────────────────────┐
    │  Lucene 읽기 경로:                                    │
    │                                                     │
    │  세그먼트 파일 → mmap → 가상 메모리 주소                   │
    │  FST 탐색 = 가상 메모리 주소 접근                         │
    │                                                     │
    │  Page Cache HIT:  메모리 접근 속도 (~100ns)             │
    │  Page Cache MISS: 디스크 읽기 (~100μs) + 캐시 적재       │
    │                                                     │
    │  → 세그먼트가 Page Cache에 있으면 디스크 I/O 없이 검색       │
    └─────────────────────────────────────────────────────┘

  "Near Real-Time"의 "Near":
    인덱싱 → 즉시 검색 가능하지 않음 (버퍼에 있음)
    refresh 후 → 검색 가능 (Page Cache에 세그먼트 존재)
    refresh 기본값: 1초
    → 최대 1초의 지연 = "Near"
    → 완전한 "Real-Time"(즉시)가 아님
```

### 3. Translog — 두 flush 사이의 안전망

```
Translog의 역할:

  상황: refresh는 됐지만 flush는 아직 (세그먼트가 Page Cache에만 있음)
        이 시점에 노드 재시작 → Page Cache 휘발 → 세그먼트 손실!

  Translog가 해결:
    모든 인덱싱/삭제 작업을 순서대로 translog에 기록
    translog는 즉시 디스크에 fsync (durability: request 기본)
    → 세그먼트가 Page Cache에만 있어도 translog에는 영구 기록됨

  노드 재시작 시 복구:
    마지막 flush checkpoint 확인
    flush 이후의 translog 읽기
    translog 항목을 순서대로 replay
    → 데이터 완전 복구

  ┌─────────────────────────────────────────────────────┐
  │  translog 내용 예시:                                  │
  │                                                     │
  │  [op_1] INDEX doc_id=42, {"title": "keyboard"}      │
  │  [op_2] INDEX doc_id=43, {"title": "mouse"}         │
  │  [op_3] DELETE doc_id=10                            │
  │  [op_4] INDEX doc_id=44, {"title": "monitor"}       │
  │  ...                                                │
  │                                                     │
  │  flush 발생:                                         │
  │    세그먼트 → 디스크 fsync                              │
  │    translog checkpoint = op_4 기록                   │
  │    op_1 ~ op_4 translog 항목 삭제 (이미 세그먼트에 반영)   │
  └─────────────────────────────────────────────────────┘

Translog 크기:
  flush 전까지 translog에 누적
  기본 flush 트리거:
    translog 크기 > 512MB
    또는 30초마다 (index.translog.flush_threshold_period)

  translog 크기와 복구 시간 관계:
    translog 크기 크면 → 복구 시 replay 시간 증가
    → 512MB 기본값이 복구 시간과 쓰기 성능의 균형
```

### 4. refresh와 flush의 독립적 동작

```
refresh와 flush는 독립적:

  타임라인 예시 (기본 설정):

  T=0s   인덱싱 → 버퍼 + translog
  T=1s   refresh → 세그먼트 생성 (Page Cache), 검색 가능
  T=2s   인덱싱 → 버퍼 + translog
  T=3s   refresh → 세그먼트 생성
  ...
  T=30s  flush  → 세그먼트 fsync, translog 초기화

  refresh는 검색 가능 여부를 결정
  flush는 내구성을 결정
  둘은 서로 기다리지 않음

명시적 호출:
  POST /myindex/_refresh
    → 즉시 세그먼트 생성 (검색 가능)
    → 테스트 코드에서 자주 사용

  POST /myindex/_flush
    → 즉시 세그먼트 fsync + translog 초기화
    → 명시적 내구성 보장 필요 시

  POST /myindex/_flush/synced  (ES 7.6 이전)
    → 모든 샤드를 동시에 flush
    → 재시작 속도 향상 목적 (ES 7.6+ 자동 처리)
```

### 5. 세그먼트 생성 빈도와 성능

```
refresh_interval에 따른 세그먼트 생성 패턴:

  refresh_interval: 1s (기본)
    1초마다 세그먼트 생성
    1분 = 60개의 작은 세그먼트
    TieredMergePolicy가 자동으로 병합

  refresh_interval: 100ms (빠른 검색)
    0.1초마다 세그먼트 생성
    1분 = 600개의 매우 작은 세그먼트
    병합 부하 급증 → CPU/I/O 증가
    → 검색 지연 감소 but 쓰기 성능 저하

  refresh_interval: -1 (대량 적재)
    세그먼트 생성 없음
    인메모리 버퍼에 누적
    → 버퍼가 가득 차면 자동으로 세그먼트 생성 (인덱스 버퍼 기반)
    → 큰 세그먼트 하나로 생성 → 병합 부하 최소

세그먼트 크기와 검색 성능:
  작은 세그먼트 많음:
    각 세그먼트마다 탐색 → 탐색 수 많음
    Posting List merge 비용 증가
  큰 세그먼트 하나:
    단일 탐색 → 최적 검색 성능

  결론: 대량 적재 후 refresh_interval 복원 + 한 번의 큰 세그먼트가
        검색 성능 최적 상태에 가까움
```

---

## 💻 실전 실험

```bash
# refresh_interval 설정 변경
PUT /search-test/_settings
{
  "index.refresh_interval": "5s"
}

# 문서 인덱싱 후 바로 검색 (refresh 전)
POST /search-test/_doc
{ "title": "NRT Test Document" }

# refresh 전 → 검색 안 됨
GET /search-test/_search
{ "query": { "match": { "title": "NRT" } } }

# 명시적 refresh 후 검색
POST /search-test/_refresh

# refresh 후 → 검색 됨
GET /search-test/_search
{ "query": { "match": { "title": "NRT" } } }

# translog 현황 확인
GET /search-test/_stats/translog?pretty
# translog.operations: 현재 translog 항목 수
# translog.size_in_bytes: translog 크기

# translog 설정 확인
GET /search-test/_settings?include_defaults=true
# index.translog.durability: request/async
# index.translog.flush_threshold_size: 기본 512mb
# index.translog.sync_interval: async 모드의 fsync 주기

# 명시적 flush
POST /search-test/_flush

# flush 후 translog 확인 (operations이 줄어야 함)
GET /search-test/_stats/translog?pretty

# 대량 적재 최적 설정
PUT /search-test/_settings
{
  "index.refresh_interval": "-1",
  "index.number_of_replicas": 0,
  "index.translog.durability": "async"
}

# 대량 적재 후 복원
PUT /search-test/_settings
{
  "index.refresh_interval": "1s",
  "index.number_of_replicas": 1,
  "index.translog.durability": "request"
}
POST /search-test/_refresh
POST /search-test/_flush
```

---

## 📊 refresh vs flush vs fsync 비교

| 동작 | 트리거 | 결과 | 내구성 | 비용 |
|------|-------|------|-------|------|
| `refresh` | 1초마다 (기본) | 세그먼트 생성 → 검색 가능 | 없음 (Page Cache) | 낮음 |
| `flush` | translog 512MB 또는 30초 | 세그먼트 fsync + translog 초기화 | 보장 | 중간 |
| `fsync` | flush 내부 | 실제 디스크 기록 완료 | 완전 보장 | 높음 |
| translog write | 매 인덱싱 | 작업 로그 기록 | 즉시 보장 | 낮음 |

---

## ⚖️ 트레이드오프

```
refresh_interval 짧게:
  이득: 검색 지연 감소
  비용: 세그먼트 생성 빈도 증가 → 병합 부하 → 쓰기 성능 저하

refresh_interval 길게 (또는 -1):
  이득: 큰 세그먼트 → 병합 효율, 쓰기 성능 향상
  비용: 검색 지연 증가 (최대 refresh_interval만큼)

translog durability: request vs async:
  request:
    이득: 매 쓰기마다 fsync → 최대 내구성 (손실 없음)
    비용: 매 요청마다 디스크 쓰기 → 쓰기 성능 저하

  async:
    이득: 주기적 fsync → 높은 쓰기 성능
    비용: 장애 시 최대 sync_interval(기본 5s) 데이터 손실 가능

flush 빈도:
  잦은 flush → 세그먼트 빠른 내구성, translog 빠른 초기화
            → 쓰기 I/O 증가
  드문 flush → 쓰기 성능 향상, 재시작 시 replay 시간 증가
```

---

## 📌 핵심 정리

```
3단계 흐름:
  인덱싱 → 인메모리 버퍼 + translog (검색 불가, 내구성 있음)
  refresh → OS Page Cache 세그먼트 (검색 가능, 내구성 아직 없음)
  flush   → 디스크 fsync (검색 가능, 내구성 완전 보장)

"Near Real-Time"의 의미:
  refresh(기본 1초) 후에만 검색 가능
  완전한 실시간(즉시)이 아닌 최대 1초 지연

translog의 역할:
  flush 이전에 노드 재시작해도 데이터 복구
  매 쓰기마다 기록 (durability: request 기본)
  flush 후 해당 항목 삭제 (크기 초기화)

OS Page Cache 연결:
  세그먼트 파일 = mmap으로 접근
  Page Cache HIT → 디스크 I/O 없이 검색
  힙 50% 제한의 이유 → 나머지 50%를 Page Cache로 활용

설정 기준:
  쓰기 성능 최대화 → refresh_interval: -1, durability: async
  내구성 최대화   → refresh_interval: 1s, durability: request
```

---

## 🤔 생각해볼 문제

**Q1.** refresh 없이 검색이 가능한 `?refresh=true` 옵션은 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

인덱싱 요청에 `?refresh=true`를 추가하면 해당 샤드에 대해 즉시 refresh를 수행하고 응답한다. 즉, 인덱싱 완료 후 refresh까지 기다렸다가 응답하는 동기 방식이다. 단, refresh는 샤드 수준에서 발생하므로 해당 인덱스의 모든 최근 변경이 검색 가능해진다. `?refresh=wait_for`는 다음 자동 refresh까지 기다리는 방식으로 refresh를 직접 트리거하지 않아 부하가 적다. 두 옵션 모두 자주 사용하면 refresh 부하가 증가하므로 테스트 외에는 피하는 것이 좋다.

</details>

**Q2.** 여러 노드가 동시에 재시작될 때 translog replay 순서는 어떻게 보장되는가?

<details>
<summary>해설 보기</summary>

각 샤드는 독립적인 translog를 가진다. Primary Shard의 translog는 해당 샤드에 속한 문서의 모든 변경 이력을 순서대로 담고 있다. 재시작 시 각 노드는 자신이 보유한 샤드의 translog를 독립적으로 replay한다. Primary와 Replica 간 동기화는 IRS(In-sync Replica Set) 메커니즘으로 보장되며, Replica가 Primary보다 뒤처졌으면 Primary로부터 누락된 작업을 다시 받아 적용한다.

</details>

**Q3.** `index.refresh_interval: -1`로 설정했을 때 언제 세그먼트가 생성되는가?

<details>
<summary>해설 보기</summary>

자동 refresh가 비활성화된 상태에서도 세그먼트는 두 가지 경우에 생성된다. 첫째, 인덱스 버퍼가 가득 찰 때다. `indices.memory.index_buffer_size` 설정(기본 10%, 힙의 10%)에 해당하는 메모리가 차면 자동으로 세그먼트를 생성한다. 둘째, 명시적으로 `POST /index/_refresh`를 호출할 때다. 대량 적재가 끝나면 수동으로 refresh를 호출해서 모든 적재 문서를 한 번에 검색 가능 상태로 만드는 것이 권장 패턴이다.

</details>

---

<div align="center">

**[⬅️ 이전: Lucene Segment — 불변 세그먼트와 병합 정책](./04-lucene-segment.md)** | **[홈으로 🏠](../README.md)** | **[다음: doc_values vs fielddata ➡️](./06-doc-values-vs-fielddata.md)**

</div>
