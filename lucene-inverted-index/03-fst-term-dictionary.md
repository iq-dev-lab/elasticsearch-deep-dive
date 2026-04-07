# FST — Term Dictionary를 메모리에 압축 저장하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Term Dictionary를 HashMap이 아닌 FST(Finite State Transducer)로 저장하는 이유는?
- FST가 공통 접두사와 접미사를 공유해 메모리를 절약하는 원리는 무엇인가?
- FST에서 O(len) 탐색이 보장되는 이유는 무엇인가?
- Lucene이 직접 FST를 구현한 배경과 다른 구현체와의 차이는?
- 접두사 쿼리(prefix query)가 FST에서 어떻게 효율적으로 처리되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

ES 클러스터에서 힙 사용량이 계속 높은 이유 중 하나는 Term Dictionary 크기다. 수천만 개의 고유 단어를 가진 인덱스에서 Term Dictionary가 어떻게 압축되는지 모르면 힙 설정이나 인덱스 설계를 최적화할 근거가 없다. FST는 추상적인 CS 개념처럼 보이지만, 실제로 ES의 검색 속도와 메모리 사용량을 결정하는 핵심 자료구조다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
오해 1: "Term Dictionary = HashMap이겠지"
  실제 HashMap이라면:
    키: 단어 문자열
    값: Posting List 위치 포인터

    문제:
      단어 1,000만 개 × 평균 10바이트 = 100MB 키 저장
      HashMap 오버헤드 (버킷 배열, 포인터) = 추가 수십 MB
      Java 객체 헤더 = 단어당 최소 16바이트 추가 오버헤드
      → 실제 단어 크기보다 2~5배 메모리 필요

      더 큰 문제:
      HashMap은 전체를 힙에 올려야 함
      → GC 대상 → GC pause 유발
      → ES 힙에 큰 Term Dictionary = GC 지옥

오해 2: "접두사 탐색에 Trie(트라이)를 쓰면 되지"
  Trie의 문제:
    각 문자마다 노드 생성
    "mechanical"과 "mechanism"의 공통 접두사 "mechani" 공유는 가능
    하지만 접미사는 공유 불가
    → 단어마다 별도 노드 체인 → 메모리 비효율

    예:
      "car" → c-a-r (3 노드)
      "cat" → c-a-t (공유 2개 + 새 1개)
      "bat" → b-a-t (1개 공유 없음)
    접미사 "at"는 "cat"과 "bat"에서 공유 불가

오해 3: "FST는 너무 복잡해서 실제로는 안 쓰겠지"
  실제:
    Lucene은 FST를 핵심 자료구조로 사용
    Lucene의 FST 구현: org.apache.lucene.util.fst 패키지
    Term Dictionary뿐 아니라 Suggest(자동완성), Synonym에도 활용
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
FST의 특성을 활용한 설계:

  고유 단어 수 최소화:
    Mapping Explosion → 고유 단어 수 폭증 → FST 크기 증가
    불필요한 동의어, n-gram 설정 → Term Dictionary 비대화
    → FST가 아무리 효율적이어도 단어 수 자체를 줄이는 것이 최우선

  접두사 쿼리 최적화:
    prefix query → FST 접두사 탐색 → 효율적
    wildcard query (?*)  → FST를 통한 열거 → 비용 있음
    regex query → FST로 처리 어려움 → 피할 것

  segments와 FST:
    각 세그먼트마다 독립적인 FST 보유
    세그먼트 많을수록 FST 메모리 합산 증가
    → 세그먼트 병합(Merge)이 FST 수를 줄이는 효과도 있음
```

---

## 🔬 내부 동작 원리

### 1. FST(Finite State Transducer)란 무엇인가

```
FST = 유한 상태 변환기

  Finite State Automaton(FSA, 유한 상태 인식기)의 확장
    FSA: 문자열이 집합에 속하는지 인식 (yes/no)
    FST: 문자열을 입력으로 받아 출력값(포인터 등)을 반환

  핵심 특성:
    ① 접두사 공유: 공통 시작 부분을 하나의 경로로 표현
    ② 접미사 공유: 공통 끝 부분을 하나의 상태로 공유 (Trie와의 차이!)
    ③ 결정적: 하나의 입력 → 하나의 출력, 분기 없음
    ④ 최소화: 같은 언어를 인식하는 가장 작은 오토마톤

  Trie vs FST:
    Trie: 접두사만 공유
    FST:  접두사 + 접미사 모두 공유 → 훨씬 적은 노드
```

### 2. FST 구축 과정 — 단어 4개 예시

```
단어 집합 (정렬된 상태 필수):
  "car"  → 포인터 A
  "cat"  → 포인터 B
  "dog"  → 포인터 C
  "dot"  → 포인터 D

Trie로 표현하면:
         [root]
        /      \
       c        d
       |        |
       a        o
      / \      / \
     r   t    g   t
   (A) (B)  (C) (D)

  노드 수: 9개

FST로 표현하면 (접미사 공유 추가):

         [root]
        /      \
       c        d
       |        |
       a        o
      / \      / \
     r   t    g   t
   (A)  |   (C)  |
        └────┘
         공유 상태 → 포인터 B 또는 D 출력

  핵심: "cat"의 "t"와 "dot"의 "t"가 공통 접미사
         → 같은 상태 노드 공유!
  노드 수: 7개 (Trie보다 2개 절약)

실제 대규모에서의 효과:
  단어 1,000만 개, 평균 길이 8자
  Trie: 약 8,000만 노드 × 노드 크기 = 수 GB
  FST: 공통 접두사·접미사 공유로 수십~수백 MB로 압축
  → HashMap 대비 10~50배 메모리 효율
```

### 3. FST 탐색 — O(len) 보장

```
탐색: "keyboard"를 Term Dictionary FST에서 찾기

  FST 탐색 과정:
    시작 상태 S0
    읽기: 'k' → S0에서 'k' 전이 → S1
    읽기: 'e' → S1에서 'e' 전이 → S2
    읽기: 'y' → S2에서 'y' 전이 → S3
    읽기: 'b' → S3에서 'b' 전이 → S4
    읽기: 'o' → S4에서 'o' 전이 → S5
    읽기: 'a' → S5에서 'a' 전이 → S6
    읽기: 'r' → S6에서 'r' 전이 → S7
    읽기: 'd' → S7에서 'd' 전이 → S8 (최종 상태)
    → 포인터 P 반환 (Posting List 위치)

  비용: 단어 길이만큼만 상태 전이 = O(len)
  "keyboard" = 8자 → 8번 상태 전이로 완료

  O(len)이 보장되는 이유:
    각 문자마다 정확히 하나의 전이만 발생 (결정적 오토마톤)
    전체 단어 수(N)와 무관
    단어 수 100만 → 1,000만이 되어도 탐색 비용 변화 없음

  HashMap과 비교:
    HashMap: O(1) 평균, but 해시 충돌 시 O(len) 재계산 + 버킷 탐색
    FST:     O(len) 항상 보장, 메모리 효율 훨씬 높음
    → 대용량에서 FST가 더 예측 가능한 성능
```

### 4. 접두사 탐색 — prefix query의 효율

```
접두사 탐색: "mech"로 시작하는 모든 단어 찾기

  FST에서:
    S0 -m→ S1 -e→ S2 -c→ S3 -h→ S4 [접두사 완료]
    S4에서 도달 가능한 모든 최종 상태 열거
    → "mechanical", "mechanism", "mechanic", ...
    → 해당 단어들의 Posting List 포인터 수집

  비용: O(접두사 길이 + 결과 단어 수)
  → prefix 쿼리가 wildcard(*)보다 빠른 이유

  wildcard 쿼리 (*mech*):
    앞에 * 있음 → 접두사 활용 불가
    FST 전체 열거 또는 별도 처리 필요
    → 훨씬 비쌈
    → "*로 시작하는 wildcard는 Full Scan에 가까움"

  접미사 탐색 (~ical로 끝나는 단어):
    표준 FST: 접미사 탐색 어려움
    → 역방향 FST (단어를 뒤집어 저장) 별도 구축 필요
    ES: 일반적으로 suffix 전용 쿼리보다 n-gram 방식 권장
```

### 5. Lucene FST 구현의 특징

```
Lucene FST (org.apache.lucene.util.fst):

  특징 1: 오프-힙(Off-Heap) 직렬화
    FST를 바이트 배열로 직렬화하여 메모리 매핑(mmap) 가능
    → JVM 힙이 아닌 OS 파일 시스템 캐시에 올라감
    → GC 영향 없음
    → 단, 힙에 올려두는 경우도 있음 (세그먼트 크기에 따라)

  특징 2: 빌드 순서 제약
    FST는 정렬된 입력만 처리 가능 (Builder 패턴)
    → Term Dictionary는 정렬된 단어 순서로 구성
    → Lucene이 세그먼트 내 단어를 항상 정렬된 상태로 유지하는 이유

  특징 3: 아크(Arc) 인코딩
    상태 전이(아크)를 compact하게 인코딩
    고정 배열(Fixed Array Arc): 전이가 많은 상태에 사용 → 배열 인덱스로 O(1) 접근
    가변 인코딩: 전이가 적은 상태 → 공간 절약

  특징 4: 출력값 공유
    아크에 출력값(partial output)을 분산 저장
    공통 접두사 부분의 출력을 앞에 배치
    → 전체 Posting List 포인터를 하나씩 복사하지 않고
       경로를 따라가며 조각을 합산

  Lucene FST vs 일반 FST:
    일반 학술 FST: 이론적 구조, 실용적 구현 고려 적음
    Lucene FST: 직렬화, mmap, compact 인코딩, 순서 제약
               → 실무 검색 엔진에 맞게 최적화된 구현
```

### 6. Term Dictionary에서 Posting List 탐색까지 전체 흐름

```
검색어 "elasticsearch" 탐색 전체 흐름:

  1. 메모리에 올라온 FST (Term Dictionary 인덱스)
     "elasticsearch" → 블록 오프셋 P

  2. 디스크의 .tim 파일 (Term Dictionary 실제 데이터)
     블록 오프셋 P에서 "elasticsearch" 엔트리 찾기
     → Posting List 파일(.doc) 오프셋 Q 획득

  3. 디스크의 .doc 파일 (Posting List)
     오프셋 Q에서 delta-encoded 문서 ID + TF 목록 읽기
     → [doc_1, doc_42, doc_117, ...] 복원

  4. (phrase 쿼리이면) .pos 파일에서 Position 읽기

  5. BM25 점수 계산 후 상위 K개 반환

  중요:
    FST는 메모리 상주 (빠른 포인터 조회)
    실제 Posting List는 디스크 (OS Page Cache 활용)
    → FST 크기를 줄이면 힙 절약 + Page Cache 더 활용 가능
```

---

## 💻 실전 실험

```bash
# FST 효과 간접 확인: Term Dictionary 메모리 사용량
GET _nodes/stats/indices/segments?pretty
# segments.memory_in_bytes: 세그먼트 관련 메모리 (FST 포함)
# segments.terms_memory_in_bytes: Term Dictionary 메모리

# 인덱스별 세그먼트 메모리 상세
GET /search-test/_stats/segments?pretty

# 단어 수 확인 (FST 크기에 비례)
GET /search-test/_stats/fielddata?pretty
GET _cat/fielddata?v&fields=*

# prefix 쿼리 (FST 접두사 탐색 활용)
GET /search-test/_search
{
  "query": {
    "prefix": { "title": "mech" }
  }
}

# wildcard 쿼리와 비교 (앞에 * = FST 활용 불가)
GET /search-test/_search
{
  "query": {
    "wildcard": { "title": "*board*" }
  }
}

# 두 쿼리를 profile로 비교
GET /search-test/_search
{
  "profile": true,
  "query": { "prefix": { "title": "mech" } }
}

# 세그먼트 수에 따른 FST 수 확인
GET /search-test/_segments?pretty
# num_search_segments: 현재 세그먼트 수
# 각 세그먼트마다 독립적인 FST 존재

# 강제 병합으로 세그먼트 수 줄이기 (FST 수도 감소)
POST /search-test/_forcemerge?max_num_segments=1
```

---

## 📊 Term Dictionary 자료구조 비교

| 자료구조 | 탐색 비용 | 메모리 효율 | 접두사 탐색 | GC 영향 |
|---------|---------|-----------|----------|--------|
| HashMap | O(1) 평균 | 낮음 (오버헤드 큼) | 불가 | 큼 (힙 상주) |
| Trie | O(len) | 중간 (접두사만 공유) | 가능 | 큼 (힙 상주) |
| **FST** | **O(len)** | **매우 높음 (접두사+접미사)** | **가능** | **낮음 (mmap 가능)** |
| 정렬 배열 | O(log N) | 높음 | 어려움 | 낮음 |

---

## ⚖️ 트레이드오프

```
FST의 이득:
  메모리 효율: HashMap 대비 10~50배 압축
  예측 가능한 O(len) 탐색
  접두사 쿼리 자연스럽게 지원
  mmap 가능 → GC 영향 최소화

FST의 비용:
  구성 시간: 빌드가 HashMap보다 복잡 (정렬 필요, 최소화 알고리즘)
  수정 불가: 구성 후 수정 불가 → 세그먼트가 불변인 이유와 연결
  구현 복잡도: HashMap보다 훨씬 복잡한 구현

수정 불가와 세그먼트 불변성의 관계:
  FST는 한 번 구축하면 수정 불가 자료구조
  → Term Dictionary를 수정하려면 세그먼트 전체를 다시 만들어야 함
  → 이것이 Lucene 세그먼트가 불변인 근본 이유 중 하나
  → 새 문서 → 새 세그먼트 + 새 FST 구축
  → 세그먼트 병합 → 기존 FST들을 합쳐 새 FST 구축
```

---

## 📌 핵심 정리

```
FST = Finite State Transducer:
  정렬된 단어 목록을 오토마톤으로 표현
  접두사 + 접미사 동시 공유 → Trie보다 압축률 높음
  탐색: O(len), 단어 수(N)와 무관

Term Dictionary에서 FST 역할:
  수천만 단어를 수십 MB로 압축
  mmap 지원 → 힙 외부에 올릴 수 있음 → GC 부담 감소
  접두사 탐색 지원 → prefix 쿼리 효율적 처리

FST와 세그먼트 불변성:
  FST는 구축 후 수정 불가
  → Term Dictionary 수정 = 세그먼트 재구성
  → Lucene이 세그먼트를 불변으로 설계한 이유와 연결

실무 영향:
  고유 단어 수 줄이기 → FST 크기 → 힙 사용량 감소
  세그먼트 병합 → FST 수 감소 → 메모리 효율 향상
  prefix 쿼리 선호, 앞에 * 있는 wildcard 지양
```

---

## 🤔 생각해볼 문제

**Q1.** 한국어 인덱스에서 형태소 분석을 하지 않고 n-gram(2-gram)으로 인덱싱하면 FST 크기에 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

n-gram 방식은 "검색엔진"을 "검색", "색엔", "엔진"으로 분리하는 대신, 2-gram이면 "검색", "색엔", "엔진"처럼 모든 연속 문자 쌍을 토큰으로 만든다. 형태소 분석보다 훨씬 많은 고유 토큰이 생성되며, Term Dictionary의 단어 수가 급격히 증가한다. FST는 단어 수에 비례해 크기가 커지므로 n-gram 인덱싱은 FST 크기, 나아가 힙 사용량과 세그먼트 크기를 크게 늘린다. 검색 품질은 일부 향상되지만 메모리와 스토리지 비용이 상당히 증가한다는 트레이드오프가 있다.

</details>

**Q2.** 세그먼트 병합(Merge)이 검색 성능에 미치는 영향은 무엇인가?

<details>
<summary>해설 보기</summary>

검색 시 Coordinating이 각 샤드의 모든 세그먼트에 대해 쿼리를 실행한다. 세그먼트 10개 → 10개 FST 탐색 + 10개 Posting List 병합. 세그먼트 1개로 병합하면 1개 FST 탐색 + 1개 Posting List만 처리하면 된다. 또한 세그먼트가 많으면 각 FST가 메모리를 차지하는데, 병합으로 FST 수가 줄면 힙 또는 OS Page Cache 절약 효과도 있다. 단, 병합 작업 자체가 CPU와 I/O를 많이 사용하므로 운영 중 강제 병합(`_forcemerge`)은 신중하게 실행해야 한다.

</details>

**Q3.** `fuzzy` 쿼리("keyboard"를 "kyeboard"로 오타 입력해도 찾아주는)는 FST에서 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

FST는 정확한 문자 경로 탐색에 최적화되어 있어 fuzzy 탐색을 직접 지원하지 않는다. Lucene의 fuzzy 쿼리는 Levenshtein 오토마톤을 FST와 결합해 처리한다. 허용 편집 거리(예: 1)에 해당하는 모든 가능한 변형을 FST에서 열거하는 방식이다. 비용이 일반 term 쿼리보다 높지만, 사용자 오타 허용이 중요한 경우 실용적인 접근이다. 편집 거리가 커질수록(fuzzy: 2 이상) 후보가 기하급수적으로 늘어 성능 부담이 증가한다.

</details>

---

<div align="center">

**[⬅️ 이전: 역색인 구조 완전 분해](./02-inverted-index-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: Lucene Segment — 불변 세그먼트와 병합 정책 ➡️](./04-lucene-segment.md)**

</div>
