---
layout: post
title: "1주차 WIL Facade 패턴 도입해보기"
date: 2025-10-29
categories: [architecture, spring]
---

## 처음엔 왜 이렇게 복잡하게 만들었나 싶었습니다

과제 템플릿 프로젝트가 Facade 패턴으로 구성되어 있었습니다. Service 하나에 다 넣으면 되지 않나? 그래서 멘토한테 직접 물어봤습니다.

돌아온 비유가 기가 막혔습니다.

> "검지 서비스, 중지 서비스, 약지 서비스 — 각각 손가락을 펴는 순수한 서비스들이 있다. 이게 순수한 도메인 비즈니스고, 사전적 정의에 가까운 순수한 개념이다. 근데 주먹 서비스를 만드려면? 모든 손가락을 조합해야 한다. 이게 애플리케이션 비즈니스이다."

이걸 듣고 감이 왔습니다. 서비스는 각자 하나의 동작에 집중하고, Facade가 그걸 조립하는 겁니다. 조리사가 재료를 조합하듯이요.

## 근데 Facade가 너무 뚱뚱해지지 않나?

솔직히 이게 첫 번째 걸림돌이었습니다. Facade에 의존성이 잔뜩 쌓이면 그게 그거 아닌가 싶었습니다.

그런데 생각해보면, 누군가는 이 복잡성을 떠안아야 합니다. 문제는 **어디에** 두느냐였습니다.

서비스가 떠안으면? 도메인이 오염되고, 재사용이 안 되고, 테스트가 꼬입니다. Facade가 떠안으면? 도메인은 깔끔하게 남고, 서비스별로 독립 테스트가 됩니다.

복잡성은 사라지지 않습니다. Facade에 몰아넣으면 적어도 도메인 서비스는 단일 책임을 지킬 수 있습니다.

## 실제 코드로 보면

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

`ProductService`는 `ProductRepository`에만 의존합니다. 쿠폰도 모르고 포인트도 모릅니다. 이게 핵심이었습니다.

## 코드 짤 때 자문해볼 것

- 이 서비스가 자기 도메인 로직만 갖고 있는가?
- Facade만 여러 서비스를 조합하고 있는가?
- 서비스가 자기 도메인 Repository 외에 다른 걸 의존하고 있진 않은가?

기능을 추가할 때 서비스를 건드려야 하면 뭔가 잘못된 겁니다. Facade만 수정하면 되는 구조가 맞다고 느꼈습니다.

## 돌아보면

처음엔 Facade가 쓸데없이 복잡하다고 생각했습니다. 그런데 "주먹 서비스" 비유를 듣고 나서 설계 의도가 한 번에 들어왔습니다. 도메인 서비스는 손가락 하나처럼 순수해야 하고, 그걸 쥐어서 주먹을 만드는 건 Facade가 할 일이었습니다. 이 분리를 하니까 테스트 짤 때도, 기능 추가할 때도 덜 겁이 났습니다.
