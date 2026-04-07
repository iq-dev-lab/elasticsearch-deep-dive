# 힙 메모리 튜닝 — RAM 50% 제한과 OS Page Cache

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JVM 힙을 RAM의 50% 이하로 제한해야 하는 이유는 무엇인가?
- 힙을 30GB 이상으로 설정하면 안 되는 이유는 무엇인가?
- G1GC와 ZGC 중 어떤 GC를 선택해야 하는가?
- GC 오버헤드가 발생했을 때 어떤 지표로 감지하는가?
- `_nodes/stats/jvm`으로 GC 현황을 어떻게 모니터링하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

ES 노드가 갑자기 응답 없음 상태가 되거나, GC pause로 인해 클러스터에서 노드가 이탈하는 사고는 힙 설정이 원인인 경우가 많다. "힙을 크게 주면 좋겠지"라는 직관이 실제로는 Lucene의 OS Page Cache를 빼앗아 역효과를 낸다. 힙 크기를 잘못 설정하면 인덱싱 성능, 검색 성능, 클러스터 안정성 모두 영향을 받는다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 힙을 RAM의 75%로 설정
  설정:
    32GB RAM → ES_JAVA_OPTS="-Xms24g -Xmx24g"

  문제:
    힙: 24GB
    남은 RAM: 8GB → OS + 다른 프로세스 + Page Cache
    Lucene 세그먼트가 Page Cache에 들어갈 공간 부족
    → 세그먼트 접근 시마다 디스크 I/O → 검색 느려짐
    → GC: 24GB 힙 관리 → GC pause 길어짐

실수 2: 힙을 32GB 이상으로 설정
  설정:
    64GB RAM → ES_JAVA_OPTS="-Xms40g -Xmx40g"

  문제:
    JVM의 Compressed Ordinary Object Pointer(COOPS):
    힙 크기 < 32GB → 4바이트 포인터 (메모리 효율적)
    힙 크기 ≥ 32GB → 8바이트 포인터 (메모리 2배)
    → 힙 30GB vs 32GB: 30GB가 실제로 더 효율적!

실수 3: -Xms와 -Xmx를 다르게 설정
  설정:
    -Xms4g -Xmx16g

  문제:
    JVM 시작: 4GB 힙
    필요에 따라 최대 16GB까지 동적 확장
    확장 시 OS에서 메모리 할당 → 지연 발생
    확장/축소 반복 → GC 비용 증가

  올바른 설정:
    -Xms = -Xmx (고정 크기)
    JVM 시작 시 최대 힙 미리 확보
    동적 확장 없음 → 예측 가능한 성능
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
힙 설정 공식:

  RAM 64GB 이하:
    힙 = min(RAM × 50%, 30GB)
    → 64GB RAM → 힙 30GB (31~32GB가 아님! COOPS 경계)
    → 32GB RAM → 힙 16GB
    → 16GB RAM → 힙 8GB

  RAM 64GB 이상:
    힙 30GB + 나머지를 Page Cache로
    또는 힙 30GB × N개 노드로 분산 (단일 노드 힙은 30GB 이하)

  elasticsearch.yml:
    # 직접 설정 (권장)
    heap.size 설정 또는 jvm.options:
    -Xms16g
    -Xmx16g

  또는 환경 변수:
    ES_JAVA_OPTS="-Xms16g -Xmx16g"

  Docker:
    ES_JAVA_OPTS="-Xms8g -Xmx8g"
    또는 ES_HEAP_SIZE=8g (편의 변수)

  GC 선택:
    ES 8.x 기본: G1GC
    힙 > 8GB + 낮은 지연 요구: ZGC 고려
    기본값 유지 권장 (테스트 없이 변경 위험)
```

---

## 🔬 내부 동작 원리

### 1. 힙 50% 제한의 이유 — OS Page Cache

```
메모리 분배 관점:

  노드 총 RAM: 32GB
  할당 방식 비교:

  나쁜 설정 (힙 24GB):
    JVM 힙:       24GB → ES 객체, 집계 버킷, fielddata, 쿼리 캐시
    OS + 기타:    2GB
    Page Cache:   6GB  ← Lucene 세그먼트 캐시 공간 부족!

  좋은 설정 (힙 16GB):
    JVM 힙:       16GB → ES 객체, 집계 버킷, fielddata, 쿼리 캐시
    OS + 기타:    2GB
    Page Cache:   14GB ← Lucene 세그먼트를 충분히 캐시

Page Cache와 Lucene 성능:
  Lucene 세그먼트 파일(.tim, .doc, .dvd, .fdt)
  → mmap으로 접근 → OS Page Cache에 올라옴
  → Page Cache HIT: RAM 속도로 데이터 접근 (~100 ns)
  → Page Cache MISS: 디스크 읽기 후 캐시 (~100 μs, 1000배 느림)

  Page Cache가 클수록:
    더 많은 세그먼트 파일이 메모리에 상주
    → 검색 시 디스크 I/O 없이 응답 → 수십 ms → 수 ms
    → 인덱싱 시 doc_values, Posting List 읽기도 빠름

힙 안에 있는 것:
  ES 내부 Java 객체 (클러스터 상태, 라우팅 테이블)
  집계 버킷 (Terms Aggregation 결과 등)
  fielddata (text 필드 집계 시)
  Query Cache (Filter Cache 비트셋 일부)
  기타 Java 구조체

힙 밖에 있는 것 (OS Page Cache):
  Lucene 세그먼트 파일 전체
  doc_values (.dvd 파일)
  Posting List (.doc 파일)
  Term Dictionary (.tim 파일, FST)
  → 이것들이 핵심 검색 성능을 결정

결론:
  힙을 줄이면 → Page Cache 늘어남 → 세그먼트 캐시 hit율 증가 → 검색 빠름
  힙을 늘리면 → Page Cache 줄어남 → 세그먼트 disk I/O 증가 → 검색 느림
```

### 2. 32GB 힙 한계 — Compressed OOPs

```
JVM Compressed OOPs (Ordinary Object Pointers):

  목적: 힙 오브젝트 참조를 4바이트로 표현 (8바이트 대신)
  조건: 힙 크기 < 약 32GB (정확한 임계값은 JVM 버전/OS마다 다름)

  힙 < 32GB:
    포인터 = 4바이트 × 객체 수
    예: 1,000만 객체 × 4바이트 = 40MB

  힙 ≥ 32GB:
    포인터 = 8바이트 × 객체 수
    예: 1,000만 객체 × 8바이트 = 80MB
    → 힙 사용량 급증 → 같은 힙에 더 적은 객체

  실제 효과:
    힙 30GB: CompressedOOPs 사용 → 효율적
    힙 32GB: 일반 OOPs → 힙 사용량 증가
    → 32GB 힙이 30GB보다 실제로 적은 객체를 수용

  임계값 확인:
    -verbose:gc 로그에서 "CompressedOOPs" 메시지 확인
    또는 jvm.options에 -XX:+UseCompressedOops (기본 활성화)

  권장:
    힙 최대 30GB (안전한 CompressedOOPs 범위)
    RAM이 64GB 이상이면 30GB 힙 + 많은 Page Cache
    또는 노드를 분산 (30GB × 3 노드 = 90GB 힙 총량)
```

### 3. GC 알고리즘 선택 — G1GC vs ZGC

```
G1GC (Garbage First GC):
  ES 8.x 기본 GC
  특징:
    힙을 Region으로 나누어 관리
    목표 pause time 설정 가능 (-XX:MaxGCPauseMillis=200)
    Young/Old 세대 개념 유지
    대부분의 ES 워크로드에 잘 작동
  pause 시간: 수십 ms ~ 수백 ms (힙 크기에 따라)
  적합: 일반적인 ES 운영

ZGC (Z Garbage Collector):
  극저지연 GC (ES 7.x+에서 지원)
  특징:
    Concurrent GC (대부분의 GC 작업을 Stop-The-World 없이)
    목표: pause time < 10ms (힙 크기 무관)
    힙 크기 클수록 G1GC 대비 이점 커짐
  적합: 실시간성 중요 (검색 지연 < 100ms 요구)
  주의: 처리량 약간 감소 (GC 작업에 CPU 지속 사용)

설정:
  jvm.options (ES 8.x):
    -XX:+UseG1GC      ← 기본값 (명시 불필요)

  ZGC로 변경:
    -XX:-UseG1GC
    -XX:+UseZGC

  주의:
    GC 변경은 충분한 테스트 없이 프로덕션에 적용 위험
    기본 G1GC가 대부분의 경우 최선
    ZGC는 검색 지연이 크리티컬한 경우에만 고려

GC 로그 활성화:
  jvm.options:
    -Xlog:gc*,gc+age=trace:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m
  → GC 패턴 분석에 필수
```

### 4. GC 문제 진단

```
GC 지표 모니터링:
  GET _nodes/stats/jvm?pretty

  주요 지표:
    jvm.gc.collectors.young.collection_count:           Young GC 횟수
    jvm.gc.collectors.young.collection_time_in_millis:  Young GC 누적 시간
    jvm.gc.collectors.old.collection_count:             Old GC 횟수
    jvm.gc.collectors.old.collection_time_in_millis:    Old GC 누적 시간

  계산:
    Young GC 평균 시간 = young_time / young_count
    Old GC 평균 시간 = old_time / old_count

  위험 신호:
    Old GC 빈번 (분당 1~2회 이상) → 힙 부족
    Old GC pause 길음 (1초 이상) → 검색 지연 직접 영향
    Young GC pause 비율 > 5% → 힙 또는 객체 생성 패턴 문제

GC overhead 경보:
  JVM이 GC에 시간의 98% 이상 사용하면 → OutOfMemoryError: GC overhead limit exceeded
  → ES가 OOM으로 인식하고 노드 재시작 (또는 프로세스 종료)

힙 덤프 분석:
  jvm.options 추가:
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/var/log/elasticsearch/

  OOM 발생 시 자동으로 heap dump 생성
  → Eclipse MAT, VisualVM으로 분석
  → 어떤 객체가 힙을 점유하는지 파악

실시간 힙 상태:
  GET _nodes/stats/jvm?pretty | grep heap_used_percent
  → 현재 힙 사용률

  연속 모니터링:
    while true; do
      curl -s "http://localhost:9200/_nodes/stats/jvm" | \
        python3 -c "import sys,json; d=json.load(sys.stdin); \
        [print(n,v['jvm']['mem']['heap_used_percent'],'%') \
        for n,v in d['nodes'].items()]"
      sleep 5
    done
```

---

## 💻 실전 실험

```bash
# 현재 힙 설정 확인
GET _nodes/jvm?pretty
# nodes[*].jvm.mem.heap_max_in_bytes: 최대 힙 크기
# nodes[*].jvm.version: JVM 버전

# 현재 힙 사용 현황
GET _nodes/stats/jvm?pretty

# 핵심 지표만 추출
GET _cat/nodes?v&h=name,heap.percent,heap.current,heap.max

# GC 통계 확인
GET _nodes/stats/jvm?pretty
# jvm.gc.collectors.young.collection_count
# jvm.gc.collectors.old.collection_count
# jvm.gc.collectors.old.collection_time_in_millis 확인

# 힙 사용률이 높은 경우 fielddata 확인
GET _nodes/stats/indices/fielddata?pretty

# fielddata 강제 제거 (힙 압박 즉각 해소)
POST _cache/clear?fielddata=true

# Circuit Breaker 상태 (힙 보호 확인)
GET _nodes/stats/breaker?pretty
# tripped_count > 0 이면 Circuit Breaker 동작 중

# CompressedOOPs 확인 (힙이 32GB 미만이면 활성화)
GET _nodes/jvm?pretty
# 응답에서 jvm.using_compressed_ordinary_object_pointers 확인

# OS 메모리 전체 현황 (Page Cache 포함)
GET _nodes/stats/os?pretty
# os.mem.used_in_bytes, os.mem.free_in_bytes
# → used + free = total (Page Cache는 free에 포함될 수 있음)
```

---

## 📊 힙 크기별 권장 설정

| RAM | 힙 크기 | Page Cache | 비고 |
|-----|-------|-----------|------|
| 8GB | 4GB | 3~4GB | 소규모 개발 |
| 16GB | 8GB | 7~8GB | 중소규모 |
| 32GB | 16GB | 14~15GB | 일반 프로덕션 |
| 64GB | 30GB | 32GB+ | 최적 (CompressedOOPs 유지) |
| 128GB | 30GB | 95GB+ | 힙은 30GB 고정, Page Cache로 활용 |
| 256GB | 30GB | 220GB+ | 노드 분산 또는 30GB 힙 고정 |

---

## ⚖️ 트레이드오프

```
힙 크게:
  이득: 많은 Java 객체 (복잡한 집계, 큰 fielddata)
  비용: Page Cache 감소 → 세그먼트 접근 느림
        GC pause 증가 (힙 클수록 GC 시간 증가)
        32GB 초과 시 CompressedOOPs 비활성화

힙 작게:
  이득: Page Cache 증가 → 검색 빠름
        GC pause 감소
  비용: 복잡한 집계, 대용량 fielddata 사용 제한
        Circuit Breaker 더 자주 동작

G1GC vs ZGC:
  G1GC: 처리량 높음, pause 예측 가능
  ZGC: pause 극소, 처리량 약간 낮음
  → 대부분의 ES 워크로드: G1GC (기본값) 유지

-Xms = -Xmx (고정 힙):
  이득: 예측 가능한 성능, 동적 확장 없음
  비용: 시작부터 최대 메모리 점유
  → 항상 권장 (가변 힙 피하기)
```

---

## 📌 핵심 정리

```
힙 50% 제한의 이유:
  나머지 50% → OS Page Cache
  Lucene 세그먼트가 Page Cache에 상주 → 디스크 I/O 없이 검색
  힙 줄이면 Page Cache 늘어남 → 검색 성능 향상

32GB 힙 한계 (CompressedOOPs):
  힙 < 32GB: 4바이트 포인터 → 힙 효율 높음
  힙 ≥ 32GB: 8바이트 포인터 → 힙 효율 낮음
  → 힙 최대 30GB 권장

힙 설정 공식:
  min(RAM × 50%, 30GB)
  -Xms = -Xmx (고정)

GC 선택:
  기본: G1GC (ES 8.x 기본, 대부분 최선)
  극저지연: ZGC (충분한 테스트 후 적용)

GC 모니터링:
  _nodes/stats/jvm: Old GC 횟수/시간 추적
  Old GC 빈번(분당 1+) → 힙 부족 신호
  힙 85%+ → 위험, fielddata 제거 또는 힙 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 64GB RAM 서버에서 힙을 30GB로 설정했는데 검색이 느리다. Page Cache가 원인인지 확인하는 방법은?

<details>
<summary>해설 보기</summary>

`GET _nodes/stats/os?pretty`에서 `os.mem.free_in_bytes`를 확인한다. 실제 사용 가능한 메모리가 낮다면 Page Cache가 충분하지 않다. 더 직접적인 방법은 ES 노드에서 `free -h` 명령으로 buff/cache(=Page Cache) 크기를 확인하는 것이다. Page Cache가 작으면 Lucene 세그먼트 접근 시마다 디스크 I/O가 발생한다. `_nodes/stats/indices/store`의 `size_in_bytes`와 Page Cache 크기를 비교해, 인덱스 데이터가 Page Cache에 들어갈 수 있는지 판단한다.

</details>

**Q2.** Young GC는 자주 발생하지만 Old GC는 거의 없다. 이 상황은 문제인가?

<details>
<summary>해설 보기</summary>

Young GC가 자주 발생하는 것은 일반적인 현상으로 Young Generation에 단명 객체(검색 요청마다 생성되는 임시 객체)가 생성되고 빠르게 수집된다. Young GC는 보통 수 ms 이내로 빠르고 서비스에 큰 영향을 주지 않는다. 반면 Old GC는 장수 객체(fielddata, 클러스터 상태)가 Old Generation으로 이동 후 수집되는 것으로, 이게 적은 것은 좋은 신호다. 다만 Young GC가 너무 빈번(초당 수 회)하고 pause time이 길다면 Young Generation 크기 조정이나 객체 생성 패턴을 검토해야 한다.

</details>

**Q3.** AWS EC2 `r5.8xlarge` (256GB RAM)에서 ES를 운영할 때 힙 설정은?

<details>
<summary>해설 보기</summary>

단일 노드라면 힙 30GB, 나머지 225GB를 Page Cache로 활용한다. 256GB RAM에서 힙을 30GB로만 설정하는 것이 아깝게 느껴질 수 있지만, 힙 30GB를 넘기면 CompressedOOPs 문제가 발생하고 GC pause도 증가한다. 대안으로 동일 머신에 ES 노드를 2개 실행할 수도 있다(각 힙 30GB × 2 = 60GB 힙, 나머지 195GB Page Cache). 이 경우 클러스터 수준의 고가용성은 별도로 설계해야 한다. 실제로 대형 서비스에서는 96~128GB RAM 서버에 30GB 힙을 설정하는 것이 Page Cache와 GC 효율의 균형점으로 권장된다.

</details>

---

<div align="center">

**[⬅️ 이전: 클러스터 성능 진단](./02-cluster-performance-diagnosis.md)** | **[홈으로 🏠](../README.md)** | **[다음: 쓰기 성능 최적화 ➡️](./04-write-performance.md)**

</div>
