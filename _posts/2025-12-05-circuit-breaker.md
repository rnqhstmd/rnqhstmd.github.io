---
layout: post
title: "Circuit Breaker로 장애 전파 막기"
date: 2025-12-05
categories: [spring, architecture]
---

## 문제: PG 장애가 전체 주문을 멈추게 한다

PG(Payment Gateway) 시스템이 간헐적으로 응답이 느려지는 상황이 발생했습니다. 결과는 예상보다 심각했습니다. PG 응답을 기다리는 스레드들이 쌓이면서 전체 주문 API가 응답 불능 상태가 됐습니다.

PG 하나의 장애가 주문 서비스 전체를 마비시켰습니다.

## 단계적 해결: Timeout → Retry → Circuit Breaker

### 1단계: Timeout

일단 무한정 기다리지 않도록 타임아웃을 설정했습니다.

```java
@FeignClient(name = "pg-client", configuration = PgClientConfig.class)
public interface PgClient {

    @PostMapping("/payments")
    PaymentResponse pay(@RequestBody PaymentRequest request);
}

@Configuration
public class PgClientConfig {

    @Bean
    public Request.Options options() {
        return new Request.Options(
            1000, TimeUnit.MILLISECONDS,  // connectTimeout
            3000, TimeUnit.MILLISECONDS,  // readTimeout
            true
        );
    }
}
```

이제 3초 이상 기다리지 않습니다. 그런데 3초씩 실패하는 요청들이 쌓이면 여전히 느립니다.

### 2단계: Retry

일시적인 장애라면 재시도로 해결될 수 있습니다.

```java
@Configuration
public class RetryConfig {

    @Bean
    public Retryer retryer() {
        return new Retryer.Default(
            100,   // 초기 대기 시간 (ms)
            1000,  // 최대 대기 시간 (ms)
            3      // 최대 재시도 횟수
        );
    }
}
```

그런데 여기서 함정이 있습니다. **PG 시스템이 이미 결제를 처리했는데 응답만 늦은 경우**, 재시도를 하면 중복 결제가 발생할 수 있습니다.

재시도는 멱등성이 보장되는 경우에만 안전합니다.

### 3단계: Circuit Breaker

재시도로도 해결되지 않는 상황, 즉 PG 시스템이 완전히 다운된 경우에는 Circuit Breaker가 필요합니다.

Circuit Breaker의 상태:
- **CLOSED**: 정상 동작. 요청이 그대로 전달됩니다.
- **OPEN**: 장애 감지. 요청을 차단하고 즉시 Fallback을 반환합니다.
- **HALF_OPEN**: 일부 요청을 통과시켜 복구 여부를 확인합니다.

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      pg-payment:
        sliding-window-size: 10           # 최근 10개 요청 기준
        failure-rate-threshold: 50        # 50% 이상 실패 시 OPEN
        wait-duration-in-open-state: 10s  # 10초 후 HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 3
        minimum-number-of-calls: 5        # 최소 5개 요청 후 판단
```

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PgClient pgClient;

    @CircuitBreaker(name = "pg-payment", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "pg-payment")
    public CompletableFuture<PaymentResponse> pay(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> pgClient.pay(request));
    }

    private CompletableFuture<PaymentResponse> paymentFallback(
            PaymentRequest request, Exception e) {
        log.warn("PG 결제 실패, Fallback 처리: {}", e.getMessage());
        return CompletableFuture.completedFuture(
            PaymentResponse.pending(request.getOrderId())
        );
    }
}
```

Circuit Breaker가 OPEN 상태가 되면, PG에 요청하지 않고 즉시 Fallback을 반환합니다. 응답 시간이 3초 → 50ms로 줄었습니다.

## 타임아웃 문제: 결제는 됐는데 응답이 없다

Circuit Breaker를 적용했지만 새로운 문제가 생겼습니다. 타임아웃으로 실패 처리됐는데, 실제로는 PG에서 결제가 완료된 경우입니다. 이 상태에서 재시도하면 중복 결제 위험이 있고, 재시도하지 않으면 결제는 됐는데 주문이 생성되지 않습니다.

해결책: **Reconcile 스케줄러**

```java
@Component
@RequiredArgsConstructor
public class PaymentReconcileScheduler {

    private final OrderRepository orderRepository;
    private final PgClient pgClient;

    @Scheduled(fixedDelay = 60000) // 1분마다
    public void reconcile() {
        // PENDING 상태의 결제를 조회
        List<Order> pendingOrders = orderRepository.findByPaymentStatus(PaymentStatus.PENDING);

        for (Order order : pendingOrders) {
            try {
                // PG에 결제 상태 조회
                PaymentStatusResponse status = pgClient.getPaymentStatus(
                    order.getPaymentKey()
                );

                if (status.isPaid()) {
                    order.completePayment();
                } else if (status.isFailed()) {
                    order.failPayment();
                }
                // 아직 처리 중이면 다음 사이클에 다시 확인
            } catch (Exception e) {
                log.error("결제 조정 실패: orderId={}", order.getId(), e);
            }
        }
    }
}
```

## Retry와 Circuit Breaker 조합 시 함정

Resilience4j에서 Retry와 Circuit Breaker를 함께 쓸 때 **순서가 중요합니다**.

```java
// 잘못된 순서: CircuitBreaker가 Retry를 감싸야 한다
@Retry(name = "pg")
@CircuitBreaker(name = "pg")  // 이렇게 하면 Retry 실패까지 Circuit Breaker가 카운트
public PaymentResponse pay(PaymentRequest request) { ... }

// 올바른 순서
@CircuitBreaker(name = "pg")  // 바깥쪽: Circuit Breaker가 전체를 감싼다
@Retry(name = "pg")           // 안쪽: Retry는 Circuit Breaker 안에서 동작
public PaymentResponse pay(PaymentRequest request) { ... }
```

올바른 순서: `CircuitBreaker > Retry > TimeLimiter > Bulkhead`

## 개선 결과

| 항목 | Before | After |
|------|--------|-------|
| PG 장애 시 응답 시간 | 3초 (타임아웃) | 50ms (Fallback) |
| PG 장애 전파 | 전체 주문 API 영향 | 격리됨 |
| 중복 결제 처리 | 없음 | Reconcile 스케줄러 |

## 정리

장애는 전파됩니다. Circuit Breaker는 장애를 격리해 전파를 막습니다.

- **Timeout**: 무한 대기를 방지합니다
- **Retry**: 일시적 장애를 흡수합니다 (멱등성 주의)
- **Circuit Breaker**: 지속적 장애에서 빠른 Fallback을 제공합니다
- **Reconcile**: 타임아웃으로 인한 데이터 불일치를 해소합니다

각각이 해결하는 문제가 다릅니다. 조합하되, 순서를 이해하고 사용해야 합니다.
