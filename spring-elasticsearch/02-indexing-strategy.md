# 인덱싱 전략 — 동기·비동기 인덱싱과 Bulk 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단건 인덱싱과 Bulk 인덱싱의 성능 차이가 얼마나 나고 그 이유는?
- Spring에서 비동기 Bulk 인덱싱을 구현하는 실용적인 패턴은?
- 인덱싱 실패 시 재처리 전략(retry, dead-letter)은 어떻게 설계하는가?
- `bulkIndex()`에서 부분 실패가 발생했을 때 어떻게 처리해야 하는가?
- 대용량 초기 적재 시 Spring에서 어떻게 성능을 극대화하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"상품 데이터 10만 건을 ES에 동기화해야 한다"는 요구사항에서 단건 인덱싱 루프로 시작하면 수십 분이 걸린다. Bulk 인덱싱과 비동기 처리를 적용하면 수 분 이내로 줄일 수 있다. 더 중요한 것은 인덱싱이 실패했을 때 데이터 누락 없이 재처리하는 안정적인 설계다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 루프에서 단건 인덱싱
  코드:
    for (Product product : products) {
        productRepository.save(product);  // 매번 HTTP 요청
    }

  성능:
    10만 건 × 10ms/건 = 1,000초 ≈ 17분
    HTTP 연결 10만 번 생성/해제
    트랜잭션 처리 시 Translog fsync 10만 번

실수 2: Bulk 실패를 전체 실패로 처리
  코드:
    try {
        operations.bulkIndex(queries, Product.class);
    } catch (Exception e) {
        log.error("Bulk indexing failed: " + e.getMessage());
        // 전체 실패 처리 → 성공한 문서도 재처리
    }

  문제:
    Bulk API는 부분 성공 가능
    전체 예외가 아닌 각 문서별 성공/실패 확인 필요
    성공한 문서를 다시 인덱싱 → 중복 처리

실수 3: @Transactional과 ES 인덱싱 혼합
  코드:
    @Transactional
    public void saveProductWithSearch(Product product) {
        productRepository.save(product);      // MySQL 저장
        esProductRepository.save(esProduct);  // ES 인덱싱
    }

  문제:
    @Transactional은 MySQL 트랜잭션만 관리
    ES 인덱싱은 트랜잭션 외부 → MySQL 롤백 시 ES는 그대로
    → MySQL/ES 불일치 발생
    올바른 접근: 트랜잭션 완료 후 ES 인덱싱 (TransactionalEventListener)
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
인덱싱 전략 선택:

  단건 실시간 인덱싱 (요청당 1건):
    용도: 사용자 행동 즉시 반영 (리뷰 작성, 상품 등록)
    방식: 비동기 단건 인덱싱 (Non-blocking)
    패턴: @TransactionalEventListener(AFTER_COMMIT) + @Async

  소규모 배치 (10~1000건):
    용도: 변경 감지 후 주기적 동기화
    방식: Bulk 인덱싱 (배치 당 1 HTTP 요청)
    패턴: bulkIndex() + 부분 실패 처리

  대규모 초기 적재 (100만+ 건):
    용도: 초기 데이터 이관, 전체 재인덱싱
    방식: Bulk + 병렬 처리 + 최적화 설정
    패턴: CompletableFuture 병렬 Bulk + refresh_interval=-1
```

---

## 🔬 내부 동작 원리

### 1. 단건 vs Bulk 성능 비교

```
단건 인덱싱 (save/index):
  ┌────────────────────────────────────────────────────┐
  │ 문서 1건마다:                                         │
  │   HTTP 연결 → 요청 전송 → ES 처리 → 응답 → 연결 종료       │
  │   Translog fsync (durability: request 기본)         │
  │   비용: ~5~50ms (로컬), ~50~500ms (원격)              │
  └────────────────────────────────────────────────────┘
  10만 건 × 20ms = 2,000초 = 33분

Bulk 인덱싱 (bulkIndex):
  ┌────────────────────────────────────────────────────┐
  │ 1000건 묶어서:                                       │
  │   HTTP 연결 1번 → 요청 전송 (1000건) → ES 처리          │
  │   → 응답 → 연결 종료                                  │
  │   Translog fsync 1번 (1000건 한번에)                 │
  │   비용: ~50~200ms for 1000건                        │
  └────────────────────────────────────────────────────┘
  10만 건 / 1000건 배치 = 100 HTTP 요청
  100 × 100ms = 10초 → 33분 대비 198배 빠름
```

### 2. @TransactionalEventListener로 안전한 단건 인덱싱

```java
// 도메인 이벤트 정의
public record ProductIndexEvent(Product product) {}

// 서비스: MySQL 저장 + 이벤트 발행
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductJpaRepository productJpaRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Product createProduct(ProductRequest request) {
        Product product = Product.from(request);
        Product saved = productJpaRepository.save(product);

        // 트랜잭션 내부에서 이벤트 발행
        // 실제 실행은 AFTER_COMMIT 이후
        eventPublisher.publishEvent(new ProductIndexEvent(saved));
        return saved;
    }
}

// 이벤트 핸들러: MySQL 커밋 후 ES 인덱싱
@Component
@RequiredArgsConstructor
public class ProductIndexEventHandler {

    private final ElasticsearchOperations operations;

    @Async("esIndexingExecutor")  // 비동기 실행
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleProductIndex(ProductIndexEvent event) {
        try {
            EsProduct esProduct = EsProduct.from(event.product());
            IndexQuery query = new IndexQueryBuilder()
                .withId(esProduct.getId())
                .withObject(esProduct)
                .build();
            operations.index(query, IndexCoordinates.of("products"));

        } catch (Exception e) {
            // 실패 시 재처리 큐로 전달
            log.error("ES indexing failed for product {}: {}",
                event.product().getId(), e.getMessage());
            // retry 또는 dead-letter 처리
        }
    }
}

// 비동기 ThreadPool 설정
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("esIndexingExecutor")
    public Executor esIndexingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("es-index-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

### 3. Bulk 인덱싱 + 부분 실패 처리

```java
@Service
@RequiredArgsConstructor
public class ProductBulkIndexService {

    private final ElasticsearchOperations operations;
    private final FailedIndexRepository failedIndexRepository;

    private static final int BATCH_SIZE = 1000;
    private static final int MAX_RETRY = 3;

    /**
     * Bulk 인덱싱 + 부분 실패 처리
     */
    public BulkIndexResult bulkIndex(List<EsProduct> products) {
        List<IndexQuery> queries = products.stream()
            .map(p -> new IndexQueryBuilder()
                .withId(p.getId())
                .withObject(p)
                .build())
            .toList();

        // Bulk 실행 (부분 실패 허용)
        List<IndexedObjectInformation> results =
            operations.bulkIndex(queries, IndexCoordinates.of("products"));

        // 결과 분석 (성공/실패 분리)
        List<String> failedIds = new ArrayList<>();
        for (int i = 0; i < results.size(); i++) {
            IndexedObjectInformation result = results.get(i);
            if (result.id() == null) {  // 실패한 경우 null
                failedIds.add(products.get(i).getId());
            }
        }

        if (!failedIds.isEmpty()) {
            log.warn("Bulk indexing partial failure: {} out of {} failed",
                failedIds.size(), products.size());
            // 실패한 항목만 재처리 큐에 추가
            handleFailures(products, failedIds);
        }

        return new BulkIndexResult(products.size() - failedIds.size(), failedIds);
    }

    /**
     * 실패 항목 재처리 (exponential backoff retry)
     */
    @Retryable(
        retryFor = ElasticsearchException.class,
        maxAttempts = MAX_RETRY,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void retryIndex(EsProduct product) {
        IndexQuery query = new IndexQueryBuilder()
            .withId(product.getId())
            .withObject(product)
            .build();
        operations.index(query, IndexCoordinates.of("products"));
    }

    /**
     * Dead-letter: 재처리도 실패한 경우 별도 저장
     */
    @Recover
    public void recover(ElasticsearchException e, EsProduct product) {
        log.error("ES indexing permanently failed for {}: {}", product.getId(), e.getMessage());
        failedIndexRepository.save(FailedIndex.of(product.getId(), e.getMessage()));
    }

    private void handleFailures(List<EsProduct> all, List<String> failedIds) {
        Set<String> failedIdSet = new HashSet<>(failedIds);
        all.stream()
            .filter(p -> failedIdSet.contains(p.getId()))
            .forEach(p -> {
                try {
                    retryIndex(p);
                } catch (Exception ex) {
                    recover((ElasticsearchException) ex, p);
                }
            });
    }
}
```

### 4. 대규모 초기 적재 — Spring 배치 최적화

```java
@Configuration
@RequiredArgsConstructor
public class ProductInitialLoadConfig {

    private final ElasticsearchOperations operations;
    private final ProductJpaRepository jpaRepository;

    private static final int BATCH_SIZE = 5000;
    private static final int THREAD_COUNT = 4;

    /**
     * 대규모 초기 적재 (초기화 시 또는 재인덱싱)
     */
    @Bean
    @ConditionalOnProperty("es.initial-load.enabled")
    public CommandLineRunner initialLoad() {
        return args -> {
            log.info("ES initial load started");
            long startTime = System.currentTimeMillis();

            // 1. 인덱스 최적화 설정 (쓰기 성능 극대화)
            disableRefreshAndReplicas();

            try {
                // 2. JPA에서 배치로 읽어 병렬 Bulk 인덱싱
                ExecutorService executor =
                    Executors.newFixedThreadPool(THREAD_COUNT);
                List<CompletableFuture<Void>> futures = new ArrayList<>();

                long totalCount = jpaRepository.count();
                long offset = 0;

                while (offset < totalCount) {
                    final long currentOffset = offset;
                    CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                        List<Product> batch = jpaRepository.findAllWithPaging(
                            currentOffset, BATCH_SIZE);
                        List<EsProduct> esProducts = batch.stream()
                            .map(EsProduct::from)
                            .toList();
                        bulkIndexBatch(esProducts);
                    }, executor);

                    futures.add(future);
                    offset += BATCH_SIZE;
                }

                // 모든 배치 완료 대기
                CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                    .join();

            } finally {
                // 3. 설정 복원 + 수동 refresh
                restoreSettings();
                log.info("ES initial load completed in {}ms",
                    System.currentTimeMillis() - startTime);
            }
        };
    }

    private void disableRefreshAndReplicas() {
        UpdateSettingsRequest request = UpdateSettingsRequest.of(u -> u
            .index("products")
            .settings(s -> s
                .refreshInterval(ri -> ri.time("-1"))
                .numberOfReplicas("0")));
        // 설정 적용
    }

    private void restoreSettings() {
        // refresh_interval 복원
        // number_of_replicas 복원
        // 수동 refresh 실행
        operations.indexOps(IndexCoordinates.of("products")).refresh();
    }

    private void bulkIndexBatch(List<EsProduct> esProducts) {
        List<IndexQuery> queries = esProducts.stream()
            .map(p -> new IndexQueryBuilder()
                .withId(p.getId())
                .withObject(p)
                .build())
            .toList();
        operations.bulkIndex(queries, IndexCoordinates.of("products"));
    }
}
```

---

## 💻 실전 실험

```java
// 단건 vs Bulk 성능 비교 테스트
@SpringBootTest
@Slf4j
class IndexingPerformanceTest {

    @Autowired
    private ElasticsearchOperations operations;

    @Test
    void compareIndexingPerformance() {
        List<EsProduct> products = generateTestProducts(10000);

        // 단건 인덱싱
        long singleStart = System.currentTimeMillis();
        products.forEach(p -> {
            IndexQuery query = new IndexQueryBuilder()
                .withId(p.getId()).withObject(p).build();
            operations.index(query, IndexCoordinates.of("test-products"));
        });
        long singleTime = System.currentTimeMillis() - singleStart;

        // Bulk 인덱싱
        long bulkStart = System.currentTimeMillis();
        List<IndexQuery> queries = products.stream()
            .map(p -> new IndexQueryBuilder()
                .withId(p.getId()).withObject(p).build())
            .toList();
        operations.bulkIndex(queries, IndexCoordinates.of("test-products"));
        long bulkTime = System.currentTimeMillis() - bulkStart;

        log.info("Single: {}ms, Bulk: {}ms, Ratio: {}x",
            singleTime, bulkTime, singleTime / bulkTime);
    }
}
```

```bash
# 인덱싱 처리량 모니터링
GET _nodes/stats/indices/indexing?pretty
# index_total, index_time_in_millis 확인

# Bulk 요청 상태 확인
GET _tasks?actions=*bulk*&detailed=true
```

---

## 📊 인덱싱 전략 비교

| 전략 | 속도 | 안정성 | 복잡도 | 적합 상황 |
|------|------|-------|-------|---------|
| 단건 동기 | 느림 | 높음 | 낮음 | 중요 문서 1건 |
| 단건 비동기 (@Async) | 중간 | 중간 | 중간 | 실시간 단건 반영 |
| Bulk 동기 | 빠름 | 중간 | 중간 | 배치 동기화 |
| Bulk 비동기 병렬 | 매우 빠름 | 복잡 | 높음 | 초기 대량 적재 |

---

## ⚖️ 트레이드오프

```
비동기 인덱싱:
  이득: MySQL 응답 시간에 ES 영향 없음
  비용: 인덱싱 실패 시 MySQL과 불일치 가능성
  → 반드시 재처리 메커니즘 필요

@TransactionalEventListener(AFTER_COMMIT):
  이득: MySQL 롤백 시 ES 인덱싱 실행 안 됨 (일관성)
  비용: MySQL 커밋 후 ES 실패 시 불일치 가능성
  → 멱등성 보장 + 재처리 큐 필수

Bulk 배치 크기:
  너무 작음 → HTTP 오버헤드 높음 (성능 저하)
  너무 큼  → 단일 실패 시 대량 재처리 필요, 메모리 압박
  5~15MB   → 대부분 최적
```

---

## 📌 핵심 정리

```
단건 vs Bulk:
  단건: 10만 건 × 20ms = 33분
  Bulk(1000건): 100회 × 100ms = 10초 → 198배 빠름

안전한 단건 인덱싱:
  @TransactionalEventListener(AFTER_COMMIT) + @Async
  → MySQL 커밋 후 비동기 ES 인덱싱
  → MySQL 롤백 시 ES 인덱싱 실행 안 됨

부분 실패 처리:
  bulkIndex()는 부분 성공 가능
  결과 리스트에서 실패 문서 ID 추출
  실패 문서만 retry (exponential backoff)
  최종 실패 → dead-letter 저장 + 모니터링

초기 대량 적재:
  refresh_interval: -1 + replicas: 0
  병렬 Bulk (THREAD_COUNT = Primary Shard 수)
  완료 후 설정 복원 + 수동 refresh
```

---

## 🤔 생각해볼 문제

**Q1.** `bulkIndex()` 결과에서 실패를 감지하는 올바른 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

`operations.bulkIndex()`는 `List<IndexedObjectInformation>`을 반환한다. 각 항목은 성공한 경우 `id` 필드에 인덱싱된 문서 ID가 있고, 실패한 경우 `id`가 null이거나 빈 문자열이다. 입력 리스트와 결과 리스트의 인덱스가 1:1 대응하므로, 실패한 인덱스 위치로 원본 문서를 찾아 재처리할 수 있다. 더 세밀한 오류 정보(원인, HTTP 상태 코드)는 `BulkItemResponse`를 직접 파싱해야 하며, 이를 위해 `ElasticsearchClient`의 `bulk()` 메서드를 직접 사용하는 방법도 있다.

</details>

**Q2.** MySQL 트랜잭션 내부에서 ES 인덱싱을 함께 실행하면 안 되는 이유는?

<details>
<summary>해설 보기</summary>

`@Transactional` 메서드 내에서 ES 인덱싱을 하면 두 가지 문제가 발생한다. 첫째, MySQL이 롤백되어도 ES 인덱싱은 이미 완료되어 취소가 불가능하다(ES는 트랜잭션 지원 안 함). 둘째, ES 인덱싱이 실패하면 예외가 던져져 MySQL 트랜잭션도 롤백된다. 즉, ES 장애가 MySQL 저장 실패로 이어진다. `@TransactionalEventListener(AFTER_COMMIT)`을 사용하면 MySQL 커밋이 완료된 후에만 ES 인덱싱을 실행해 롤백 시 ES 인덱싱을 건너뛸 수 있다. ES 실패는 MySQL 저장에 영향을 주지 않으며, 별도 재처리 메커니즘으로 해결한다.

</details>

**Q3.** 비동기 인덱싱 큐(ThreadPool)가 가득 찼을 때 `CallerRunsPolicy`의 동작은?

<details>
<summary>해설 보기</summary>

`CallerRunsPolicy`는 큐가 가득 찼을 때 작업을 제출한 스레드(보통 HTTP 요청 처리 스레드)가 직접 해당 작업을 실행한다. 이는 백프레셔(Backpressure) 효과를 낸다. 비동기 ES 인덱싱 큐가 포화 상태이면, 새 HTTP 요청 처리가 ES 인덱싱을 직접 수행하느라 느려진다. 이를 통해 시스템이 자연스럽게 속도를 조절한다. 대안으로 `AbortPolicy`는 예외를 던져 인덱싱 요청을 버리고, `DiscardPolicy`는 조용히 버린다. `CallerRunsPolicy`는 데이터 손실 없이 자동 속도 조절이 필요할 때 권장된다.

</details>

---

<div align="center">

**[⬅️ 이전: Spring Data Elasticsearch](./01-spring-data-elasticsearch.md)** | **[홈으로 🏠](../README.md)** | **[다음: 검색 쿼리 작성 ➡️](./03-search-query-building.md)**

</div>
