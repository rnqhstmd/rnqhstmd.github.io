---
layout: post
title: "실시간 랭킹 시스템 구현기 (feat. 콜드 스타트 해결 과정)"
date: 2025-12-26
categories: [redis, spring]
---

## 인기 상품 랭킹 시스템 요구사항

- 실시간으로 인기 상품 TOP 10을 보여줍니다
- 인기도 기준: 조회수, 좋아요, 주문 수를 가중치를 두어 합산
- 일간 집계 기준 (오늘 발생한 이벤트만 반영)

## Redis Sorted Set (ZSET) 선택

랭킹 시스템에 Redis ZSET을 선택한 이유입니다.

- `ZADD`: O(log N)으로 점수 업데이트
- `ZREVRANGE`: O(log N + M)으로 상위 N개 조회
- 아토믹 연산: `ZINCRBY`로 동시성 문제 없이 점수 증가

```java
@Service
@RequiredArgsConstructor
public class ProductRankingService {

    private final RedisTemplate<String, String> redisTemplate;

    // 가중치
    private static final double VIEW_WEIGHT = 0.1;
    private static final double LIKE_WEIGHT = 0.2;
    private static final double ORDER_WEIGHT = 0.7;

    private String getDailyKey() {
        return "ranking:daily:" + LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);
        // 예: "ranking:daily:20251226"
    }

    public void recordView(Long productId) {
        redisTemplate.opsForZSet().incrementScore(
            getDailyKey(),
            String.valueOf(productId),
            VIEW_WEIGHT
        );
    }

    public void recordLike(Long productId) {
        redisTemplate.opsForZSet().incrementScore(
            getDailyKey(),
            String.valueOf(productId),
            LIKE_WEIGHT
        );
    }

    public void recordOrder(Long productId) {
        redisTemplate.opsForZSet().incrementScore(
            getDailyKey(),
            String.valueOf(productId),
            ORDER_WEIGHT
        );
    }

    public List<Long> getTopProducts(int limit) {
        Set<String> topIds = redisTemplate.opsForZSet().reverseRange(
            getDailyKey(), 0, limit - 1
        );

        return topIds.stream()
            .map(Long::parseLong)
            .collect(Collectors.toList());
    }
}
```

## 문제: 콜드 스타트

자정이 지나면 날짜가 바뀌고 새로운 키가 생성됩니다. 새벽 00:00에는 오늘 데이터가 전혀 없습니다. 랭킹이 비어있는 상태입니다.

사용자가 자정 직후에 랭킹을 조회하면 아무것도 나오지 않습니다. 처음엔 이걸 간과했습니다.

### 잘못된 접근: 단순 복사

처음에는 자정에 전날 키를 오늘 키로 복사하는 방식을 생각했습니다.

```java
// 잘못된 접근
@Scheduled(cron = "0 0 0 * * *")
public void copyYesterdayRanking() {
    String yesterday = "ranking:daily:" + LocalDate.now().minusDays(1)...;
    String today = getDailyKey();
    redisTemplate.rename(yesterday, today); // 이렇게 하면 안 된다
}
```

문제가 있습니다. 오늘 활동이 쌓이면서 전날 데이터가 섞입니다. 오늘 주문 0건인 상품이 어제 주문 많다는 이유로 상위에 남습니다.

### 올바른 접근: Score Carry-Over (점수 이월)

핵심 아이디어는 전날 점수의 일부(10%)를 오늘로 이월하는 겁니다. 당일 활동이 쌓이면 이월 점수의 영향이 자연스럽게 줄어듭니다.

```java
@Scheduled(cron = "0 50 23 * * *") // 매일 23:50
public void carryOverScore() {
    String todayKey = getDailyKey();
    String tomorrowKey = "ranking:daily:" + LocalDate.now().plusDays(1)
        .format(DateTimeFormatter.BASIC_ISO_DATE);

    // 오늘 TOP 50의 점수를 10%만 내일로 이월
    Set<TypedTuple<String>> topProducts = redisTemplate.opsForZSet()
        .reverseRangeWithScores(todayKey, 0, 49);

    if (topProducts != null) {
        for (TypedTuple<String> tuple : topProducts) {
            // 이미 오늘 활동이 있는 상품은 이월 점수로 덮어쓰지 않습니다 (멱등성 보장)
            Double existingScore = redisTemplate.opsForZSet().score(tomorrowKey, tuple.getValue());
            if (existingScore != null) {
                continue;
            }
            double carryOverScore = tuple.getScore() * 0.1;
            redisTemplate.opsForZSet().add(
                tomorrowKey,
                tuple.getValue(),
                carryOverScore
            );
        }
    }

    // TTL 설정: 48시간 (여유롭게)
    redisTemplate.expire(tomorrowKey, Duration.ofHours(48));
}
```

23:50에 실행해서 자정 전에 내일 키를 미리 준비합니다. 자정이 되면 이월된 점수로 시작하고, 실제 활동이 쌓이면서 이월 점수의 비중이 줄어듭니다.

## 멱등성 보장

스케줄러가 여러 번 실행되더라도 결과가 동일해야 합니다. `ZADD`의 `NX` 옵션을 사용하면 이미 존재하는 멤버는 업데이트하지 않습니다.

```java
// 이미 이월이 완료된 상품은 덮어쓰지 않는다
redisTemplate.opsForZSet().addIfAbsent(
    tomorrowKey,
    tuple.getValue(),
    carryOverScore
);
```

이 방식은 "아직 오늘 활동이 없는 신규 상품만 이월 점수로 초기화"하는 의미입니다. 오늘 이미 활동이 있었다면 이월 점수는 무시됩니다.

## TTL 전략

키 관리를 자동화합니다.

```java
public void recordView(Long productId) {
    String key = getDailyKey();
    redisTemplate.opsForZSet().incrementScore(key, String.valueOf(productId), VIEW_WEIGHT);

    // 키 첫 생성 시 TTL 설정 (48시간)
    if (redisTemplate.getExpire(key) == -1) {
        redisTemplate.expire(key, Duration.ofHours(48));
    }
}
```

TTL을 48시간으로 설정하면 어제 키가 오늘 자정 이후에도 24시간 더 유지됩니다. 어제 랭킹 조회가 필요할 때 사용할 수 있습니다.

## 전체 흐름 정리

```
23:50 스케줄러 실행
  └── 오늘 TOP 50 조회
  └── 10% 점수로 내일 키에 ZADD (addIfAbsent)
  └── 내일 키 TTL 48시간 설정

00:00 날짜 변경
  └── getDailyKey()가 새로운 날짜 키 반환
  └── 이미 이월된 점수로 초기화된 상태

이후 활동 발생
  └── 조회: +0.1점
  └── 좋아요: +0.2점
  └── 주문: +0.7점
  └── 당일 활동이 쌓이면서 이월 점수의 영향 감소
```

## 정리

Redis ZSET은 랭킹 구현에 최적화된 자료구조입니다. `ZINCRBY`의 아토믹 연산 덕분에 동시성 처리가 자연스럽습니다.

콜드 스타트 문제는 실시간 랭킹 시스템의 숨겨진 함정입니다. 단순히 "자정에 초기화"가 아니라, 연속성 있는 랭킹을 위해 이월 전략이 필요합니다.

Score Carry-Over 방식의 장점입니다.
- 자정에 랭킹이 비지 않습니다
- 당일 활동이 없는 상품은 자연스럽게 순위가 낮아집니다
- 전날 인기 상품이 당일 초기에도 어느 정도 반영됩니다
