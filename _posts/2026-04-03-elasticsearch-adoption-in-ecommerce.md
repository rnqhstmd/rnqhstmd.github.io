---
layout: post
title: "이커머스 검색에 Elasticsearch를 붙이면서 배운 것들"
date: 2026-04-03
categories: [spring, architecture]
tags: [elasticsearch, nori, ecommerce, search, circuit-breaker]
---

"운동화를"로 검색하면 "운동화"가 안 나왔습니다.

이커머스 프로젝트의 상품 검색은 MySQL `LIKE '%keyword%'`로 구현되어 있었습니다. 키워드가 상품명에 포함되어 있으면 결과가 나오고, 아니면 0건. 단순했고, 처음에는 충분했습니다.

문제는 한글이었습니다. "운동화를"에서 조사 "를"이 붙으면 "운동화"와 다른 문자열이 됩니다. MySQL은 한글 형태소 분석을 지원하지 않아서, 사용자가 자연스럽게 입력한 검색어가 매칭에 실패했습니다. "나이키"를 입력해도 "Nike"를 찾을 수 없고, "나이케"라고 오타를 내면 결과는 0건이었습니다.

검색이 안 되면 사용자는 떠납니다. Elasticsearch를 도입하기로 했습니다.

---

## 기존 검색이 어떻게 생겼는지

QueryDSL로 동적 쿼리를 조합하는 구조였습니다.

```java
public Page<Product> findProducts(ProductSearchCondition condition) {
    BooleanBuilder builder = new BooleanBuilder();

    if (condition.keyword() != null && !condition.keyword().isBlank()) {
        // LIKE '%keyword%' — 앞뒤 와일드카드로 인덱스 사용 불가
        builder.and(product.name.containsIgnoreCase(condition.keyword().trim()));
    }
    if (condition.brandId() != null) {
        builder.and(product.brandId.eq(condition.brandId()));
    }
    if (condition.minPrice() != null) {
        builder.and(product.price.value.goe(condition.minPrice()));
    }
    if (condition.maxPrice() != null) {
        builder.and(product.price.value.loe(condition.maxPrice()));
    }

    builder.and(product.deletedAt.isNull());

    List<Product> content = queryFactory
            .selectFrom(product)
            .where(builder)
            .offset(condition.pageable().getOffset())
            .limit(condition.pageable().getPageSize())
            .fetch();

    Long total = queryFactory
            .select(product.count())
            .from(product)
            .where(builder)
            .fetchOne();

    return new PageImpl<>(content, condition.pageable(), total != null ? total : 0L);
}
```

코드 자체는 깔끔합니다. 문제는 `containsIgnoreCase`가 만들어내는 SQL입니다.

```sql
SELECT * FROM product WHERE name LIKE '%운동화%' AND deleted_at IS NULL
```

`LIKE '%keyword%'`는 앞에 와일드카드가 있어서 B-Tree 인덱스를 못 탑니다. 상품이 1,000개일 때는 괜찮지만, 10만 개가 되면 매 검색마다 풀 테이블 스캔입니다. 거기에 content 쿼리와 count 쿼리를 따로 돌리니까 부하가 두 배입니다.

정리하면 이랬습니다.

| 항목 | 상태 |
|------|------|
| 검색 대상 | `name` 필드 하나 |
| 매칭 방식 | `LIKE '%keyword%'` (substring) |
| 한글 형태소 분석 | 없음 |
| 관련도 점수 | 없음 — 매칭 여부만 판단 |
| 자동완성 | 없음 |
| 오타 교정 | 없음 |

상품명만 검색 가능하고, 브랜드명이나 카테고리명으로는 검색이 안 됐습니다. "나이키 운동화"라고 입력하면 상품명에 "나이키 운동화"가 통째로 들어있어야 결과가 나왔습니다.

---

## 왜 Elasticsearch인가

Elasticsearch의 핵심은 **역인덱스(Inverted Index)**입니다. MySQL의 B-Tree 인덱스가 "이 row에 어떤 값이 있는가"를 찾는다면, 역인덱스는 "이 단어가 어떤 document에 들어있는가"를 찾습니다.

```
일반 인덱스:    Document → Terms
역인덱스:      Term → Documents

"나이키" → [doc1, doc7, doc15]
"운동화" → [doc1, doc3, doc7, doc22]
"에어맥스" → [doc1, doc15]
```

"나이키 운동화"를 검색하면 두 단어의 역인덱스를 교차해서 doc1, doc7을 찾습니다. 상품이 몇 개든 토큰 단위 조회라 속도가 일정합니다. 10만 건 기준으로 MySQL LIKE가 200ms 이상 걸리던 게 ES에서는 10-20ms로 줄었습니다.

여기에 한글 형태소 분석기(Nori)를 붙이면 "운동화를" → "운동화"로 분석해서 조사를 떼어냅니다. 관련도 점수(BM25)로 검색 결과를 정렬할 수 있고, 자동완성이나 집계(Aggregation) 같은 기능도 기본 제공합니다.

---

## 아키텍처 — 검색은 ES, 트랜잭션은 MySQL

ES를 도입한다고 MySQL을 버리는 건 아닙니다. ES는 검색에 특화된 엔진이지, RDB를 대체하는 게 아닙니다. 트랜잭션, 외래 키, ACID 보장은 MySQL이 해야 합니다.

설계 원칙은 **검색 트래픽을 ES로 분리하고, MySQL은 트랜잭션에 집중**하는 겁니다.

**검색 흐름:**

```
Client → Controller → Facade → ProductSearchService (@CircuitBreaker)
  → ProductSearchPort (도메인 인터페이스)
    → ElasticsearchProductSearchAdapter (구현체)
      → Elasticsearch (역인덱스 검색 → ID 목록 반환)
  → ProductService.getProductsByIds(ids)
    → MySQL (상세 정보 조회)
```

ES에서는 검색 조건에 맞는 **상품 ID 목록만** 가져옵니다. 상세 정보(가격, 재고, 설명 등)는 MySQL에서 조회합니다. ES는 검색 색인이지 데이터의 원본(source of truth)이 아닙니다.

**CUD + 인덱스 동기화:**

```
Client → Controller → Facade → ProductService → MySQL (CUD 처리)
  → 트랜잭션 커밋
  → @TransactionalEventListener(AFTER_COMMIT)
    → ProductIndexer → Elasticsearch (인덱스 동기화)
```

상품이 생성/수정/삭제되면 MySQL 트랜잭션이 커밋된 후에 ES 인덱스를 업데이트합니다. 이전에 Kafka 이벤트 발행에서 `AFTER_COMMIT`을 쓴 것과 같은 이유입니다 — 트랜잭션이 롤백됐는데 ES에는 반영되면 데이터가 꼬입니다.

도메인 레이어에 `ProductSearchPort` 인터페이스를 두고, 인프라 레이어에서 `ElasticsearchProductSearchAdapter`로 구현했습니다. ES에 대한 의존이 도메인으로 새어나가지 않습니다. 나중에 ES를 다른 검색 엔진으로 바꿔도 도메인 코드는 안 건드립니다.

---

## 인덱스 설계

### Nori 한글 분석기

ES에 한글 형태소 분석을 시키려면 Nori 플러그인이 필요합니다. Docker 이미지를 빌드할 때 설치했습니다.

```dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:8.17.0
RUN bin/elasticsearch-plugin install --batch analysis-nori
```

Nori는 은전한닢 기반 한글 형태소 분석기입니다. "나이키 에어맥스 운동화를 샀다"를 토크나이징하면:

```
입력: "나이키 에어맥스 운동화를 샀다"
토큰: ["나이키", "에어", "맥스", "운동화", "사다"]
```

조사 "를"이 떨어지고, "샀다"가 원형 "사다"로 바뀝니다. 이제 "운동화"로 검색해도 "운동화를"이 포함된 상품을 찾을 수 있습니다.

### Edge N-gram으로 자동완성

자동완성은 사용자가 타이핑하는 중간에 결과를 보여줘야 합니다. "나이"까지 입력했을 때 "나이키 에어맥스"가 나와야 합니다.

Edge N-gram은 단어의 앞에서부터 잘라서 토큰을 만듭니다.

```
"나이키" → ["나", "나이", "나이키"]
```

색인 시에는 edge_ngram_analyzer로 이렇게 쪼개서 저장하고, 검색 시에는 standard 분석기로 입력값을 그대로 매칭합니다. 색인과 검색의 분석기를 다르게 설정하는 게 핵심입니다.

```json
{
  "name": {
    "type": "text",
    "analyzer": "korean",
    "fields": {
      "autocomplete": {
        "type": "text",
        "analyzer": "edge_ngram_analyzer",
        "search_analyzer": "edge_ngram_search_analyzer"
      }
    }
  }
}
```

`name` 필드 하나에 두 가지 분석기를 붙였습니다. 본문 검색은 `korean` 분석기로, 자동완성은 `name.autocomplete`로 edge_ngram 분석기를 사용합니다.

### 비정규화 — 브랜드명, 카테고리명을 문서에 포함

MySQL에서는 브랜드명을 가져오려면 JOIN이 필요합니다. ES에서는 JOIN이 없습니다(정확히는 있지만 성능이 나쁩니다). 그래서 `ProductDocument`에 브랜드명과 카테고리명을 비정규화해서 넣었습니다.

```java
@Document(indexName = "products", createIndex = false)
public class ProductDocument {
    @Id
    private Long id;
    private String name;
    private Long brandId;
    private String brandName;      // 비정규화
    private Long categoryId;
    private String categoryName;   // 비정규화
    private Long price;
    private Long likeCount;
    private String createdAt;
    private String deletedAt;
}
```

덕분에 "나이키"를 검색하면 상품명, 브랜드명, 카테고리명을 한꺼번에 검색할 수 있습니다. 대신 브랜드명이 바뀌면 해당 브랜드의 모든 상품 문서를 업데이트해야 하는 트레이드오프가 있습니다. 브랜드명이 자주 바뀌는 서비스라면 고민이 필요하지만, 이커머스에서 브랜드명 변경은 드문 일이라 괜찮다고 판단했습니다.

---

## 검색 구현

### multi_match — 여러 필드에서 검색

```java
Query mainQuery;
if (keyword == null || keyword.isBlank()) {
    mainQuery = Query.of(q -> q.matchAll(m -> m));
} else {
    mainQuery = Query.of(q -> q
            .multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name", "brandName", "categoryName")
                    .type(TextQueryType.BestFields)
                    .tieBreaker(0.3)
            )
    );
}
```

`multi_match`로 `name`, `brandName`, `categoryName` 세 필드를 동시에 검색합니다.

- `BestFields`: 가장 점수가 높은 필드를 기준으로 정렬합니다. "나이키"가 상품명에 있는 것과 카테고리명에만 있는 것의 관련도가 다르니까요.
- `tieBreaker(0.3)`: 가장 높은 점수에 나머지 필드 점수의 30%를 더합니다. "나이키"가 상품명과 브랜드명 둘 다에 매칭되면 한 필드에만 매칭된 것보다 점수가 높아집니다.

필터 조건(브랜드, 가격 범위 등)은 `bool` 쿼리의 `filter` context로 넣었습니다. filter context는 점수 계산에 영향을 안 주고, ES가 결과를 캐싱합니다.

```java
SearchResponse<ProductDocument> response = esClient.search(s -> s
        .index("products")
        .query(q -> q.bool(b -> {
            b.must(mainQuery);
            filters.forEach(b::filter);
            return b;
        }))
        .from(page * size)
        .size(size)
        .sort(sortOptions),
    ProductDocument.class
);
```

검색 결과에서는 **상품 ID만** 뽑아서 MySQL에서 상세 정보를 조회합니다. ES를 데이터 원본으로 쓰지 않는다는 원칙을 지킨 겁니다.

### 자동완성

```java
Query prefixQuery = Query.of(q -> q
        .matchPhrasePrefix(mp -> mp
                .field("name.autocomplete")
                .query(prefix)
        )
);
```

`name.autocomplete` 필드에 `match_phrase_prefix`를 씁니다. 사용자가 "나이"까지 입력하면, 색인 시 edge_ngram으로 만들어둔 ["나", "나이", "나이키"] 토큰과 매칭되어 "나이키 에어맥스" 같은 결과가 나옵니다.

### Faceted Search — 브랜드별, 카테고리별, 가격대별 집계

쇼핑몰에서 검색 결과 왼쪽에 "브랜드: 나이키(15), 아디다스(8)" 같은 필터가 나오는 거, 그게 Faceted Search입니다. ES의 Aggregation으로 구현했습니다.

```java
esClient.search(s -> s
    .size(0)  // 검색 결과 자체는 필요 없고, 집계만 수행
    .aggregations("brand_facets", a -> a
            .terms(t -> t.field("brandName.keyword").size(50)))
    .aggregations("category_facets", a -> a
            .terms(t -> t.field("categoryName.keyword").size(50)))
    .aggregations("price_ranges", a -> a
            .range(r -> r.field("price")
                    .ranges(rng -> rng.key("~10,000").to(10000.0))
                    .ranges(rng -> rng.key("10,000~50,000").from(10000.0).to(50000.0))
                    .ranges(rng -> rng.key("50,000~100,000").from(50000.0).to(100000.0))
                    .ranges(rng -> rng.key("100,000~").from(100000.0))
            )),
    ProductDocument.class
);
```

`.size(0)`으로 검색 결과는 안 가져오고 집계 데이터만 받습니다. `brandName.keyword`처럼 `.keyword` 서브필드를 쓰는 이유는, 집계는 분석되지 않은 원본 값 기준으로 해야 하기 때문입니다. `korean` 분석기를 거치면 "나이키 코리아"가 ["나이키", "코리아"]로 쪼개져서 집계가 엉망이 됩니다.

---

## 데이터 동기화

MySQL이 원본이고 ES는 검색용 복제본이니까, 상품 데이터가 바뀔 때마다 ES도 업데이트해야 합니다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleProductCreated(ProductCreatedEvent event) {
    indexProduct(event.productId(), "생성");
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleProductUpdated(ProductUpdatedEvent event) {
    indexProduct(event.productId(), "수정");
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleProductDeleted(ProductDeletedEvent event) {
    productSearchPort.deleteProduct(event.productId());
}
```

`AFTER_COMMIT`을 쓴 이유는 앞서 말한 대로입니다. 트랜잭션이 확정된 후에만 ES에 반영해야 데이터 불일치를 막을 수 있습니다.

동기화 전략을 고를 때 세 가지를 비교했습니다.

| 전략 | 장점 | 단점 | 적합 시나리오 |
|------|------|------|-------------|
| **Application Event** (현재) | 구현 단순, 기존 이벤트 재활용 | 동기화 실패 시 불일치 | 모놀리식 단일 앱 |
| Kafka CDC | 비동기, 느슨한 결합 | 지연 발생, 인프라 복잡 | MSA 환경 |
| Debezium | MySQL binlog 기반, 코드 변경 없음 | 운영 복잡도 높음 | 대규모 시스템 |

프로젝트가 모놀리식 단일 앱이고, `ProductCreatedEvent` 같은 도메인 이벤트가 이미 있었습니다. 추가 인프라 없이 `@TransactionalEventListener`만 붙이면 됐으니까 Application Event를 선택했습니다.

MSA로 전환하거나 서비스가 커지면 Kafka CDC나 Debezium을 검토해야겠지만, 지금 단계에서 과한 인프라를 깔 이유는 없었습니다.

### 전체 재인덱싱

인덱스 매핑을 바꾸거나 데이터 불일치가 의심될 때 쓰는 배치 재인덱싱도 만들었습니다.

```java
@Transactional(readOnly = true)
public long reindexAll() {
    productSearchPort.deleteAllDocuments();

    long indexed = 0;
    int page = 0;

    while (true) {
        Page<Product> productPage = productRepository.findAllPaged(
            PageRequest.of(page, BATCH_SIZE)  // BATCH_SIZE = 1000
        );

        Map<Long, Brand> brandMap = brandService.getBrandsByIds(brandIds);
        Map<Long, Category> categoryMap = categoryService.getCategoriesByIds(categoryIds);

        for (Product product : products) {
            productSearchPort.indexProduct(product, brandName, categoryName);
            indexed++;
        }

        if (!productPage.hasNext()) break;
        page++;
    }
    return indexed;
}
```

1,000건 단위로 페이지를 넘기면서 처리합니다. 브랜드/카테고리 조회를 페이지 단위로 배치해서 N+1을 방지했습니다. Admin 전용 API로 수동 실행합니다.

---

## ES가 죽으면? — Circuit Breaker + MySQL 폴백

ES는 외부 인프라입니다. 죽을 수 있습니다. ES가 죽었다고 검색이 아예 안 되면 안 됩니다.

Resilience4j Circuit Breaker를 걸고, ES 장애 시 기존 MySQL LIKE 쿼리로 폴백하게 만들었습니다.

```java
@CircuitBreaker(name = "elasticsearchSearch", fallbackMethod = "searchFallback")
public ProductSearchInfo search(ProductGetListCommand command) {
    // ES 검색
}

private ProductSearchInfo searchFallback(ProductGetListCommand command, Throwable t) {
    log.warn("ES 검색 Circuit Breaker 폴백 -> MySQL LIKE: {}", t.getMessage());
    Page<Product> page = productService.getProducts(condition);
    // MySQL LIKE 검색으로 폴백
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      elasticsearchSearch:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
```

최근 10개 요청 중 실패율이 50%를 넘으면 Circuit Breaker가 열립니다. 열린 동안에는 ES로 요청을 보내지 않고 바로 MySQL 폴백을 탑니다. 10초 후 반만 열어서(HALF_OPEN) 3개 요청을 시도하고, 성공하면 다시 ES를 씁니다.

| 상태 | 동작 |
|------|------|
| CLOSED | 정상 — ES 검색 |
| OPEN | 실패율 초과 — 검색은 MySQL LIKE, 자동완성/집계는 빈 결과 |
| HALF_OPEN | 3개 요청을 ES로 시도하여 복구 판단 |

자동완성과 집계는 폴백할 대상이 없으니 빈 결과를 반환합니다. 검색은 되는데 자동완성이 안 되는 건 서비스에 치명적이지 않습니다.

ES 인덱스 자동 생성도 같은 원칙입니다. 앱 기동 시 ES 인덱스를 만들려고 시도하는데, 실패해도 예외를 삼키고 앱은 정상 시작됩니다. Circuit Breaker가 MySQL로 폴백해주니까요.

```java
@PostConstruct
public void createIndexIfNotExists() {
    try {
        // 인덱스 존재 확인 → 없으면 JSON 설정으로 생성
    } catch (Exception e) {
        log.error("ES '{}' 인덱스 생성 실패: {}", INDEX_NAME, e.getMessage(), e);
        // 앱 기동은 차단하지 않음
    }
}
```

이전 이커머스 포스트에서 Kafka 이벤트 발행에 Circuit Breaker를 걸었던 것과 같은 패턴입니다. 외부 인프라는 항상 죽을 수 있다는 전제로, 폴백을 준비해두는 게 핵심입니다.

---

## 전후 비교

| 항목 | Before (MySQL LIKE) | After (Elasticsearch) |
|------|--------------------|-----------------------|
| 검색 대상 | `name` 1개 | `name` + `brandName` + `categoryName` |
| 한글 처리 | 없음 | Nori 형태소 분석 |
| 매칭 방식 | substring | 토큰 매칭 + BM25 관련도 |
| 자동완성 | 없음 | Edge N-gram |
| 집계 | 없음 | Terms/Range Aggregation |
| 10만 건 검색 속도 | ~200ms+ (풀스캔) | ~10-20ms (역인덱스) |
| DB 부하 | 검색 + 트랜잭션 공유 | 검색은 ES, 트랜잭션은 MySQL 분리 |

---

## 돌이켜보며

ES를 붙이면서 가장 많이 고민한 건 "ES가 죽으면 어떡하지?"였습니다. 검색 품질을 올리려고 도입한 건데, 장애 포인트가 하나 더 생긴 셈이니까요. Circuit Breaker + MySQL 폴백으로 해결했지만, 폴백 시에는 검색 품질이 원래 수준으로 떨어집니다. 결국 "검색이 아예 안 되는 것보다는 품질이 낮아도 되는 게 낫다"는 판단이었습니다.

데이터 동기화도 고민이었습니다. Application Event 방식은 단순하지만, 이벤트 처리가 실패하면 MySQL과 ES 사이에 불일치가 생깁니다. 지금은 에러 로그 + 수동 재인덱싱으로 대응하고 있는데, 서비스가 커지면 Kafka CDC 같은 방식으로 바꿔야 할 수 있습니다.

아직 구현하지 못한 것도 있습니다. 오타 교정(Fuzzy Query), 동의어 사전("운동화" ↔ "스니커즈"), 검색어 하이라이팅. 하나씩 붙여나갈 생각입니다.

이커머스 프로젝트를 처음 만들 때부터 반복되는 패턴이 있습니다. **단순하게 시작하고, 부족한 지점을 발견하고, 그때 복잡도를 올린다.** MySQL LIKE로 시작해서, 한글 검색이 안 되는 걸 발견하고, ES를 붙인 것도 같은 흐름입니다. 처음부터 ES를 설계했으면 이 판단의 맥락을 몰랐을 겁니다.
