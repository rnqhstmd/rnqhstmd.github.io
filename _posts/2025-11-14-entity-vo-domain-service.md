---
layout: post
title: "Entity, Value Object, Domain Service 구분이 실제로 뭘 바꿨나"
date: 2025-11-14
categories: [architecture, spring]
---

## DDD 개념 정리

처음에는 Entity, VO, Domain Service를 구분하는 게 이론적인 개념 정리 정도로만 느꼈습니다. 실제 코드에 적용해보니 설계가 달라졌습니다.

### Entity

- 고유한 식별자가 있습니다 (`id`)
- 시간이 지남에 따라 상태가 변합니다
- 같은 `id`를 가지면 동일한 객체입니다

```java
@Entity
public class Order {
    @Id
    private Long id;
    private Long userId;
    private OrderStatus status;
    // ...
}
```

### Value Object (VO)

- 값 자체가 본질입니다
- 불변(Immutable)입니다
- 값이 같으면 동일한 객체입니다 (`id` 필요 없음)

```java
@Embeddable
public class Money {
    private final int amount;

    public Money(int amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("금액은 0 이상이어야 합니다.");
        }
        this.amount = amount;
    }

    public Money add(Money other) {
        return new Money(this.amount + other.amount);
    }

    public Money subtract(Money other) {
        return new Money(this.amount - other.amount);
    }
}
```

### Domain Service

- 상태가 없습니다 (Stateless)
- 여러 Entity나 VO 간의 협력이 필요한 로직을 담당합니다
- 특정 Entity에 속하지 않는 도메인 로직입니다

## 실제 적용: OrderItemPrice와 OrderTotalAmount를 VO로

기존 코드에서는 금액을 `int`로 다루고 있었습니다.

```java
// 변경 전
@Entity
public class OrderItem {
    private int price;
    private int quantity;

    public int calculateSubtotal() {
        return price * quantity;
    }
}

// Order에서
int totalAmount = 0;
for (OrderItem item : items) {
    totalAmount += item.calculateSubtotal();
}
if (totalAmount < 0) { // 여기저기 검증 코드가 흩어진다
    throw new IllegalStateException("...");
}
```

금액 검증 로직이 여러 곳에 흩어져 있고, `int`로 다루다 보니 음수 체크를 매번 해야 했습니다. 솔직히 이게 첫 번째 걸림돌이었습니다.

VO로 추출하면:

```java
// 변경 후
@Embeddable
public class OrderItemPrice {
    private final Money price;
    private final int quantity;

    public OrderItemPrice(Money price, int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("수량은 1 이상이어야 합니다.");
        }
        this.price = price;
        this.quantity = quantity;
    }

    public Money calculateSubtotal() {
        return new Money(price.getAmount() * quantity);
    }
}

@Embeddable
public class OrderTotalAmount {
    private final Money amount;

    public OrderTotalAmount(List<OrderItemPrice> items) {
        this.amount = items.stream()
            .map(OrderItemPrice::calculateSubtotal)
            .reduce(new Money(0), Money::add);
    }

    public OrderTotalAmount applyDiscount(Money discountAmount) {
        return new OrderTotalAmount(this.amount.subtract(discountAmount));
    }
}
```

금액 연산과 검증이 VO 내부에 캡슐화됐습니다. `Order` 코드가 단순해졌습니다.

```java
@Entity
public class Order {
    @Embedded
    private OrderTotalAmount totalAmount;

    public void applyDiscount(Money discountAmount) {
        this.totalAmount = totalAmount.applyDiscount(discountAmount);
    }
}
```

## 무엇이 달라졌나

**버그 원천 차단:** 음수 금액은 `Money` 생성 시점에 차단됩니다. 이전에는 연산 결과를 확인해야 했습니다.

**테스트 용이성:** `Money`, `OrderItemPrice` 같은 VO는 독립적으로 테스트할 수 있습니다. 의존성이 없습니다.

```java
@Test
void 금액은_음수가_될_수_없다() {
    assertThatThrownBy(() -> new Money(-1))
        .isInstanceOf(IllegalArgumentException.class);
}

@Test
void 금액_덧셈() {
    Money a = new Money(1000);
    Money b = new Money(500);
    assertThat(a.add(b)).isEqualTo(new Money(1500));
}
```

**코드 가독성:** `int totalAmount` 대신 `OrderTotalAmount totalAmount`를 보면 의도가 명확합니다.

## @Embeddable 활용

JPA에서 VO를 사용할 때 `@Embeddable`을 활용하면 별도 테이블 없이 값 객체를 매핑할 수 있습니다.

```java
@Entity
public class Order {
    @Id
    private Long id;

    @Embedded
    private OrderTotalAmount totalAmount;

    @ElementCollection
    private List<OrderItemPrice> items;
}
```

DB 스키마는 단순하게 유지하면서 도메인 모델은 풍부하게 가져갈 수 있습니다.

## 정리

Entity, VO, Domain Service 구분은 코드 구조의 문제입니다. VO로 값을 표현하면 그 값에 관련된 로직과 검증이 한 곳에 모입니다. Entity는 상태 변화와 식별에 집중할 수 있습니다. 저는 이 분리가 실제로 버그를 줄이고 테스트를 단순하게 만들었다고 느꼈습니다.
