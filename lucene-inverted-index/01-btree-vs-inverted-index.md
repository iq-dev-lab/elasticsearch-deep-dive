# B-Tree vs 역색인 — 왜 검색에는 역색인인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MySQL `LIKE '%keyword%'`는 왜 인덱스를 사용하지 못하고 Full Scan을 하는가?
- 역색인은 "elasticsearch 가 포함된 문서"를 어떻게 O(1)에 가깝게 찾는가?
- B-Tree가 강한 영역과 역색인이 강한 영역은 어디서 갈라지는가?
- 역색인의 구조적 제약은 무엇이고, 그 제약을 어떻게 해결하는가?
- `database-internals`에서 배운 B-Tree 지식이 왜 여기서 출발점이 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"MySQL에 이미 인덱스가 있는데 왜 Elasticsearch를 따로 써야 하죠?"라는 질문의 답이 여기 있다. B-Tree와 역색인은 둘 다 "빠른 탐색"을 위한 자료구조지만, 최적화된 질문의 종류가 근본적으로 다르다. 이 차이를 모르면 Elasticsearch를 MySQL의 대체제로 오해하거나, 역으로 MySQL에서 무리하게 전문 검색을 구현하려다 성능 문제를 만난다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
상황: 상품 검색 기능 구현

MySQL로 시도:
  SELECT * FROM products
  WHERE title LIKE '%mechanical keyboard%';

  결과:
    인덱스가 있어도 LIKE '%...%' 앞에 %가 있으면 인덱스 사용 불가
    → Full Table Scan: 모든 행을 읽으며 문자열 비교
    → 1,000만 건이면 수 초~수십 초

  왜 인덱스를 못 쓰는가:
    B-Tree는 왼쪽부터 정렬된 구조
    "mech"로 시작하는 것은 B-Tree에서 범위 탐색 가능
    하지만 "중간에 포함"은 B-Tree가 알 방법이 없음
    → 전체를 스캔해야만 찾을 수 있음

오해:
  "FULLTEXT INDEX를 쓰면 되지 않나?"
  실제:
    MySQL FULLTEXT INDEX는 간단한 전문 검색 가능
    하지만 BM25 스코어링, 형태소 분석, 복잡한 쿼리 조합,
    집계(Aggregation), 분산 처리 — 모두 한계 있음
    Elasticsearch 역색인 = FULLTEXT INDEX의 수십 배 기능과 성능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
역할에 맞는 저장소 선택:

  MySQL (B-Tree):
    정확 일치:   WHERE id = 42             → O(log N), 매우 빠름
    범위 조회:   WHERE price BETWEEN 1 2   → O(log N + K)
    전위 매칭:   WHERE title LIKE 'mech%'  → O(log N + K)
    트랜잭션:    ACID 보장
    → 구조화된 데이터, 정확한 조건, 트랜잭션 필요 시

  Elasticsearch (역색인):
    전문 검색:   "mechanical keyboard 포함 문서" → O(len) 탐색
    관련성 정렬: BM25로 "얼마나 관련있는가" 점수 계산
    집계 분석:   카테고리별 집계, 가격 히스토그램
    분산 처리:   수억 문서도 샤드로 분산
    → 검색어 기반 탐색, 관련성 정렬, 대규모 집계

  실무 패턴:
    MySQL   → 주 데이터 저장소 (트랜잭션, 정합성)
    ES      → 검색 인덱스 (MySQL 데이터를 ES에 동기화)
    → MySQL이 원본, ES가 검색 전용 복사본
```

---

## 🔬 내부 동작 원리

### 1. B-Tree 구조와 전문 검색의 한계

```
B-Tree 인덱스 (database-internals 연결):

  인덱스 컬럼: title
  데이터:
    id=1: "Mechanical Keyboard Cherry MX"
    id=2: "Wireless Mouse Logitech"
    id=3: "Mechanical Switch Tester"
    id=4: "Gaming Keyboard RGB"

  B-Tree 리프 노드 (정렬 순서):
  ┌──────────────────────────────────────────────────────┐
  │ "Gaming Keyboard RGB"     → id=4                     │
  │ "Mechanical Keyboard..."  → id=1                     │
  │ "Mechanical Switch..."    → id=3                     │
  │ "Wireless Mouse..."       → id=2                     │
  └──────────────────────────────────────────────────────┘

  B-Tree가 잘하는 것:
    WHERE title = 'Gaming Keyboard RGB'
    → 루트에서 리프까지 O(log N) 탐색 → id=4 즉시 반환

    WHERE title LIKE 'Mech%'      (접두사 매칭)
    → 'Mech'로 시작하는 첫 번째 위치부터 순차 탐색
    → id=1, id=3 반환 (B-Tree 범위 탐색 활용)

  B-Tree가 못하는 것:
    WHERE title LIKE '%keyboard%'  (중간 포함)
    → B-Tree는 왼쪽(접두사)을 기준으로 정렬
    → 어느 위치에 "keyboard"가 있는지 알 수 없음
    → 전체 행 스캔 (4행 → 전부 읽고 문자열 비교)
    → 1,000만 행이면 1,000만 번 비교

  B-Tree의 근본적 질문:
    "이 값과 정확히 같은(또는 범위 내인) 레코드는 어디있나?"
    → 값 → 레코드 위치: 순방향 매핑에 최적화
```

### 2. 역색인의 발상 전환 — 뒤집힌 매핑

```
역색인의 발상:
  "단어 → 그 단어가 등장하는 문서 목록"을 미리 만들어두자

  기존 B-Tree: 레코드 → 값
  역색인:      단어(Term) → 문서 ID 목록(Posting List)

문서 집합:
  doc_1: "Mechanical Keyboard Cherry MX"
  doc_2: "Wireless Mouse Logitech"
  doc_3: "Mechanical Switch Tester"
  doc_4: "Gaming Keyboard RGB"

Analyzer 처리 (소문자화, 토큰 분리):
  doc_1 → ["mechanical", "keyboard", "cherry", "mx"]
  doc_2 → ["wireless", "mouse", "logitech"]
  doc_3 → ["mechanical", "switch", "tester"]
  doc_4 → ["gaming", "keyboard", "rgb"]

역색인 구성:
  Term          Posting List (문서 ID)
  ──────────────────────────────────────
  "cherry"    → [1]
  "gaming"    → [4]
  "keyboard"  → [1, 4]         ← 핵심!
  "logitech"  → [2]
  "mechanical"→ [1, 3]
  "mouse"     → [2]
  "mx"        → [1]
  "rgb"       → [4]
  "switch"    → [3]
  "tester"    → [3]
  "wireless"  → [2]

검색: "keyboard 가 포함된 문서"
  1. Term Dictionary에서 "keyboard" 탐색 → O(log N) 또는 O(len) (FST)
  2. Posting List: [1, 4]
  3. doc_1, doc_4 즉시 반환
  → 전체 문서 스캔 없음!

검색: "mechanical AND keyboard"
  "mechanical" → [1, 3]
  "keyboard"   → [1, 4]
  교집합:         [1]          ← doc_1만 반환
  → 두 Posting List의 AND 연산 (merge join)
```

### 3. 역색인의 탐색 비용

```
B-Tree vs 역색인 탐색 비용 비교:

  전체 문서 수: N
  검색어 길이: L
  검색 결과 수: K

  B-Tree (LIKE '%keyword%'):
    비용: O(N) — 전체 스캔
    N=1,000만 → 1,000만 번 비교

  역색인:
    Step 1: Term Dictionary에서 단어 탐색
      정렬된 배열 이진 탐색: O(log V)  (V = 전체 단어 수)
      FST 사용 시:          O(L)       (L = 검색어 길이)
    Step 2: Posting List 읽기
      O(K)                             (K = 결과 문서 수)
    총 비용: O(L + K)

  실제 차이:
    N=10,000,000 (1,000만 문서)
    L=10 (10글자 검색어)
    K=100 (결과 100개)

    B-Tree:  O(10,000,000)
    역색인:  O(10 + 100) = O(110)
    → 약 90,000배 차이
```

### 4. 역색인의 구조적 한계와 해결

```
역색인이 못하는 것:

  역방향 조회: "doc_1의 title 필드 값은?"
    역색인: 단어 → 문서 ID (순방향만)
    문서 ID → 단어: 역색인에서 불가능
    → _source JSON을 별도로 저장하는 이유
    → doc_values로 특정 필드 값 조회

  범위 쿼리: "price > 100 AND price < 200"
    역색인: 정확한 단어만 탐색
    숫자 범위는 역색인보다 BKD-Tree(Block KD-Tree)가 적합
    → ES는 숫자/날짜 필드에 역색인 대신 BKD-Tree 사용

  정렬: "결과를 price 순으로 정렬"
    역색인: 단어를 알면 문서를 찾지만, 문서의 price 값을 모름
    → doc_values(컬럼 지향 저장소)로 해결 (Ch2-06 참고)

  UPDATE: "doc_1의 price를 150으로 변경"
    역색인 재구성 필요 (단어 목록 변경 가능)
    세그먼트가 불변인 이유 → 수정 = 삭제 마킹 + 새 문서 삽입
    → UPDATE가 비싸다는 의미 (Ch2-04 참고)

해결 구조:
  ┌─────────────────────────────────────────────────────┐
  │              Elasticsearch의 필드 저장 전략             │
  │                                                     │
  │  text 필드: 역색인 (전문 검색)                           │
  │  keyword 필드: 역색인 (정확 매칭) + doc_values (정렬)     │
  │  숫자/날짜: BKD-Tree (범위 쿼리) + doc_values (정렬)     │
  │  _source: 원본 JSON 전체 (Fetch Phase에서 반환)         │
  └─────────────────────────────────────────────────────┘
```

### 5. B-Tree와 역색인이 공존하는 영역

```
Elasticsearch에서 B-Tree 계열도 사용:

  Term Dictionary 내부 정렬:
    단어 목록은 정렬된 상태로 저장 (이진 탐색 가능)
    메모리에 FST(Finite State Transducer)로 압축 적재

  BKD-Tree (Block KD-Tree):
    숫자, 날짜, geo_point 필드의 범위 쿼리
    range 쿼리: "2024-01-01 ~ 2024-12-31 사이 문서"
    → 역색인 대신 BKD-Tree로 처리

  결론:
    역색인이 B-Tree를 완전히 대체하는 것이 아님
    각 쿼리 유형에 맞는 자료구조를 필드 타입별로 선택
    ES는 한 문서에 여러 자료구조를 동시에 유지:
      역색인 + BKD-Tree + doc_values + _source 저장
```

---

## 💻 실전 실험

```bash
# 실험용 인덱스 생성
PUT /search-test
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "price": { "type": "integer" },
      "category": { "type": "keyword" }
    }
  }
}

# 테스트 문서 삽입
POST /search-test/_bulk
{ "index": { "_id": "1" } }
{ "title": "Mechanical Keyboard Cherry MX", "price": 150000, "category": "keyboard" }
{ "index": { "_id": "2" } }
{ "title": "Wireless Mouse Logitech", "price": 80000, "category": "mouse" }
{ "index": { "_id": "3" } }
{ "title": "Mechanical Switch Tester", "price": 30000, "category": "accessory" }
{ "index": { "_id": "4" } }
{ "title": "Gaming Keyboard RGB", "price": 120000, "category": "keyboard" }

# 역색인 확인 — "keyboard"가 어떤 문서 ID와 연결되는지
# term vector로 역색인 내 정보 조회
GET /search-test/_termvectors/1
{
  "fields": ["title"],
  "offsets": true,
  "positions": true,
  "term_statistics": true,
  "field_statistics": true
}

# 전문 검색 (역색인 활용)
GET /search-test/_search
{
  "query": { "match": { "title": "keyboard" } }
}
# → doc_1, doc_4 반환 (역색인에서 즉시 조회)

# B-Tree 계열: 정확 일치 (keyword 필드)
GET /search-test/_search
{
  "query": { "term": { "category": "keyboard" } }
}

# B-Tree 계열: 범위 쿼리 (BKD-Tree 내부 사용)
GET /search-test/_search
{
  "query": {
    "range": { "price": { "gte": 100000, "lte": 200000 } }
  }
}

# _explain으로 역색인 점수 계산 과정 확인
GET /search-test/_explain/1
{
  "query": { "match": { "title": "keyboard" } }
}
# description에서 "weight(title:keyboard in 0)" 확인
# → 역색인에서 "keyboard"를 찾아 BM25 점수 계산
```

---

## 📊 B-Tree vs 역색인 완전 비교

| 항목 | B-Tree (MySQL) | 역색인 (Elasticsearch) |
|------|---------------|----------------------|
| 최적 쿼리 | 정확 일치, 범위, 전위 매칭 | 전문 검색, 단어 포함 여부 |
| `LIKE '%word%'` | Full Scan O(N) | O(단어 길이 + 결과 수) |
| 정렬 | 인덱스 컬럼 기준 O(1) | doc_values 필요 |
| UPDATE 비용 | O(log N) | 삭제 마킹 + 재삽입 (비쌈) |
| 관련성 점수 | 없음 | BM25 점수 계산 |
| 트랜잭션 | ACID 완전 지원 | 문서 단위 원자성만 |
| 저장 방식 | 행 지향 (Row-oriented) | 문서 지향 + 역색인 분리 |
| 범위 쿼리 숫자/날짜 | B-Tree 범위 탐색 | BKD-Tree 별도 사용 |

---

## ⚖️ 트레이드오프

```
역색인의 이득:
  전문 검색 O(단어 길이 + 결과 수) — B-Tree 불가 영역
  관련성 점수(BM25)로 검색 결과 순위 제공
  분산 처리 — 샤드로 역색인 분산

역색인의 비용:
  인덱싱 시간 — 문서 저장 시 Analyzer 처리 + 역색인 구성
  인덱스 크기 — 원본 데이터보다 역색인이 더 클 수 있음
  UPDATE 비용 — 불변 세그먼트 구조상 수정 = 삭제 + 재삽입
  역방향 조회 불가 — 문서 ID → 필드 값은 doc_values 별도 필요
  트랜잭션 없음 — 여러 문서 원자적 수정 불가

선택 기준:
  "사용자가 입력한 단어가 포함된 문서를 찾고 관련성 순 정렬"
    → Elasticsearch (역색인)

  "특정 조건을 만족하는 레코드를 정확히 찾고 트랜잭션 필요"
    → MySQL (B-Tree)

  현실:
    대부분 둘 다 필요 → MySQL(원본) + ES(검색 인덱스) 조합
```

---

## 📌 핵심 정리

```
B-Tree의 한계:
  LIKE '%word%' → Full Scan (앞에 % 있으면 인덱스 사용 불가)
  "포함 검색"은 B-Tree의 정렬 구조와 근본적으로 맞지 않음

역색인의 발상:
  단어 → 문서 ID 목록을 미리 만들어둠 (뒤집힌 매핑)
  검색어를 받으면 Term Dictionary → Posting List → 즉시 반환
  탐색 비용: O(단어 길이 + 결과 수), Full Scan 없음

각자의 강점:
  B-Tree: 정확 일치, 범위, ACID, 낮은 UPDATE 비용
  역색인: 전문 검색, 관련성 점수, 분산 처리

ES는 둘 다 활용:
  text → 역색인 (전문 검색)
  숫자/날짜 → BKD-Tree (범위 쿼리)
  keyword/숫자 → doc_values (정렬·집계)
  _source → 원본 JSON (Fetch Phase 반환)
```

---

## 🤔 생각해볼 문제

**Q1.** Elasticsearch에서 `term` 쿼리와 `match` 쿼리의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

`match` 쿼리는 검색어를 Analyzer로 처리(토큰화, 소문자화 등)한 후 역색인에서 탐색한다. `term` 쿼리는 Analyzer 처리 없이 그대로 역색인에서 탐색한다. text 필드는 인덱싱 시 "Mechanical Keyboard"를 ["mechanical", "keyboard"]로 분리해서 저장한다. `match`로 "Mechanical"을 검색하면 "mechanical"로 변환 후 탐색해서 문서를 찾는다. `term`으로 "Mechanical"(대문자)을 검색하면 역색인에는 소문자 "mechanical"만 있어서 결과가 없다. text 필드엔 `match`, keyword 필드엔 `term`을 사용하는 이유다.

</details>

**Q2.** 역색인 구조상 "가격이 10만 원 이상인 키보드" 같은 복합 쿼리는 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

전문 검색 조건("키보드")은 역색인으로, 숫자 범위 조건("10만 원 이상")은 BKD-Tree로 각각 처리한다. 두 자료구조에서 각각 문서 ID 집합을 추출한 후 교집합(AND)으로 최종 결과를 만든다. `bool` 쿼리의 `must`로 두 조건을 조합하면 ES가 내부적으로 각 조건에 맞는 자료구조를 선택해 처리한다.

</details>

**Q3.** 역색인을 구성하면 인덱싱 속도가 느려지는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

문서 인덱싱 시 Analyzer 체인(Character Filter → Tokenizer → Token Filter)을 통해 텍스트를 토큰으로 분리하고, 각 토큰에 대해 Term Dictionary에 추가하고 Posting List를 업데이트해야 한다. 또한 세그먼트 구조상 인메모리 버퍼에 누적 후 디스크에 불변 세그먼트로 플러시하는 과정이 있다. MySQL INSERT가 단순히 B-Tree에 키-값 하나를 삽입하는 것에 비해, ES 인덱싱은 Analyzer 처리 + 역색인 구성 + doc_values 갱신 + _source 저장의 여러 단계를 거친다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 역색인 구조 완전 분해 ➡️](./02-inverted-index-structure.md)**

</div>
