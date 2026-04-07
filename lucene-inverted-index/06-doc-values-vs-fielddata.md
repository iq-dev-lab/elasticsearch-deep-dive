# doc_values vs fielddata — 정렬·집계를 위한 컬럼 지향 저장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 역색인으로 검색은 되는데 왜 정렬과 집계에는 다른 자료구조가 필요한가?
- doc_values는 어떻게 컬럼 지향으로 저장되고 mmap으로 접근하는가?
- fielddata가 힙에 올라와 OOM을 일으키는 구체적인 경로는 무엇인가?
- text 필드에 doc_values가 기본 비활성화된 이유는 무엇인가?
- keyword와 text 필드를 같이 설정하는 multi-field 패턴은 왜 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Aggregation 요청 하나로 ES 노드 OOM이 발생하는 사고의 근본 원인이 여기 있다. text 필드에 집계를 실행했을 때 fielddata가 힙을 폭발적으로 점유하는 메커니즘을 이해하지 못하면 예방도, 진단도 불가능하다. doc_values와 fielddata의 차이는 Aggregation 챕터(Ch5)의 기반 지식이기도 하다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: text 필드에 집계 시도 → OOM
  요청:
    GET /products/_search
    {
      "size": 0,
      "aggs": {
        "by_description": {
          "terms": { "field": "description" }
        }
      }
    }

  오류: Fielddata is disabled on text fields by default.
  해결 시도: fielddata: true 설정
  결과: 집계는 되지만 힙 사용량 폭등 → OOM

  올바른 접근:
    text + keyword 멀티필드 매핑 설계
    description.keyword (keyword 타입) → doc_values 자동 활성화
    → 집계는 .keyword 필드로

실수 2: keyword 필드를 text로 잘못 설정
  상황:
    "category": { "type": "text" }  // 실수
    → "electronics", "keyboard" 같은 카테고리 값

  문제:
    text는 분석됨 → "electronics"가 토큰화
    term 쿼리로 "electronics" 검색 시 결과 없을 수 있음
    집계 시 fielddata 필요 → OOM 위험
    정렬 불가

  올바른 설정:
    "category": { "type": "keyword" }
    → 분석 안 됨, 정확 매칭, doc_values 자동 활성화, 집계/정렬 가능

실수 3: doc_values를 모든 필드에서 비활성화
  최적화 시도:
    "price": { "type": "integer", "doc_values": false }

  문제:
    doc_values 없으면 이 필드로 정렬/집계 불가
    → 해당 필드 기준 sort, terms agg, range agg 모두 작동 안 함
    스토리지 절약 = 기능 손실

  올바른 용도:
    정렬·집계가 절대 필요 없는 필드에만 doc_values: false
    (예: 검색 전용 텍스트, 순수 필터링용 정수)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
역할별 필드 타입 설계:

  전문 검색만:
    "body": { "type": "text" }
    → 역색인 활성화, doc_values 없음 (text는 기본)
    → 전문 검색 가능, 집계/정렬 불가

  집계·정렬만:
    "category": { "type": "keyword" }
    → doc_values 활성화 (keyword 기본)
    → 정확 매칭, 집계, 정렬 가능

  전문 검색 + 집계 모두:
    "title": {
      "type": "text",
      "fields": {
        "keyword": { "type": "keyword" }
      }
    }
    → title: 전문 검색 (역색인)
    → title.keyword: 집계/정렬 (doc_values)

  숫자 (집계+범위 쿼리):
    "price": { "type": "integer" }
    → doc_values 자동 활성화
    → BKD-Tree로 범위 쿼리, doc_values로 집계/정렬

  정렬·집계 불필요한 대용량 텍스트:
    "raw_log": { "type": "text", "doc_values": false, "norms": false }
    → 스토리지 최소화
```

---

## 🔬 내부 동작 원리

### 1. 역색인의 역방향 조회 문제

```
역색인의 구조:
  단어(Term) → 문서 ID 목록 (순방향)
  "keyboard" → [doc_1, doc_4]

정렬 요청:
  GET /products/_search
  { "sort": [{ "price": "asc" }] }

  역색인으로 정렬하려면?
  "price" 값들의 역색인:
    100000 → [doc_1]
    120000 → [doc_4]
    150000 → [doc_2]

  정렬 알고리즘이 필요한 것:
    doc_1의 price 값은? → 역색인을 뒤집어야 함
    doc_2의 price 값은? → 역색인을 뒤집어야 함

  역색인은 이 질문에 직접 답할 수 없음:
    "doc_id → 필드 값" = 역방향 조회
    역색인 = "필드 값 → doc_id" = 순방향

  해결:
    doc_id → 필드 값 을 별도로 저장해두자
    → doc_values (컬럼 지향 저장소)
```

### 2. doc_values — 컬럼 지향 저장소

```
doc_values 저장 구조:

  행 지향 (MySQL, _source):
    doc_1: { title: "Keyboard", price: 150000, category: "electronics" }
    doc_2: { title: "Mouse",    price: 80000,  category: "mouse" }
    doc_3: { title: "Monitor",  price: 300000, category: "monitor" }

  컬럼 지향 (doc_values):
    doc_id → price
    [0] → 150000
    [1] → 80000
    [2] → 300000

    doc_id → category
    [0] → "electronics"
    [1] → "mouse"
    [2] → "monitor"

  정렬 시:
    price 컬럼 전체 읽기 → [150000, 80000, 300000]
    → 정렬 → [80000(doc_2), 150000(doc_1), 300000(doc_3)]
    → 각 행의 price를 뒤지는 것보다 훨씬 효율적

  집계(Terms Agg) 시:
    category 컬럼 전체 읽기 → ["electronics", "mouse", "monitor"]
    → 카운팅 → {electronics: 1, mouse: 1, monitor: 1}

  컬럼 지향이 유리한 이유:
    분석 쿼리는 특정 컬럼 전체를 순차 읽기
    CPU 캐시 라인 효율: 연속 메모리 → 캐시 히트율 높음
    벡터 연산 최적화 가능
```

### 3. doc_values의 물리적 저장

```
파일 구조:
  _0.dvd → doc_values 실제 데이터
  _0.dvm → doc_values 메타데이터 (컬럼 타입, 압축 정보)

저장 방식 (정수형 예시):
  price 컬럼: [150000, 80000, 300000, 120000, 50000]

  압축 1: Sorted와 Ordinals
    문자열(keyword) → 정수 ordinal로 변환
    "electronics" → 0
    "monitor"     → 1
    "mouse"       → 2
    ordinals: [0, 2, 1, 0, 2] (정수로 저장)

  압축 2: 비트 패킹
    값의 범위에 따라 최소 비트 수 사용
    최대값 300000 → 18비트로 표현 가능
    4바이트 정수 대신 18비트로 압축

mmap 접근:
  .dvd 파일 → mmap
  → OS Page Cache에 올라옴
  → JVM 힙 밖 (off-heap)
  → GC 대상이 아님
  → 대용량 doc_values도 힙 부담 없음

  ┌─────────────────────────────────────────────────────┐
  │  메모리 배치:                                          │
  │                                                     │
  │  JVM 힙 (ES_JAVA_OPTS -Xmx)                          │
  │    역색인 관련 메모리, 쿼리 처리, GC 대상                   │
  │                                                     │
  │  OS Page Cache (힙 외부)                              │
  │    .dvd 파일 (mmap)  ← doc_values 여기!               │
  │    .tim 파일 (FST)   ← Term Dictionary 일부           │
  │    .doc 파일         ← Posting List                  │
  │                                                     │
  │  힙을 RAM 50% 이하로 제한하는 이유:                       │
  │    나머지 50%를 Page Cache로 확보                       │
  │    doc_values, Posting List 캐시 효율 극대화            │
  └─────────────────────────────────────────────────────┘
```

### 4. fielddata — 힙에 올라오는 역방향 매핑

```
fielddata란:
  text 필드에 집계/정렬이 필요할 때 사용하는 인메모리 자료구조
  doc_values와 동일한 목적이지만 구현 방식이 완전히 다름

왜 text 필드에 doc_values가 없는가:
  text 필드: Analyzer가 단어 분리
  "Mechanical Keyboard Cherry MX" → ["mechanical", "keyboard", "cherry", "mx"]
  하나의 문서가 여러 토큰 → 어떤 값을 doc_values에 저장?
  원본 문자열 전체? → 너무 큰 값, 집계 의미 없음
  토큰 목록? → 멀티값, doc_values 구조와 맞지 않음
  → text 필드는 doc_values 대신 fielddata 사용 (비활성화 기본)

fielddata 생성 과정 (fielddata: true 설정 시):

  첫 번째 집계 요청 수신
    ↓
  해당 text 필드의 역색인 전체 읽기
  (모든 단어 → 문서 ID 목록)
    ↓
  역방향 매핑 구성 (문서 ID → 단어 목록)
    ↓
  힙(JVM Heap)에 전체 적재

  ┌─────────────────────────────────────────────────────┐
  │  fielddata 힙 점유 예시:                               │
  │                                                     │
  │  인덱스: 100만 문서, description 필드 평균 500바이트       │
  │                                                     │
  │  역색인 크기 ≈ 문서 수 × 평균 단어 수 × 단어 크기             │
  │  ≈ 1,000,000 × 50단어 × 8바이트 = 400MB                │
  │                                                     │
  │  fielddata 힙 점유: ~400MB ~ 1GB                      │
  │  힙이 2GB라면 → 힙의 50% 이상 fielddata가 차지            │
  │  → GC 압박 → GC pause → 서비스 응답 지연                 │
  │  → 추가 요청 → OOM (OutOfMemoryError)                 │
  └─────────────────────────────────────────────────────┘

Circuit Breaker (자세한 내용: Ch5-04):
  fielddata 로드 전 힙 사용량 추정
  임계값 초과 시 요청 거부 (OOM 방지)
  indices.breaker.fielddata.limit: 기본 힙의 40%
```

### 5. text + keyword 멀티필드 — 실무 패턴

```
멀티필드(Multi-field) 매핑:

  PUT /products
  {
    "mappings": {
      "properties": {
        "title": {
          "type": "text",           ← 전문 검색용 (역색인)
          "fields": {
            "keyword": {
              "type": "keyword",    ← 집계/정렬용 (doc_values)
              "ignore_above": 256   ← 256자 초과 시 인덱싱 안 함
            }
          }
        }
      }
    }
  }

  사용:
    전문 검색: GET /products/_search { "query": { "match": { "title": "keyboard" } } }
    정확 매칭: GET /products/_search { "query": { "term": { "title.keyword": "Mechanical Keyboard" } } }
    집계:      "terms": { "field": "title.keyword" }
    정렬:      "sort": [{ "title.keyword": "asc" }]

  저장 구조:
    title       → 역색인 (분석된 토큰들)
    title.keyword → 역색인 (원본 문자열 그대로) + doc_values

  dynamic mapping의 기본 동작:
    문자열 자동 감지 → title과 title.keyword 자동 생성
    → 편리하지만 불필요한 필드 생성 → Mapping Explosion 주의
```

### 6. doc_values를 비활성화할 수 있는 경우

```
doc_values: false 가 적합한 경우:

  조건:
    해당 필드로 정렬, 집계, scripted field를 절대 사용 안 함
    순수 검색 필터링 또는 _source 반환 목적

  이득:
    .dvd 파일 없음 → 디스크 절약
    인덱싱 시 doc_values 구성 생략 → 인덱싱 속도 약간 향상

  비용:
    정렬 시도 → 오류 또는 script 필요
    집계 시도 → 오류

  예시:
    "raw_content": {
      "type": "text",
      "doc_values": false,   ← 집계/정렬 안 함
      "norms": false,        ← 점수 계산 안 함 (필터 전용)
      "index_options": "docs" ← Position 저장 안 함
    }
    → 순수 필터링 전용 텍스트 필드 최적화

  반대로 절대 하면 안 되는 것:
    "price": { "type": "integer", "doc_values": false }
    → price 집계, 정렬 모두 불가 → 가격 필터만 가능한 필드 됨
```

---

## 💻 실전 실험

```bash
# 멀티필드 매핑으로 인덱스 생성
PUT /field-test
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "price": { "type": "integer" },
      "description": {
        "type": "text",
        "fielddata": false
      }
    }
  }
}

# 테스트 문서 삽입
POST /field-test/_bulk
{ "index": { "_id": "1" } }
{ "title": "Mechanical Keyboard", "price": 150000, "description": "Cherry MX switches" }
{ "index": { "_id": "2" } }
{ "title": "Wireless Mouse", "price": 80000, "description": "Ergonomic design" }
{ "index": { "_id": "3" } }
{ "title": "Gaming Monitor", "price": 350000, "description": "144Hz refresh rate" }

# title (text): 전문 검색
GET /field-test/_search
{ "query": { "match": { "title": "keyboard" } } }

# title.keyword: 정확 매칭
GET /field-test/_search
{ "query": { "term": { "title.keyword": "Mechanical Keyboard" } } }

# title.keyword: 집계
GET /field-test/_search
{
  "size": 0,
  "aggs": { "by_title": { "terms": { "field": "title.keyword" } } }
}

# price: 집계 (doc_values 자동 활성화)
GET /field-test/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "price_range": { "range": { "field": "price", "ranges": [
      { "to": 100000 },
      { "from": 100000, "to": 200000 },
      { "from": 200000 }
    ]}}
  }
}

# description (fielddata: false): 집계 시 오류 확인
GET /field-test/_search
{
  "size": 0,
  "aggs": { "by_desc": { "terms": { "field": "description" } } }
}
# 오류: Fielddata is disabled on text fields by default

# fielddata 메모리 사용량 확인
GET _nodes/stats/indices/fielddata?pretty
GET _cat/fielddata?v&fields=*

# doc_values vs fielddata 메모리 위치 확인
GET _nodes/stats/indices/segments?pretty
# segments.memory_in_bytes: JVM 힙 내 세그먼트 메모리
# (doc_values는 off-heap이므로 여기 포함 안 됨)

# fielddata 캐시 비우기 (OOM 상황 임시 해결)
POST /field-test/_cache/clear?fielddata=true
```

---

## 📊 doc_values vs fielddata 비교

| 항목 | doc_values | fielddata |
|------|-----------|-----------|
| 기본 활성화 | keyword, 숫자, 날짜 | 비활성화 (text 기본) |
| 저장 위치 | 디스크 (.dvd 파일) | JVM 힙 |
| 메모리 영향 | off-heap (Page Cache) | 힙 점유 → GC/OOM 위험 |
| 생성 시점 | 인덱싱 시 (사전 구성) | 첫 집계 요청 시 (지연 로딩) |
| text 필드 | 기본 불가 (토큰 문제) | fielddata: true로 활성화 가능 |
| 성능 | 매우 빠름 (mmap) | 첫 요청 느림, 이후 캐시 |
| 권장 | 항상 권장 | 불가피한 경우에만 |

---

## ⚖️ 트레이드오프

```
doc_values의 이득:
  off-heap (OS Page Cache) → GC 부담 없음
  인덱싱 시 구성 → 검색 시 즉시 사용
  압축 효율 (ordinals, 비트 패킹)

doc_values의 비용:
  인덱싱 시 추가 저장 → 디스크 공간 증가 (보통 원본의 20~50%)
  인덱싱 속도 약간 저하 (doc_values 구성 포함)

fielddata의 이득:
  text 필드에 집계 가능 (유일한 방법)
  한 번 로딩 후 재사용

fielddata의 비용:
  힙 점유 → GC 부담 → OOM 위험
  첫 요청 시 느림 (역색인 전체 읽기 + 역방향 매핑 구성)
  Circuit Breaker 설정 필수

멀티필드의 이득과 비용:
  이득: text + keyword 동시 가능 → 전문 검색 + 집계 모두
  비용: 같은 데이터를 두 번 저장 (역색인 + keyword 역색인 + doc_values)
        인덱스 크기 증가
```

---

## 📌 핵심 정리

```
역색인의 역방향 조회 한계:
  역색인: 단어 → 문서 ID (순방향만)
  정렬·집계에는 문서 ID → 필드 값 (역방향) 필요
  → doc_values 또는 fielddata로 해결

doc_values:
  컬럼 지향 저장소 (.dvd 파일)
  인덱싱 시 사전 구성 → 검색 시 즉시 사용
  off-heap (OS Page Cache 활용) → GC 부담 없음
  keyword, 숫자, 날짜 필드에 기본 활성화

fielddata:
  text 필드 집계 시 사용
  힙에 역방향 매핑 구성 → OOM 위험
  기본 비활성화, 활성화 시 Circuit Breaker 필수

실무 패턴:
  text + keyword 멀티필드 → 전문 검색 + 집계 모두
  집계/정렬 없는 필드 → doc_values: false로 디스크 절약
  text 필드 집계 → .keyword 서브필드로 대체 (fielddata 지양)

힙 50% 제한의 이유:
  나머지 50% = OS Page Cache
  doc_values(mmap), Posting List, FST 캐시
  → Page Cache가 충분해야 I/O 없이 검색 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `ignore_above: 256` 설정이 keyword 필드에 있을 때 어떤 문서가 집계에서 제외되는가?

<details>
<summary>해설 보기</summary>

`ignore_above: 256`은 원본 문자열이 256자를 초과하면 해당 keyword 필드에 인덱싱하지 않는다는 설정이다. 해당 문서가 집계에서 완전히 제외되는 것이 아니라, 그 필드의 keyword 값만 없는 것으로 처리된다. 즉, 256자 초과인 문서는 terms aggregation 결과에 해당 값으로 집계되지 않고, `doc_count_error_upper_bound`나 `sum_other_doc_count`에도 포함되지 않는다. text 필드로의 전문 검색과 _source 반환은 여전히 정상 동작한다.

</details>

**Q2.** doc_values가 off-heap에 있는데 ES 힙 크기를 늘리면 doc_values 성능이 왜 향상되지 않는가?

<details>
<summary>해설 보기</summary>

doc_values는 OS Page Cache에 캐시되며 JVM 힙과 무관하다. 힙을 늘리면 오히려 OS가 Page Cache에 할당할 수 있는 메모리가 줄어들어 역효과가 날 수 있다. doc_values 성능을 높이려면 힙을 RAM의 50% 이하로 제한해 Page Cache 공간을 충분히 확보하는 것이 맞다. doc_values .dvd 파일이 Page Cache에 상주하면 mmap 접근 시 디스크 I/O 없이 메모리 속도로 읽을 수 있다.

</details>

**Q3.** 같은 값이 많은 keyword 필드(예: status가 "active"/"inactive" 두 가지)에서 doc_values의 ordinals 압축이 효과적인 이유는?

<details>
<summary>해설 보기</summary>

ordinals는 고유 값을 정수로 매핑하는 압축 방식이다. status 필드가 "active"와 "inactive" 두 가지만 있다면 ordinals는 {0: "active", 1: "inactive"} 테이블과 각 문서의 ordinal 배열([0, 1, 0, 0, 1, ...])로 구성된다. 문서 수가 100만 개이더라도 ordinal 배열은 1비트로 표현 가능하다(0 또는 1). 원본 문자열("active" = 6바이트)을 100만 번 저장하는 것보다 테이블(두 항목) + 배열(100만 × 1비트)이 압도적으로 작다. 카디널리티(고유 값 수)가 낮을수록 ordinals 압축이 효과적이다.

</details>

---

<div align="center">

**[⬅️ 이전: Near Real-Time 검색](./05-near-real-time-search.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — 분석 파이프라인 ➡️](../text-analysis/01-analysis-pipeline.md)**

</div>
