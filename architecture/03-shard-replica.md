# 샤드와 레플리카 — Primary/Replica 역할과 라우팅 수식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Primary Shard와 Replica Shard의 역할은 어떻게 다른가?
- 문서가 어느 샤드에 저장되는지 결정하는 라우팅 수식은 무엇인가?
- 샤드 수를 인덱스 생성 후에 변경할 수 없는 이유는 무엇인가?
- `_shrink`, `_split`, `_clone` API는 어떻게 이 제약을 우회하는가?
- 레플리카 수는 운영 중에도 변경 가능한데, Primary는 왜 안 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"데이터가 늘어서 샤드를 추가하면 되지 않나요?"라는 질문은 ES를 처음 운영하는 사람에게서 자주 나온다. 답은 "불가"다. 그 이유를 모르면 처음 인덱스 설계에서 실수하고, 나중에 Reindex라는 비용이 큰 작업을 피할 수 없다. 라우팅 수식이 샤드 수에 의존한다는 사실을 이해하면, 왜 이 제약이 존재하는지 자연스럽게 납득된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 샤드 수를 1로 시작
  상황: "일단 1개로 시작하고 나중에 늘리자"

  결과:
    데이터가 100GB로 늘어남
    단일 샤드 → 단일 노드에 부하 집중
    샤드 수 변경 불가 → Reindex 필요
    100GB Reindex = 수 시간, 서비스 중단 또는 이중 운영

실수 2: 샤드 수를 지나치게 많이 설정
  상황: "나중을 위해 샤드 1,000개로 시작하자"

  결과:
    데이터 1GB에 샤드 1,000개
    샤드당 1MB → Lucene 오버헤드가 데이터보다 큼
    클러스터 상태에 1,000개 샤드 정보 → 상태 비대화
    검색 시 1,000개 샤드에 병렬 요청 → Coordinating 노드 부하 급증
    → "Too many shards" 문제 (ES 권장 상한: 노드당 약 20~25개 샤드)

실수 3: 레플리카도 변경 불가라고 오해
  상황: 레플리카 수를 줄이거나 늘리는 것도 안 된다고 생각

  실제:
    레플리카 수는 언제든지 변경 가능
    PUT /myindex/_settings { "number_of_replicas": 2 }
    → 즉시 새 Replica 샤드 생성 시작
    Primary 수만 고정, Replica 수는 동적으로 조정 가능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
올바른 샤드 수 설계:

  목표: 샤드 하나당 10~50GB

  계산 예시:
    현재 데이터: 50GB, 예상 1년 후 데이터: 200GB
    여유 계수 적용: 200GB × 1.3 = 260GB
    목표 샤드 크기: 30GB
    Primary 샤드 수: 260GB / 30GB ≈ 9개 → 올림하여 10개 설정

  샤드 증가 없이 용량 늘리는 방법:
    시계열 데이터 → 인덱스 롤오버 (날짜별 인덱스 분리)
    logs-2024-01, logs-2024-02, ... (각각 독립 샤드 설계)
    → 전체 데이터 증가해도 각 인덱스의 샤드 수는 그대로

  불가피한 경우: _split API 활용
    기존 샤드 수의 배수로만 분할 가능
    Shard 3개 → 6개, 9개 (불가: 5개)
    데이터 재배치 없이 논리적 분할 (빠름)
    단, _split은 완벽한 해결책이 아님 (모든 새 샤드가 1개 기존 샤드에서 파생)
```

---

## 🔬 내부 동작 원리

### 1. 라우팅 수식 — 문서가 어느 샤드로 가는가

```
문서 라우팅 수식:

  shard_num = hash(routing) % number_of_primary_shards

  routing 기본값: 문서의 _id
  number_of_primary_shards: 인덱스 생성 시 고정된 값

예시:
  인덱스: products (Primary Shard 3개)
  문서 ID: "abc123"

  hash("abc123") = 1,847,293,041  (murmur3 해시)
  1,847,293,041 % 3 = 0
  → Shard 0에 저장

  문서 ID: "xyz789"
  hash("xyz789") = 2,304,857,122
  2,304,857,122 % 3 = 2
  → Shard 2에 저장

왜 Primary 샤드 수가 고정이어야 하는가:

  현재 상태:
    Primary Shard 3개
    hash("abc123") % 3 = 0 → Shard 0

  만약 Primary Shard 5개로 늘린다면:
    hash("abc123") % 5 = ?  (전혀 다른 값)
    → Shard 0에 있는 문서를 Shard 0으로 찾지 못함
    → 기존 모든 문서의 위치가 틀어짐
    → 전체 Reindex 없이는 문서를 찾을 수 없음

  결론:
    라우팅 수식이 number_of_primary_shards에 의존
    → Primary 샤드 수 변경 = 전체 데이터 재라우팅 필요
    → ES는 이를 Reindex로만 허용
```

### 2. Primary vs Replica 역할 분리

```
Primary Shard:
  ┌─────────────────────────────────────────────────────┐
  │                  Primary Shard                      │
  │                                                     │
  │  쓰기 진입점:                                          │
  │    모든 인덱싱 요청이 먼저 Primary에 도달                  │
  │    Primary가 유효성 검증 + 로컬 인덱싱 수행                │
  │    → Replica에 복제 명령 (병렬)                         │
  │    → 모든 Replica 확인 후 클라이언트 응답                  │
  │                                                     │
  │  복제 오케스트레이터:                                    │
  │    Replica가 실패하면 클러스터에 보고                      │
  │    클러스터가 해당 Replica를 In-sync set에서 제거          │
  │                                                     │
  │  장애 복구 기준:                                       │
  │    Primary 장애 시 → Replica 중 하나가 Primary로 승격    │
  │    승격 기준: in-sync replica set 내에서 선택            │
  └─────────────────────────────────────────────────────┘

Replica Shard:
  ┌─────────────────────────────────────────────────────┐
  │                  Replica Shard                      │
  │                                                     │
  │  읽기 분산:                                           │
  │    검색 요청 시 Primary/Replica 중 선택 (부하 분산)        │
  │    → 레플리카가 많을수록 검색 처리량 수평 확장                │
  │                                                     │
  │  장애 대비:                                           │
  │    Primary 장애 시 자동으로 Primary 승격                 │
  │    데이터 손실 없이 서비스 계속                            │
  │                                                     │
  │  제약:                                               │
  │    항상 Primary와 다른 노드에 배치 (같은 노드 = 의미 없음)    │
  │    → 가용 노드 < 1 + replica 수이면 일부 Replica 미할당    │
  └─────────────────────────────────────────────────────┘

In-sync Replica Set (IRS):
  Primary와 동기화된 Replica 목록
  복제 확인이 완료된 Replica만 포함
  Primary 장애 시 IRS 내에서만 새 Primary 선출
  → IRS 밖의 Replica는 최신 데이터 없음 → Primary 승격 불가
```

### 3. 커스텀 라우팅 — 샤드를 직접 지정하는 방법

```
기본 라우팅 (ID 기반):
  PUT /products/_doc/1
  { "title": "keyboard", "category": "electronics" }
  → hash("1") % 3 = 어떤 샤드로든 갈 수 있음

커스텀 라우팅:
  PUT /products/_doc/1?routing=electronics
  { "title": "keyboard", "category": "electronics" }
  → hash("electronics") % 3 = 항상 같은 샤드

  GET /products/_search?routing=electronics
  { "query": { "term": { "category": "electronics" } } }
  → electronics 라우팅 키의 샤드 하나만 탐색
  → 모든 샤드 Scatter 없이 단일 샤드 탐색 가능!

사용 사례:
  테넌트(고객) ID로 라우팅
    → 특정 고객 데이터가 항상 같은 샤드에 저장
    → 고객별 검색 시 단일 샤드만 탐색 → 빠름

주의:
  커스텀 라우팅 키 분포가 불균일하면 샤드 불균형 발생
  특정 샤드에만 데이터 집중 (Hot Shard 문제)
  → 라우팅 키 카디널리티가 충분히 높아야 함
```

### 4. _split, _shrink, _clone — Primary 수 변경 우회

```
_split API (샤드 수 증가):
  조건: 기존 샤드 수의 배수만 가능
  조건: 인덱스가 read-only 상태여야 함

  예시: 3샤드 → 6샤드
  PUT /products/_split/products-v2
  {
    "settings": {
      "index.number_of_shards": 6
    }
  }

  내부 동작:
    기존 Shard 0 → 새 Shard 0, Shard 3 (파생)
    기존 Shard 1 → 새 Shard 1, Shard 4
    기존 Shard 2 → 새 Shard 2, Shard 5
    하드 링크(hard link)로 파일 복사 없이 빠른 분할
    단, 모든 새 샤드의 데이터는 원래 샤드에서 파생
    → 진정한 재분산은 아님, 추후 문서 추가부터 새 라우팅 적용

_shrink API (샤드 수 감소):
  조건: 현재 샤드 수의 약수만 가능
  조건: 모든 Primary가 같은 노드에 있어야 함 (이동 후 작업)
  6샤드 → 3샤드, 2샤드, 1샤드 가능

_clone API (동일 샤드 수로 복사):
  샤드 수 유지, 인덱스 자체 복사
  설정 변경 후 실험용도로 활용

공통 한계:
  세 API 모두 Reindex만큼 유연하지 않음
  근본적인 샤드 재설계는 Reindex가 유일한 선택
```

---

## 💻 실전 실험

```bash
# 문서가 어느 샤드에 저장됐는지 확인
GET /products/_search_shards?routing=abc123

# 특정 문서의 라우팅 정보 확인
GET /products/_doc/1?routing=electronics

# 샤드별 문서 수 확인 (불균형 감지)
GET _cat/shards/products?v&s=shard&h=index,shard,prirep,state,docs,store,node

# 레플리카 수 동적 변경 (Primary는 불가, Replica는 가능)
PUT /products/_settings
{
  "number_of_replicas": 2
}

# _split 전 인덱스를 read-only로 변경
PUT /products/_settings
{
  "index.blocks.write": true
}

# 샤드 3개 → 6개로 split
PUT /products/_split/products-v2
{
  "settings": {
    "index.number_of_shards": 6,
    "index.blocks.write": null
  }
}

# 라우팅 키로 특정 샤드만 검색
GET /products/_search?routing=electronics
{
  "query": {
    "term": { "category": "electronics" }
  }
}

# 샤드 배치 현황 (Primary P, Replica R 구분)
GET _cat/shards?v&h=index,shard,prirep,state,docs,store,ip,node
```

---

## 📊 샤드 수에 따른 성능 트레이드오프

| 샤드 수 | 장점 | 단점 |
|---------|------|------|
| 너무 적음 (1~2개) | 관리 단순, Coordinating 부하 낮음 | 병렬 검색 없음, 용량 확장 불가 |
| 적절함 (5~10개) | 검색 병렬화, 적절한 용량 | 균형 잡힌 트레이드오프 |
| 너무 많음 (100개+) | 매우 세밀한 병렬화 | Coordinating 부담, 클러스터 상태 비대, 세그먼트 오버헤드 |

**Elastic 공식 권장**: 샤드 하나당 10~50GB, 노드당 샤드 수 20~25개 이하

---

## ⚖️ 트레이드오프

```
레플리카 많을수록:
  이득: 읽기 처리량 증가, 노드 장애 내성 향상
  비용: 저장 공간 (데이터 × (1 + replica 수))
        쓰기 비용 증가 (Primary + 모든 Replica에 복제)
        노드 수가 충분해야 Replica 배치 가능

레플리카 없을 경우 (number_of_replicas: 0):
  이득: 쓰기 성능 극대화, 저장 공간 절약
  사용: 대규모 초기 데이터 적재(Bulk Indexing) 시 임시 설정
  주의: 노드 1개 장애 → 해당 샤드 데이터 유실

커스텀 라우팅의 트레이드오프:
  이득: 특정 검색의 샤드 탐색 범위 축소 → 빠름
  비용: 라우팅 키 불균형 → Hot Shard → 성능 저하
        커스텀 라우팅 없는 검색 시 모든 샤드 탐색 여전히 필요
```

---

## 📌 핵심 정리

```
라우팅 수식:
  shard_num = hash(_id 또는 routing 키) % number_of_primary_shards
  → Primary 샤드 수가 바뀌면 모든 기존 문서를 찾을 수 없게 됨
  → Primary 샤드 수는 인덱스 생성 시 영구 고정

Primary vs Replica:
  Primary: 쓰기 진입점, 복제 오케스트레이터, IRS 관리
  Replica: 읽기 분산, 장애 시 Primary 승격 대기
  Replica 수: 운영 중 동적 변경 가능 (Primary와 다름)

샤드 수 설계 원칙:
  샤드 하나당 10~50GB 목표
  너무 적으면 용량·병렬성 제한
  너무 많으면 Coordinating 부하·클러스터 상태 비대
  시계열 데이터 → 인덱스 롤오버로 샤드 수 분산

불가피한 변경: _split(배수 증가), _shrink(약수 감소), Reindex(완전 재설계)
```

---

## 🤔 생각해볼 문제

**Q1.** `number_of_replicas: 1`로 설정한 인덱스에서 노드가 2개뿐이라면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

정상 동작한다. 2개 노드에 Primary와 Replica가 각각 다른 노드에 배치된다. 단, 세 번째 노드 없이 `number_of_replicas: 2`로 늘리면 두 번째 Replica를 배치할 노드가 없어 Unassigned 상태가 되고 클러스터는 Yellow가 된다. Primary와 Replica는 항상 서로 다른 노드에 있어야 하기 때문이다.

</details>

**Q2.** Primary Shard가 장애 시 Replica가 자동으로 Primary로 승격되는데, 데이터 손실은 없는가?

<details>
<summary>해설 보기</summary>

In-sync Replica Set(IRS)에 포함된 Replica만 승격 대상이 된다. IRS에 포함되려면 Primary의 모든 쓰기를 확인받아야 한다. `wait_for_active_shards` 설정으로 복제 확인 수를 제어한다. 기본값(1)은 Primary만 확인하면 응답하는 것으로, Replica에 복제되기 전 Primary 장애 시 최대 1개 문서 손실 가능성이 있다. `wait_for_active_shards: all`로 설정하면 손실 없지만 쓰기 지연이 발생한다.

</details>

**Q3.** 커스텀 라우팅 키를 사용할 때 반드시 검색 요청에도 같은 라우팅 키를 지정해야 하는 이유는?

<details>
<summary>해설 보기</summary>

커스텀 라우팅으로 인덱싱하면 문서가 `hash(routing_key) % shard_count`에 해당하는 샤드에만 저장된다. 검색 시 라우팅 키를 지정하지 않으면 Coordinating 노드가 모든 샤드에 요청을 보내 해당 샤드에서 문서를 찾긴 하지만, 라우팅 키 없는 _search는 커스텀 라우팅의 성능 이점을 전혀 활용하지 못한다. 또한, 라우팅 키로 인덱싱한 문서를 라우팅 키 없이 GET으로 조회하면 ES가 기본 라우팅(ID 기반)으로 샤드를 찾아 문서를 찾지 못할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: 클러스터 상태와 Master 노드](./02-cluster-state-master.md)** | **[홈으로 🏠](../README.md)** | **[다음: 쓰기 경로 ➡️](./04-write-path.md)**

</div>
