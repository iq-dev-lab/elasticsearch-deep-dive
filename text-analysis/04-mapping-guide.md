# 매핑(Mapping) 완전 가이드 — text·keyword·dynamic mapping

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `text`와 `keyword` 타입의 근본적인 차이는 무엇이고, 언제 각각을 써야 하는가?
- dynamic mapping이 편리하지만 위험한 이유는 무엇인가?
- Mapping Explosion은 왜 발생하고 클러스터에 어떤 영향을 주는가?
- `dynamic: strict`로 의도치 않은 필드를 막는 방법은?
- 이미 설정된 매핑을 변경하려면 어떻게 해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

매핑은 ES에서 MySQL의 스키마와 같다. 처음 설계를 잘못하면 운영 중에 수정이 매우 어렵고, 잘못된 dynamic mapping 설정 하나로 클러스터 전체가 불안정해질 수 있다. Mapping Explosion은 실제로 대형 서비스에서 발생한 장애 원인 중 하나다. 매핑 설계를 제대로 이해하지 않으면 검색 기능이 동작하지 않거나 성능이 예측 불가능해진다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: text 필드로 집계 시도
  매핑:
    "category": { "type": "text" }

  집계 요청:
    "aggs": { "by_cat": { "terms": { "field": "category" } } }

  오류: Fielddata is disabled on text fields by default.
        Set fielddata=true on [category]

  근본 원인:
    text는 역색인 전문 검색용 → 집계·정렬에 doc_values 없음
    keyword는 집계·정렬·정확 매칭용

  올바른 설계:
    "category": { "type": "keyword" }  (집계/정렬용)
    또는
    "category": { "type": "text", "fields": { "keyword": { "type": "keyword" } } }

실수 2: keyword 필드로 전문 검색 시도
  매핑:
    "description": { "type": "keyword" }

  검색:
    "query": { "match": { "description": "fast search" } }

  결과: 의도와 다른 동작
    keyword는 분석 안 됨
    역색인에 "fast search" 전체 문자열이 하나의 토큰
    match 쿼리가 keyword 필드에 적용되면 내부적으로 term 쿼리로 변환
    → "fast search"가 정확히 일치하는 문서만 반환

  올바른 설계:
    "description": { "type": "text" } (전문 검색용)

실수 3: dynamic mapping 방치로 Mapping Explosion
  상황:
    로그 데이터를 ES에 인덱싱
    로그마다 다른 키를 가진 JSON

  {"timestamp": "...", "user_id": 1, "action": "login"}
  {"timestamp": "...", "order_id": 123, "status": "pending", "product": {...}}
  {"timestamp": "...", "error_code": "E001", "stack_trace": "..."}

  dynamic: true (기본값) 상태에서:
    새로운 필드가 나올 때마다 자동으로 매핑 추가
    → 수천 개의 서로 다른 필드가 클러스터 상태에 추가됨
    → 클러스터 상태 크기 폭증
    → Master 노드 부하 증가 → 클러스터 불안정
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
매핑 설계 원칙:

  1. 항상 명시적 매핑 먼저 정의
     PUT /index { "mappings": { "properties": { ... } } }
     dynamic: false 또는 strict로 자동 생성 제한

  2. 필드 목적별 타입 선택
     전문 검색 → text
     집계·정렬·정확 일치 → keyword
     숫자 범위 → integer, float, double
     날짜 → date
     불린 → boolean
     둘 다 필요 → text + keyword 멀티필드

  3. dynamic 설정 선택
     dynamic: true (기본) → 새 필드 자동 생성 (개발 초기에만 허용)
     dynamic: false → 새 필드 무시 (인덱싱은 됨, 검색만 안 됨)
     dynamic: strict → 새 필드 오류 반환 (프로덕션 권장)

  4. 중첩 객체 주의
     nested 타입: 내부 객체 관계 보존 (별도 Lucene 문서)
     object 타입: 내부 객체를 flat하게 처리 (기본)
```

---

## 🔬 내부 동작 원리

### 1. text vs keyword 내부 차이

```
text 타입:
  ┌────────────────────────────────────────────────────────┐
  │  "Mechanical Keyboard Cherry MX"                       │
  │                    │                                   │
  │             Analyzer 실행                               │
  │                    │                                   │
  │   ["mechanical", "keyboard", "cherry", "mx"]           │
  │                    │                                   │
  │              역색인 저장                                  │
  │   mechanical → [doc_1]                                 │
  │   keyboard   → [doc_1, doc_4]                          │
  │   cherry     → [doc_1]                                 │
  │   mx         → [doc_1]                                 │
  └────────────────────────────────────────────────────────┘

  특징:
    Analyzer로 분석 (소문자화, 토큰 분리 등)
    전체 문자열이 아닌 토큰으로 역색인 저장
    doc_values 기본 비활성화 (집계 불가)
    match, match_phrase 쿼리에 적합
    term 쿼리로 "Mechanical"(대문자) 검색 → 없음 (역색인엔 소문자만)

keyword 타입:
  ┌────────────────────────────────────────────────────────┐
  │  "Mechanical Keyboard Cherry MX"                       │
  │                    │                                   │
  │          Analyzer 없음 (그대로 저장)                       │
  │                    │                                   │
  │   ["Mechanical Keyboard Cherry MX"] (단일 토큰)          │
  │                    │                                   │
  │              역색인 저장                                  │
  │   "Mechanical Keyboard Cherry MX" → [doc_1]            │
  │                    │                                   │
  │              doc_values 저장                            │
  │   doc_1 → "Mechanical Keyboard Cherry MX"              │
  └────────────────────────────────────────────────────────┘

  특징:
    분석 없음 → 원본 문자열 그대로 단일 토큰
    대소문자 구분 (정확 일치)
    doc_values 기본 활성화 (집계·정렬 가능)
    term 쿼리, terms 집계, sort에 적합
    "Mechanical"로 검색 → 실패 (전체 문자열과 불일치)
```

### 2. 필드 타입별 내부 자료구조

```
숫자 타입 (integer, long, float, double):
  역색인: BKD-Tree (범위 쿼리 최적화)
    range: { price: { gte: 10000, lte: 50000 } }
    → BKD-Tree에서 O(log N) 범위 탐색
  doc_values: 컬럼 지향 저장 (집계·정렬)
  term 쿼리도 가능 (정확한 값 일치)

date 타입:
  내부적으로 epoch milliseconds로 저장
  역색인: BKD-Tree (날짜 범위 쿼리)
  doc_values: epoch ms 값 저장 (날짜 집계)
  format 설정: "yyyy-MM-dd", "epoch_millis" 등

boolean 타입:
  역색인: "T" (true) / "F" (false) 단일 토큰
  doc_values: 0 / 1 숫자로 저장

object 타입 (내부 JSON 객체):
  {"user": {"name": "Alice", "age": 30}}
  → Flat 변환:
    user.name → "Alice"
    user.age  → 30

nested 타입:
  object과 달리 내부 객체를 별도 Lucene 문서로 저장
  내부 필드 간 관계 보존

  object 문제:
    {"tags": [{"name": "red", "count": 10}, {"name": "blue", "count": 5}]}
    object flat 변환:
      tags.name  → ["red", "blue"]
      tags.count → [10, 5]
    쿼리: tags.name="red" AND tags.count=5
    → object: red와 5가 같은 배열에 있으므로 매칭 (잘못된 결과!)
    → nested: 각 태그가 별도 문서 → 정확한 필터링 가능
```

### 3. Dynamic Mapping — 자동 타입 추론

```
dynamic: true (기본) 시 자동 타입 매핑:

  값 예시            → 추론 타입
  ─────────────────────────────────
  "hello"           → text + keyword (멀티필드)
  123               → long
  3.14              → float
  true/false        → boolean
  "2024-01-01"      → date (날짜 패턴 인식 시)
  {"nested": "obj"} → object
  [1, 2, 3]         → long (배열의 첫 요소 타입)

문제: 타입 추론 오류
  첫 문서: {"code": "123"}   → code 타입: text (문자열)
  두 번째: {"code": 456}     → code = 456 (숫자) → text 필드에 숫자 저장
  → 매핑 충돌: 이미 text로 설정된 필드에 숫자 저장 시도

  반대 상황:
  첫 문서: {"version": 1}    → version 타입: long
  두 번째: {"version": "1.2.3"} → long 필드에 문자열 저장
  → 인덱싱 실패!

dynamic: false:
  새 필드 자동 매핑 안 함
  데이터는 _source에 저장되지만 역색인 없음 → 검색 불가
  기존 매핑된 필드는 정상 작동
  용도: 일부 필드만 검색 가능하도록 제한

dynamic: strict:
  새 필드 발견 시 인덱싱 오류 반환
  {"unknown_field": "value"} → strict_dynamic_mapping_exception
  용도: 스키마를 엄격하게 관리하는 프로덕션 환경
```

### 4. Mapping Explosion — 왜 위험한가

```
Mapping Explosion 발생 시나리오:

  로그 데이터 인덱싱 (dynamic: true):
    {"ts": 1, "data": {"user_123": {...}}}
    {"ts": 2, "data": {"user_456": {...}}}
    {"ts": 3, "data": {"order_789": {...}}}

    → data.user_123, data.user_456, data.order_789
       → 매 문서마다 새로운 필드가 자동 생성

  시간이 지나면:
    수천 개의 고유 필드가 매핑에 추가
    클러스터 상태에 수백 MB의 매핑 정보 누적

클러스터 영향:
  ┌──────────────────────────────────────────────────────┐
  │  클러스터 상태 크기 폭증                                   │
  │     ↓                                                │
  │  Master 노드가 클러스터 상태 배포 시간 증가                  │
  │     ↓                                                │
  │  새 인덱스 생성, 샤드 재배치 등 모든 클러스터 작업 느려짐         │
  │     ↓                                                │
  │  Master Election 타임아웃 위험                          │
  │     ↓                                                │
  │  클러스터 불안정 → 서비스 영향                              │
  └──────────────────────────────────────────────────────┘

방어 설정:
  index.mapping.total_fields.limit: 1000 (기본값)
  → 인덱스당 최대 필드 수 제한
  → 초과 시 인덱싱 오류

  index.mapping.depth.limit: 20 (기본값)
  → 중첩 객체 최대 깊이 제한

  index.mapping.nested_fields.limit: 50 (기본값)
  → nested 타입 필드 최대 수 제한

근본 해결:
  dynamic: strict로 명시적 매핑만 허용
  가변 키를 가진 데이터 → flattened 타입 사용
```

### 5. flattened 타입 — Mapping Explosion 해결책

```
문제:
  로그 데이터에서 키가 동적으로 변하는 JSON 객체

  {"labels": {"env": "prod", "region": "us-east", "version": "1.2.3"}}
  → labels 하위 키마다 별도 매핑 생성 위험

flattened 타입:
  PUT /logs
  {
    "mappings": {
      "properties": {
        "labels": { "type": "flattened" }
      }
    }
  }

  동작:
    labels 하위 모든 키-값을 단일 역색인으로 처리
    labels.env, labels.region, labels.version → 별도 매핑 생성 안 함
    대신 "env.prod", "region.us-east" 형태로 단일 역색인

  제한:
    정확 매칭 쿼리만 가능 (term, terms)
    범위 쿼리, 집계 제한적
    전문 검색(match) 불가

  용도:
    키가 동적으로 변하는 메타데이터, 태그, 라벨
    Kubernetes labels, 사용자 정의 속성 등
```

### 6. 매핑 변경의 제약

```
변경 가능:
  기존 필드에 새 멀티필드 추가
  "title" → "title.keyword" 서브필드 추가
  dynamic, index 설정 변경
  새 필드 추가

변경 불가 (재인덱싱 필요):
  기존 필드의 타입 변경 (text → keyword 등)
  기존 필드의 Analyzer 변경
  기존 필드 삭제 (삭제처럼 보이게만 가능)

이유:
  이미 인덱싱된 데이터는 기존 타입/Analyzer로 저장됨
  타입이 변경되면 기존 역색인과 새 타입 불일치
  → 전체 재인덱싱 필요

재인덱싱 절차:
  1. 새 매핑으로 새 인덱스 생성 (products-v2)
  2. Reindex API로 기존 데이터 복사
  3. 별칭(alias)을 새 인덱스로 전환
  4. 기존 인덱스 삭제

  POST _reindex
  {
    "source": { "index": "products-v1" },
    "dest":   { "index": "products-v2" }
  }

  별칭 전환 (무중단):
  POST _aliases
  {
    "actions": [
      { "remove": { "index": "products-v1", "alias": "products" } },
      { "add":    { "index": "products-v2", "alias": "products" } }
    ]
  }
```

---

## 💻 실전 실험

```bash
# 명시적 매핑 설계 (권장 패턴)
PUT /products-explicit
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "description": { "type": "text" },
      "price":       { "type": "integer" },
      "category":    { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "created_at":  { "type": "date", "format": "yyyy-MM-dd HH:mm:ss||epoch_millis" },
      "in_stock":    { "type": "boolean" },
      "rating":      { "type": "float" },
      "metadata":    { "type": "flattened" }
    }
  }
}

# dynamic: strict 테스트 (새 필드 오류 확인)
POST /products-explicit/_doc
{
  "title": "Test Product",
  "price": 10000,
  "unknown_field": "should fail"
}
# 오류: strict_dynamic_mapping_exception

# 정상 인덱싱
POST /products-explicit/_doc
{
  "title": "Mechanical Keyboard",
  "price": 150000,
  "category": "keyboard",
  "tags": ["gaming", "mechanical"],
  "created_at": "2024-01-15 10:30:00",
  "in_stock": true,
  "rating": 4.5,
  "metadata": { "color": "black", "weight_g": 850 }
}

# text vs keyword 동작 비교
GET /products-explicit/_search
{
  "query": { "match": { "title": "mechanical" } }       // text: 분석 후 탐색
}

GET /products-explicit/_search
{
  "query": { "term": { "title.keyword": "Mechanical Keyboard" } } // keyword: 정확 일치
}

# keyword 집계
GET /products-explicit/_search
{
  "size": 0,
  "aggs": { "by_category": { "terms": { "field": "category" } } }
}

# 매핑 확인
GET /products-explicit/_mapping?pretty

# 필드 수 확인 (Mapping Explosion 모니터링)
GET /products-explicit/_mapping?pretty
# properties 개수 확인

# Reindex 예시
PUT /products-v2
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "title":    { "type": "text" },
      "price":    { "type": "double" },  // integer → double 변경
      "category": { "type": "keyword" }
    }
  }
}

POST _reindex
{
  "source": { "index": "products-explicit", "size": 1000 },
  "dest":   { "index": "products-v2" }
}
```

---

## 📊 타입 선택 가이드

| 용도 | 타입 | 이유 |
|------|------|------|
| 전문 검색 (단어 포함 여부) | `text` | 역색인, Analyzer 적용 |
| 정확 일치, 집계, 정렬 | `keyword` | doc_values, 분석 없음 |
| 전문 검색 + 집계 | `text` + `keyword` 멀티필드 | 둘 다 필요 |
| 숫자 범위 쿼리 | `integer`, `long`, `float` | BKD-Tree |
| 날짜 범위, 집계 | `date` | epoch ms 저장 |
| 참/거짓 필터 | `boolean` | 단순 필터 최적화 |
| 동적 키 JSON | `flattened` | Mapping Explosion 방지 |
| 배열 내 객체 관계 | `nested` | 정확한 관계 쿼리 |

---

## ⚖️ 트레이드오프

```
dynamic: true vs strict:
  true   → 개발 편의, 스키마 유연성 | Mapping Explosion 위험
  false  → 새 필드 무시, 안전 | 검색 불가 필드 생김 (디버깅 어려움)
  strict → 가장 안전, 스키마 강제 | 모든 필드 사전 정의 필요

text vs keyword 선택:
  전문 검색 서비스: text (사용자가 단어로 검색)
  정확한 값 필터링: keyword (코드, 상태값, 카테고리)
  둘 다: 멀티필드 (저장 비용은 2배이지만 기능은 2배)

nested 사용:
  이득: 배열 내 객체 관계 정확한 쿼리
  비용: 별도 Lucene 문서 → 인덱스 크기 증가, 쿼리 복잡도 증가
  → 실제로 관계 쿼리가 필요할 때만 nested 사용
```

---

## 📌 핵심 정리

```
text vs keyword:
  text:    Analyzer 적용 → 토큰 분리 → 전문 검색
  keyword: 분석 없음 → 원본 그대로 → 집계·정렬·정확 일치
  둘 다: 멀티필드 패턴 (text + keyword 서브필드)

dynamic mapping:
  true:   새 필드 자동 생성 (개발용)
  false:  새 필드 무시 (중간 안전)
  strict: 새 필드 오류 (프로덕션 권장)

Mapping Explosion:
  동적 키 데이터가 dynamic: true에서 필드 폭증
  클러스터 상태 비대화 → Master 노드 부하 → 클러스터 불안정
  해결: dynamic: strict + flattened 타입 활용

매핑 변경:
  타입·Analyzer 변경 → 재인덱싱 필수
  새 필드 추가 → 즉시 가능
  별칭(alias)으로 무중단 재인덱싱 구현
```

---

## 🤔 생각해볼 문제

**Q1.** `dynamic: false`로 설정한 인덱스에 새 필드가 있는 문서를 인덱싱하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`dynamic: false`이면 새 필드가 포함된 문서 인덱싱은 성공한다. 인덱싱 오류가 발생하는 것은 `strict` 모드다. `false` 모드에서는 새 필드의 값이 `_source`에는 저장되지만 역색인이 생성되지 않아 그 필드로 검색할 수 없다. 즉, 데이터는 _source에 보관되지만 검색 불가능한 상태다. 나중에 해당 필드를 검색 가능하게 만들려면 매핑에 명시적으로 추가하고 전체 재인덱싱해야 한다.

</details>

**Q2.** 이미 운영 중인 인덱스에서 `text` 필드를 `keyword`로 변경하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

기존 필드 타입 변경은 ES에서 불가능하다. 해결 방법은 새 매핑으로 새 인덱스를 생성하고 Reindex API로 데이터를 복사하는 것이다. 무중단으로 전환하려면 별칭(alias)을 활용한다. 운영 중 애플리케이션이 인덱스 이름 대신 별칭을 바라보고 있어야 하며, 재인덱싱 완료 후 별칭을 새 인덱스로 원자적으로 전환하면 다운타임 없이 전환된다. 재인덱싱 중 새로 들어오는 쓰기를 처리하려면 두 인덱스에 동시에 쓰거나, 재인덱싱 완료 후 차이분을 추가로 복사하는 패턴을 사용한다.

</details>

**Q3.** `ignore_above: 256`이 설정된 keyword 필드에 300자짜리 문자열을 인덱싱하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

해당 keyword 필드는 인덱싱(역색인 저장)을 건너뛴다. 단, `_source`에는 원본 300자 문자열이 그대로 저장된다. 검색 시 term 쿼리, terms 집계, sort는 해당 문서를 찾지 못하지만, `_source`로 직접 조회하거나 match 쿼리를 사용하는 text 필드 검색에는 영향이 없다. `ignore_above`는 과도하게 긴 값(예: base64 인코딩된 데이터, URL)이 keyword 필드에 저장되어 메모리를 낭비하는 것을 방지하기 위한 설정이다.

</details>

---

<div align="center">

**[⬅️ 이전: 커스텀 Analyzer 설계](./03-custom-analyzer.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스 시점 vs 검색 시점 분석 ➡️](./05-index-vs-search-time-analysis.md)**

</div>
