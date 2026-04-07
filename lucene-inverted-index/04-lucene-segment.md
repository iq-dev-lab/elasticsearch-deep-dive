# Lucene Segment — 불변 세그먼트와 병합 정책

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Lucene 세그먼트가 불변(Immutable)으로 설계된 이유는 무엇인가?
- 문서 삭제와 수정이 실제로 어떻게 처리되는가? (삭제 마킹의 실체)
- 세그먼트 병합(Merge)은 언제, 어떻게 발생하며 무엇을 해결하는가?
- TieredMergePolicy는 세그먼트를 어떤 기준으로 병합하는가?
- 삭제된 문서가 많을 때 검색 성능이 저하되는 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"문서를 1,000만 건 삭제했는데 디스크 공간이 안 줄어요", "업데이트를 많이 했더니 검색이 느려졌어요", "force merge는 언제 써야 하나요?" — 이 질문들의 답이 전부 세그먼트 불변성과 병합 정책에 있다. 세그먼트가 어떻게 생성되고 병합되는지 모르면 인덱스 크기 예측도, 성능 튜닝도 근거 없는 조작이 된다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: DELETE 후 디스크가 줄지 않는다고 당황
  상황:
    1,000만 건 DELETE 요청 완료
    disk usage: 변화 없음

  이유:
    DELETE = 삭제 마킹(Live Documents 비트셋에 0 표시)
    실제 데이터는 세그먼트에 그대로 존재
    → 세그먼트 병합 시 삭제 마킹 문서를 제외하고 새 세그먼트 생성
    → 그때 비로소 디스크 반환

  해결:
    _forcemerge 호출 → 세그먼트 즉시 병합 → 삭제 마킹 문서 제거
    단, 운영 중 _forcemerge는 I/O 집중 → 주의

실수 2: 잦은 UPDATE로 인한 세그먼트 폭증 인식 못함
  패턴:
    하루 100만 건 UPDATE (상품 가격 변경)

  내부에서:
    UPDATE = DELETE 마킹 + 새 문서 INSERT
    → 매 UPDATE마다 삭제 마킹 1개 + 새 세그먼트(또는 버퍼) 1개 문서 추가
    → 삭제 마킹 비율 증가 → 검색 시 불필요한 마킹 문서 건너뜀 비용

  신호:
    _segments API에서 deleted_docs 비율이 높음
    (삭제 마킹 비율 > 20%이면 성능 영향 시작)

실수 3: _forcemerge를 너무 자주 호출
  상황:
    성능 향상을 위해 매 시간 _forcemerge 호출

  문제:
    강제 병합은 CPU, I/O 집중 작업
    병합 중 검색 성능 저하
    → 빠르게 쓰는 인덱스에서 강제 병합은 역효과
    → 자동 병합(TieredMergePolicy)을 신뢰하고
       정적 인덱스(더 이상 쓰기 없음)에만 _forcemerge 사용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
세그먼트 생명주기를 이해한 운영:

  일반 운영:
    TieredMergePolicy가 자동으로 세그먼트 병합
    삭제 비율, 세그먼트 크기를 기준으로 병합 결정
    개입 최소화 → 자동 병합 신뢰

  쓰기 집중 시:
    refresh_interval 늘리기 → 세그먼트 생성 빈도 감소
    → 작은 세그먼트가 적게 생기므로 병합 필요성 감소

  정적 인덱스 (더 이상 쓰기 없는):
    _forcemerge?max_num_segments=1
    → 단일 세그먼트로 병합 → 최적 검색 성능
    → 로그 인덱스, ILM에서 Cold 단계 진입 시 적합

  삭제 마킹 모니터링:
    GET /index/_stats → index.total.docs.deleted 확인
    삭제 비율 = deleted / (docs + deleted)
    20% 이상이면 병합 정책 점검 또는 _forcemerge 고려
```

---

## 🔬 내부 동작 원리

### 1. 세그먼트 불변성의 이유

```
왜 세그먼트는 불변인가?

이유 1: 역색인 재구성 비용
  역색인의 핵심: 단어 → 문서 ID 목록 (정렬됨)
  "keyboard"의 Posting List: [1, 4, 7, ...]
  문서 4를 수정하면:
    "keyboard"의 Posting List에서 4를 제거하고 새 ID 추가
    → Term Dictionary 전체 재구성 필요
    → FST 전체 재구성 필요 (FST는 수정 불가)
    → 모든 세그먼트 파일(.tim, .doc, .pos) 재작성
    → 문서 하나 수정 = 세그먼트 전체 재생성 → 너무 비쌈

이유 2: 동시성 단순화
  불변 세그먼트:
    한 번 작성하면 수정 없음
    → 읽기 스레드가 락 없이 세그먼트 읽기 가능
    → 여러 검색 요청이 동시에 같은 세그먼트를 읽어도 충돌 없음

  가변 세그먼트라면:
    수정 중 읽기 = 잠금 필요
    잠금 = 동시 검색 처리량 감소

이유 3: OS Page Cache 최적화
  불변 파일 = 캐시에 올라간 내용이 변하지 않음
  → OS가 적극적으로 Page Cache에 올림
  → mmap으로 파일을 메모리처럼 접근 가능
  가변 파일: 무효화(invalidation) 관리 필요 → 캐시 효율 저하
```

### 2. 삭제와 수정의 실제 처리

```
삭제 처리 (DELETE):

  DELETE /products/_doc/42

  내부 동작:
    ① 클러스터 상태에서 doc_id=42가 있는 세그먼트 확인
    ② 해당 세그먼트의 .liv 파일 (Live Documents 비트셋) 업데이트
       .liv: 비트 배열, 각 비트 = 해당 doc_id의 생존 여부
       doc_id=42 → 비트 0으로 변경 (삭제 마킹)
    ③ 세그먼트 데이터 자체는 변경 없음

  삭제 마킹 문서의 영향:
    검색 시: 삭제 마킹된 문서는 결과에서 제외
    비용: Posting List를 탐색하다 삭제 마킹 문서를 만나면 건너뜀
         삭제 비율 높을수록 → 건너뜀 횟수 증가 → 탐색 비용 증가

  ┌─────────────────────────────────────────────────────┐
  │           세그먼트 내부 (삭제 후)                        │
  │                                                     │
  │  실제 데이터:  doc_1 | doc_42 | doc_7 | doc_100       │
  │  .liv 비트셋:    1   |   0    |   1   |    1         │
  │                              ↑ 삭제 마킹              │
  │                                                     │
  │  검색: Posting List에서 doc_42 발견                    │
  │        .liv에서 doc_42 비트 확인 → 0 → 건너뜀            │
  └─────────────────────────────────────────────────────┘

수정 처리 (UPDATE):

  PUT /products/_doc/42 { "price": 200 }

  내부 동작:
    ① 기존 doc_42에 삭제 마킹 (.liv 비트 → 0)
    ② 새 문서를 인메모리 버퍼에 추가 (새 doc_id 할당)
    ③ refresh 후 새 세그먼트에 새 doc_id로 저장

  결과:
    기존 세그먼트: doc_42 삭제 마킹 (데이터는 존재, 무효화만)
    새 세그먼트:   새 doc_id로 업데이트된 문서 존재
    → 세그먼트가 늘어남
    → 삭제 마킹 문서 증가 → 병합 필요성 증가
```

### 3. 세그먼트 병합 (Merge) — 삭제 문서 실제 제거

```
병합 트리거:
  TieredMergePolicy가 주기적으로 판단:
    세그먼트 크기가 너무 작은 것이 너무 많음
    삭제 마킹 비율이 임계값 초과
    → 병합 대상 세그먼트 선정

병합 과정:

  Before:
    Segment_0: doc_1, doc_42(삭제), doc_7    (3개 문서, 1개 삭제)
    Segment_1: doc_100, doc_200              (2개 문서)
    Segment_2: doc_300(삭제), doc_400        (2개 문서, 1개 삭제)

  병합 작업:
    ① Segment_0, Segment_1, Segment_2를 병합 대상으로 선정
    ② 살아있는 문서만 읽기 (삭제 마킹 건너뜀):
       doc_1, doc_7, doc_100, doc_200, doc_400
    ③ 새 세그먼트 Segment_3 생성:
       새 역색인 구성 (FST 새로 구축)
       새 doc_id 재할당 (0부터 순차)
       삭제 마킹 문서 완전히 제외
    ④ 기존 세그먼트 파일 삭제 (디스크 반환!)

  After:
    Segment_3: new_doc_0(구 doc_1), new_doc_1(구 doc_7),
               new_doc_2(구 doc_100), new_doc_3(구 doc_200),
               new_doc_4(구 doc_400)
    → 삭제 마킹 문서 완전 제거
    → 디스크 공간 반환
    → 단일 세그먼트 → 검색 탐색 범위 감소

병합의 이득:
  삭제 문서 실제 제거 → 디스크 공간 반환
  세그먼트 수 감소 → 검색 시 탐색 세그먼트 수 감소
  새 FST 구축 → 삭제 문서 없는 깔끔한 역색인
  삭제 마킹 건너뜀 비용 제거

병합의 비용:
  CPU: 새 역색인(FST) 구성
  I/O: 기존 세그먼트 읽기 + 새 세그먼트 쓰기
  일시적으로 두 배 이상의 디스크 공간 필요 (병합 중)
```

### 4. TieredMergePolicy — 병합 결정 알고리즘

```
TieredMergePolicy (Lucene/ES 기본 병합 정책):

핵심 파라미터:
  maxMergeAtOnce:        한 번에 병합할 최대 세그먼트 수 (기본 10)
  segmentsPerTier:       각 계층당 세그먼트 수 (기본 10)
  maxMergedSegmentMB:    병합 후 세그먼트 최대 크기 (기본 5GB)

계층(Tier) 구조:
  세그먼트를 크기로 계층화

  Tier 0 (소형):  < 10MB    → 최대 10개 허용
  Tier 1 (중형):  10~100MB  → 최대 10개 허용
  Tier 2 (대형):  100MB~5GB → 최대 10개 허용

  Tier 0에 11개가 되면:
    → 소형 세그먼트 10개 병합 → 중형 세그먼트 1개 생성
    Tier 1에 합류

병합 스코어 계산:
  병합 후보 선정 기준 (높을수록 병합 우선순위 높음):
    1. 삭제 마킹 비율: deletedRatio 높을수록 병합 급함
    2. 세그먼트 크기 편차: 유사한 크기끼리 병합 효율
    3. 전체 세그먼트 수: 많을수록 병합 필요

실제 설정 (ES에서 조정 가능):
  PUT /myindex/_settings
  {
    "index.merge.policy.max_merge_at_once": 10,
    "index.merge.policy.segments_per_tier": 10,
    "index.merge.policy.max_merged_segment": "5gb",
    "index.merge.policy.expunge_deletes_allowed": 10
  }

  expunge_deletes_allowed: 삭제 비율 몇 % 이상이면 병합 강제?
    기본 10% → 삭제 마킹이 10% 넘으면 병합 우선순위 높아짐
```

### 5. 세그먼트와 검색 성능의 관계

```
세그먼트 수와 검색 비용:

  세그먼트 10개:
    각 세그먼트마다 별도 FST 탐색
    각 Posting List를 merge (10개 결과를 합산)
    → 10배 탐색 비용 (단일 세그먼트 대비)

  세그먼트 1개 (forcemerge 후):
    단일 FST 탐색
    단일 Posting List 읽기
    → 최소 탐색 비용

  실제로는 OS Page Cache가 완화:
    자주 읽는 세그먼트 → Page Cache에 상주
    세그먼트가 많아도 캐시 히트율 높으면 영향 적음

  삭제 마킹 비율과 검색 비용:
    Posting List: [1, 4(삭제), 7, 42(삭제), 100]
    탐색 중 삭제 마킹 문서를 만날 때마다 .liv 파일 조회 + 건너뜀
    삭제 비율 50% → Posting List 절반이 버려짐 → 비효율
```

---

## 💻 실전 실험

```bash
# 현재 세그먼트 상태 확인
GET /search-test/_segments?pretty
# 각 세그먼트의:
#   num_docs: 살아있는 문서 수
#   deleted_docs: 삭제 마킹된 문서 수
#   size_in_bytes: 세그먼트 크기
#   committed: 디스크 fsync 여부

# 세그먼트 요약 통계
GET _cat/segments/search-test?v&h=index,shard,segment,docs.count,docs.deleted,size

# 삭제 마킹 문서 생성 실험
DELETE /search-test/_doc/1
DELETE /search-test/_doc/2
DELETE /search-test/_doc/3

# 삭제 후 세그먼트 확인 (docs.deleted 증가 확인)
GET _cat/segments/search-test?v&h=segment,docs.count,docs.deleted,size

# 인덱스 통계에서 삭제 비율 확인
GET /search-test/_stats?pretty
# _all.total.docs.count: 살아있는 문서
# _all.total.docs.deleted: 삭제 마킹 문서

# 강제 병합 (삭제 마킹 문서 실제 제거)
POST /search-test/_forcemerge?max_num_segments=1&only_expunge_deletes=false

# 병합 후 세그먼트 확인 (docs.deleted = 0이어야 함)
GET _cat/segments/search-test?v

# 병합 정책 설정 조회
GET /search-test/_settings?pretty
# index.merge.policy.* 설정 확인

# UPDATE 패턴의 세그먼트 영향 실험
# 10번 연속 업데이트 후 세그먼트 확인
PUT /search-test/_doc/10 { "title": "update test v1", "price": 1 }
PUT /search-test/_doc/10 { "title": "update test v2", "price": 2 }
PUT /search-test/_doc/10 { "title": "update test v3", "price": 3 }
# ...

GET _cat/segments/search-test?v
# docs.deleted 증가 확인
```

---

## 📊 삭제 마킹 비율에 따른 영향

| 삭제 비율 | 검색 성능 영향 | 디스크 낭비 | 권장 조치 |
|---------|------------|----------|---------|
| 0~10% | 미미 | 낮음 | 자동 병합 신뢰 |
| 10~30% | 약간 저하 | 중간 | TieredMergePolicy 모니터링 |
| 30~50% | 체감 저하 | 높음 | 병합 정책 튜닝 고려 |
| 50%+ | 명확한 저하 | 매우 높음 | `_forcemerge?only_expunge_deletes=true` |

---

## ⚖️ 트레이드오프

```
불변 세그먼트의 이득:
  동시성 단순화 (락 없는 읽기)
  OS Page Cache 최적화
  FST 등 불변 자료구조 활용 가능

불변 세그먼트의 비용:
  UPDATE/DELETE = 삭제 마킹 + 새 삽입 → 세그먼트 증가
  삭제 마킹 문서 = 디스크 공간 낭비 + 탐색 비용 증가
  병합이 이를 해결하지만 병합 자체가 CPU/I/O 비용

병합 빈도의 트레이드오프:
  병합 자주 → 검색 최적화, 삭제 문서 빠른 제거
           → CPU/I/O 부하, 쓰기 성능 저하
  병합 드물게 → 쓰기 성능 유지
             → 세그먼트 증가, 삭제 마킹 누적

forcemerge vs 자동 병합:
  forcemerge → 즉각적인 최적화, but 운영 중 I/O spike
  자동 병합 → 점진적, 운영 영향 분산
  → 정적 인덱스: forcemerge OK
  → 활성 인덱스: 자동 병합 신뢰
```

---

## 📌 핵심 정리

```
세그먼트 불변성 이유:
  역색인 재구성 비용 (FST 수정 불가)
  동시성 단순화 (락 없는 읽기)
  OS Page Cache 최적화

삭제/수정 처리:
  DELETE → .liv 비트셋에 삭제 마킹 (데이터는 남음)
  UPDATE → 삭제 마킹 + 새 문서 삽입
  실제 제거는 세그먼트 병합 시 발생

병합(Merge)의 역할:
  삭제 마킹 문서 실제 제거 → 디스크 반환
  세그먼트 수 감소 → 검색 성능 향상
  새 FST 구축 → 깔끔한 역색인

TieredMergePolicy:
  세그먼트를 크기로 계층화
  유사 크기끼리 병합 → 효율적인 I/O
  삭제 비율 높으면 병합 우선순위 높임

forcemerge 사용 기준:
  정적 인덱스 (더 이상 쓰기 없음) → OK
  활성 쓰기 인덱스 → 자동 병합 신뢰
```

---

## 🤔 생각해볼 문제

**Q1.** 세그먼트 병합 중에도 검색이 가능한 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

기존 세그먼트는 병합이 완료될 때까지 그대로 유지된다. 병합 과정에서 새 세그먼트가 생성되는 동안 검색은 여전히 기존 세그먼트를 읽는다. 새 세그먼트 생성이 완료되면 Lucene이 atomic하게 IndexReader를 교체한다(세그먼트 목록 갱신). 이후 검색은 새 세그먼트를 읽고, 기존 세그먼트는 더 이상 참조하는 IndexReader가 없을 때 삭제된다. 이 과정에서 락이 필요 없는 이유가 불변 세그먼트 덕분이다.

</details>

**Q2.** `only_expunge_deletes=true`로 forcemerge를 실행하면 무엇이 다른가?

<details>
<summary>해설 보기</summary>

`only_expunge_deletes=true`는 삭제 마킹 문서가 있는 세그먼트만 선택적으로 병합한다. `max_num_segments=1`처럼 모든 세그먼트를 단일 세그먼트로 합치는 것이 아니라, 삭제 문서를 제거하는 것이 목표일 때 I/O 비용을 줄인다. 삭제 마킹이 없는 깨끗한 세그먼트는 그대로 유지되어 불필요한 재구성을 피한다.

</details>

**Q3.** 인덱스에 쓰기는 없고 읽기만 있는 상황에서 단일 세그먼트(`max_num_segments=1`)로 병합하면 어떤 이점이 있는가?

<details>
<summary>해설 보기</summary>

단일 세그먼트이면 검색 시 탐색할 FST가 하나, Posting List도 하나로 줄어든다. 세그먼트 간 결과 merge 연산이 없어지고, 삭제 마킹 문서도 모두 제거된 상태다. 또한 OS는 단일 파일을 통째로 Page Cache에 올리는 것이 여러 파일보다 관리하기 쉬어 캐시 효율이 높다. 시계열 인덱스에서 더 이상 쓰기가 없는 과거 인덱스(예: ILM Cold 단계)를 forcemerge하는 것이 권장되는 이유다.

</details>

---

<div align="center">

**[⬅️ 이전: FST — Term Dictionary 압축 저장](./03-fst-term-dictionary.md)** | **[홈으로 🏠](../README.md)** | **[다음: Near Real-Time 검색 ➡️](./05-near-real-time-search.md)**

</div>
