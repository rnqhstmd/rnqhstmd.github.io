---
layout: post
title: "@MockitoSpyBean으로 통합 테스트의 순수성 지키기"
date: 2025-10-29
categories: [testing, spring]
---

## 문제: 통합 테스트인데 실제 DB를 전혀 검증하지 않았다

도메인 통합 테스트를 작성하던 중 이상한 방향으로 흘러가고 있다는 생각이 들었다. 서비스 레이어 테스트인데 Repository를 `mock()`으로 처리하고 있었다.

```java
// 변경 전: 이게 정말 통합 테스트인가?
@ExtendWith(MockitoExtension.class)
class PointServiceTest {

    @Mock
    private PointRepository pointRepository;

    @InjectMocks
    private PointService pointService;

    @Test
    void 포인트_충전() {
        when(pointRepository.findByUserId(1L))
            .thenReturn(Optional.of(new Point(1L, 1000)));

        pointService.charge(1L, 500);

        verify(pointRepository).save(any());
    }
}
```

`when().thenReturn()`으로 모든 동작을 미리 정의해두니, 실제 JPA 쿼리가 제대로 동작하는지 전혀 검증되지 않았다. 이건 단위 테스트다. 통합 테스트라고 부를 수 없다.

## @MockitoSpyBean이란?

`@MockitoSpyBean`은 실제 Bean을 Spring Context에서 가져오되, 특정 메서드만 선택적으로 stubbing할 수 있다.

핵심은 **기본 동작이 "진짜 동작"**이라는 점이다.

| 구분 | Mock | Spy |
|------|------|-----|
| 기본 동작 | 모든 메서드를 가짜로 | 실제 동작 |
| stubbing | 원하는 메서드를 가짜로 교체 | 원하는 메서드만 가짜로 교체 |
| 용도 | 단위 테스트 | 통합 테스트에서 일부만 격리 |

## 변경 후: 진짜 통합 테스트

```java
@SpringBootTest
class PointServiceIntegrationTest {

    @Autowired
    private PointService pointService;

    @Autowired
    private PointRepository pointRepository;

    @MockitoSpyBean
    private ExternalNotificationClient notificationClient;

    @Test
    @Transactional
    void 포인트_충전_후_실제_DB에_저장된다() {
        // given
        Point point = pointRepository.save(new Point(1L, 1000));

        // 외부 알림은 실제 호출하지 않고 stubbing
        doNothing().when(notificationClient).sendChargeNotification(any());

        // when
        pointService.charge(1L, 500);

        // then - 실제 DB에서 조회해서 검증
        Point saved = pointRepository.findByUserId(1L).orElseThrow();
        assertThat(saved.getBalance()).isEqualTo(1500);

        // 알림 클라이언트가 호출됐는지 검증
        verify(notificationClient).sendChargeNotification(any());
    }
}
```

이제 실제 JPA 쿼리가 수행되고, 실제 DB에 저장되는지 검증한다. 외부 시스템 연동(알림 클라이언트)만 선택적으로 stubbing했다.

## 인사이트

**Spy는 "실제 동작 + 검증 도구"의 조합이다.** 실제 동작을 유지하면서 호출 여부, 호출 횟수, 인자 등을 검증할 수 있다.

**통합 테스트는 "결과"를 검증한다.** Mock 기반 테스트는 "행동"을 검증한다. 통합 테스트에서 `verify`만 쓰고 `assertThat`이 없다면 테스트 전략을 다시 생각해볼 필요가 있다.

## 테스트 전략 레이어링

테스트는 목적에 따라 계층을 나눠야 한다.

```
단위 테스트 (@Mock + @InjectMocks)
  └── 도메인 로직의 순수 검증
  └── 빠른 피드백

통합 테스트 (@MockitoSpyBean + @SpringBootTest)
  └── 실제 DB 저장/조회 검증
  └── 레이어 간 협력 검증
  └── 외부 의존성만 선택적 격리

E2E 테스트 (TestRestTemplate / MockMvc)
  └── API 레벨 전체 흐름 검증
```

각 레이어가 검증해야 할 것을 명확히 하고, 그에 맞는 도구를 선택해야 한다. `@MockitoSpyBean`은 통합 테스트 레이어에서 외부 의존성을 격리할 때 가장 적합한 도구다.
