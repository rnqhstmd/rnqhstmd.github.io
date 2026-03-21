---
layout: post
title: "1주차 WIL Facade 패턴 도입해보기"
date: 2025-10-29
categories: [architecture, spring]
---

## Facade 패턴을 왜 쓰는가

과제의 템플릿 프로젝트가 Facade 패턴으로 구성되어 있었다. 처음엔 왜 이렇게 복잡하게 만들었나 싶었다. 그래서 멘토에게 직접 물어봤다.

멘토로부터 받은 핵심 비유는 이랬다.

> "검지 서비스, 중지 서비스, 약지 서비스 각각의 손가락을 펴는 순수한 서비스들이 있다. 이게 순수한 도메인 비즈니스고, 사전적 정의에 가까운 순수한 개념이다. 근데 주먹 서비스를 만드려면? 모든 손가락을 조합해야하는 일이 발생한다. 이게 애플리케이션 비즈니스이다."

즉, 각 서비스는 도메인의 사전적 정의에 집중하고, Facade는 조리사처럼 재료를 조합하는 책임을 담당한다.

## Facade가 너무 많은 의존성을 지니는데 괜찮을까?

누군가는 이 복잡성을 감당해야 한다. 선택지는 두 가지다.

**서비스가 복잡성을 담당하는 경우:**
- 도메인이 오염된다
- 재사용이 불가능해진다
- 테스트가 어려워진다

**Facade가 복잡성을 담당하는 경우:**
- 도메인은 깔끔하게 유지된다
- 각 서비스를 독립적으로 테스트할 수 있다
- 조합 로직이 한 곳에 집중된다

결국 복잡성은 사라지지 않는다. 어디에 위치시킬 것인지의 문제다. Facade에 복잡성을 몰아넣으면, 도메인 서비스는 단일 책임을 유지할 수 있다.

## 구조 예시

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
        // 재고 차감
        productService.deductStock(command.getProductId(), command.getQuantity());

        // 쿠폰 적용
        int discountAmount = couponService.apply(command.getCouponId());

        // 포인트 차감
        pointService.deduct(command.getUserId(), command.getPointAmount());

        // 주문 생성
        return orderService.create(command, discountAmount);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public void deductStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        product.deductStock(quantity);
    }
}
```

`ProductService`는 오직 `ProductRepository`에만 의존한다. 다른 도메인을 전혀 모른다.

## 실무 체크리스트

이 패턴을 적용할 때 스스로 확인해야 할 질문들이다.

- 서비스가 도메인 비즈니스만 담당하는가?
- Facade는 애플리케이션 비즈니스를 조합하고 있는가?
- 서비스는 본인 도메인의 Repository만 의존하는가?

세 가지 질문에 모두 "예"라고 답할 수 있다면 올바른 구조다.

## 정리

Facade 패턴을 처음 접했을 때는 불필요하게 복잡하다고 느꼈다. 하지만 "주먹 서비스" 비유를 듣고 나서야 설계 의도가 명확하게 이해됐다. 도메인 서비스는 하나의 손가락처럼 순수해야 하고, 손가락들을 조합하는 건 Facade의 역할이다. 이 분리가 도메인의 재사용성과 테스트 용이성을 동시에 확보해준다.
