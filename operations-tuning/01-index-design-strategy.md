# 인덱스 설계 전략 — 샤드 크기·롤오버·ILM 핫-웜-콜드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 샤드당 10~50GB 가이드라인은 어디서 나오는 수치이고 왜 그 범위인가?
- 시계열 인덱스에서 인덱스 롤오버(Rollover)를 사용하는 이유는 무엇인가?
- ILM의 Hot → Warm → Cold → Delete 단계는 각각 무엇을 의미하는가?
- 인덱스 별칭(alias)과 데이터 스트림(data stream)은 어떻게 다른가?
- 샤드 수 설계 실수를 인덱스 생성 후 수정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

인덱스 설계는 한 번 잘못하면 운영 중 수정이 매우 어렵다. 샤드가 너무 적으면 용량·병렬성 한계, 너무 많으면 오버헤드. 시계열 데이터(로그, 이벤트)를 단일 인덱스에 무한정 쌓으면 샤드 크기가 폭발한다. ILM 없이 데이터 보관 기간을 관리하려면 수동 삭제 스크립트가 필요하다. 처음부터 올바른 전략을 세우면 이 모든 문제를 예방할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 단일 인덱스에 로그를 무한정 쌓기
  설정:
    PUT /application-logs (샤드 5개, 레플리카 1개)
    → 모든 로그를 이 인덱스에 계속 인덱싱

  6개월 후:
    인덱스 크기: 500GB
    샤드당 크기: 100GB (10~50GB 권장의 2~10배)
    검색: 느림 (세그먼트 너무 많음, 병합 부하)
    삭제: 오래된 데이터 삭제 = 부분 삭제 불가
          → DELETE by query (비쌈) 또는 전체 인덱스 삭제만 가능

  올바른 접근:
    날짜별 인덱스: application-logs-2024-01, application-logs-2024-02, ...
    ILM으로 자동 롤오버 + 오래된 인덱스 자동 삭제

실수 2: 샤드 수를 너무 많이 설정
  설정:
    PUT /products (number_of_shards: 50)

  실제 데이터: 2GB
  샤드당: 40MB → Lucene 오버헤드가 데이터보다 큼
  클러스터 상태: 50 Primary + 50 Replica = 100개 샤드 정보
  검색: 50개 샤드에 Scatter → Coordinator 병합 부하
  → "너무 많은 샤드" 문제

실수 3: ILM 없이 수동 데이터 관리
  운영:
    crontab으로 매일 오래된 인덱스 삭제 스크립트
    샤드 크기 모니터링 스크립트 별도 작성

  문제:
    스크립트 실패 시 오래된 데이터 무한 누적
    핫/웜/콜드 구성 없이 고비용 SSD에 모든 데이터
    → ILM이 이 모든 것을 자동화
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
인덱스 설계 체크리스트:

  □ 샤드 크기 목표 설정 (10~50GB)
    예상 1년 데이터 크기 / 목표 샤드 크기 = Primary 샤드 수

  □ 시계열 데이터 → 롤오버 + ILM 설정
    날짜별 자동 인덱스 생성
    크기/날짜 기준 자동 롤오버
    핫-웜-콜드 티어로 비용 최적화

  □ 정적 데이터 (카탈로그, 사용자) → 단일 인덱스
    샤드 수 = 데이터 크기 / 30GB
    ILM 불필요 (Delete 단계 없음)

  □ 인덱스 별칭(alias) 또는 데이터 스트림 사용
    애플리케이션은 별칭을 바라봄
    실제 인덱스는 내부에서 교체 가능

  □ 레플리카 수
    개발: 0 (빠른 인덱싱, 단일 노드)
    프로덕션: 1 이상 (장애 내성)
    읽기 집중: 2~3 (읽기 처리량 확장)
```

---

## 🔬 내부 동작 원리

### 1. 샤드 크기 가이드라인의 근거

```
왜 샤드당 10~50GB를 권장하는가?

너무 작은 샤드 (예: 1GB):
  문제 1: 오버헤드 > 데이터
    각 Lucene 인스턴스마다 메모리 오버헤드 발생
    JVM 객체, 세그먼트 메타데이터, 파일 핸들
    → 샤드 크기가 작을수록 오버헤드 비율 증가

  문제 2: 과도한 샤드 수
    1TB 데이터를 1GB 샤드로 → 1,000개 샤드
    클러스터 상태: 1,000 × (1 Primary + 1 Replica) = 2,000개 샤드 정보
    Master 노드 부하, Coordinator 병합 부하 증가

  문제 3: 비효율적인 세그먼트
    Lucene 세그먼트 병합 정책이 작은 인덱스에서 비효율
    검색 성능 최적화가 어려움

너무 큰 샤드 (예: 200GB):
  문제 1: 리밸런싱 비용
    노드 추가 시 200GB 샤드를 다른 노드로 이동
    → 수십 분 ~ 수 시간의 네트워크 전송
    이동 중 성능 저하

  문제 2: 복구 시간
    노드 장애 후 Primary 복구
    200GB 세그먼트 복사 → 오랜 Yellow/Red 상태

  문제 3: 세그먼트 병합 부하
    단일 샤드에 수억 개 문서 → 세그먼트 병합이 빈번하고 비쌈

10~50GB 권장 근거:
  오버헤드와 데이터 크기의 균형
  노드 간 리밸런싱 시 수 분 이내 이동 가능
  세그먼트 병합 효율적
  GC 부담 적정 수준
  Elastic 공식 가이드 + 수많은 운영 경험 기반
```

### 2. 인덱스 롤오버 — 시계열 데이터 관리

```
롤오버 패턴:
  단일 거대 인덱스 → 시간/크기 기준으로 새 인덱스 자동 생성

  write alias: logs-write (항상 최신 인덱스를 가리킴)
  read alias:  logs-read (모든 인덱스를 포함)

  타임라인:
    2024-01: logs-000001 (write alias → 이 인덱스)
    1월 31일: 롤오버 조건 충족 (크기 or 기간)
    2024-02: logs-000002 생성 (write alias 자동 이동)
    2024-03: logs-000003 생성

  롤오버 조건:
    - 인덱스 크기 초과 (max_primary_shard_size)
    - 문서 수 초과 (max_docs)
    - 기간 초과 (max_age: "30d")

  이점:
    각 인덱스가 적절한 크기 유지
    오래된 인덱스를 통째로 삭제 → DELETE by query 불필요
    핫-웜-콜드 단계별 다른 설정 적용 가능

데이터 스트림 (Data Stream, ES 7.9+):
  롤오버 + 별칭 + 타임스탬프 필드를 통합한 상위 레벨 추상화

  PUT _data_stream/logs-app
  → 자동으로 .ds-logs-app-2024.01-000001 같은 인덱스 생성
  → 롤오버 자동 관리
  → write는 항상 최신 backing index
  → read는 모든 backing index 탐색

  데이터 스트림 vs 별칭:
    데이터 스트림: 타임스탬프 필드 필수, 더 간단한 관리
    별칭:          유연성 높음, 직접 관리 필요
    → 시계열 데이터는 데이터 스트림 권장 (ES 7.9+)
```

### 3. ILM — Index Lifecycle Management

```
ILM 4단계:

  ┌────────────────────────────────────────────────────────────┐
  │                    ILM 생명주기                              │
  │                                                            │
  │  HOT (핫)           WARM (웜)          COLD (콜드)           │
  │  ────────────        ────────           ──────────         │
  │  최신 데이터         조회만              장기 보관                │
  │  빠른 쓰기/읽기      읽기 최적화          저렴한 스토리지            │
  │  SSD               SSD 또는 HDD        HDD 또는 오브젝트       │
  │  레플리카 1+         레플리카 1+          레플리카 0 가능          │
  │  forcemerge 안 함   forcemerge 실행     freeze 가능           │
  │                                                            │
  │  조건: max_age/size  조건: hot 이후      조건: warm 이후        │
  │                                                            │
  │                                              DELETE        │
  │                                              ──────        │
  │                                              인덱스 삭제      │
  │                                              보관 기간 만료   │
  └────────────────────────────────────────────────────────────┘

ILM 정책 설정 예시:
  PUT _ilm/policy/logs-policy
  {
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_primary_shard_size": "40gb",
              "max_age": "7d"
            },
            "set_priority": { "priority": 100 }
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "shrink": { "number_of_shards": 1 },
            "forcemerge": { "max_num_segments": 1 },
            "set_priority": { "priority": 50 }
          }
        },
        "cold": {
          "min_age": "30d",
          "actions": {
            "freeze": {},
            "set_priority": { "priority": 0 }
          }
        },
        "delete": {
          "min_age": "90d",
          "actions": { "delete": {} }
        }
      }
    }
  }

각 단계의 핵심 액션:
  hot:   rollover (새 인덱스로 전환)
  warm:  forcemerge (단일 세그먼트로 압축) + shrink (샤드 수 감소)
  cold:  freeze (메모리에서 내리기, 조회만) 또는 searchable snapshot
  delete: 인덱스 삭제

shrink 액션:
  hot 단계에서 샤드 5개 → warm에서 1개로 축소
  → 오래된 데이터: 쓰기 없음 → 단일 샤드로 검색 충분
  → 클러스터 샤드 수 감소 → Master 노드 부담 감소

forcemerge:
  단일 세그먼트로 병합 → 검색 최적화
  삭제 마킹 문서 실제 제거 → 공간 반환
  I/O 집중 → warm 단계에서 실행 (hot 단계 성능 영향 없도록)
```

### 4. 실전 ILM + 데이터 스트림 구성

```
전체 구성 흐름:

Step 1: 컴포넌트 템플릿 생성
  PUT _component_template/logs-settings
  {
    "template": {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1,
        "index.lifecycle.name": "logs-policy",
        "index.lifecycle.rollover_alias": "logs"
      }
    }
  }

  PUT _component_template/logs-mappings
  {
    "template": {
      "mappings": {
        "properties": {
          "@timestamp": { "type": "date" },
          "level":      { "type": "keyword" },
          "message":    { "type": "text" },
          "service":    { "type": "keyword" }
        }
      }
    }
  }

Step 2: 인덱스 템플릿 생성
  PUT _index_template/logs-template
  {
    "index_patterns": ["logs-*"],
    "data_stream": {},              ← 데이터 스트림 사용 시
    "composed_of": ["logs-settings", "logs-mappings"],
    "priority": 200
  }

Step 3: 데이터 스트림 생성
  PUT _data_stream/logs-app

Step 4: 데이터 인덱싱
  POST logs-app/_doc
  {
    "@timestamp": "2024-01-15T10:30:00",
    "level": "ERROR",
    "message": "Connection timeout",
    "service": "payment-service"
  }

Step 5: ILM 상태 확인
  GET logs-app/_ilm/explain
  → 각 backing index의 현재 ILM 단계 확인
```

---

## 💻 실전 실험

```bash
# ILM 정책 생성
PUT _ilm/policy/demo-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_primary_shard_size": "1gb", "max_age": "1d" }
        }
      },
      "delete": {
        "min_age": "7d",
        "actions": { "delete": {} }
      }
    }
  }
}

# 인덱스 템플릿 생성
PUT _index_template/demo-template
{
  "index_patterns": ["demo-logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "demo-policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" },
        "level":      { "type": "keyword" }
      }
    }
  }
}

# 데이터 스트림 생성
PUT _data_stream/demo-logs-app

# 데이터 인덱싱
POST demo-logs-app/_doc
{
  "@timestamp": "now",
  "level": "INFO",
  "message": "Application started"
}

# 데이터 스트림 상태 확인
GET _data_stream/demo-logs-app

# ILM 상태 확인
GET demo-logs-app/_ilm/explain?pretty

# 수동 롤오버 테스트
POST demo-logs-app/_rollover

# 샤드 크기 확인
GET _cat/shards/demo-logs-app?v&h=index,shard,prirep,state,docs,store,node

# 인덱스 크기별 샤드 확인
GET _cat/indices?v&s=store.size:desc&h=index,health,pri,rep,docs.count,store.size
```

---

## 📊 인덱스 전략별 비교

| 전략 | 적합 데이터 | 장점 | 단점 |
|------|---------|------|------|
| 단일 인덱스 | 카탈로그, 사용자, 소규모 | 단순 관리 | 데이터 증가 시 샤드 크기 폭발 |
| 날짜별 인덱스 | 로그, 이벤트 | 기간별 삭제 용이 | 수동 관리 필요 |
| ILM + 데이터 스트림 | 시계열 모든 데이터 | 자동 관리, 비용 최적화 | 초기 설정 복잡 |
| Rollup/Transform | 집계 결과 저장 | 빠른 집계 조회 | 실시간성 없음 |

---

## ⚖️ 트레이드오프

```
샤드 수 많음:
  이득: 병렬 검색·인덱싱, 노드 간 균등 분산
  비용: 클러스터 상태 비대화, Coordinator 부하, 오버헤드 증가

ILM forcemerge:
  이득: 단일 세그먼트로 검색 최적화
  비용: CPU/I/O 집중 → warm 단계에서 실행 (hot 단계 영향 없도록)

Cold 단계 freeze:
  이득: 메모리에서 내려 비용 절약
  비용: 조회 시 재로딩 필요 → 응답 지연
  → Searchable Snapshot: 오브젝트 스토리지 + 빠른 접근 (ES 8.x)

데이터 스트림 vs 별칭:
  데이터 스트림: 간단, 타임스탬프 필수, 시계열 전용
  별칭: 유연, 직접 관리, 다양한 데이터 패턴
```

---

## 📌 핵심 정리

```
샤드 크기 가이드라인:
  10~50GB per shard
  너무 작음 → 오버헤드 과다, 클러스터 상태 비대
  너무 큼 → 리밸런싱 비용, 복구 시간 증가

시계열 데이터 전략:
  데이터 스트림 + ILM 조합
  자동 롤오버 (크기/기간 기준)
  핫-웜-콜드 티어로 비용 최적화

ILM 주요 액션:
  hot:   rollover (크기/기간 기준 새 인덱스 전환)
  warm:  forcemerge (단일 세그먼트) + shrink (샤드 축소)
  cold:  freeze 또는 searchable snapshot
  delete: 인덱스 삭제

인덱스 별칭/데이터 스트림:
  애플리케이션은 별칭/데이터 스트림을 통해 접근
  실제 인덱스는 내부에서 롤오버/전환
  → 애플리케이션 코드 변경 없이 인덱스 교체 가능
```

---

## 🤔 생각해볼 문제

**Q1.** 기존에 단일 인덱스(`logs`)로 운영하다가 ILM으로 전환하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

새 ILM 정책과 인덱스 템플릿을 먼저 생성한다. 그 후 기존 `logs` 인덱스에 write alias를 추가하거나, 새 인덱스(`logs-000001`)를 생성하고 `logs-write` alias를 부여한다. 애플리케이션이 `logs-write` alias로 쓰도록 변경하면 ILM이 자동으로 롤오버를 관리한다. 기존 `logs` 인덱스는 read alias에 추가해 기존 데이터도 검색 가능하게 유지한다. 이후 ILM Delete 정책에 따라 오래된 인덱스가 자동 삭제된다.

</details>

**Q2.** warm 단계에서 `shrink`로 샤드 수를 줄이면 어떤 이점이 있는가?

<details>
<summary>해설 보기</summary>

hot 단계에서 샤드 5개로 병렬 쓰기 성능을 확보했던 인덱스가 warm으로 넘어가면 더 이상 쓰기가 없다. 이때 샤드 5개를 유지하면 클러스터 상태에 5개 샤드 정보가 남고 Master 노드 부담이 된다. shrink로 1개 샤드로 줄이면 클러스터 상태 크기가 감소하고, 단일 샤드로 검색 시 Scatter 병합 오버헤드도 없어진다. 단, shrink는 모든 Primary가 같은 노드에 있어야 하므로 사전에 샤드 이동이 필요하고, 별도 인덱스로 재생성되는 작업이다.

</details>

**Q3.** 데이터 스트림과 일반 인덱스를 하나의 별칭으로 묶어서 검색할 수 있는가?

<details>
<summary>해설 보기</summary>

가능하다. 데이터 스트림과 일반 인덱스 모두 별칭에 추가할 수 있다. 예를 들어 `all-logs` 별칭에 `logs` 일반 인덱스(과거 데이터)와 `app-logs` 데이터 스트림(신규 데이터)을 모두 추가하면, `all-logs`로 검색 시 두 곳 모두 탐색한다. 다만 쓰기는 별칭이 지정한 단일 쓰기 대상으로만 가능하고, 데이터 스트림의 쓰기는 항상 최신 backing index로 향한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 클러스터 성능 진단 ➡️](./02-cluster-performance-diagnosis.md)**

</div>
