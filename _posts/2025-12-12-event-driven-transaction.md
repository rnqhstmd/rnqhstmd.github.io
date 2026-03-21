---
layout: post
title: "무거운 트랜잭션, 이벤트로 가볍게 만들기"
date: 2025-12-12
categories: [spring, architecture]
---

## 2.8초짜리 트랜잭션

주문 생성 API의 응답 시간을 측정했더니 2.8초가 나왔습니다. 트랜잭션 안에서 일어나는 일들을 나열해봤습니다.

```
주문 생성 트랜잭션 (2.8초)
├── 재고 차감 (150ms) - 핵심 비즈니스
├── 주문 생성 (100ms) - 핵심 비즈니스
├── 쿠폰 사용 처리 (200ms) - 부가 작업
├── 결제 데이터 전송 (2000ms) - 외부 시스템 호출
└── 데이터 웨어하우스 전송 (350ms) - 분석용 부가 작업
```

대부분의 시간을 외부 시스템 호출과 부가 작업이 차지하고 있었습니다. 핵심 작업(재고 + 주문)은 250ms입니다.

## 핵심과 부가를 분리하는 기준

모든 작업이 같은 트랜잭션에 있을 필요가 없습니다. 분리 기준을 정했습니다.

**동기로 처리해야 하는 것 (핵심):**
- 재고 차감 - 즉시 반영이 필요합니다
- 주문 생성 - 주문 결과를 응답해야 합니다

**비동기로 처리해도 되는 것 (부가):**
- 쿠폰 사용 처리 - 주문 성공 후 처리해도 됩니다
- 결제 데이터 전송 - 외부 시스템이 잠깐 늦어도 됩니다
- 데이터 웨어하우스 전송 - 분석 데이터는 약간의 지연이 허용됩니다

## Spring ApplicationEvent로 비동기 분리

```java
// 주문 생성 완료 이벤트
public class OrderCreatedEvent {
    private final Long orderId;
    private final Long userId;
    private final Long couponId;
    private final PaymentData paymentData;

    // constructor, getters
}
```

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {

    private final ProductService productService;
    private final OrderService orderService;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public OrderResult order(OrderCommand command) {
        // 핵심: 재고 차감 + 주문 생성 (동기)
        productService.deductStock(command.getProductId(), command.getQuantity());
        Order order = orderService.create(command);

        // 이벤트 발행 (트랜잭션 커밋 후 처리)
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            command.getUserId(),
            command.getCouponId(),
            command.getPaymentData()
        ));

        return OrderResult.from(order);
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class OrderEventHandler {

    private final CouponService couponService;
    private final DataWarehouseClient dataWarehouseClient;
    private final PaymentDataClient paymentDataClient;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 쿠폰 처리 (별도 트랜잭션) - 실패해도 다음 작업은 계속 진행합니다
        if (event.getCouponId() != null) {
            try {
                couponService.markAsUsed(event.getCouponId());
            } catch (Exception e) {
                log.error("쿠폰 처리 실패: couponId={}", event.getCouponId(), e);
            }
        }

        // 결제 데이터 전송
        try {
            paymentDataClient.send(event.getPaymentData());
        } catch (Exception e) {
            log.error("결제 데이터 전송 실패: orderId={}", event.getOrderId(), e);
        }

        // 데이터 웨어하우스 전송
        try {
            dataWarehouseClient.send(event.getOrderId());
        } catch (Exception e) {
            log.error("데이터 웨어하우스 전송 실패: orderId={}", event.getOrderId(), e);
        }
    }
}
```

`@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` 는 트랜잭션이 커밋된 후에만 이벤트를 처리합니다. 주문이 롤백됐는데 쿠폰이 처리되는 불상사를 막습니다.

`@Async`는 별도 스레드에서 비동기로 처리합니다. `@EnableAsync`를 설정 클래스에 추가해야 합니다.

## 별도 트랜잭션 처리

이벤트 핸들러는 새로운 트랜잭션에서 동작해야 합니다.

```java
@Service
@RequiredArgsConstructor
public class CouponService {

    private final CouponRepository couponRepository;

    @Transactional
    public void markAsUsed(Long couponId) {
        Coupon coupon = couponRepository.findById(couponId)
            .orElseThrow();
        coupon.use();
    }
}
```

`@Async` 핸들러는 별도 스레드에서 실행되므로 기존 트랜잭션 컨텍스트가 전파되지 않습니다. 따라서 `REQUIRES_NEW`가 아닌 기본 `@Transactional`만으로도 새 트랜잭션이 생성됩니다. 동기 이벤트 핸들러(`@Async` 없이)에서 기존 트랜잭션과 완전히 분리된 새 트랜잭션이 필요한 경우에는 `REQUIRES_NEW`가 필요합니다. 쿠폰 처리가 실패해도 주문은 이미 커밋됐으므로 영향받지 않습니다.

## 좋아요 기능에도 적용

상품 좋아요 기능에서도 락 경합 문제가 있었습니다. 좋아요를 누를 때마다 `product` 테이블의 `like_count`를 업데이트하는데, 동시 요청이 많으면 락 경합이 발생했습니다.

```java
// Before: 매 요청마다 직접 업데이트
@Transactional
public void like(Long userId, Long productId) {
    productLikeRepository.save(new ProductLike(userId, productId));

    // 이 업데이트가 락 경합 유발
    Product product = productRepository.findByIdWithLock(productId);
    product.increaseLikeCount();
}
```

이벤트로 분리했습니다.

```java
@Transactional
public void like(Long userId, Long productId) {
    productLikeRepository.save(new ProductLike(userId, productId));
    eventPublisher.publishEvent(new ProductLikedEvent(productId));
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async
public void handleProductLiked(ProductLikedEvent event) {
    // 비동기로 처리하거나, 배치로 집계
    productService.incrementLikeCount(event.getProductId());
}
```

좋아요 저장과 카운트 업데이트를 분리해서 락 경합을 줄였습니다.

## 개선 결과

```
Before: 주문 생성 응답 시간 = 2.8초
After:  주문 생성 응답 시간 = 280ms (재고 + 주문만 동기 처리)
```

약 10배 개선입니다.

## 한계: 이 방식의 문제점

그런데 이 구조에는 한계가 있습니다.

**이벤트 유실:** 애플리케이션이 이벤트를 발행하고 처리하기 전에 재시작되면 이벤트가 사라집니다. `ApplicationEvent`는 메모리 내 이벤트이기 때문입니다.

**순서 보장 불가:** 여러 이벤트 핸들러가 병렬로 실행되면 순서가 보장되지 않습니다.

**재시도 없음:** 핸들러가 실패하면 이벤트가 손실됩니다.

이 한계들을 해결하려면 메시지 브로커가 필요합니다. 다음 단계는 Kafka 도입입니다.
