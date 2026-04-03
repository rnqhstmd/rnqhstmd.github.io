---
layout: post
title: "약속 시간 역산 앱을 만들며 고민했던 것들 — 백엔드 편"
date: 2026-03-29
categories: [spring, architecture]
tags: [redis, kafka, async, scheduler, cache, fallback, spring-batch]
---

친구들과의 약속에 늦을 때마다 "출발했어?" 카톡이 오고, 이동시간을 매번 검색하는 게 귀찮아서 앱을 만들기 시작했습니다. 약속 시간에서 이동시간을 역산해서 "지금 출발해"라고 알려주는 앱, 오디야(어디야)입니다.

3주 반 동안 41개 API와 6종의 스케줄러를 구현하면서, "이게 맞나?" 싶은 순간이 반복됐습니다.

## 외부 API를 매번 호출하면 무료 티어가 하루 만에 사라진다

### 문제를 만난 순간

이동시간 계산은 외부 API에 의존합니다. 카카오모빌리티(자동차)는 일 10,000건, ODsay(대중교통)는 일 1,000건이 무료 한도입니다. 사용자가 6명뿐인데도, 약속 하나에 참여자 전원의 이동시간을 계산하면 6건. 약속 수정 한 번에 또 6건. 당일 재계산까지 더하면 무료 티어가 하루 만에 소진될 수 있었습니다.

처음엔 "사용자가 6명인데 뭐"라고 생각했지만, 앱이 성장하면 어떻게 될지 생각하니 API 호출을 줄이는 구조가 필요했습니다.

### Redis Cache-Aside 패턴으로 해결

좌표를 소수점 4자리(~11m 정밀도)로 해싱해서 캐시 키를 만들었습니다. 같은 동네에서 출발하는 사람들의 이동시간은 하나의 캐시로 묶이는 셈입니다.

```java
// 캐시 조회 — replica에서 읽기
public Optional<Integer> get(double originLat, double originLng,
                             double destLat, double destLng,
                             TransportType transportType) {
    try {
        String key = buildKey(originLat, originLng, destLat, destLng, transportType);
        String value = replicaRedisTemplate.opsForValue().get(key);
        return Optional.ofNullable(value).map(Integer::parseInt);
    } catch (Exception e) {
        log.warn("이동시간 캐시 조회 실패 (graceful degradation)", e);
        return Optional.empty();  // Redis가 죽어도 캐시 미스로 처리
    }
}
```

이동수단마다 TTL을 다르게 잡은 게 핵심이었습니다.

| 이동수단 | TTL | 근거 |
|----------|-----|------|
| 자동차 | 30분 | 교통 상황이 빠르게 변함 |
| 대중교통 | 6시간 | 시간표 기반, 비교적 안정 |
| 도보 | 무기한 | 거리가 변하지 않음 |

자동차 TTL을 30분으로 잡은 건 당일 재계산 주기와 맞추기 위해서입니다. 어차피 약속 1시간 전부터 30분 간격으로 재계산하니까, 30분 이상 된 캐시는 쓸모가 없습니다.

장소 검색에도 같은 패턴을 적용했습니다. 검색어를 SHA-256으로 해싱해서 24시간 TTL로 캐시하고, `X-Cache: HIT/MISS` 헤더를 응답에 넣어서 실제로 캐시가 동작하는지 확인할 수 있게 했습니다.

---

## 재촉하기 버튼을 연타하면 알림이 두 번 간다

### Race Condition을 실제로 만나다

재촉하기(Nudge) 기능에는 동일 대상에게 5분 쿨다운이 있습니다. 처음에는 "쿨다운 있는지 확인 → 없으면 설정"을 2단계로 구현했습니다.

```java
// 처음 구현
if (!nudgeCooldownRepository.existsCooldown(appointmentId, senderId, receiverId)) {
    // 이 사이에 다른 요청이 끼어들 수 있다
    nudgeCooldownRepository.setCooldown(appointmentId, senderId, receiverId);
    sendNudgeNotification(...);
}
```

테스트에서는 잘 동작했습니다. 그런데 PR #10 전체 리팩토링에서 3개 병렬 에이전트로 코드를 분석했을 때, "두 요청이 동시에 existsCooldown을 호출하면 둘 다 false를 받는다"는 Race Condition이 발견됐습니다.

### Redis SETNX로 원자성 확보

이커머스 프로젝트에서 쿠폰 선착순 발급을 `SETNX`로 해결했던 경험이 떠올랐습니다. 같은 패턴이었습니다.

```java
public boolean trySetCooldown(Long appointmentId, Long senderId, Long receiverId) {
    String key = buildKey(appointmentId, senderId, receiverId);
    return Boolean.TRUE.equals(
        redisTemplate.opsForValue().setIfAbsent(key, "1", TTL)  // 5분 TTL
    );
}
```

`setIfAbsent`는 Redis `SETNX` 명령어입니다. 키가 없을 때만 값을 설정하는 원자적 연산이라, 동시에 100개 요청이 와도 정확히 1개만 `true`를 받습니다. 확인과 설정이 하나의 명령으로 끝나니 사이에 끼어들 틈이 없습니다.

---

## 이동시간 계산 때문에 약속 생성이 3초 걸린다

### 동기 처리의 함정

약속을 만들면 참여자 전원의 이동시간을 계산해야 합니다. 외부 API 호출은 건당 200~500ms. 참여자가 6명이면 최대 3초입니다. "약속 만들기" 버튼을 누르고 3초를 기다리는 건 사용자 경험으로 용납이 안 됐습니다.

### 호스트만 동기, 나머지는 비동기

약속을 만든 사람(호스트)의 이동시간은 응답에 바로 보여줘야 하니까 동기로 계산하고, 나머지 참여자는 트랜잭션 커밋 후 비동기로 처리하도록 분리했습니다.

```java
// AppointmentFacade.java
public AppointmentResponse create(CreateAppointmentRequest request, Long userId) {
    Appointment appointment = appointmentService.create(request);

    // 호스트만 동기 계산
    AppointmentParticipant host = appointment.findParticipant(userId);
    travelTimeService.calculateAndSave(host);

    // 나머지 참여자는 비동기 이벤트 발행
    List<Long> asyncIds = appointment.getParticipants().stream()
        .filter(p -> !p.getUserId().equals(userId))
        .map(AppointmentParticipant::getId)
        .toList();
    if (!asyncIds.isEmpty()) {
        applicationEventPublisher.publishEvent(new TravelTimeCalculateEvent(asyncIds));
    }

    return AppointmentResponse.from(appointment);
}
```

```java
// TravelTimeAsyncService.java
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async("travelTimeExecutor")
public void handleTravelTimeCalculation(TravelTimeCalculateEvent event) {
    for (Long participantId : event.participantIds()) {
        try {
            AppointmentParticipant participant =
                participantRepository.findById(participantId).orElse(null);
            if (participant == null) continue;
            travelTimeService.calculateAndSave(participant);
        } catch (Exception e) {
            log.warn("비동기 이동시간 계산 실패: participantId={}", participantId, e);
        }
    }
}
```

`AFTER_COMMIT`이 핵심입니다. 이전 이커머스 프로젝트에서도 동일한 패턴을 사용했는데, 트랜잭션 커밋 전에 이벤트를 발행하면 Consumer가 이벤트를 받았는데 원본 트랜잭션이 롤백되는 상황이 생깁니다. 약속이 롤백됐는데 이동시간이 계산되는 불일치를 원천 방지한 겁니다.

응답 시간이 참여자 수 × 500ms에서 호스트 1명분(~500ms)으로 줄었고, 한 참여자의 계산 실패가 다른 참여자에 영향을 주지 않게 됐습니다.

---

## 카카오모빌리티가 점검 중이면 약속을 못 만든다

### 외부 API 장애 = 서비스 장애

이동시간 계산이 약속 생성 경로에 있었기 때문에, 카카오모빌리티가 응답하지 않으면 약속 자체를 만들 수 없었습니다. "우리 서비스는 멀쩡한데 남의 서비스 때문에 멈추는" 상황이었습니다.

### Haversine 직선거리로 fallback

외부 API가 실패하면 지구 표면의 두 점 사이 직선거리(Haversine 공식)에 이동수단별 보정계수를 곱해서 추정값을 계산합니다.

```java
private int calculateHaversineFallback(double lat1, double lng1,
                                       double lat2, double lng2,
                                       TransportType transportType) {
    double distanceMeters = haversineDistance(lat1, lng1, lat2, lng2);
    return switch (transportType) {
        case CAR_PARKING, CAR_PICKUP -> {
            double km = distanceMeters / 1000.0;
            yield (int) Math.ceil((km * 1.4) / 40.0 * 60);  // 우회 1.4배, 시속 40km
        }
        case TRANSIT -> {
            double km = distanceMeters / 1000.0;
            yield (int) Math.ceil((km * 1.5) / 30.0 * 60);  // 환승 1.5배, 시속 30km
        }
        case WALKING -> (int) Math.ceil(distanceMeters / 80.0);  // 분속 80m
    };
}
```

여기서 중요한 설계 결정이 하나 있었습니다. fallback 결과는 캐시하지 않습니다.

```java
// TravelTime 엔티티
@Column(nullable = false)
private boolean isFallback = false;
```

fallback으로 저장된 이동시간은 `isFallback = true`로 표시됩니다. 캐시에 넣지 않았기 때문에, 다음에 같은 구간을 계산할 때 캐시 미스가 발생하고 외부 API를 다시 시도합니다. API가 복구되면 자연스럽게 정확한 값으로 교체됩니다.

전체 흐름을 정리하면 이렇습니다.

```
50m 이내? → 0분 반환
    ↓ No
Redis 캐시 히트? → 캐시값 반환
    ↓ Miss
외부 API 호출 → 성공 → 캐시 저장 + 반환
             → 실패 → Haversine 추정값 반환 (캐시 안 함)
```

---

## 출발 알림이 3시간 전 기준으로 간다

### 아침에 계산한 이동시간으로 퇴근 시간에 알림을 보내면

이동시간은 약속 생성 시 1회 계산됩니다. 아침 10시에 "강남까지 30분"으로 계산됐는데, 실제 약속이 오후 6시면 퇴근 시간 교통 체증 때문에 90분이 걸릴 수 있습니다. 출발 알림은 여전히 30분 기준이니까, 사용자는 확실히 지각합니다.

### 임박도에 따른 동적 재계산

```java
@Scheduled(fixedDelay = 60_000, initialDelay = 30_000)
public void execute() {
    String sql = """
        SELECT tt.id, ...
        FROM travel_times tt
        JOIN appointments a ON ...
        WHERE a.date_time > NOW()
          AND a.date_time <= DATE_ADD(NOW(), INTERVAL 60 MINUTE)
          AND tt.transport_type IN ('CAR_PARKING', 'CAR_PICKUP')
          AND (
            -- 60~30분 전: 30분 간격 재계산
            (a.date_time > DATE_ADD(NOW(), INTERVAL 30 MINUTE)
             AND tt.calculated_at < DATE_SUB(NOW(), INTERVAL 30 MINUTE))
            OR
            -- 30~0분 전: 10분 간격 재계산
            (a.date_time <= DATE_ADD(NOW(), INTERVAL 30 MINUTE)
             AND tt.calculated_at < DATE_SUB(NOW(), INTERVAL 10 MINUTE))
          )
        """;

    for (Map<String, Object> row : jdbcTemplate.queryForList(sql)) {
        try {
            int oldDuration = ((Number) row.get("duration_minutes")).intValue();
            int newDuration = batchKakaoMobilityClient.getDuration(...);

            // 출발 알림 시각 재계산
            LocalDateTime newAlert = appointmentTime
                .minusMinutes(newDuration)
                .minusMinutes(parkingBuffer)
                .minusMinutes(extraMinutes);

            jdbcTemplate.update(UPDATE_SQL, newDuration, newAlert, LocalDateTime.now(), rowId);

            // 5분 이상 변동 시 알림
            if (Math.abs(newDuration - oldDuration) >= 5) {
                notificationProcessor.processTravelTimeChanged(row, oldDuration, newDuration);
            }
        } catch (Exception e) {
            log.warn("이동시간 재계산 실패: id={}", row.get("id"), e);
        }
    }
}
```

대중교통과 도보는 재계산하지 않습니다. 대중교통은 시간표 기반이라 거의 변하지 않고, 도보는 완전히 고정입니다. 자동차만 실시간 교통 상황에 따라 크게 달라지니까요.

여기서 모듈 구조에 대한 고민이 있었습니다. 재계산 스케줄러는 `odiya-batch` 모듈에 있는데, 카카오모빌리티 API 클라이언트는 `odiya-api` 모듈에 있었습니다. batch에서 api 모듈을 의존하면 모듈 간 결합이 생깁니다.

`BatchKakaoMobilityClient`를 batch 모듈에 독립적으로 구현했습니다. 코드 중복이 조금 생기지만, 모듈 간 의존성보다 낫다고 판단했습니다.

---

## @Transactional이 안 먹는다

### self-invocation 문제

출발 알림 스케줄러를 구현하고 테스트했을 때, 알림이 발송되는데 DB 상태가 갱신되지 않아서 같은 알림이 계속 중복 발송되는 현상이 발생했습니다.

원인은 Spring의 self-invocation 문제였습니다. 같은 클래스 내에서 `@Transactional` 메서드를 호출하면, Spring AOP 프록시를 거치지 않아서 트랜잭션이 적용되지 않습니다.

```java
// 문제 코드: 같은 클래스 내 호출
@Scheduled(fixedDelay = 60_000)
public void execute() {
    List<Row> targets = findTargets();
    for (Row row : targets) {
        processReminder(row);  // this.processReminder() — 프록시 우회
    }
}

@Transactional  // 적용 안 됨
public void processReminder(Row row) { ... }
```

### Bean 분리로 해결

트랜잭션 처리가 필요한 로직을 `NotificationProcessor`라는 별도 Bean으로 분리했습니다.

```java
// 스케줄러 — 스케줄링만 담당
@Component
@RequiredArgsConstructor
public class DepartureReminderScheduler {
    private final NotificationProcessor notificationProcessor;

    @Scheduled(fixedDelay = 60_000)
    public void execute() {
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(QUERY);
        for (Map<String, Object> row : rows) {
            try {
                notificationProcessor.processDepartureReminder(row);
            } catch (Exception e) {
                log.error("출발 알림 실패: {}", row, e);
            }
        }
    }
}

// 프로세서 — 트랜잭션 처리 전담
@Component
public class NotificationProcessor {

    @Transactional  // 별도 Bean이니까 프록시가 정상 동작
    public void processDepartureReminder(Map<String, Object> row) {
        if (isDuplicate(row, "DEPARTURE_REMINDER")) return;  // 중복 방지
        Long notificationId = insertNotification(row);
        eventPublisher.publish(buildEvent(row, notificationId));
    }
}
```

중복 발송 방지도 함께 강화했습니다. 처음에는 `SENT` 상태만 체크했는데, `PENDING`(발송 요청됨, 아직 미발송) 상태에서도 중복이 생길 수 있어서 `SENT + PENDING` 모두 확인하도록 바꿨습니다.

---

## 3개 모듈에 같은 코드가 복붙되어 있다

### 알림 이벤트 클래스가 세 군데에

PR #8에서 푸시 알림 시스템을 처음 만들 때, `NotificationEvent` 클래스가 `odiya-api`, `odiya-streamer`, `odiya-batch` 세 모듈에 각각 복사되어 있었습니다. Kafka 토픽 이름도 `"notification.send"` 문자열이 여기저기 하드코딩되어 있었습니다.

한 모듈에서 이벤트 필드를 추가하면 다른 두 모듈도 동기화해야 하는 번거로운 상황이었습니다.

### 공유 모듈로 통합

PR #10 전체 리팩토링에서 이벤트 관련 클래스를 `modules/kafka` 공유 모듈로 추출했습니다.

```java
// modules/kafka — 모든 모듈이 공유
public record NotificationEvent(
    String eventId, Long notificationId, Long receiverId,
    List<String> tokens, String type, String title, String body,
    Long referenceId, String referenceType
) {}

public final class NotificationTopics {
    public static final String SEND = "notification.send";
    public static final String RESULT = "notification.result";
    public static final String DLQ = "notification.send.dlq";
    private NotificationTopics() {}
}
```

같은 리팩토링에서 함께 잡은 것들이 있습니다.

| 오류 | 수정 |
|------|------|
| kafka.yml의 `value-serializer` (consumer인데 serializer 키 사용) | `value-deserializer`로 수정 |
| `confg` 패키지 오타 | `config`으로 rename |
| `application.name` 오타 | 수정 |
| AppointmentFacade가 Repository 직접 참조 | Service 위임으로 전환 |
| Facade @Transactional 누락 3건 | 일괄 추가 |

31개 파일을 수정했는데 코드는 38줄 순감소(-134 +96)했습니다. 코드를 추가한 게 아니라 정리한 리팩토링이었습니다.

---

## Kafka 파이프라인 — 알림이 실패해도 유실되면 안 된다

### FCM 발송 실패 시 재시도와 DLQ

푸시 알림 발송은 Kafka 파이프라인으로 처리합니다. API 서버가 이벤트를 발행하면, Streamer 모듈의 Consumer가 Firebase Admin SDK로 FCM을 발송합니다.

FCM 발송이 실패할 수 있습니다. 네트워크 일시 장애, Firebase 서버 과부하 등. 재시도 로직을 구현했습니다.

```java
// NotificationKafkaConsumer.java — 최대 3회 재시도 + DLQ
private static final int MAX_RETRY = 3;
private static final long[] BACKOFF_MS = {1000, 2000, 4000};

@KafkaListener(topics = NotificationTopics.SEND)
public void consume(NotificationEvent event, Acknowledgment ack) {
    boolean success = false;
    List<String> invalidTokens = new ArrayList<>();

    for (int attempt = 0; attempt < MAX_RETRY && !success; attempt++) {
        try {
            if (attempt > 0) Thread.sleep(BACKOFF_MS[attempt]);
            invalidTokens = firebaseMessagingService.send(event);
            success = true;
        } catch (Exception e) {
            log.warn("FCM 발송 실패 (attempt {}): {}", attempt + 1, e.getMessage());
        }
    }

    if (!success) {
        // DLQ로 이동 — 나중에 수동으로 확인/재처리
        kafkaTemplate.send(NotificationTopics.DLQ, event.eventId(), event);
    }

    // 결과 이벤트 발행 — 알림 상태 SENT/FAILED 갱신
    publishResult(event, success, invalidTokens);
    ack.acknowledge();
}
```

exponential backoff(1초 → 2초 → 4초)로 일시적 장애를 넘기고, 3회 모두 실패하면 DLQ(Dead Letter Queue)로 보내서 유실을 방지합니다. 발송 결과는 별도 Kafka 토픽으로 발행하여 알림 상태를 `SENT`/`FAILED`로 갱신하고, 무효한 FCM 토큰은 비활성화합니다.

---

## 매 PR마다 보안 감사를 돌린 이유

처음에는 기능 구현에만 집중했습니다. 그런데 PR #2에서 LIKE 와일드카드 인젝션이, PR #5에서 `@Valid` 누락이, PR #6에서 participantIds 무제한 DoS 가능성이 발견됐습니다. 다 코드 리뷰에서 잡은 겁니다.

이후로 매 PR에 "Audit Summary"를 달기 시작했습니다. CRITICAL/HIGH/MEDIUM/LOW 심각도를 매기고, 해결 여부를 명시합니다.

| PR | 발견 | 해결 |
|----|------|------|
| #2 | LIKE 인젝션 + IDOR | 이스케이프 + 소유권 검증 |
| #6 | participantIds 무제한 DoS | `@Size(max=30)` |
| #8 | FCM 초기화 실패 시 기동 계속됨 | 초기화 실패 → 기동 중단 |
| #10 | Nudge Race Condition | SETNX 원자적 연산 |
| #10 | 입력 검증 부재 7건 | @NotBlank, @Size, @Pattern |

범위 밖 이슈는 "이월"로 기록하고 후속 PR에서 해결합니다. PR #5에서 "Facade @Transactional 미적용"을 Warning으로 남겼고, PR #10에서 일괄 해결했습니다.

모든 문제를 당장 해결할 수는 없지만, 기록해두면 잊어버리지 않습니다. PR #5에서 남긴 Warning이 PR #10에서 해결된 것처럼요.

---

## 되돌아보며

3주 반을 돌아보면, 이커머스 프로젝트에서 배웠던 패턴들이 반복해서 등장했습니다.

- **SETNX**: 쿠폰 선착순에서 배운 원자적 연산이 Nudge 쿨다운에 그대로 적용됐습니다.
- **AFTER_COMMIT**: 주문 이벤트에서 배운 트랜잭션 커밋 후 발행을 이동시간 비동기 계산에 썼습니다.
- **Cache-Aside + TTL**: 이커머스의 Redis 캐시 패턴을 이동시간과 장소 검색에 가져왔습니다.
- **Bean 분리**: 한 번 겪어본 self-invocation 문제라 스케줄러에서 바로 알아채고 처리할 수 있었습니다.

결국 가장 빠른 학습은 "같은 문제를 다른 도메인에서 다시 만나는 것"이었습니다. 처음 만났을 때는 원인 파악에 시간이 걸리지만, 두 번째부터는 패턴이 보여서 빠르게 해결할 수 있습니다.

처음부터 모든 걸 고려하면 아무것도 못 만듭니다. 일단 동작하게 만들고, 깨지는 지점을 발견하면 그때 필요한 만큼만 복잡도를 올리는 게 결국 가장 빠른 길이었습니다.
