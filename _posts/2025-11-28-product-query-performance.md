---
layout: post
title: "10만 건 상품 데이터 조회 성능 개선기 - 인덱스와 캐시 전략"
date: 2025-11-28
categories: [spring, redis]
---

## 문제 정의

상품 목록 조회 API가 느렸다. 10만 건의 데이터를 기준으로 브랜드별 가격 범위 필터링 + 정렬 + 페이징을 처리하는데 체감상 느리다는 피드백이 있었다. 측정을 시작했다.

## 현실적인 테스트 데이터 생성

성능 테스트는 실제 데이터와 유사한 분포로 해야 의미 있다.

```sql
-- 10만 건 데이터 생성 프로시저
DELIMITER //
CREATE PROCEDURE generate_products()
BEGIN
    DECLARE i INT DEFAULT 0;
    WHILE i < 100000 DO
        INSERT INTO product (name, brand, price, like_count, created_at)
        VALUES (
            CONCAT('상품_', i),
            ELT(1 + FLOOR(RAND() * 10),
                'Nike', 'Adidas', 'Puma', 'New Balance', 'Converse',
                'Vans', 'Reebok', 'Asics', 'Saucony', 'Brooks'),
            FLOOR(10000 + RAND() * 490000), -- 10,000 ~ 500,000원
            FLOOR(RAND() * 10000),
            DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY)
        );
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;

CALL generate_products();
```

브랜드 분포를 균등하게, 가격은 실제 상품처럼 분포시켰다.

## 1단계: 쿼리 분석

```sql
EXPLAIN SELECT *
FROM product
WHERE brand = 'Nike'
  AND price BETWEEN 50000 AND 200000
ORDER BY price ASC
LIMIT 20 OFFSET 0;
```

`EXPLAIN` 결과를 보니 `type: ALL`이었다. 풀 테이블 스캔이다. 10만 건을 모두 읽고 필터링하고 있었다.

## 2단계: 복합 인덱스 설계

단순 인덱스를 각각 거는 것보다, 쿼리 패턴에 맞는 복합 인덱스가 효과적이다.

```sql
-- 브랜드 + 가격 복합 인덱스
CREATE INDEX idx_brand_price ON product(brand, price);
```

인덱스 컬럼 순서가 중요하다. `WHERE brand = ?` 조건이 먼저 오고 `ORDER BY price`가 이어지는 패턴이므로 `(brand, price)` 순서가 맞다.

### 성능 비교

```sql
-- 인덱스 없이 강제 실행 (IGNORE INDEX)
SELECT SQL_NO_CACHE *
FROM product IGNORE INDEX (idx_brand_price)
WHERE brand = 'Nike'
  AND price BETWEEN 50000 AND 200000
ORDER BY price ASC
LIMIT 20;
-- 실행 시간: 847ms

-- 인덱스 활용
SELECT SQL_NO_CACHE *
FROM product
WHERE brand = 'Nike'
  AND price BETWEEN 50000 AND 200000
ORDER BY price ASC
LIMIT 20;
-- 실행 시간: 350ms
```

**58.6% 개선.** 그런데 아직 350ms는 느리다. 상품 목록은 자주 조회되는 API이기 때문에 캐시를 적용하기로 했다.

## 3단계: Redis 캐시 적용

브랜드별 가격 범위 필터는 파라미터 조합이 많지만, 자주 사용되는 조합은 제한적이다. Redis에 결과를 캐싱한다.

```java
@Service
@RequiredArgsConstructor
public class ProductQueryService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final Duration CACHE_TTL = Duration.ofMinutes(5);

    public Page<ProductResponse> getProducts(ProductSearchRequest request, Pageable pageable) {
        String cacheKey = buildCacheKey(request, pageable);

        // 캐시 조회
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return (Page<ProductResponse>) cached;
        }

        // DB 조회
        Page<ProductResponse> result = productRepository.findByCondition(request, pageable);

        // 캐시 저장
        redisTemplate.opsForValue().set(cacheKey, result, CACHE_TTL);

        return result;
    }

    private String buildCacheKey(ProductSearchRequest request, Pageable pageable) {
        return String.format("products:brand:%s:minPrice:%d:maxPrice:%d:page:%d:size:%d:sort:%s",
            request.getBrand(),
            request.getMinPrice(),
            request.getMaxPrice(),
            pageable.getPageNumber(),
            pageable.getPageSize(),
            pageable.getSort().toString()
        );
    }
}
```

### 캐시 적용 후 성능

```
인덱스만: 350ms
인덱스 + Redis 캐시 (캐시 히트): 72ms
```

**79.4% 추가 개선.** 인덱스 대비 총 91.5% 개선이다.

## 4단계: 비정규화 - likeCount 컬럼

"좋아요 많은 순" 정렬 기능이 있었는데, 이 경우 `like` 테이블을 COUNT하는 쿼리가 실행됐다.

```sql
-- 느린 쿼리
SELECT p.*, COUNT(l.id) as like_count
FROM product p
LEFT JOIN product_like l ON p.id = l.product_id
WHERE p.brand = 'Nike'
GROUP BY p.id
ORDER BY like_count DESC;
```

`product` 테이블에 `like_count` 컬럼을 추가해 비정규화했다.

```java
@Entity
public class Product {
    // ...
    private int likeCount;

    public void increaseLikeCount() {
        this.likeCount++;
    }

    public void decreaseLikeCount() {
        if (this.likeCount > 0) {
            this.likeCount--;
        }
    }
}
```

JOIN 없이 단순 컬럼 정렬로 처리 가능해졌다.

## 5단계: 캐시 무효화 전략

캐시를 사용하면 데이터 일관성 문제가 생긴다. 상품 정보나 좋아요 수가 바뀌면 캐시를 어떻게 처리할까?

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    @Transactional
    public void updateProduct(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        product.update(request);

        // 해당 브랜드의 캐시 무효화
        invalidateBrandCache(product.getBrand());
    }

    private void invalidateBrandCache(String brand) {
        Set<String> keys = redisTemplate.keys("products:brand:" + brand + ":*");
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
}
```

패턴 기반 키 삭제는 `KEYS` 명령어를 사용하므로 프로덕션에서는 주의가 필요하다. 큰 규모라면 `SCAN` 명령어를 활용하거나, 캐시 키 구조를 변경해 태그 기반 무효화를 고려해야 한다.

## 정리

성능 개선은 측정에서 시작한다. 추측으로 최적화하면 의미 없는 작업이 된다.

적용한 전략 요약:

1. **복합 인덱스**: 쿼리 패턴에 맞는 (brand, price) 인덱스로 58.6% 개선
2. **Redis 캐싱**: 캐시 히트 시 79.4% 추가 개선
3. **비정규화**: like_count 컬럼으로 JOIN 제거
4. **캐시 무효화**: 데이터 변경 시 관련 캐시 삭제

각 전략은 트레이드오프가 있다. 인덱스는 쓰기 성능에 영향을 주고, 캐시는 일관성 관리가 필요하며, 비정규화는 업데이트 복잡도를 높인다. 측정하고 판단하는 과정이 중요하다.
