# 검색 쿼리 작성 — NativeQuery·CriteriaQuery·StringQuery 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- NativeQuery, CriteriaQuery, StringQuery는 각각 어떤 상황에 적합한가?
- 동적 쿼리에서 null 조건을 안전하게 처리하는 패턴은?
- `SearchHits`에서 점수, 하이라이트, 집계 결과를 어떻게 추출하는가?
- NativeQuery로 bool 쿼리와 집계를 함께 작성하는 방법은?
- Pageable과 SearchAfter를 이용한 대용량 페이지네이션은 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

검색 서비스의 복잡도는 쿼리에서 드러난다. "카테고리 + 가격 범위 + 키워드 + 정렬 + 하이라이팅 + 카테고리별 집계"를 한 번에 처리해야 하는 쿼리를 Repository 메서드명으로 표현하는 것은 한계가 있다. NativeQuery로 ES Query DSL을 그대로 사용하면서 Spring의 타입 안전성과 의존성 주입을 활용하는 방법을 알면 복잡한 검색 기능도 유지보수 가능하게 구현할 수 있다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 동적 쿼리에서 null 조건 처리 실수
  코드:
    CriteriaQuery query = new CriteriaQuery(
        Criteria.where("category").is(category)  // category가 null이면?
            .and("price").lessThanEqual(maxPrice)
    );

  문제:
    category = null → Criteria.where("category").is(null) → NullPointerException
    또는 { "term": { "category": null } } → ES 오류
    → 동적 쿼리에서 null 필드 미포함 로직 필요

실수 2: StringQuery로 쿼리 파라미터 직접 삽입 (Injection 위험)
  코드:
    String query = "{\"match\": {\"title\": \"" + userInput + "\"}}";
    new StringQuery(query);

  문제:
    userInput = "test\"}}...악의적인 내용"
    → 쿼리 구조 변경 가능 (Query Injection)
    → NativeQuery의 QueryBuilders 사용으로 파라미터 안전 처리

실수 3: SearchHits의 하이라이트를 가져오지 못함
  코드:
    SearchHits<Product> hits = operations.search(query, Product.class);
    hits.forEach(hit -> {
        Product product = hit.getContent();  // 하이라이트 없음
    });

  문제:
    하이라이트는 hit.getHighlightField("title")로 별도 추출
    hit.getContent()는 원본 문서만, 하이라이팅 정보 없음
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)

```
쿼리 타입 선택 기준:

  NativeQuery (권장):
    ES Query DSL을 Java 빌더로 표현
    타입 안전, 자동 완성 지원
    bool 쿼리, 집계, 하이라이팅, kNN 모두 지원
    → 대부분의 경우 NativeQuery 사용

  CriteriaQuery:
    Spring Data 스타일 빌더
    단순 조건 쿼리에 간결
    복잡한 ES 기능 표현 제한적
    → 간단한 필터링에만 사용

  StringQuery:
    직접 JSON 문자열로 쿼리 작성
    기존 JSON 쿼리를 그대로 사용할 때 유용
    파라미터 바인딩 없어 Injection 위험
    → Kibana에서 검증된 쿼리를 그대로 이식할 때 임시 사용

  @Query (Repository):
    단순 고정 쿼리에 사용
    복잡한 동적 쿼리에 부적합
    → 단순 필터 쿼리에만
```

---

## 🔬 내부 동작 원리

### 1. NativeQuery — 완전한 ES Query DSL

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    /**
     * 동적 조건 + 하이라이팅 + 집계를 포함한 복합 검색
     */
    public ProductSearchResult search(ProductSearchRequest request) {

        // 동적 bool 쿼리 구성
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // 키워드 검색 (must: 점수 계산)
        if (StringUtils.hasText(request.keyword())) {
            boolQuery.must(m -> m
                .multiMatch(mm -> mm
                    .query(request.keyword())
                    .fields("title^3", "description")  // title 가중치 3배
                    .type(TextQueryType.BestFields)
                    .tieBreaker(0.3)));
        }

        // 카테고리 필터 (filter: 점수 없음, 캐시)
        if (StringUtils.hasText(request.category())) {
            boolQuery.filter(f -> f
                .term(t -> t
                    .field("category")
                    .value(request.category())));
        }

        // 가격 범위 필터
        if (request.minPrice() != null || request.maxPrice() != null) {
            boolQuery.filter(f -> f
                .range(r -> {
                    RangeQuery.Builder range = r.field("price");
                    if (request.minPrice() != null)
                        range.gte(JsonData.of(request.minPrice()));
                    if (request.maxPrice() != null)
                        range.lte(JsonData.of(request.maxPrice()));
                    return range;
                }));
        }

        // 재고 있는 상품만
        if (Boolean.TRUE.equals(request.inStockOnly())) {
            boolQuery.filter(f -> f
                .term(t -> t.field("inStock").value(true)));
        }

        // 정렬 설정
        List<SortOptions> sorts = buildSortOptions(request.sortBy());

        // 하이라이팅 설정
        Highlight highlight = Highlight.of(h -> h
            .fields("title", hf -> hf
                .preTags("<em>")
                .postTags("</em>")
                .numberOfFragments(3)));

        // 카테고리별 집계
        Map<String, Aggregation> aggregations = Map.of(
            "by_category", Aggregation.of(a -> a
                .terms(t -> t
                    .field("category")
                    .size(20))),
            "price_stats", Aggregation.of(a -> a
                .stats(s -> s.field("price")))
        );

        // 최종 NativeQuery 조합
        NativeQuery nativeQuery = NativeQuery.builder()
            .withQuery(q -> q.bool(boolQuery.build()))
            .withSort(sorts)
            .withHighlightQuery(new HighlightQuery(
                new HighlightParameters.HighlightParametersBuilder()
                    .build(), null))
            .withAggregation("by_category", aggregations.get("by_category"))
            .withAggregation("price_stats", aggregations.get("price_stats"))
            .withPageable(PageRequest.of(request.page(), request.size()))
            .build();

        SearchHits<EsProduct> hits = operations.search(
            nativeQuery, EsProduct.class);

        return buildResult(hits);
    }

    private List<SortOptions> buildSortOptions(String sortBy) {
        return switch (sortBy) {
            case "price_asc"  -> List.of(SortOptions.of(s -> s
                .field(f -> f.field("price").order(SortOrder.Asc))));
            case "price_desc" -> List.of(SortOptions.of(s -> s
                .field(f -> f.field("price").order(SortOrder.Desc))));
            case "latest"     -> List.of(SortOptions.of(s -> s
                .field(f -> f.field("createdAt").order(SortOrder.Desc))));
            default           -> List.of(SortOptions.of(s -> s
                .score(sc -> sc.order(SortOrder.Desc))));  // 기본: 관련성
        };
    }
}
```

### 2. SearchHits에서 데이터 추출

```java
private ProductSearchResult buildResult(SearchHits<EsProduct> hits) {

    // 총 결과 수
    long totalHits = hits.getTotalHits();

    // 문서 목록 + 하이라이트
    List<ProductSearchItem> items = hits.getSearchHits().stream()
        .map(hit -> {
            EsProduct product = hit.getContent();
            float score = hit.getScore();

            // 하이라이트 추출
            Map<String, List<String>> highlightFields = hit.getHighlightFields();
            List<String> titleHighlights =
                highlightFields.getOrDefault("title", List.of());

            return ProductSearchItem.builder()
                .id(product.getId())
                .title(product.getTitle())
                .titleHighlight(titleHighlights.isEmpty()
                    ? product.getTitle()
                    : String.join("...", titleHighlights))
                .price(product.getPrice())
                .score(score)
                .build();
        })
        .toList();

    // 집계 결과 추출
    Map<String, ElasticsearchAggregation> aggs = hits.getAggregations()
        .aggregations().stream()
        .collect(Collectors.toMap(
            ElasticsearchAggregation::getName,
            Function.identity()
        ));

    // Terms 집계 (카테고리별 수)
    List<CategoryCount> categories = List.of();
    if (aggs.containsKey("by_category")) {
        categories = aggs.get("by_category")
            .aggregation().getAggregate()
            .sterms().buckets().array().stream()
            .map(b -> new CategoryCount(b.key().stringValue(), b.docCount()))
            .toList();
    }

    return ProductSearchResult.builder()
        .totalHits(totalHits)
        .items(items)
        .categories(categories)
        .build();
}
```

### 3. 대용량 페이지네이션 — Search After

```java
/**
 * 대용량 페이지네이션: search_after 패턴
 * from+size 방식의 Deep Pagination 문제 해결
 */
public SearchAfterResult searchWithSearchAfter(
        ProductSearchRequest request,
        List<Object> searchAfterValues) {  // 이전 페이지 마지막 정렬값

    NativeQuery.Builder queryBuilder = NativeQuery.builder()
        .withQuery(q -> q.bool(buildBoolQuery(request)))
        .withSort(List.of(
            SortOptions.of(s -> s.score(sc -> sc.order(SortOrder.Desc))),
            SortOptions.of(s -> s.field(f -> f    // 정렬 일관성을 위해 _id 추가
                .field("_id").order(SortOrder.Asc)))
        ))
        .withPageable(PageRequest.of(0, request.size()));  // from=0 고정

    // search_after 값 지정 (두 번째 페이지부터)
    if (searchAfterValues != null && !searchAfterValues.isEmpty()) {
        queryBuilder.withSearchAfter(searchAfterValues);
    }

    NativeQuery nativeQuery = queryBuilder.build();
    SearchHits<EsProduct> hits = operations.search(nativeQuery, EsProduct.class);

    // 다음 페이지를 위한 마지막 정렬값 추출
    List<Object> nextSearchAfter = null;
    if (!hits.getSearchHits().isEmpty()) {
        SearchHit<EsProduct> lastHit = hits.getSearchHits()
            .get(hits.getSearchHits().size() - 1);
        nextSearchAfter = lastHit.getSortValues();
    }

    return new SearchAfterResult(
        hits.getSearchHits().stream()
            .map(SearchHit::getContent)
            .toList(),
        nextSearchAfter,
        hits.getTotalHits()
    );
}
```

### 4. CriteriaQuery vs NativeQuery 비교

```java
// CriteriaQuery: 간단하지만 표현력 제한
CriteriaQuery criteriaQuery = new CriteriaQuery(
    Criteria.where("category").is("keyboard")
        .and("price").lessThanEqual(200000)
        .and("inStock").is(true)
);
// 문제: BM25 스코어링, 하이라이팅, 집계, 복잡한 bool 표현 어려움

// NativeQuery: 완전한 표현력
NativeQuery nativeQuery = NativeQuery.builder()
    .withQuery(q -> q
        .bool(b -> b
            .filter(f -> f.term(t -> t.field("category").value("keyboard")))
            .filter(f -> f.range(r -> r.field("price").lte(JsonData.of(200000))))
            .filter(f -> f.term(t -> t.field("inStock").value(true)))))
    .build();
// → 동일 결과, 하지만 집계/하이라이팅 추가 가능

// StringQuery: Kibana에서 검증된 쿼리 이식 (임시 용도)
StringQuery stringQuery = new StringQuery("""
    {
      "bool": {
        "must": [{"match": {"title": "keyboard"}}],
        "filter": [{"term": {"category": "keyboard"}}]
      }
    }
    """);
// 주의: 파라미터 동적 바인딩 없음 → 정적 쿼리에만
```

---

## 💻 실전 실험

```java
// 복합 검색 서비스 테스트
@SpringBootTest
class ProductSearchServiceTest {

    @Autowired
    private ProductSearchService searchService;

    @Test
    void testComplexSearch() {
        ProductSearchRequest request = ProductSearchRequest.builder()
            .keyword("기계식 키보드")
            .category("keyboard")
            .minPrice(50000.0)
            .maxPrice(200000.0)
            .inStockOnly(true)
            .sortBy("price_asc")
            .page(0)
            .size(10)
            .build();

        ProductSearchResult result = searchService.search(request);

        assertThat(result.totalHits()).isGreaterThan(0);
        assertThat(result.items()).hasSizeLessThanOrEqualTo(10);
        // 하이라이트 확인
        result.items().forEach(item ->
            assertThat(item.titleHighlight()).contains("<em>")
        );
        // 집계 확인
        assertThat(result.categories()).isNotEmpty();
    }

    @Test
    void testSearchAfterPagination() {
        // 첫 페이지
        SearchAfterResult firstPage = searchService
            .searchWithSearchAfter(request, null);

        // 두 번째 페이지
        SearchAfterResult secondPage = searchService
            .searchWithSearchAfter(request, firstPage.nextSearchAfter());

        // 결과 중복 없어야 함
        Set<String> firstIds = firstPage.items().stream()
            .map(EsProduct::getId)
            .collect(Collectors.toSet());
        secondPage.items().forEach(item ->
            assertThat(firstIds).doesNotContain(item.getId())
        );
    }
}
```

```bash
# NativeQuery가 실제로 보내는 ES 쿼리 확인
# Spring 로그 레벨 설정
logging:
  level:
    org.springframework.data.elasticsearch.client.WIRE: TRACE

# TRACE 로그에서 실제 요청/응답 JSON 확인
```

---

## 📊 쿼리 타입 비교

| 항목 | NativeQuery | CriteriaQuery | StringQuery |
|------|-----------|--------------|------------|
| ES 기능 완전 지원 | ✅ | 부분적 | ✅ |
| 타입 안전성 | ✅ (빌더) | ✅ | ❌ |
| 동적 쿼리 | ✅ | ✅ | ❌ |
| 가독성 | 중간 | 높음 | 낮음 |
| Injection 방지 | ✅ | ✅ | ❌ |
| 권장 상황 | 대부분 | 단순 필터 | 정적 JSON 이식 |

---

## ⚖️ 트레이드오프

```
NativeQuery 복잡도:
  이득: ES 모든 기능 활용 (집계, 하이라이팅, kNN)
  비용: 코드 장황해짐, ES API 변경 시 코드 수정 필요
  → 쿼리 Builder 패턴으로 복잡도 분산

CriteriaQuery:
  이득: Spring Data 스타일로 간결
  비용: 복잡한 ES 기능 표현 한계
  → 단순 CRUD 레이어에서만 사용

Search After vs from+size:
  from+size: 이전 페이지로 돌아갈 수 있음, 깊어질수록 비용 증가
  search_after: 이전 페이지 불가, 항상 O(size) 비용
  → 무한 스크롤: search_after / 페이지 네비게이션: from+size (10페이지 이내)
```

---

## 📌 핵심 정리

```
쿼리 타입 선택:
  NativeQuery → 대부분의 검색 (복잡한 bool, 집계, 하이라이팅)
  CriteriaQuery → 단순 필터링
  StringQuery → 임시/정적 쿼리

동적 쿼리 null 처리:
  if (condition != null) { boolQuery.filter(...) }
  → null인 조건은 쿼리에 추가하지 않음

SearchHits 데이터 추출:
  hit.getContent() → 원본 문서
  hit.getScore() → BM25 점수
  hit.getHighlightFields() → 하이라이트 결과
  hit.getSortValues() → search_after용 정렬값
  hits.getAggregations() → 집계 결과

대용량 페이지네이션:
  search_after + 일관된 정렬(+_id) → Deep Pagination 해결
  이전 페이지 마지막 sortValues를 다음 요청의 searchAfter로 전달
```

---

## 🤔 생각해볼 문제

**Q1.** `multiMatch`에서 `title^3`과 같은 필드 가중치(boost)는 BM25 점수에 어떻게 영향을 미치는가?

<details>
<summary>해설 보기</summary>

boost 값은 해당 필드의 BM25 점수에 곱해진다. `title^3`이면 title 필드에서 계산된 BM25 점수에 3을 곱해 최종 점수를 구성한다. 같은 검색어가 title에 있는 문서는 description에 있는 문서보다 3배 높은 점수를 받는다. 이는 사용자가 검색하는 맥락에서 제목이 설명보다 관련성이 높을 것이라는 가정에 기반한다. `_explain` API로 실제 boost가 적용된 점수 계산 과정을 확인할 수 있다.

</details>

**Q2.** `search_after`를 사용할 때 정렬 기준에 반드시 `_id`를 추가해야 하는 이유는?

<details>
<summary>해설 보기</summary>

`search_after`는 이전 페이지의 마지막 정렬 값을 기준으로 그 이후 문서를 가져온다. 정렬 기준이 점수나 날짜처럼 동점(tie)이 발생할 수 있는 필드라면, 동점인 문서들이 페이지 경계에서 누락되거나 중복될 수 있다. `_id`를 마지막 정렬 기준으로 추가하면 모든 문서가 고유한 정렬 값을 가져 동점을 결정론적으로 처리한다. 이를 통해 페이지 간 문서 누락이나 중복 없이 안정적인 페이지네이션이 보장된다.

</details>

**Q3.** NativeQuery 빌더에서 `withQuery`와 `withFilter`를 동시에 사용하면 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

Spring Data ES에서 `NativeQuery.withFilter()`는 내부적으로 `bool.filter`에 해당 조건을 추가한다. 즉, `withQuery`로 지정한 Query Context 쿼리와 `withFilter`로 지정한 Filter Context 조건이 함께 적용된다. Query DSL의 `bool.must` + `bool.filter` 조합과 동일하다. 그러나 복잡한 쿼리에서는 이 분리보다 `NativeQuery.withQuery(q -> q.bool(b -> b.must(...).filter(...)))`처럼 명시적으로 bool 쿼리를 작성하는 것이 의도가 더 명확하고 혼동을 줄인다.

</details>

---

<div align="center">

**[⬅️ 이전: 인덱싱 전략](./02-indexing-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데이터 동기화 패턴 ➡️](./04-data-sync-patterns.md)**

</div>
