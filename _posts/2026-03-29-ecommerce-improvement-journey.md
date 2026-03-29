---
layout: post
title: "이커머스 플랫폼을 만들며 고민했던 것들"
date: 2026-03-29
categories: [spring, architecture]
tags: [ecommerce, redis, kafka, concurrency, jwt, clean-architecture]
---

이커머스 플랫폼을 처음부터 만들었습니다. 기능을 구현할 때마다 "이게 맞나?" 싶은 순간이 반복됐고, 처음에 돌아가던 코드가 깨지는 상황을 만나면서 하나씩 고쳐나갔습니다.

## 재고 차감에서 시작된 동시성 고민

### 단순한 코드가 깨지는 순간

처음 주문 기능을 만들 때, 재고 차감은 별 생각 없이 구현했습니다. `product.decreaseStock(quantity)` 한 줄이면 되니까요.

문제는 동시에 여러 주문이 들어올 때 생겼습니다.

```
Thread A: read stock=10 → decrease → stock=8
Thread B: read stock=10 → decrease → stock=7
결과: 5개가 빠져야 하는데 stock=7 (3개 유실)
```

두 트랜잭션이 같은 row를 동시에 읽으면, 나중에 커밋하는 쪽이 앞선 변경을 덮어씁니다. 교과서에서 봤던 lost update를 실제로 만난 겁니다.

### 비관적 락 + 데드락 방지

비관적 락(`PESSIMISTIC_WRITE`)을 걸어서 해결했습니다. `SELECT ... FOR UPDATE`로 DB 레벨에서 동시 읽기를 막는 방식입니다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id IN :ids ORDER BY p.id ASC")
List<Product> findAllByIdsWithLock(@Param("ids") List<Long> ids);
```

여기서 `ORDER BY p.id ASC`가 중요합니다. 복수 상품을 동시에 락 잡을 때, 주문 A가 상품 [1, 2] 순서로, 주문 B가 상품 [2, 1] 순서로 락을 잡으면 데드락이 걸립니다. 모든 트랜잭션이 같은 순서로 락을 잡게 강제해서 데드락을 아예 막았습니다.

```java
List<Long> sortedProductIds = commands.stream()
        .map(StockDeductionCommand::productId)
        .sorted()
        .toList();
List<Product> products = productService.getProductsByIdsWithLock(sortedProductIds);
```

`Propagation.MANDATORY`도 걸었습니다. 이 메서드가 트랜잭션 없이 혼자 호출되면 재고는 차감됐는데 주문은 안 만들어지는 사고가 날 수 있어서, 반드시 기존 트랜잭션 안에서만 실행되게 했습니다.

---

## 쿠폰 선착순 발급 — 세 번 고친 이야기

프로젝트에서 가장 여러 번 고친 부분입니다. 코드 리뷰를 받을 때마다 빈틈이 발견됐고, 결국 3단계에 걸쳐 동시성 보호 수준을 올렸습니다.

### 1차: hasKey + set (실패)

처음 구현은 이랬습니다.

```java
Boolean hasKey = redisTemplate.hasKey(stockKey);
if (Boolean.FALSE.equals(hasKey)) {
    int remaining = policy.getTotalQuantity() - policy.getIssuedQuantity();
    redisTemplate.opsForValue().set(stockKey, String.valueOf(remaining));
}
Long remaining = redisTemplate.opsForValue().decrement(stockKey);
```

`hasKey` 체크와 `set` 사이에 다른 스레드가 끼어드는 게 문제였습니다. 잔여 수량이 3인데 두 스레드가 동시에 `hasKey=false`를 읽으면, 둘 다 `set(3)`을 실행합니다. 이미 차감된 수량이 리셋되는 겁니다.

### 2차: setIfAbsent로 원자성 확보

코드 리뷰에서 이 문제를 지적받고, `setIfAbsent`(= Redis SETNX)로 바꿨습니다.

```java
redisTemplate.opsForValue().setIfAbsent(stockKey, String.valueOf(remainingStock));
```

키가 없을 때만 값을 세팅하는 원자적 연산이라, 동시에 여러 스레드가 호출해도 최초 1개만 성공합니다. catch 범위도 `CoreException`에서 `Exception`으로 넓혀서, DB 예외가 나도 Redis 수량이 복구되게 했습니다.

### 3차: Redisson 분산 락까지

운영 안정성을 고민하면서 한 단계 더 올렸습니다. SETNX + DECR만으로는 Redis와 DB 사이의 정합성 문제가 남았습니다.

```java
public CouponInfo issueCoupon(Long couponPolicyId, String userId) {
    String lockKey = COUPON_LOCK_KEY_PREFIX + couponPolicyId;
    return distributedLockService.executeWithLock(
            lockKey, waitTimeSeconds, leaseTimeSeconds, TimeUnit.SECONDS,
            () -> doIssueCoupon(couponPolicyId, userId));
}

private CouponInfo doIssueCoupon(Long couponPolicyId, String userId) {
    return transactionTemplate.execute(status -> {
        CouponPolicy policy = couponService.getCouponPolicyWithLock(couponPolicyId);
        // ... setIfAbsent + DECR ...
    });
}
```

결국 3중 보호 구조가 됐습니다.

| 계층 | 역할 | 혼자서는 부족한 이유 |
|------|------|---------------------|
| Redisson 분산 락 | 인스턴스 간 직렬화 | Redis 단일 장애 시 보호 불가 |
| DB 비관적 락 | issuedQuantity lost update 방지 | 분산 환경에서 Redis 수량과 동기화 불가 |
| Redis SETNX + DECR | 빠른 수량 소진 판단 | DB 반영 실패 시 불일치 발생 가능 |

과한 걸까? 처음엔 저도 그렇게 생각했습니다. 그런데 각 계층 하나만 빼도 엣지 케이스가 남는 걸 확인하고 나니, 이게 적정선이라는 판단이 들었습니다.

### DistributedLockService

분산 락 서비스는 범용적으로 만들어서 쿠폰 외에도 쓸 수 있게 했습니다.

```java
public <T> T executeWithLock(String key, long waitTime, long leaseTime,
                              TimeUnit unit, Supplier<T> supplier) {
    RLock lock = redissonClient.getLock(key);
    boolean acquired = false;
    try {
        acquired = lock.tryLock(waitTime, leaseTime, unit);
        if (!acquired) {
            throw new CoreException(ErrorType.CONFLICT, "다른 요청이 처리 중입니다.");
        }
        return supplier.get();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new CoreException(ErrorType.INTERNAL_ERROR, "락 획득 중 인터럽트가 발생했습니다.");
    } finally {
        if (acquired && lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

`tryLock`으로 일정 시간만 대기하게 해서 무한 대기를 방지했고, `leaseTime`으로 프로세스가 죽어도 락이 자동 해제되게 했습니다. `waitTime`과 `leaseTime`은 `@ConfigurationProperties`로 빼서 운영 중에 조정할 수 있게 했습니다.

---

## Redis를 어디에, 어떻게 쓸지

### 장바구니 — Redis Hash

장바구니를 RDB에 넣을지 Redis에 넣을지 고민했습니다. 장바구니는 임시 데이터이고, 수량 변경이 잦고, 사용자별로 독립적입니다. RDB에 넣으면 수량 바꿀 때마다 UPDATE 쿼리가 나가고, 유효기간 만료 처리를 위한 배치도 따로 돌려야 합니다.

Redis Hash를 선택했습니다.

```java
// key: cart:{userId}, field: productId, value: quantity
public long addItem(String userId, Long productId, int quantity) {
    String key = cartKey(userId);
    Long result = redisTemplate.opsForHash().increment(key, productId.toString(), quantity);
    return result != null ? result : quantity;
}
```

`HINCRBY`가 원자적 연산이라 동시에 같은 상품을 담아도 수량이 정확합니다. TTL 7일을 줘서 방치된 장바구니는 알아서 정리됩니다. `checkout` 시에는 기존 `placeOrder` 플로우를 그대로 재사용해서 주문 로직을 건드리지 않았습니다.

### 인기 상품 — Sorted Set

인기 상품은 매번 `ORDER BY like_count DESC`로 DB를 때릴 수도 있지만, 트래픽이 몰리면 부하가 집중됩니다. Redis Sorted Set을 캐시로 넣었습니다.

```java
// 캐시 히트
Set<ZSetOperations.TypedTuple<String>> cached = redisTemplate.opsForZSet()
        .reverseRangeWithScores(POPULAR_KEY, 0, limit - 1);

// 캐시 미스: DB 조회 → Sorted Set 적재 → TTL 1시간
redisTemplate.opsForZSet().add(POPULAR_KEY, tuples);
redisTemplate.expire(POPULAR_KEY, 1, TimeUnit.HOURS);
```

TTL은 1시간으로 잡았습니다. 인기 상품 순위가 1시간 정도 지연되는 건 사용자 경험에 거의 영향이 없고, DB 부하 감소 효과가 훨씬 큽니다. `@Retry` + fallback을 걸어서 Redis가 죽어도 DB에서 직접 조회하게 했습니다.

### 좋아요 수 비정규화

`likeCount`를 매번 `SELECT COUNT(*)`로 계산하고 있었습니다. 상품 목록에서 각 상품마다 COUNT 쿼리가 나가는 N+1 문제였습니다.

Product 엔티티에 `likeCount` 필드를 추가하고, 좋아요 추가/제거 시 원자적으로 업데이트하는 방식으로 바꿨습니다.

```java
productRepository.incrementLikeCount(productId);
productService.evictProductCache(productId);
```

읽기 빈도가 쓰기보다 압도적으로 높은 경우, 비정규화의 이점이 큽니다. `@CacheEvict`로 캐시 무효화도 같이 걸어서 정합성을 챙겼습니다.

### Spring Cache 체계화

초반에는 `redisTemplate`으로 직접 get/set 하는 방식이었습니다. 캐시 키가 여기저기 흩어져 있고, TTL 설정도 코드 곳곳에 박혀 있어서 관리가 힘들었습니다.

Spring Cache 추상화를 도입하고 `RedisCacheManager`로 캐시별 TTL을 한 곳에서 관리하게 바꿨습니다.

```java
Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
cacheConfigurations.put("product", defaultConfig.entryTtl(Duration.ofMinutes(10)));
cacheConfigurations.put("brands", defaultConfig.entryTtl(Duration.ofHours(1)));
```

```java
@Cacheable(value = "brands", key = "#pageable.pageNumber + '-' + #pageable.pageSize + '-' + #pageable.sort")
public Page<Brand> getBrands(Pageable pageable) { ... }
```

비즈니스 로직과 캐시 로직이 분리되니 코드가 깔끔해졌고, TTL 변경도 한 곳만 고치면 됩니다. `@ConditionalOnProperty`로 local/test에서는 캐시를 끄게 해서 테스트 격리도 챙겼습니다.

### Rate Limiting — Lua 스크립트로 원자성 확보

Rate Limiting을 처음 만들 때 INCR과 EXPIRE를 따로 호출했습니다.

```java
Long count = redisTemplate.opsForValue().increment(key);
if (count != null && count == 1L) {
    redisTemplate.expire(key, windowSeconds, TimeUnit.SECONDS);
}
```

INCR 후 프로세스가 죽으면 EXPIRE가 안 걸려서 해당 키가 영구적으로 남습니다. 해당 사용자가 영구 차단되는 셈입니다. 쿠폰 때와 같은 원자성 문제였습니다.

Lua 스크립트로 두 연산을 묶었습니다.

```lua
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return count
```

사용자 식별 방식도 바꿨습니다. `X-Forwarded-For` 헤더는 클라이언트가 조작할 수 있어서, `SecurityContext`의 userId + `remoteAddr` 폴백으로 변경했습니다.

---

## 동기 처리의 한계, 이벤트로 풀기

### AFTER_COMMIT을 써야 하는 이유

이전에는 주문 후 알림 발송 같은 후속 작업을 동기로 처리했습니다. 주문 API 응답 시간에 알림 시간이 포함되고, 알림이 실패하면 주문 자체가 실패하는 문제가 있었습니다.

Kafka 이벤트로 비동기 분리했는데, 여기서 `AFTER_COMMIT`이 중요했습니다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@CircuitBreaker(name = "kafkaPublisher", fallbackMethod = "handleOrderPlacedFallback")
public void handle(OrderPlacedEvent event) {
    kafkaTemplate.send(TOPIC_ORDER_PLACED, String.valueOf(event.orderId()), event)
            .get(SEND_TIMEOUT_SECONDS, TimeUnit.SECONDS);
}
```

만약 트랜잭션 커밋 전에 Kafka에 이벤트를 보내면, Consumer가 이벤트를 받았는데 원본 트랜잭션이 롤백되는 상황이 생깁니다. 존재하지 않는 주문에 대한 알림이 나가는 거죠. `AFTER_COMMIT`으로 트랜잭션이 확정된 후에만 발행하게 해서 이걸 막았습니다.

### 보상 트랜잭션 — 주문 취소의 복잡함

주문 취소 기능을 처음 설계할 때, 생각보다 원복해야 하는 항목이 많아서 당황했습니다. 주문 상태 변경, 재고 복구, 포인트 환불, 쿠폰 사용 플래그 복원 — 4가지가 하나라도 빠지면 데이터 불일치가 생깁니다.

```java
@Transactional
public OrderInfo.CancelInfo cancelOrder(Long orderId, String userId) {
    Order order = orderService.getOrderByIdWithLock(orderId);
    order.cancel();
    restoreStock(order);
    pointService.refundPoint(userId, order.getActualPaymentAmount());
    if (order.getUserCouponId() != null) {
        couponService.restoreCoupon(order.getUserCouponId());
    }
    eventPublisher.publishEvent(OrderCancelledEvent.from(order));
    return OrderInfo.CancelInfo.from(order);
}
```

단일 `@Transactional` 안에서 모든 보상 작업을 수행합니다. 하나라도 실패하면 전체가 롤백됩니다. `getActualPaymentAmount()`로 쿠폰 할인 후 실제 결제 금액만 환불하는 것도 처음에는 놓쳤다가 테스트에서 잡았습니다.

### Circuit Breaker — 장애 전파 차단

Kafka나 Redis 같은 외부 의존성은 언제든 장애가 날 수 있습니다. Kafka가 죽으면 이벤트 발행이 타임아웃되고, 그게 주문 API 응답 지연으로 번지고, 전체 시스템이 느려집니다.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      kafkaPublisher:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
  retry:
    instances:
      redisRetry:
        max-attempts: 3
        wait-duration: 500ms
```

Kafka 이벤트 발행에는 Circuit Breaker를, Redis 인기 상품 조회에는 Retry + fallback을 걸었습니다. Redis가 죽어도 DB에서 직접 조회하면 되니까, 사용자에게 에러를 보여줄 이유가 없습니다.

### Graceful Shutdown

서버를 재시작할 때 진행 중인 주문이 강제 종료되면 데이터가 꼬일 수 있습니다.

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

종료 신호가 오면 새 요청 수신을 멈추고, 진행 중인 요청은 최대 30초까지 완료를 기다립니다. `RedisHealthIndicator`로 `/actuator/health`에 Redis 상태를 노출해서, 로드밸런서가 인스턴스를 트래픽에서 빼는 판단에 쓸 수 있게 했습니다.

---

## Offset 페이지네이션이 느려질 때

### 문제 발견

초반에는 Spring Data의 기본 `Pageable`을 썼습니다.

```sql
SELECT * FROM orders WHERE user_id = ? LIMIT 20 OFFSET 10000;
```

데이터가 쌓이니 offset이 커질수록 느려졌습니다. DB가 앞의 10,000개 row를 읽고 버리는 비효율이 원인이었습니다. 페이지를 넘기는 도중에 데이터가 삽입/삭제되면 같은 항목이 중복 표시되거나 누락되는 문제도 있었습니다.

### Cursor 기반으로 전환

```java
public static <T, E> CursorPageResponse<T> of(
        List<E> entities, int size,
        Function<E, Long> idExtractor, Function<E, T> mapper) {
    boolean hasNext = entities.size() > size;
    List<E> content = hasNext ? entities.subList(0, size) : entities;
    Long nextCursor = hasNext ? idExtractor.apply(content.get(content.size() - 1)) : null;
    return new CursorPageResponse<>(mapped, nextCursor, hasNext);
}
```

```sql
SELECT * FROM orders WHERE user_id = ? AND id < ? ORDER BY id DESC LIMIT 21;
```

`size + 1`개를 조회해서 별도 COUNT 쿼리 없이 `hasNext`를 판단합니다. WHERE 조건으로 이전 페이지의 마지막 ID 이후만 가져오니까, 데이터가 아무리 많아도 성능이 일정합니다.

---

## X-USER-ID의 허점

### 인증이 없는 것과 다름없었던 구조

초기 설계에서는 클라이언트가 보내는 `X-USER-ID` 헤더를 그대로 신뢰했습니다.

```java
@RequestHeader(value = "X-USER-ID", required = false) String userId
```

아무 검증 없이 클라이언트가 보내는 userId를 그대로 썼습니다. 누구든 다른 사용자의 ID를 헤더에 넣어서 요청할 수 있었고, 역할 구분이 없어서 모든 사용자가 관리자 API를 호출할 수 있었습니다. 솔직히 인증이 없는 것과 같았습니다.

### Spring Security + JWT로 전면 교체

```java
public String createAccessToken(String userId, Role role) {
    return Jwts.builder()
            .subject(userId)
            .claim("role", role.name())
            .issuedAt(now)
            .expiration(new Date(now.getTime() + accessExpiration))
            .signWith(secretKey)
            .compact();
}
```

Access Token 30분, Refresh Token 7일로 설정했습니다. Refresh Token은 Redis에 저장하고, 갱신 시 이전 토큰을 무효화하는 Token Rotation을 적용했습니다.

RBAC도 도입했습니다.

```java
.requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
.requestMatchers(HttpMethod.POST, "/api/v1/products").hasRole("ADMIN")
.anyRequest().authenticated()
```

상품 조회는 누구나, 상품 생성은 ADMIN만, 나머지는 로그인 필수. 단순하지만 이전에는 아예 없던 구분입니다.

### 컨트롤러 전환

8개 컨트롤러에 흩어져 있던 `@RequestHeader("X-USER-ID")`를 `SecurityContextHelper`로 전부 바꿨습니다.

```java
String userId = SecurityContextHelper.getCurrentUserId();      // 인증 필수
String userId = SecurityContextHelper.getCurrentUserIdOrNull(); // 선택적 (isLiked 등)
```

E2E 테스트도 10개 전부 JWT 방식으로 전환했습니다. `TestAuthHelper`를 만들어서 테스트에서도 실제 인증 플로우를 거치게 했습니다.

| 이전 | 이후 |
|------|------|
| X-USER-ID 헤더 (검증 없음) | JWT 서명 검증 |
| 역할 구분 없음 | USER/ADMIN RBAC |
| 만료 없음 | Access 30분 + Refresh 7일 |

---

## 도메인 모델에서 고민한 것들

### 주문 상태 머신

처음에는 `PENDING` → `PAID` 단방향만 있었습니다. 취소 기능을 추가하면서 `PAID` → `CANCELLED` 전이를 넣었는데, 상태 전이 규칙을 어디에 둘지 고민했습니다.

서비스 레이어에 두면 여러 서비스에서 호출할 때 규칙이 깨지기 쉽습니다. 엔티티 안에 넣었습니다.

```java
public void cancel() {
    if (this.status != OrderStatus.PAID) {
        throw new CoreException(ErrorType.BAD_REQUEST,
            "결제 완료 상태의 주문만 취소할 수 있습니다.");
    }
    this.status = OrderStatus.CANCELLED;
    this.cancelledAt = ZonedDateTime.now();
}
```

어떤 서비스에서 `cancel()`을 호출하든 같은 규칙이 적용됩니다.

### 포인트 이력 — Audit Trail

처음에는 Point 엔티티에 balance만 있었습니다. 충전/사용/환불 이력이 없어서 잔액이 어떻게 변했는지 추적할 방법이 없었습니다. "내 포인트가 왜 이만큼이죠?" 같은 질문에 답할 수 없는 구조였습니다.

PointHistory 엔티티를 추가했습니다.

```java
private Point updatePointAndLog(String userId, Long amount, PointHistoryType type, ...) {
    Point point = getPointWithLock(userId);
    operation.accept(point, amount);
    pointRepository.save(point);
    pointHistoryRepository.save(
            PointHistory.create(point, type, amount, point.getBalanceValue()));
    return point;
}
```

`balanceAfter` 필드가 핵심입니다. 변동 후 잔액을 스냅샷으로 남기기 때문에, 이력만 쭉 훑으면 현재 잔액이 맞는지 검증할 수 있습니다. 이력 저장이 포인트 변동과 같은 트랜잭션 안에서 일어나기 때문에, 포인트는 바뀌었는데 이력이 안 남는 상황은 없습니다.

### Facade 패턴 — 왜 레이어를 하나 더 뒀는가

Interface Layer  → Controller, DTO (HTTP 요청 처리)
Application Layer → Facade, Command, Info (애플리케이션 비즈니스)
Domain Layer     → Entity, Service, Repository(인터페이스) (핵심 도메인 로직)
Infrastructure   → RepositoryImpl, JpaRepository (데이터 및 외부 연동)
Infrastructure   → RepositoryImpl, JpaRepository
```

주문 하나 처리하려면 재고 + 포인트 + 쿠폰 + 이벤트를 다 건드려야 합니다. 이 조합 로직이 컨트롤러에 들어가면 컨트롤러가 뚱뚱해지고, 서비스끼리 직접 호출하면 순환 의존이 생겼습니다.

Facade가 트랜잭션 경계이자 오케스트레이션 지점이 되면서, 컨트롤러는 HTTP만, 도메인 서비스는 자기 도메인만 담당하게 됐습니다. 입력(Command)과 출력(Info)을 record로 분리한 것도 API 응답 형태를 바꿔도 도메인이 영향받지 않게 하기 위해서입니다.

---

## 되돌아보며

돌이켜 보면 하나의 패턴이 반복됐습니다. **단순하게 시작하고, 깨지는 지점을 발견하고, 그때 필요한 만큼만 복잡도를 올린다.**

- 동시성: 비관적 락 → SETNX → 분산 락. 한 번에 3중 보호를 설계한 게 아니라, 코드 리뷰에서 빈틈이 발견될 때마다 한 단계씩 올렸습니다.
- 캐시: `redisTemplate` 직접 호출 → Spring Cache 추상화. 캐시 키가 흩어져서 관리가 힘들어질 때 바꿨습니다.
- 인증: X-USER-ID → JWT + RBAC. 인증이 없는 것과 다름없다는 걸 인식하고 나서 전면 교체했습니다.

"이걸 왜 처음부터 안 했지?"라는 생각이 들 때도 있지만, 처음부터 모든 걸 고려했으면 아무것도 못 만들었을 겁니다. 일단 동작하는 코드를 만들고, 그 코드가 실패하는 시나리오를 직접 만나면서 고치는 게 결국 가장 빠른 길이었습니다.
