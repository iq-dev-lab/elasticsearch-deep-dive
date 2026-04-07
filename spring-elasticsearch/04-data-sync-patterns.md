# 데이터 동기화 패턴 — CDC·Debezium·검색-DB 정합성 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MySQL → Elasticsearch 동기화의 3가지 방식은 무엇이고 각각의 트레이드오프는?
- Debezium이 MySQL binlog를 어떻게 읽어 Kafka로 전송하는가?
- DB와 ES 사이의 일시적 불일치를 허용 가능한 설계로 만드는 방법은?
- 재인덱싱(Reindex)이 필요한 상황과 무중단 재인덱싱 절차는?
- Spring에서 Application-level 이중 쓰기를 안전하게 구현하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

MySQL이 원본 데이터, Elasticsearch가 검색 인덱스인 아키텍처에서 가장 어려운 문제는 정합성이다. MySQL에 저장됐는데 ES에 반영이 안 되거나, MySQL 롤백됐는데 ES는 이미 인덱싱된 상황이 발생한다. 이 문제를 해결하는 세 가지 패턴(이중 쓰기, 배치 동기화, CDC)의 원리와 트레이드오프를 이해하면 서비스 특성에 맞는 선택을 할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: @Transactional 안에서 이중 쓰기
  코드:
    @Transactional
    public void saveProduct(Product product) {
        productJpaRepository.save(product);     // MySQL
        esProductRepository.save(esProduct);    // ES (트랜잭션 밖!)
    }

  문제:
    ES는 트랜잭션에 참여하지 않음
    MySQL 롤백 → ES 저장 취소 안 됨 → 불일치
    ES 오류 → MySQL도 롤백됨 → ES 장애가 서비스 장애로

실수 2: 배치 동기화에서 updated_at 기준 미사용
  코드:
    @Scheduled(fixedDelay = 60000)
    public void syncAll() {
        List<Product> all = productJpaRepository.findAll(); // 전체!
        // ES 전체 재인덱싱
    }

  문제:
    상품 100만 건 → 매 분 전체 동기화
    MySQL + ES 모두 과부하
    → 마지막 동기화 이후 변경된 것만 동기화 (incremental)

실수 3: 재인덱싱 중 서비스 중단
  패턴:
    인덱스 삭제 → 재생성 → 인덱싱
    인덱싱 완료까지 검색 불가 → 서비스 중단

  올바른 패턴:
    새 인덱스 생성(v2) → 데이터 복사 → 별칭 전환
    → 무중단 재인덱싱
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
동기화 방식 선택 기준:

  Application-level 이중 쓰기:
    적합: 단순 아키텍처, 실시간성 중요, 소규모
    방법: @TransactionalEventListener(AFTER_COMMIT) + 재처리 큐
    지연: 수십 ms ~ 수 초

  배치 동기화:
    적합: 실시간 불필요, 단순 운영, 소규모
    방법: 스케줄러 + updated_at 기준 incremental 동기화
    지연: 분 ~ 시간

  CDC (Change Data Capture, Debezium):
    적합: 대규모, 여러 컨슈머 필요, 실시간성 + 안정성
    방법: MySQL binlog → Kafka → Consumer → ES
    지연: 수십 ms ~ 수 초

  선택 공식:
    규모 작음 + 단순함 원함: 이중 쓰기
    실시간 불필요: 배치 동기화
    대규모 + 안정성 중요: CDC
```

---

## 🔬 내부 동작 원리

### 1. Application-level 이중 쓰기 — 안전한 구현

```java
// 방법 1: TransactionalEventListener 패턴
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductJpaRepository jpaRepository;
    private final ApplicationEventPublisher eventPublisher;
    private final FailedSyncRepository failedSyncRepository;

    @Transactional
    public Product createProduct(ProductCreateRequest request) {
        Product product = new Product(request);
        Product saved = jpaRepository.save(product);

        // 트랜잭션 내 이벤트 발행 (AFTER_COMMIT에 실행됨)
        eventPublisher.publishEvent(
            new ProductSavedEvent(saved.getId(), "CREATE"));
        return saved;
    }

    @Transactional
    public Product updateProduct(Long id, ProductUpdateRequest request) {
        Product product = jpaRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        product.update(request);
        // dirty checking → MySQL UPDATE
        eventPublisher.publishEvent(
            new ProductSavedEvent(id, "UPDATE"));
        return product;
    }

    @Transactional
    public void deleteProduct(Long id) {
        jpaRepository.deleteById(id);
        eventPublisher.publishEvent(
            new ProductSavedEvent(id, "DELETE"));
    }
}

// ES 인덱싱 핸들러
@Component
@RequiredArgsConstructor
public class ProductEsEventHandler {

    private final ProductJpaRepository jpaRepository;
    private final ElasticsearchOperations operations;
    private final FailedSyncRepository failedSyncRepository;

    @Async("esIndexingExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Retryable(
        retryFor = Exception.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void handleProductSaved(ProductSavedEvent event) {
        try {
            switch (event.operation()) {
                case "CREATE", "UPDATE" -> {
                    // DB에서 최신 데이터 조회 (이벤트 데이터는 스냅샷일 수 있음)
                    Product product = jpaRepository.findById(event.productId())
                        .orElseThrow();
                    EsProduct esProduct = EsProduct.from(product);
                    IndexQuery query = new IndexQueryBuilder()
                        .withId(esProduct.getId())
                        .withObject(esProduct)
                        .build();
                    operations.index(query, IndexCoordinates.of("products"));
                }
                case "DELETE" -> {
                    operations.delete(
                        String.valueOf(event.productId()),
                        IndexCoordinates.of("products"));
                }
            }
        } catch (Exception e) {
            log.error("ES sync failed for product {}: {}", event.productId(), e.getMessage());
            // 재시도 모두 실패 시 dead-letter에 저장
        }
    }

    @Recover
    public void recover(Exception e, ProductSavedEvent event) {
        failedSyncRepository.save(FailedSync.builder()
            .productId(event.productId())
            .operation(event.operation())
            .errorMessage(e.getMessage())
            .createdAt(LocalDateTime.now())
            .build());
        // 모니터링 알림 발송
        log.error("ES sync permanently failed, saved to dead-letter: {}", event.productId());
    }
}

// 실패한 동기화 재처리 (별도 스케줄러)
@Component
@RequiredArgsConstructor
public class FailedSyncRetryService {

    private final FailedSyncRepository failedSyncRepository;
    private final ProductEsEventHandler handler;

    @Scheduled(fixedDelay = 60000)  // 1분마다
    public void retryFailedSync() {
        List<FailedSync> failures = failedSyncRepository
            .findByRetryCountLessThan(5)  // 최대 5회 재시도
            .stream()
            .limit(100)
            .toList();

        failures.forEach(failure -> {
            try {
                handler.handleProductSaved(
                    new ProductSavedEvent(failure.getProductId(), failure.getOperation()));
                failedSyncRepository.delete(failure);
            } catch (Exception e) {
                failure.incrementRetryCount();
                failedSyncRepository.save(failure);
            }
        });
    }
}
```

### 2. CDC — Debezium + Kafka 아키텍처

```
CDC (Change Data Capture) 전체 흐름:

  MySQL
    └── binlog (Binary Log)
           │ Debezium이 binlog 구독
           ▼
    Debezium Connector
    (MySQL Source Connector)
           │ 변경 이벤트를 JSON으로 변환
           ▼
    Apache Kafka
    Topic: dbserver1.inventory.products
           │
    ┌──────┴──────┐
    ▼             ▼
  ES Consumer   다른 Consumer들
  (Kafka          (알림 서비스,
  Consumer)       통계 서비스, ...)
    │
    ▼
  Elasticsearch

Debezium 이벤트 구조:
  {
    "before": {                    ← 변경 전 데이터 (UPDATE, DELETE)
      "id": 1,
      "title": "Keyboard Old"
    },
    "after": {                     ← 변경 후 데이터 (INSERT, UPDATE)
      "id": 1,
      "title": "Keyboard New",
      "price": 150000
    },
    "op": "u",                     ← c(insert), u(update), d(delete), r(read/snapshot)
    "ts_ms": 1705279800000
  }

Kafka Consumer (Spring Boot):
  @KafkaListener(topics = "dbserver1.inventory.products")
  public void handleProductChange(String message) {
      DebeziumEvent event = objectMapper.readValue(message, DebeziumEvent.class);

      switch (event.op()) {
          case "c", "u" -> {
              Product product = mapToProduct(event.after());
              EsProduct esProduct = EsProduct.from(product);
              IndexQuery query = new IndexQueryBuilder()
                  .withId(esProduct.getId())
                  .withObject(esProduct)
                  .build();
              operations.index(query, IndexCoordinates.of("products"));
          }
          case "d" -> {
              operations.delete(
                  String.valueOf(event.before().id()),
                  IndexCoordinates.of("products"));
          }
      }
  }
```

### 3. 배치 동기화 — Incremental Sync

```java
@Component
@RequiredArgsConstructor
public class ProductIncrementalSyncJob {

    private final ProductJpaRepository jpaRepository;
    private final ProductBulkIndexService bulkIndexService;
    private final SyncStateRepository syncStateRepository;

    @Scheduled(fixedDelay = 60000)  // 1분마다
    public void incrementalSync() {
        // 마지막 동기화 시각 로드
        SyncState syncState = syncStateRepository
            .findById("products")
            .orElse(new SyncState("products", LocalDateTime.now().minusMinutes(2)));

        LocalDateTime lastSync = syncState.getLastSyncAt();
        LocalDateTime now = LocalDateTime.now();

        // 마지막 동기화 이후 변경된 상품만 조회
        List<Product> changed = jpaRepository
            .findByUpdatedAtAfter(lastSync);

        if (!changed.isEmpty()) {
            List<EsProduct> esProducts = changed.stream()
                .map(EsProduct::from)
                .toList();
            bulkIndexService.bulkIndex(esProducts);
            log.info("Synced {} products changed after {}", changed.size(), lastSync);
        }

        // 동기화 상태 업데이트
        syncState.setLastSyncAt(now);
        syncStateRepository.save(syncState);
    }
}
```

### 4. 정합성 허용 범위 설계

```
일시적 불일치를 허용 가능하게 만드는 설계:

  1. 검색과 상세 조회 분리
     검색 결과(ES): 약간 오래된 데이터 허용
     상세 조회(MySQL): 항상 최신 데이터
     → 사용자가 검색에서 상품 클릭 → 상세 페이지는 MySQL

  2. ES 검색 결과에서 MySQL 재확인
     ES에서 상품 ID 목록 추출
     MySQL에서 실제 데이터 조회 (최신 + 존재 여부 확인)
     → ES는 "어떤 상품이 해당하는가" 답변
     → MySQL은 "실제 상품 데이터는 무엇인가" 답변

  3. 소프트 삭제 활용
     MySQL DELETE → soft delete (deleted_at 설정)
     ES에서도 deleted_at 필터 → 삭제된 상품 제외
     → 동기화 지연 중에도 삭제 상품이 검색되지 않음

  4. 버전 필드로 최신 데이터 확인
     MySQL: version 필드 (업데이트 시 증가)
     ES: version 필드 포함
     ES 검색 결과의 version vs MySQL version 비교
     → 오래된 ES 데이터 감지 가능

  5. 허용 가능한 지연 시간 명시
     이중 쓰기: ~1초
     배치 동기화: ~5분
     CDC: ~1~2초
     → SLA에 검색 인덱스 지연 시간 명시
```

### 5. 무중단 재인덱싱 절차

```
재인덱싱이 필요한 경우:
  - 매핑 변경 (필드 타입, Analyzer 변경)
  - 대규모 데이터 정합성 수정
  - 인덱스 설정 변경 (샤드 수 등)

무중단 재인덱싱 4단계:

Step 1: 새 인덱스 + 별칭 설정
  현재: products (별칭) → products-v1 (실제 인덱스)
  새 인덱스 생성: products-v2 (새 매핑 적용)
  애플리케이션: "products" 별칭으로만 접근 중

Step 2: 데이터 복사 (Reindex API)
  POST _reindex?wait_for_completion=false
  {
    "source": { "index": "products-v1" },
    "dest":   { "index": "products-v2" }
  }
  → 작업 ID 반환 → 비동기 실행
  GET _tasks/{task_id}  // 진행 상황 확인

  또는 Spring에서:
  operations.reindex(ReindexRequest.of(r -> r
      .source(s -> s.index("products-v1"))
      .dest(d -> d.index("products-v2"))
  ));

Step 3: 재인덱싱 중 신규 변경 처리
  재인덱싱 시작 후 발생한 변경:
  → products (별칭) → products-v1에 계속 인덱싱
  → 재인덱싱 완료 후 v1→v2 차이분 처리
  또는:
  → CDC: 재인덱싱 완료 시점 이후의 Kafka offset부터 v2에 적용

Step 4: 별칭 원자적 전환
  POST _aliases
  {
    "actions": [
      { "remove": { "index": "products-v1", "alias": "products" } },
      { "add":    { "index": "products-v2", "alias": "products" } }
    ]
  }
  → 이 순간부터 모든 요청이 products-v2로
  → 애플리케이션 코드 변경 없음

Step 5: 구 인덱스 삭제 (검증 후)
  DELETE /products-v1
  → 충분한 검증 후 (1~7일 유지 후 삭제)
```

---

## 💻 실전 실험

```bash
# Debezium + Kafka 없이 CDC 개념 실험 (Kibana)
# MySQL binlog 기반 CDC를 ES Reindex로 시뮬레이션

# 무중단 재인덱싱 실험
# 1. 별칭 현재 상태 확인
GET _aliases

# 2. 새 인덱스 생성 (새 매핑 포함)
PUT /products-v2
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "title":    { "type": "text", "analyzer": "nori" },
      "category": { "type": "keyword", "eager_global_ordinals": true },
      "price":    { "type": "double" }
    }
  }
}

# 3. 비동기 재인덱싱
POST _reindex?wait_for_completion=false
{
  "source": { "index": "products", "size": 1000 },
  "dest":   { "index": "products-v2" }
}
# 응답: { "task": "abc123:456" }

# 4. 재인덱싱 진행 상황 확인
GET _tasks/abc123:456

# 5. 별칭 원자적 전환
POST _aliases
{
  "actions": [
    { "remove": { "index": "products-v1", "alias": "products" } },
    { "add":    { "index": "products-v2", "alias": "products" } }
  ]
}

# 6. 검색 확인 (별칭으로)
GET /products/_search
{ "query": { "match_all": {} } }

# 7. 구 인덱스 정보 확인 후 삭제
GET _cat/indices/products-v1?v
DELETE /products-v1
```

---

## 📊 동기화 방식 비교

| 방식 | 지연 | 안정성 | 복잡도 | 다중 컨슈머 | 적합 규모 |
|------|------|-------|-------|-----------|---------|
| 이중 쓰기 | ~1초 | 중간 | 낮음 | ❌ | 소규모 |
| 배치 동기화 | 분~시간 | 높음 | 낮음 | ❌ | 소~중규모 |
| CDC (Debezium) | ~1초 | 높음 | 높음 | ✅ | 중~대규모 |

---

## ⚖️ 트레이드오프

```
이중 쓰기:
  이득: 구현 단순, Kafka 불필요
  비용: ES 오류 재처리 로직 직접 구현, 다중 컨슈머 불가
  → 소규모 서비스, 빠른 개발 필요 시

배치 동기화:
  이득: 구현 매우 단순, 안정적
  비용: 실시간성 없음 (분 단위 지연)
  → 실시간 검색 불필요, 주기적 동기화면 충분한 경우

CDC (Debezium + Kafka):
  이득: 실시간, 안정적, 다중 컨슈머 (ES + 알림 + 통계 동시)
  비용: Kafka 운영, Debezium 설정, 높은 복잡도
  → 대규모, 여러 시스템 연계, 높은 안정성 요구

허용 불일치 vs 강한 정합성:
  허용 불일치: 검색 지연 허용 → 성능 향상 (캐시 효과 극대화)
  강한 정합성: 실시간 동기화 → 복잡도 + 비용 증가
  → 대부분 허용 불일치로 충분 (검색 특성상)
```

---

## 📌 핵심 정리

```
3가지 동기화 패턴:
  이중 쓰기: @TransactionalEventListener(AFTER_COMMIT) + 재처리 큐
  배치 동기화: 스케줄러 + updated_at 기준 incremental
  CDC: MySQL binlog → Debezium → Kafka → ES Consumer

안전한 이중 쓰기:
  @Transactional 밖에서 ES 인덱싱
  @TransactionalEventListener로 MySQL 커밋 후 실행
  재처리 큐 + dead-letter 패턴

정합성 허용 설계:
  검색(ES) + 상세조회(MySQL) 분리
  허용 지연 시간 SLA 명시
  소프트 삭제 + 버전 필드 활용

무중단 재인덱싱:
  새 인덱스 생성 → 데이터 복사 → 별칭 원자적 전환
  애플리케이션은 별칭만 바라봄 → 코드 변경 없음
  구 인덱스는 충분한 검증 후 삭제
```

---

## 🤔 생각해볼 문제

**Q1.** CDC 방식에서 Debezium Connector가 중단됐다가 재시작될 때 데이터 유실이 없는 이유는?

<details>
<summary>해설 보기</summary>

Debezium은 MySQL binlog의 처리 위치(offset)를 Kafka에 저장한다. 구체적으로는 Kafka Connect의 offset 저장 메커니즘을 사용해 처리한 binlog 위치(파일명 + 위치)를 주기적으로 커밋한다. Debezium이 재시작되면 마지막으로 커밋된 offset을 읽어와 그 위치부터 binlog를 다시 처리한다. MySQL binlog는 설정된 보관 기간 동안 유지되므로, 그 기간 내에 재시작하면 모든 변경 이벤트를 처리할 수 있다. 이 덕분에 Debezium 재시작, 서버 장애 등의 상황에서도 at-least-once 데이터 전달이 보장된다.

</details>

**Q2.** 이중 쓰기에서 MySQL은 성공했지만 ES 인덱싱이 반복 실패해 dead-letter에 쌓일 때 운영 절차는?

<details>
<summary>해설 보기</summary>

dead-letter에 저장된 실패 항목은 모니터링 알림으로 즉시 파악해야 한다. 원인을 파악(ES 장애, 네트워크 문제, 데이터 형식 오류)하고 해결한 후, dead-letter의 실패 항목들을 재처리한다. 재처리는 dead-letter의 product_id를 읽어 MySQL에서 최신 데이터를 가져와 ES에 인덱싱하는 방식이다. 멱등성(같은 ID로 여러 번 인덱싱해도 최신 데이터로 덮어쓰임)이 보장되므로 안전하게 재처리 가능하다. 대규모 누적 시 배치로 bulk 재처리한다.

</details>

**Q3.** 재인덱싱 완료 후 별칭 전환 직전에 발생한 MySQL 변경은 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

별칭 전환 직전까지의 변경은 기존 인덱스(v1)에만 반영된다. 전환 후에는 새 인덱스(v2)에만 반영된다. 전환 직전의 변경이 v2에 누락될 수 있다. 이를 해결하는 방법은 여러 가지다. CDC를 사용한다면 재인덱싱 시작 시점의 Kafka offset을 기록하고, 재인덱싱 완료 후 그 offset 이후의 이벤트를 v2에 적용하면 된다. 이중 쓰기 방식이라면 별칭 전환 후 v1과 v2의 최근 변경 사항 diff를 실행해 v2에 missing 데이터를 보충한다. 가장 안전한 방법은 재인덱싱 완료 후 `_doc_count`와 일부 샘플 데이터를 비교하고, 짧은 시간만 이중 쓰기(v1 + v2 동시)를 유지한 후 전환하는 것이다.

</details>

---

<div align="center">

**[⬅️ 이전: 검색 쿼리 작성](./03-search-query-building.md)** | **[홈으로 🏠](../README.md)**

</div>
