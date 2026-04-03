---
layout: post
title: "Kafka, 제대로 이해하고 쓰자"
date: 2026-04-03
categories: [kafka, architecture]
tags: [kafka, message-queue, event-driven, spring-boot]
---

이커머스 프로젝트에서 주문 후 알림을 비동기로 분리할 때 Kafka를 처음 썼습니다. `kafkaTemplate.send()` 한 줄이면 메시지가 날아갔고, `@TransactionalEventListener`로 트랜잭션 커밋 후에만 발행하게 했습니다. 동작은 했는데, 누가 "Kafka에서 메시지 순서가 보장돼?" 하고 물으면 선뜻 대답을 못 하겠더라고요.

그래서 정리했습니다. 내부 동작 원리부터, 전달 보장, Consumer 관리, 그리고 언제 Kafka를 쓰고 안 쓰는지까지.

## Kafka는 메시지 큐가 아니다

흔히 Kafka를 "메시지 큐"라고 부르는데, 정확하지 않습니다. Kafka는 **분산 이벤트 스트리밍 플랫폼**입니다. 전통적인 메시지 큐(RabbitMQ 같은)와 결정적으로 다른 점이 하나 있습니다.

**메시지를 소비해도 삭제하지 않습니다.**

RabbitMQ는 Consumer가 메시지를 가져가면 큐에서 사라집니다. Kafka는 보존 정책(retention policy)에 따라 메시지를 유지합니다. 일주일이든 한 달이든, 설정한 기간 동안 남아 있습니다. 덕분에 같은 데이터를 여러 Consumer가 독립적으로 읽을 수 있고, 장애 복구 시 과거 시점으로 되감아서 다시 처리할 수 있습니다.

Kafka 공식 문서는 이벤트 스트리밍을 "인체의 중추신경계에 해당하는 디지털 등가물"이라고 표현합니다. 좀 거창하지만, 데이터가 실시간으로 흘러다니는 파이프라인이라고 생각하면 됩니다.

---

## 핵심 구조

### Event (Record)

Kafka에서 데이터의 최소 단위입니다. "어떤 일이 일어났다"는 사실의 기록입니다.

```
key: "order-123"
value: { "status": "PAID", "amount": 35000 }
timestamp: 1712150400000
headers: { "source": "order-service" }
```

key는 선택이지만, 나중에 설명할 파티셔닝에서 중요합니다.

### Topic과 Partition

Topic은 이벤트를 분류하는 논리적 카테고리입니다. 파일시스템의 폴더 같은 겁니다. "order-events", "payment-events" 이런 식으로 이름 붙입니다.

그런데 Topic 하나가 하나의 로그 파일이면 병목이 생깁니다. 그래서 Topic을 여러 **Partition**으로 쪼갭니다.

```
Topic: order-events
├── Partition 0: [msg0, msg1, msg4, msg7, ...]
├── Partition 1: [msg2, msg3, msg6, msg9, ...]
└── Partition 2: [msg5, msg8, msg10, ...]
```

Partition은 Kafka의 병렬 처리 단위입니다. 파티션이 3개면 최대 3개의 Consumer가 동시에 읽을 수 있습니다. 파티션 수 = 최대 병렬 Consumer 수라고 생각하면 됩니다.

**순서 보장은 파티션 단위입니다.** 같은 파티션 안에서는 메시지 순서가 보장되지만, 파티션 간에는 보장되지 않습니다. 주문 이벤트의 순서가 중요하다면 주문 ID를 key로 넣어야 합니다. 같은 key는 같은 파티션으로 갑니다.

파티션 수는 운영 중에 늘릴 수 있지만 **줄일 수는 없습니다.** 처음에 너무 적게 잡으면 나중에 Consumer를 늘려도 병렬성이 안 올라갑니다. 시간당 10만 건 정도 규모라면 10개가 일반적입니다.

### Broker

Kafka 서버 하나를 Broker라고 부릅니다. 보통 여러 대를 묶어서 클러스터를 구성합니다. 각 Broker는 고유한 ID를 가지고, 파티션의 복제본을 나눠서 저장합니다.

### Replication — 데이터를 잃지 않으려면

Broker가 죽으면 그 안의 데이터도 날아갑니다. 이걸 막기 위해 파티션을 여러 Broker에 복제합니다.

```
Partition 0
├── Broker 1: Leader    ← 모든 읽기/쓰기 처리
├── Broker 2: Follower  ← Leader 데이터 복제
└── Broker 3: Follower  ← Leader 데이터 복제
```

- **Replication Factor**: 복제본 수. 보통 3으로 설정합니다.
- **Leader**: 해당 파티션의 모든 읽기/쓰기를 담당합니다.
- **Follower**: Leader의 데이터를 복제합니다. Leader가 죽으면 Follower 중 하나가 새 Leader가 됩니다.
- **ISR (In-Sync Replicas)**: Leader에 충분히 따라잡은 복제본 집합입니다. `replica.lag.time.max.ms` 안에 복제를 완료한 Follower만 ISR에 포함됩니다.

Leader 장애 시 ISR 안에 있는 Follower만 새 Leader 후보가 됩니다. ISR 밖의 Follower는 데이터가 뒤처져 있을 수 있어서요.

### Consumer Group

같은 `group.id`를 가진 Consumer들의 모음입니다. 핵심 규칙은 하나입니다.

> 하나의 파티션은 같은 Consumer Group 내에서 하나의 Consumer에만 할당된다.

```
Consumer Group A (3명)
├── Consumer 1 → Partition 0
├── Consumer 2 → Partition 1
└── Consumer 3 → Partition 2

Consumer Group B (2명)  ← 같은 Topic을 독립적으로 읽음
├── Consumer 4 → Partition 0, 1
└── Consumer 5 → Partition 2
```

Consumer Group A와 B는 서로 영향을 주지 않습니다. 같은 메시지를 각자의 속도로 읽습니다. 주문 서비스와 알림 서비스가 같은 "order-events" 토픽을 각자의 Group으로 읽는 식입니다.

### ZooKeeper에서 KRaft로

Kafka는 원래 메타데이터 관리를 ZooKeeper에 의존했습니다. 클러스터를 띄우려면 ZooKeeper부터 띄워야 했고, 운영 대상이 두 개인 셈이었습니다.

Kafka 3.3에서 KRaft(Kafka Raft) 모드가 도입됐고, **Kafka 4.0부터는 KRaft가 필수**입니다. ZooKeeper 지원이 완전히 제거됐습니다. Controller 노드가 Raft 합의 프로토콜로 메타데이터를 직접 관리합니다. 장애 조치가 밀리초 단위로 빨라졌고, 별도 ZooKeeper 클러스터를 관리할 필요가 없어졌습니다.

---

## 메시지가 유실되면 안 되는데

이커머스에서 주문 이벤트가 유실되면 큰일입니다. 재고는 차감됐는데 알림이 안 가거나, 결제는 됐는데 주문 확인이 안 되는 상황이 벌어집니다. Kafka는 메시지 전달 보장을 세 단계로 제공합니다.

### At-Most-Once (최대 한 번)

Producer가 메시지를 보내고 확인을 기다리지 않습니다. "fire and forget"입니다.

- 메시지가 유실될 수 있음
- 중복은 없음
- 지연시간이 가장 낮음

```properties
acks=0
```

로그 수집처럼 일부 유실이 괜찮은 경우에 씁니다. 주문 이벤트에는 절대 쓰면 안 됩니다.

### At-Least-Once (최소 한 번) — 기본값

Producer가 Broker의 확인(ack)을 기다립니다. 확인을 못 받으면 재전송합니다.

- 메시지 유실 없음
- 재전송으로 **중복이 생길 수 있음**

```properties
acks=all
retries=3
```

Kafka의 기본 동작입니다. 대부분의 경우 이걸로 충분하지만, Consumer 쪽에서 중복 처리를 감안한 멱등성 로직이 필요합니다. 예를 들어 같은 주문 ID로 알림이 두 번 가도 한 번만 보내지게 하는 식입니다.

### Exactly-Once (정확히 한 번)

메시지가 정확히 한 번만 전달되고 처리됩니다. 유실도 없고 중복도 없습니다. Kafka 0.11.0.0(2017년)부터 지원됩니다.

구현 원리가 좀 복잡합니다.

**1단계: Idempotent Producer**

```properties
enable.idempotence=true  # Kafka 3.0부터 기본값
```

Broker가 각 Producer에게 PID(Producer ID)를 부여하고, 메시지마다 sequence number를 추적합니다. 같은 PID + 같은 sequence number의 메시지가 오면 중복으로 판단해서 버립니다. TCP의 중복 패킷 제거와 비슷한데, 차이점은 Broker 재시작 후에도 유지된다는 겁니다. 시퀀스 번호가 복제 로그에 저장되니까요.

성능 영향은 거의 없습니다. 메시지 배치에 숫자 필드 하나 추가하는 수준입니다.

**2단계: Transactional API**

여러 파티션에 원자적으로 쓸 때 필요합니다.

```properties
transactional.id=order-service-tx-1
```

Producer가 트랜잭션을 열고 여러 토픽/파티션에 메시지를 보낸 뒤, 한 번에 커밋하거나 롤백합니다. 트랜잭션 코디네이터가 2단계 커밋 프로토콜로 관리합니다.

**3단계: Consumer 격리**

```properties
isolation.level=read_committed
```

Consumer가 커밋된 트랜잭션의 메시지만 읽게 합니다. 진행 중이거나 롤백된 트랜잭션의 메시지는 걸러집니다.

**Kafka Streams에서는 한 줄이면 됩니다:**

```properties
processing.guarantee=exactly_once_v2
```

입력 토픽 읽기 → 상태 저장소 업데이트 → 출력 토픽 쓰기를 원자적으로 처리합니다.

**성능 영향은?**

| 비교 기준 | Throughput 변화 |
|-----------|----------------|
| `acks=all` 대비 idempotent | 거의 없음 |
| `acks=all` 대비 transactional | 약 3% 감소 |
| `acks=1` 대비 transactional | 약 20% 감소 |
| Kafka Streams (100ms 커밋 간격) | 15-30% 감소 |
| Kafka Streams (30초 커밋 간격) | 거의 없음 |

Idempotent producer는 켜두지 않을 이유가 없습니다. Kafka 3.0부터 기본값이기도 하고요. Transactional API는 정합성이 정말 중요한 경우에만 쓰면 됩니다.

**한 가지 주의.** Exactly-once가 보장되는 범위는 Kafka 내부 연산(토픽 읽기 → 처리 → 토픽 쓰기)뿐입니다. 외부 DB에 쓰거나 HTTP API를 호출하는 건 보장 범위 밖입니다. 이건 Kafka만의 문제가 아니라 분산 시스템의 근본적인 한계입니다.

---

## Offset — 어디까지 읽었는지 기억하기

Consumer가 파티션에서 메시지를 읽으면, "나는 여기까지 읽었다"를 기록해야 합니다. 그래야 Consumer가 재시작돼도 이어서 읽을 수 있으니까요. 이게 offset commit입니다.

### 커밋 방식

**자동 커밋 (기본값):**

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

5초마다 자동으로 현재 위치를 커밋합니다. 편하지만, 메시지를 처리하기 전에 커밋이 될 수 있어서 장애 시 유실이 생길 수 있습니다. 반대로 커밋 후 처리 전에 장애가 나면 중복이 생깁니다.

**수동 커밋:**

```java
// 블로킹 — 커밋 완료까지 대기. 안전하지만 느림
consumer.commitSync();

// 논블로킹 — 커밋 실패해도 재시도 없음. 빠르지만 불안함
consumer.commitAsync();
```

비즈니스 로직에서 중복이나 유실이 치명적이면 수동 커밋을 써야 합니다. 자동 커밋은 "대충 at-least-once면 충분한" 경우에 쓰는 겁니다.

### auto.offset.reset

Consumer Group이 처음 시작하거나, 커밋된 offset이 이미 만료돼서 없을 때의 동작을 결정합니다.

- `earliest`: 파티션의 맨 처음부터 읽음. 과거 데이터를 모두 처리해야 할 때.
- `latest` (기본값): 지금부터 새로 들어오는 메시지만 읽음.
- `none`: 커밋된 offset이 없으면 예외를 던짐.

개발 중에는 `earliest`로 놓고 테스트하다가, 운영에서는 `latest`로 바꾸는 경우가 많습니다. 다만 "운영에서 새 Consumer Group을 배포했는데 기존 메시지를 다 처리해야 한다"면 `earliest`여야 합니다. 이건 상황에 따라 다릅니다.

### __consumer_offsets

Consumer가 커밋한 offset은 `__consumer_offsets`라는 Kafka 내부 토픽에 저장됩니다. 기본 50개 파티션으로 구성됩니다. 외부 저장소(ZooKeeper 등)에 의존하지 않고 Kafka 자체에서 관리하는 구조입니다.

### Consumer Lag

Consumer Lag = 파티션의 최신 offset - Consumer가 커밋한 offset.

Lag이 0이면 생산 속도를 따라잡고 있다는 뜻이고, 지속적으로 증가하면 Consumer 처리 속도가 못 따라가는 겁니다. 이때는 Consumer 수를 늘리거나(파티션 수 범위 내에서), 처리 로직을 최적화해야 합니다.

모니터링은 `kafka-consumer-groups.sh --describe` 명령이나, Burrow 같은 전용 도구를 씁니다.

---

## Rebalancing — 컨슈머가 바뀔 때 벌어지는 일

Consumer Group에 Consumer가 추가되거나 빠지면, 파티션을 다시 분배하는 **리밸런싱**이 발생합니다.

### 언제 발생하는가

- Consumer가 그룹에 합류하거나 떠날 때
- Consumer가 `session.timeout.ms` 안에 heartbeat를 보내지 못할 때
- Topic에 새 파티션이 추가될 때
- Group Coordinator 브로커가 변경될 때

### Eager vs Cooperative

**Eager (전통 방식):**

모든 Consumer가 자기 파티션을 전부 반납합니다. 그러고 나서 처음부터 다시 할당합니다. "Stop-the-world"입니다. 리밸런싱 동안 그룹 전체가 처리를 멈춥니다.

**Cooperative (Kafka 2.4+):**

이동이 필요한 파티션만 재할당합니다. 나머지 Consumer는 멈추지 않고 계속 처리합니다. 다운타임이 크게 줄어듭니다.

### 파티션 할당 전략

| 전략 | 특징 |
|------|------|
| RangeAssignor | 토픽별로 연속된 파티션 할당. 기본값이지만 불균형 생기기 쉬움 |
| RoundRobinAssignor | 모든 파티션을 순환 할당. 균등하지만 리밸런싱 시 파티션 이동이 많음 |
| StickyAssignor | 기존 할당을 최대한 유지. 이동 최소화 |
| **CooperativeStickyAssignor** | Sticky + Cooperative. 현재 권장 |

운영 환경에서는 `CooperativeStickyAssignor`를 쓰는 게 맞습니다. 불필요한 파티션 이동을 줄이고, 리밸런싱 동안에도 처리가 계속됩니다.

### 리밸런싱을 줄이는 설정

배포할 때마다 Consumer가 잠깐 죽었다 살아나면서 리밸런싱이 반복되는 문제가 있습니다. 이걸 줄이려면:

```properties
# 세션 타임아웃을 넉넉하게 — 일시적 지연으로 리밸런싱 발생 방지
session.timeout.ms=45000

# 폴링 간격을 실제 처리 시간보다 넉넉하게
max.poll.interval.ms=300000

# 동시 기동 시 반복 리밸런싱 방지
group.initial.rebalance.delay.ms=3000
```

**Static Group Membership**도 있습니다. 각 Consumer에 `group.instance.id`를 부여하면, 잠깐 끊겼다 재접속해도 같은 멤버로 인식합니다. 배포 시 불필요한 리밸런싱을 피할 수 있습니다.

```properties
group.instance.id=consumer-host-1
```

### ConsumerRebalanceListener

리밸런싱이 일어날 때 콜백을 받을 수 있습니다. `onPartitionsRevoked()`에서 아직 커밋하지 않은 offset을 커밋하는 게 핵심입니다. 안 그러면 파티션이 다른 Consumer에게 넘어간 뒤 같은 메시지를 중복 처리할 수 있습니다.

---

## 언제 Kafka를 쓰고, 언제 안 쓰는가

Kafka가 만능은 아닙니다. 다른 메시지 브로커와 비교하면 선택 기준이 명확해집니다.

| 기준 | Kafka | RabbitMQ | Redis Streams |
|------|-------|----------|---------------|
| 처리량 | 수백만 msg/sec | 수천 msg/sec | 수십만 msg/sec |
| 메시지 보존 | 설정에 따라 무기한 | 소비 후 삭제 | 제한적 |
| 이벤트 재생 | 가능 | 불가 | 제한적 |
| 라우팅 유연성 | Topic + key 기반 | Exchange 기반 정교한 라우팅 | 단순 |
| 운영 난이도 | 높음 | 중간 | 낮음 |

**Kafka를 쓸 때:**
- 이벤트를 여러 서비스가 독립적으로 소비해야 할 때 (다중 Consumer Group)
- 이벤트 재생이 필요할 때 (장애 복구, 데이터 재처리)
- 처리량이 초당 수만 건 이상일 때
- 로그 수집, 메트릭 파이프라인, CDC(Change Data Capture)

**RabbitMQ가 나을 때:**
- 복잡한 라우팅이 필요할 때 (우선순위 큐, 지연 큐, 데드 레터)
- 요청-응답(request-reply) 패턴이 필요할 때
- 처리량보다 메시지 라우팅 유연성이 중요할 때

**Redis Streams가 나을 때:**
- 이미 Redis를 쓰고 있고, 간단한 메시징이 필요할 때
- MVP 단계에서 빠르게 구현해야 할 때
- 메시지 장기 보존이 필요 없을 때

확신이 없으면 제일 단순한 걸로 시작해서 부족할 때 갈아타는 게 낫습니다. 이커머스 프로젝트에서 제가 Kafka를 선택한 건, 주문 이벤트를 알림 서비스와 정산 서비스가 각각 독립적으로 소비해야 했기 때문입니다. RabbitMQ였으면 같은 메시지를 두 큐에 복제하는 설정이 필요했을 겁니다.

---

## Spring Boot에서 Kafka 쓰기

### 설정

```yaml
spring:
  kafka:
    bootstrap-servers: "localhost:9092"
    producer:
      key-serializer: "org.apache.kafka.common.serialization.StringSerializer"
      value-serializer: "org.springframework.kafka.support.serializer.JsonSerializer"
      acks: "all"
      retries: 3
      properties:
        "[enable.idempotence]": true
    consumer:
      group-id: "order-service"
      key-deserializer: "org.apache.kafka.common.serialization.StringDeserializer"
      value-deserializer: "org.springframework.kafka.support.serializer.JsonDeserializer"
      auto-offset-reset: "earliest"
      properties:
        "[spring.json.trusted.packages]": "com.example.*"
    listener:
      concurrency: 3
```

`acks=all`로 모든 ISR이 확인해야 성공으로 처리하고, `enable.idempotence=true`로 중복 전송을 방지합니다. `listener.concurrency=3`은 Consumer 스레드를 3개 띄워서 파티션 3개를 병렬로 처리하겠다는 뜻입니다.

### Producer

```java
@Component
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publish(OrderEvent event) {
        kafkaTemplate.send("order-events", String.valueOf(event.orderId()), event);
    }
}
```

`KafkaTemplate`은 Spring Boot가 자동으로 구성합니다. 주입받아서 쓰면 됩니다. key에 주문 ID를 넣으면 같은 주문의 이벤트가 같은 파티션으로 가서 순서가 보장됩니다.

### Consumer

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events")
    public void handle(OrderEvent event) {
        // 메시지 처리
    }
}
```

`@KafkaListener` 하나면 됩니다. `group-id`는 `application.yml`에서 설정한 값이 적용됩니다.

### 에러 핸들링

Consumer에서 처리 실패가 나면 기본적으로 계속 재시도합니다. 무한 재시도는 위험하니까, 재시도 횟수를 제한하고 실패 메시지를 DLT(Dead Letter Topic)로 보내는 게 일반적입니다.

```java
@Bean
public CommonErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
    return new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(kafkaTemplate),
        new FixedBackOff(1000L, 3)  // 1초 간격, 최대 3회
    );
}
```

3번 재시도 후에도 실패하면 `order-events.DLT` 토픽으로 메시지가 이동합니다. DLT에 쌓인 메시지는 나중에 수동으로 원인 분석 후 재처리하면 됩니다.

### 테스트

```java
@SpringBootTest
@EmbeddedKafka(
    topics = "order-events",
    bootstrapServersProperty = "spring.kafka.bootstrap-servers"
)
class OrderEventTest {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Test
    void 주문_이벤트를_발행하고_소비한다() {
        // @EmbeddedKafka가 인메모리 Kafka Broker를 띄워줌
        kafkaTemplate.send("order-events", "1", new OrderEvent(1L, "PAID"));
        // Consumer 검증 로직
    }
}
```

`@EmbeddedKafka`를 붙이면 테스트용 인메모리 Kafka가 뜹니다. 실제 Kafka 클러스터 없이 통합 테스트가 가능합니다.

### 트랜잭션이 필요한 경우

이커머스 프로젝트에서 했던 것처럼, 트랜잭션 커밋 후에만 이벤트를 발행하려면:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderPlacedEvent event) {
    kafkaTemplate.send("order-events", String.valueOf(event.orderId()), event);
}
```

트랜잭션이 롤백되면 이벤트가 발행되지 않습니다. "존재하지 않는 주문에 대한 알림"이 나가는 걸 막을 수 있습니다.

---

## 정리

Kafka를 쓰면서 제일 중요하다고 느낀 건 세 가지입니다.

**파티션 설계를 처음에 잘 해야 합니다.** 파티션 수는 늘릴 수 있지만 줄일 수 없고, key 설계가 순서 보장과 Consumer 병렬성을 결정합니다. 나중에 바꾸기 어렵습니다.

**전달 보장 수준을 의식적으로 선택해야 합니다.** 기본값인 at-least-once가 대부분 적절하지만, Consumer 쪽에서 멱등성을 챙겨야 합니다. Exactly-once는 Kafka 내부 연산에만 보장된다는 걸 잊으면 안 됩니다.

**Consumer Lag을 모니터링해야 합니다.** Lag이 늘어나면 처리가 밀리고 있다는 신호입니다. 리밸런싱 설정도 운영 환경에 맞게 튜닝하지 않으면 배포할 때마다 처리가 멈추는 상황이 생깁니다.
