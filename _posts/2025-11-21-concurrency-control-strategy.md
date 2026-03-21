---
layout: post
title: "도메인별 동시성 제어 최적화 전략"
date: 2025-11-21
categories: [spring, architecture]
---

## 주문 기능에서 동시성 문제

주문 기능을 구현하다 보면 세 가지 도메인에서 동시성 이슈가 발생합니다.

- **재고 (Product)**: 동시에 10명이 마지막 1개를 주문하면?
- **포인트 (Point)**: 동시에 여러 결제가 같은 포인트를 차감하면?
- **쿠폰 (Coupon)**: 동시에 여러 사람이 같은 쿠폰을 사용하면?

처음에는 세 도메인 모두 비관적 락(Pessimistic Lock)을 적용했습니다. 일단 안전하게 가자는 생각이었습니다.

## 비관적 락과 낙관적 락 복습

**비관적 락 (Pessimistic Lock)**
- "충돌이 발생할 것이다"라고 가정하고 미리 잠급니다
- `SELECT ... FOR UPDATE`
- 충돌이 많을 때 유리합니다
- 데드락 위험이 있습니다

**낙관적 락 (Optimistic Lock)**
- "충돌이 없을 것이다"라고 가정하고 커밋 시점에 충돌을 감지합니다
- `@Version` 컬럼을 이용한 버전 충돌 감지
- 충돌이 적을 때 유리합니다
- 충돌 시 재시도 로직이 필요합니다

## 도메인별 특성 분석

락 전략을 결정하기 전에 각 도메인의 특성을 분석했습니다.

| 도메인 | 충돌 확률 | 예외 격리 가능성 | 선택 |
|--------|-----------|-----------------|------|
| Product (재고) | 높음 | 낮음 | 비관적 락 |
| Point (포인트) | 높음 | 낮음 | 비관적 락 |
| Coupon (쿠폰) | 낮음 | 높음 | 낙관적 락 |

**재고와 포인트**는 충돌 확률이 높습니다. 인기 상품은 동시에 수십 명이 주문할 수 있고, 포인트도 한 사용자가 여러 탭에서 동시에 결제 시도할 수 있습니다. 비관적 락이 맞습니다.

**쿠폰**은 다릅니다. 보통 쿠폰은 1인 1매 제한이 있고, 같은 쿠폰으로 동시 요청이 들어오는 경우가 드뭅니다. 낙관적 락으로 충돌을 감지하고 예외를 던지는 게 더 가볍습니다.

## 구현

### 재고: 비관적 락

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public void deductStock(Long productId, int quantity) {
        Product product = productRepository.findByIdWithLock(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        product.deductStock(quantity); // 재고 부족 시 도메인 예외
    }
}
```

### 포인트: 비관적 락

```java
public interface PointRepository extends JpaRepository<Point, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Point p WHERE p.userId = :userId")
    Optional<Point> findByUserIdWithLock(@Param("userId") Long userId);
}
```

### 쿠폰: 낙관적 락

```java
@Entity
public class Coupon {

    @Id
    private Long id;

    @Version
    private Long version;

    private boolean used;

    public void use() {
        if (this.used) {
            throw new CouponAlreadyUsedException(this.id);
        }
        this.used = true;
    }
}

public interface CouponRepository extends JpaRepository<Coupon, Long> {

    @Lock(LockModeType.OPTIMISTIC)
    @Query("SELECT c FROM Coupon c WHERE c.id = :id")
    Optional<Coupon> findByIdWithOptimisticLock(@Param("id") Long id);
}
```

낙관적 락은 `@Version` 필드를 통해 동시 수정을 감지합니다. 두 트랜잭션이 같은 `version`으로 업데이트하려 하면 `OptimisticLockException`이 발생합니다.

### OrderFacade에서 조합

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {

    private final ProductService productService;
    private final CouponService couponService;
    private final PointService pointService;
    private final OrderService orderService;

    @Transactional
    public OrderResult order(OrderCommand command) {
        // 비관적 락: 재고 차감
        productService.deductStock(command.getProductId(), command.getQuantity());

        // 낙관적 락: 쿠폰 사용 (충돌 시 OptimisticLockException)
        int discountAmount = 0;
        if (command.getCouponId() != null) {
            discountAmount = couponService.apply(command.getCouponId());
        }

        // 비관적 락: 포인트 차감
        if (command.getPointAmount() > 0) {
            pointService.deduct(command.getUserId(), command.getPointAmount());
        }

        return orderService.create(command, discountAmount);
    }
}
```

## 낙관적 락 예외 처리

낙관적 락은 예외가 발생했을 때 처리 전략이 필요합니다.

쿠폰의 경우, 두 트랜잭션이 동시에 같은 쿠폰을 사용하려 하면 하나는 버전 충돌(`OptimisticLockingFailureException`)로 실패합니다. 재시도하더라도 쿠폰은 이미 사용된 상태이므로 재시도가 아닌 명확한 에러 응답을 주는 게 맞습니다.

```java
@ExceptionHandler(OptimisticLockingFailureException.class)
public ResponseEntity<ErrorResponse> handleOptimisticLockException(
        OptimisticLockingFailureException e) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(new ErrorResponse("이미 처리된 요청입니다. 다시 시도해주세요."));
}
```

재고나 포인트처럼 재시도가 의미 있는 경우에는 `@Retryable`을 활용할 수 있지만, 이 경우는 비관적 락을 쓰는 게 더 직관적입니다.

## 정리

모든 락을 동일하게 처리하는 건 잘못된 접근입니다. 도메인의 특성을 이해하고 적절한 전략을 선택해야 합니다.

- **충돌 확률이 높고, 실패가 치명적**: 비관적 락
- **충돌 확률이 낮고, 실패를 예외로 처리 가능**: 낙관적 락

비관적 락은 안전하지만 성능 비용이 있습니다. 낙관적 락은 가볍지만 충돌 처리 로직이 필요합니다. 도메인을 이해한 설계가 과잉 동기화를 막습니다.
