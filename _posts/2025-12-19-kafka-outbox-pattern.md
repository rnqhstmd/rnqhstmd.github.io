---
layout: post
title: "Kafka 도입기: ApplicationEvent에서 신뢰할 수 있는 이벤트 파이프라인으로"
date: 2025-12-19
categories: [kafka, spring, architecture]
---

## ApplicationEvent의 한계

지난 주 `ApplicationEvent`로 트랜잭션을 분리했습니다. 그런데 이 구조의 한계가 명확해졌습니다.

**1. 메모리 기반 - 서버 재시작 시 유실**

이벤트가 메모리에만 존재합니다. 이벤트를 발행하고 핸들러가 처리하기 전에 서버가 재시작되면 이벤트는 사라집니다.

**2. 순서 보장 없음**

비동기 핸들러가 여러 개일 때 실행 순서를 보장할 수 없습니다.

**3. 재시도 없음**

핸들러가 실패하면 이벤트가 손실됩니다. 재시도 로직을 직접 구현해야 합니다.

## Kafka를 선택한 이유

메시지 브로커 선택지는 여러 가지입니다 (RabbitMQ, ActiveMQ 등). Kafka를 선택한 핵심 이유는 **재생(Replay) 기능**입니다.

- **내구성**: 메시지가 디스크에 저장됩니다. 컨슈머가 다운돼도 재시작 후 이어서 처리할 수 있습니다.
- **재생**: 과거 메시지를 다시 처리할 수 있습니다. 버그 수정 후 재처리, 새 서비스 투입 시 히스토리 처리에 유용합니다.
- **순서 보장**: 같은 파티션 내에서 순서가 보장됩니다.

## 문제: 메시지 발행과 DB 저장의 원자성

Kafka 도입 전에 해결해야 할 근본적인 문제가 있었습니다.

```java
@Transactional
public void order(OrderCommand command) {
    Order order = orderService.create(command);
    // DB 트랜잭션 커밋
    // ...
    kafkaTemplate.send("order-created", event); // 여기서 실패하면?
}
```

DB에 주문이 저장됐는데 Kafka 발행이 실패하면 이벤트가 유실됩니다. 반대로 Kafka 발행은 됐는데 DB 커밋이 실패하면 유령 이벤트가 됩니다.

DB 트랜잭션과 Kafka 발행은 원자적으로 처리돼야 합니다. 그러나 두 시스템은 같은 트랜잭션을 공유할 수 없습니다.

## 해결: Transactional Outbox Pattern

핵심 아이디어: **이벤트를 Kafka에 직접 발행하지 않고, 같은 DB 트랜잭션 안에 `outbox` 테이블에 저장합니다.**

```sql
CREATE TABLE outbox (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSON NOT NULL,
    created_at DATETIME NOT NULL,
    published BOOLEAN DEFAULT FALSE,
    published_at DATETIME
);
```

```java
@Transactional
public OrderResult order(OrderCommand command) {
    // 핵심 비즈니스: 재고 차감 + 주문 생성
    productService.deductStock(command.getProductId(), command.getQuantity());
    Order order = orderService.create(command);

    // Outbox에 이벤트 저장 (같은 트랜잭션)
    outboxRepository.save(new OutboxEvent(
        "ORDER",
        order.getId(),
        "ORDER_CREATED",
        objectMapper.writeValueAsString(OrderCreatedPayload.from(order))
    ));

    // 여기까지가 하나의 트랜잭션
    return OrderResult.from(order);
}
```

주문 생성과 이벤트 저장이 같은 트랜잭션입니다. 주문이 롤백되면 이벤트도 롤백됩니다. 원자성이 보장됩니다.

## Outbox 발행 스케줄러

```java
@Component
@RequiredArgsConstructor
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000) // 1초마다
    public void publishPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository.findByPublishedFalse();

        for (OutboxEvent event : pendingEvents) {
            try {
                // 동기 전송 (.get()으로 결과 대기)
                kafkaTemplate.send(
                    event.getEventType().toLowerCase(),
                    String.valueOf(event.getAggregateId()),
                    event.getPayload()
                ).get(); // 발행 성공 확인

                event.markAsPublished();
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Outbox 발행 실패: id={}", event.getId(), e);
                // 다음 스케줄에서 재시도
            }
        }
    }
}
```

`.get()`으로 동기 확인을 하는 이유가 있습니다. 발행 성공이 확인된 후에만 `published = true`로 표시해야 합니다. 비동기로 처리하면 발행 실패를 감지하기 어렵습니다.

## Consumer: 멱등성 보장

Outbox 패턴은 At Least Once 전송을 보장합니다. 즉, 같은 메시지가 여러 번 전달될 수 있습니다. 컨슈머는 멱등하게 처리해야 합니다.

```java
@KafkaListener(topics = "order_created", groupId = "coupon-service")
public void handleOrderCreated(ConsumerRecord<String, String> record) {
    String eventId = record.headers().lastHeader("eventId").value().toString();

    // 이미 처리된 이벤트인지 확인
    if (eventHandledRepository.existsByEventId(eventId)) {
        log.info("이미 처리된 이벤트: {}", eventId);
        return;
    }

    OrderCreatedPayload payload = objectMapper.readValue(
        record.value(), OrderCreatedPayload.class
    );

    // 실제 처리
    couponService.markAsUsed(payload.getCouponId());

    // 처리 완료 기록
    eventHandledRepository.save(new EventHandled(eventId));
}
```

`event_handled` 테이블에 처리한 이벤트 ID를 저장해서 중복 처리를 방지합니다.

## Partition Key로 순서 보장

같은 주문의 이벤트는 같은 파티션에서 처리돼야 순서가 보장됩니다.

```java
kafkaTemplate.send(
    "order-created",
    String.valueOf(order.getId()), // Partition Key = 주문 ID
    payload
);
```

같은 `orderId`를 가진 메시지는 항상 같은 파티션으로 라우팅됩니다.

## DLQ (Dead Letter Queue) 처리

처리를 반복해도 실패하는 메시지는 DLQ로 보냅니다.

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);

    // 3번 재시도 후 DLQ로 전송
    FixedBackOff backOff = new FixedBackOff(1000L, 3);

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
    return handler;
}
```

DLQ에 쌓인 메시지는 별도로 모니터링하고 처리합니다.

## At Least Once + Idempotency = Exactly Once Semantics

Kafka는 기본적으로 At Least Once를 보장합니다. 완벽한 Exactly Once를 달성하려면:

```
Outbox Pattern (발행 누락 없음)
    +
At Least Once (발행 중복 가능)
    +
Consumer 멱등성 (중복 처리 방지)
    =
Exactly Once Semantics
```

각 요소가 역할을 분담해서 전체적으로 정확히 한 번 처리를 달성합니다.

## 정리

`ApplicationEvent`의 한계를 Kafka + Transactional Outbox Pattern으로 해결했습니다.

| 항목 | ApplicationEvent | Kafka + Outbox |
|------|-----------------|----------------|
| 내구성 | 메모리, 재시작 시 유실 | 디스크 저장 |
| 원자성 | AFTER_COMMIT으로 부분 보장 | DB와 같은 트랜잭션 |
| 순서 보장 | 없음 | 파티션 키로 보장 |
| 재시도 | 없음 | 자동 재시도 + DLQ |
| 복잡도 | 낮음 | 높음 |

복잡도가 높아진 만큼 신뢰성이 올라갔습니다. 이벤트 유실이 치명적인 비즈니스 로직에는 Outbox Pattern이 적합합니다.
