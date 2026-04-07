# Spring Data Elasticsearch — Repository·Operations·매핑 어노테이션

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Document`와 `@Field` 어노테이션이 내부 ES 매핑 JSON으로 어떻게 변환되는가?
- `ElasticsearchRepository`와 `ElasticsearchOperations`의 역할은 어떻게 나뉘는가?
- 어노테이션 기반 매핑의 한계와 `@Mapping`으로 커스텀 JSON을 주입하는 이유는?
- Spring Boot Auto-configuration이 ES 클라이언트를 어떻게 자동 생성하는가?
- 인덱스 자동 생성이 프로덕션에서 위험한 이유와 비활성화 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Data Elasticsearch를 "그냥 JPA 쓰듯이" 사용하다가 실제 프로덕션에서 매핑이 의도와 다르게 생성되거나, 인덱스 자동 생성으로 운영 인덱스가 덮어써지는 사고가 발생한다. `@Field`가 내부적으로 어떤 ES 매핑 JSON을 생성하는지, 그 한계는 무엇인지 알아야 의도한 대로 ES가 동작한다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: createIndex=true로 두고 프로덕션 배포
  설정:
    @Document(indexName = "products", createIndex = true)

  문제:
    애플리케이션 시작 시 인덱스 자동 생성
    이미 존재하는 인덱스 → 기존 설정 유지 (ES는 덮어쓰지 않음)
    하지만 인덱스 없으면 → 기본 설정으로 생성 (샤드 수, 레플리카 등)
    → 프로덕션 최적화 설정 없이 인덱스 생성 가능

    더 큰 문제:
    개발 환경에서 createIndex=true → 매번 재시작 시 인덱스 상태 변화
    매핑 변경 → 자동 재생성 시도 (ES는 기존 필드 타입 변경 불가)
    → 충돌 오류

  올바른 접근:
    createIndex = false (기본값으로 비활성화)
    인덱스는 IaC(Terraform, Pulumi) 또는 별도 스크립트로 관리

실수 2: @Field 어노테이션만 믿고 복잡한 매핑 설정
  설정:
    @Field(type = FieldType.Text, analyzer = "nori_analyzer")

  문제:
    nori_analyzer가 해당 인덱스에 정의되어 있어야 함
    @Field로 Analyzer 이름만 지정, 실제 Analyzer 정의는 별도 설정 필요
    Analyzer 미정의 인덱스에서 nori_analyzer 사용 → 오류

실수 3: ElasticsearchRepository 메서드명으로 복잡한 쿼리 표현
  코드:
    List<Product> findByTitleContainingAndPriceGreaterThanAndCategoryIn(
      String title, Double price, List<String> categories);

  문제:
    메서드명으로 생성되는 쿼리: query context에서 처리
    filter 최적화 없음, 캐시 없음
    복잡한 조건 → 가독성 낮고 디버깅 어려움
    → 복잡한 쿼리는 NativeQuery 직접 작성 권장
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
Spring Data ES 설계 원칙:

  인덱스 관리:
    createIndex = false (기본)
    인덱스 생성은 외부에서 관리 (Terraform, Kibana, 별도 스크립트)
    매핑 변경 → Reindex 절차

  어노테이션 매핑:
    단순 필드 타입 지정 → @Field 활용
    복잡한 설정 (커스텀 Analyzer, 다중 필드) → @Mapping(mappingPath)
    → 실제 ES 매핑 JSON 파일을 resources에 관리

  쿼리 선택:
    단순 CRUD → ElasticsearchRepository
    복잡한 검색 → NativeQuery + ElasticsearchOperations
    동적 쿼리 → NativeQuery with QueryBuilders

  클라이언트 설정:
    ElasticsearchClient (공식 Java Client) 기반 (Spring Data ES 5.x+)
    연결 설정은 application.yml에서 관리
    프로덕션: 인증, SSL 설정 필수
```

---

## 🔬 내부 동작 원리

### 1. @Document와 @Field의 ES 매핑 변환

```java
@Document(indexName = "products", createIndex = false)
@Setting(settingPath = "es-settings/products-settings.json")
public class Product {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "nori_analyzer",
           searchAnalyzer = "nori_analyzer")
    private String title;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Date,
           format = DateFormat.date_time)
    private LocalDateTime createdAt;

    @Field(type = FieldType.Text, analyzer = "nori_analyzer",
           fields = {
               @InnerField(suffix = "keyword", type = FieldType.Keyword)
           })
    private String description;
}
```

변환 결과 (Spring Data ES가 생성하는 매핑 JSON):
```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "search_analyzer": "nori_analyzer"
      },
      "category": {
        "type": "keyword"
      },
      "price": {
        "type": "double"
      },
      "createdAt": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },
      "description": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

매핑 변환 과정:
  ① 애플리케이션 시작 시 Spring Data ES가 엔티티 클래스 스캔
  ② @Document → 인덱스 이름 추출
  ③ @Field → 각 필드의 타입, Analyzer, 옵션 추출
  ④ MappingBuilder가 ES 매핑 JSON 생성
  ⑤ createIndex=true이면 자동 생성, false이면 스캔만 (생성 안 함)

### 2. @Mapping — 커스텀 JSON 주입

어노테이션 매핑의 한계:
```
@Field로 표현 못하는 것들:
  - 커스텀 Analyzer 정의 (character filter, tokenizer, token filter 조합)
  - nested 타입 내부 세밀한 설정
  - dynamic: strict 같은 인덱스 레벨 설정
  - enable: false (검색 비활성화)
  - eager_global_ordinals: true
```

커스텀 매핑 사용:
```java
@Document(indexName = "products", createIndex = false)
@Mapping(mappingPath = "es-mappings/products-mapping.json")
public class Product {
    // @Field 어노테이션 없이 매핑 JSON 파일로 관리
    @Id
    private String id;
    private String title;
    private String category;
    private Double price;
}
```

resources/es-mappings/products-mapping.json:
```json
{
  "dynamic": "strict",
  "properties": {
    "title": {
      "type": "text",
      "analyzer": "korean_analyzer",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    },
    "category": {
      "type": "keyword",
      "eager_global_ordinals": true
    },
    "price": {
      "type": "double",
      "doc_values": true
    }
  }
}
```

### 3. ElasticsearchRepository vs ElasticsearchOperations

```java
// ① ElasticsearchRepository — CRUD + 메서드명 쿼리
public interface ProductRepository
    extends ElasticsearchRepository<Product, String> {

    // 메서드명 → 자동 쿼리 생성
    List<Product> findByCategory(String category);
    List<Product> findByPriceBetween(Double min, Double max);
    Page<Product> findByTitleContaining(String keyword, Pageable pageable);

    // @Query 어노테이션으로 직접 JSON 작성
    @Query("{\"bool\": {\"must\": [{\"match\": {\"title\": \"?0\"}}], " +
           "\"filter\": [{\"term\": {\"category\": \"?1\"}}]}}")
    List<Product> findByTitleAndCategory(String title, String category);
}

// ② ElasticsearchOperations — 세밀한 제어
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    public SearchHits<Product> searchProducts(
            String keyword, String category, Double minPrice) {

        // NativeQuery로 세밀한 쿼리 제어
        Query query = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .must(m -> m.match(mm -> mm
                        .field("title")
                        .query(keyword)))
                    .filter(f -> f.term(t -> t
                        .field("category")
                        .value(category)))
                    .filter(f -> f.range(r -> r
                        .field("price")
                        .gte(JsonData.of(minPrice))))))
            .withSort(SortOptions.of(s -> s
                .score(sc -> sc.order(SortOrder.Desc))))
            .withPageable(PageRequest.of(0, 10))
            .withHighlightQuery(HighlightQuery.of(hq -> hq
                .withHighlighter(HighlightParameters.builder()
                    .withPreTags("<em>")
                    .withPostTags("</em>")
                    .build())
                .withFields(HighlightField.of("title"))))
            .build();

        return operations.search(query, Product.class);
    }
}
```

책임 분리 기준:
```
ElasticsearchRepository:
  적합: 단순 CRUD, 조건 검색, 페이지네이션
  특징: 선언적, 간결, Spring Data 표준

ElasticsearchOperations:
  적합: 복잡한 검색, 하이라이팅, 집계, Bulk 작업
  특징: 명시적, 세밀한 제어, ES 기능 완전 활용
```

### 4. Spring Boot Auto-configuration 동작

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
    connection-timeout: 10s
    socket-timeout: 30s
```

Auto-configuration 체인:
```
ElasticsearchRestClientAutoConfiguration
  → RestClient 생성 (저수준 HTTP 클라이언트)
  → ElasticsearchClient 생성 (고수준 Java 클라이언트)

ElasticsearchDataAutoConfiguration
  → ElasticsearchOperationsImpl 생성
  → Repository 스캔 및 프록시 생성
  → 엔티티 매핑 정보 캐시
```

커스텀 클라이언트 설정:
```java
@Configuration
public class ElasticsearchConfig
    extends ElasticsearchConfiguration {

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .withConnectTimeout(Duration.ofSeconds(10))
            .withSocketTimeout(Duration.ofSeconds(30))
            .withBasicAuth("elastic", "changeme")
            .build();
    }
}
```

---

## 💻 실전 실험

```java
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
}

// Product 엔티티
@Document(indexName = "products", createIndex = false)
@lombok.Data
public class Product {
    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "nori")
    private String title;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Boolean)
    private Boolean inStock;

    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private LocalDateTime createdAt;
}

// Repository
public interface ProductRepository
    extends ElasticsearchRepository<Product, String> {

    List<Product> findByCategory(String category);
    List<Product> findByInStockTrue();
    List<Product> findByPriceLessThanEqual(Double price);
}

// 현재 생성될 매핑 확인 (Spring이 생성하는 JSON)
@SpringBootTest
class MappingTest {

    @Autowired
    private ElasticsearchOperations operations;

    @Test
    void printMapping() throws Exception {
        // 매핑 정보 출력
        IndexOperations indexOps = operations.indexOps(Product.class);
        Document mapping = indexOps.createMapping();
        System.out.println(mapping.toJson());
    }
}
```

```bash
# 어노테이션으로 생성되는 매핑 확인
# Spring이 생성하는 매핑을 Kibana Dev Tools에서 확인
GET /products/_mapping?pretty

# Nori Analyzer가 실제로 작동하는지 확인
GET /products/_analyze
{
  "analyzer": "nori",
  "text": "전자제품 검색 엔진"
}
```

---

## 📊 @Field 주요 타입 매핑

| @Field(type=) | ES 매핑 타입 | 용도 |
|------------|-----------|------|
| `FieldType.Text` | text | 전문 검색 (Analyzer 적용) |
| `FieldType.Keyword` | keyword | 집계, 정렬, 정확 매칭 |
| `FieldType.Integer` | integer | 정수 |
| `FieldType.Double` | double | 부동소수점 |
| `FieldType.Date` | date | 날짜/시간 |
| `FieldType.Boolean` | boolean | 참/거짓 |
| `FieldType.Nested` | nested | 중첩 객체 (관계 보존) |
| `FieldType.Object` | object | 단순 내부 객체 |
| `FieldType.Auto` | dynamic 추론 | 타입 자동 감지 |

---

## ⚖️ 트레이드오프

```
createIndex = true:
  이득: 개발 편의 (자동 인덱스 생성)
  비용: 프로덕션에서 의도치 않은 인덱스 생성 위험
  → 개발만 true, 프로덕션은 false

@Field 어노테이션:
  이득: 코드 가독성, 타입 안전성
  비용: 복잡한 ES 매핑 옵션 표현 한계
  → 복잡한 경우 @Mapping(mappingPath) 사용

Repository vs Operations:
  Repository: 간결, 표준 인터페이스, 기능 제한
  Operations: 완전한 ES 기능, 코드 복잡도 증가
  → 단순 CRUD: Repository / 복잡한 검색: Operations
```

---

## 📌 핵심 정리

```
@Document:
  indexName → 인덱스 이름 매핑
  createIndex = false → 프로덕션 권장
  @Setting(settingPath) → 인덱스 설정 JSON 파일

@Field:
  type → ES 필드 타입
  analyzer, searchAnalyzer → Analyzer 지정
  fields → 멀티필드 (text + keyword)
  복잡한 설정 → @Mapping(mappingPath)으로 대체

클라이언트 계층:
  ElasticsearchRepository → CRUD + 메서드명 쿼리 (선언적)
  ElasticsearchOperations → 복잡한 검색, 집계, Bulk (명시적)

Auto-configuration:
  application.yml에 spring.elasticsearch.uris 설정
  → RestClient → ElasticsearchClient → Operations → Repository 자동 구성
  커스텀 설정: ElasticsearchConfiguration 상속
```

---

## 🤔 생각해볼 문제

**Q1.** `@Field(type = FieldType.Text)`로 설정한 필드에서 Aggregation을 수행하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

기본적으로 오류가 발생한다. text 필드는 doc_values가 비활성화되어 있고 fielddata도 기본 비활성화 상태다. `terms` 집계처럼 doc_values가 필요한 집계를 실행하면 `Fielddata is disabled on text fields by default` 오류가 반환된다. 해결 방법은 `@Field(type = FieldType.Text, fielddata = true)` 설정이지만 힙 사용 문제가 있다. 올바른 방법은 `@MultiField`나 `@InnerField`로 keyword 서브필드를 추가하고 집계는 `.keyword` 필드로 하는 것이다.

</details>

**Q2.** Spring Data ES에서 인덱스 매핑을 변경해야 할 때 어떤 절차를 거쳐야 하는가?

<details>
<summary>해설 보기</summary>

ES에서 기존 필드 타입 변경은 불가능하므로 Reindex가 필요하다. 절차는 다음과 같다. 첫째, 새 매핑을 적용한 새 인덱스(`products-v2`)를 생성한다. 둘째, Reindex API로 기존 데이터를 복사한다. 셋째, 별칭을 원자적으로 교체한다(`products` 별칭을 `products-v2`로 이동). 넷째, 애플리케이션의 `@Document(indexName = "products")`는 별칭을 바라보므로 코드 변경 없이 새 인덱스를 사용한다. Spring Data ES의 Repository와 Operations는 인덱스 이름만 알면 동작하므로 별칭 기반 전환이 투명하게 된다.

</details>

**Q3.** `ElasticsearchRepository.findAll()`을 대용량 인덱스에서 사용하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

`findAll()`은 내부적으로 `match_all` 쿼리를 사용한다. 기본 `size`가 적용되지만 페이지네이션 없이 `findAll()` 호출 시 모든 문서를 메모리에 로딩한다. 1,000만 건의 인덱스라면 수십 GB의 데이터가 JVM 힙으로 올라와 OOM이 발생할 수 있다. `findAll(Pageable pageable)`로 페이지 처리하거나, Scroll API/Search After를 사용하는 대안이 필요하다. 대용량 Export는 `ElasticsearchOperations`의 Scroll 방식으로 처리해야 한다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 인덱싱 전략 ➡️](./02-indexing-strategy.md)**

</div>
