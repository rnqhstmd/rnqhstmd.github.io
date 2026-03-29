---
layout: post
title: "약속 시간 역산 앱을 만들며 고민했던 것들 — iOS 편"
date: 2026-03-29
categories: [ios, swiftui]
tags: [swiftui, kakaomap-sdk, async-await, uikit-interop, ux]
---

백엔드 41개 API를 만들어놓고 iOS 앱을 열어보니, 모든 화면이 하드코딩된 MockData로 돌아가고 있었습니다. 친구 목록도, 약속도, 알림도 앱을 재시작하면 항상 같은 데이터. 여기서부터가 진짜 시작이었습니다.

## MockData 지옥에서 탈출하기

### 화면은 있는데 데이터가 가짜다

iOS 앱의 모든 ViewModel이 이런 식이었습니다.

```swift
class MyAppointmentsViewModel: ObservableObject {
    @Published var upcomingAppointments: [Appointment] = MockData.allAppointments
    @Published var pastAppointments: [Appointment] = []

    func loadAppointments() {
        // 아무것도 하지 않음
    }
}
```

`MockData.allAppointments`는 앱에 박혀 있는 정적 배열입니다. 백엔드에 약속을 100개 만들어도 iOS에서는 항상 같은 3개 약속만 보입니다.

### 5개 PR에 걸친 체계적 전환

한 번에 바꾸면 어디서 뭐가 깨졌는지 알 수 없으니, 도메인별로 나눠서 전환했습니다. 모든 도메인에 같은 아키텍처 패턴을 적용했습니다.

```
APIEndpoint (enum) → DTO (Codable) → Repository (protocol + impl) → ViewModel
```

전환 후 코드는 이렇게 바뀌었습니다.

```swift
@MainActor
final class MyAppointmentsViewModel: ObservableObject {
    @Published var upcomingAppointments: [Appointment] = []
    @Published var pastAppointments: [Appointment] = []
    private let repository: AppointmentRepository

    func loadAppointments() async {
        async let upcoming = repository.getMyAppointments(status: .upcoming)
        async let past = repository.getMyAppointments(status: .past)

        do {
            let (upcomingResult, pastResult) = try await (upcoming, past)
            upcomingAppointments = upcomingResult.appointments.map { Appointment(from: $0) }
            pastAppointments = pastResult.appointments.map { Appointment(from: $0) }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

`async let`으로 UPCOMING과 PAST를 동시에 요청합니다. 순차 호출이면 500ms + 500ms = 1초인데, 병렬이면 ~500ms. 약속 생성 화면에서도 친구 목록과 출발지 목록을 병렬로 불러옵니다.

```swift
// CreateAppointmentViewModel.swift — init에서 병렬 로드
init(...) {
    Task {
        async let _: () = loadFriends()
        async let _: () = loadDeparturePlaces()
    }
}
```

5개 PR로 전체 도메인을 전환했습니다.

| PR | 도메인 | API 수 | 주요 패턴 |
|----|--------|--------|----------|
| #11 | 친구 | 12개 | DTO 8종, APIEndpoint 14개 |
| #12 | 약속 | 12개 | cursor 페이지네이션, `async let` 병렬 |
| #13 | 알림 | 6개 | 낙관적 읽음 처리 + 롤백 |
| #14 | 출발지/설정/태그 | 10개 | `async let` 3종 병렬, id 타입 통일 |
| #15 | 캘린더 | 1개 | 월 변경 시 자동 로드 |

### 통합하니까 빌드가 깨진다

5개 PR을 개별로 머지한 뒤 통합 빌드를 돌리니 에러가 터졌습니다. PR #16에서 잡은 문제들입니다.

- `TagResponseDTO`가 `FriendDTO.swift`와 `SettingsDTO.swift` 두 곳에 중복 정의 — 단일 정의로 통합
- `Tag.id` 타입이 어떤 곳은 `String`, 어떤 곳은 `Int64` — 백엔드 Long에 맞게 `Int64`로 통일
- `Step4ConfirmView`에서 `MockData` 직접 참조가 남아 있음 — Repository API 호출로 교체

`MockData`는 `#Preview` 안에서만 사용하고, 프로덕션 코드에서는 완전히 제거했습니다.

---

## 호스트인데 수정 버튼이 안 보인다

### 하드코딩된 호스트 판별

약속 상세 화면에서 호스트에게만 "약속 수정/취소" 메뉴가 보여야 합니다. 그런데 실제 호스트가 들어가도 메뉴가 안 보이는 버그가 있었습니다.

```swift
// Before — 참여자 배열에서 호스트를 찾아 id가 0인지 확인
private var isHost: Bool {
    viewModel.appointment.participants.first(where: { $0.isHost })?.id == 0
}
```

`id == 0`이라는 하드코딩이 문제였습니다. 실제 사용자 ID는 0이 아니니까 항상 `false`를 반환합니다. 클라이언트에서 호스트 여부를 추론하려다 생긴 버그입니다.

### 서버가 판단하게 하기

백엔드 `AppointmentDetailResponse`에는 이미 `isHost` 필드가 있었습니다. 그걸 그대로 쓰면 되는 거였습니다.

```swift
// After — 서버 응답을 그대로 사용
struct Appointment {
    let currentUserIsHost: Bool  // 백엔드 isHost 응답 매핑
}

private var isHost: Bool {
    viewModel.appointment.currentUserIsHost
}
```

```swift
// 호스트일 때만 메뉴 표시
if isHost {
    Menu {
        Button { viewModel.prepareEdit() } label: {
            Label("약속 수정", systemImage: "pencil")
        }
        Button(role: .destructive) {
            viewModel.showCancelConfirm = true
        } label: {
            Label("약속 취소", systemImage: "xmark.circle")
        }
    } label: {
        Image(systemName: "ellipsis.circle")
    }
}
```

권한 판별은 서버에서 하고, 클라이언트는 결과를 받아 쓰면 됩니다. 클라이언트에서 "참여자 중 isHost가 true인 사람의 id가 현재 사용자 id와 같은지" 같은 로직을 짜는 순간 엣지 케이스에 당합니다.

---

## 지도 자리에 회색 사각형만 보인다

### UIKit SDK를 SwiftUI에서 쓸 수 없다

약속 상세, 장소 검색, 출발지 검색 3개 화면에 지도 placeholder만 있었습니다. 카카오맵 iOS SDK v2를 통합하려 했는데, SDK가 UIKit 기반이라 SwiftUI에서 바로 쓸 수 없었습니다. 그리고 카카오 개발자 콘솔에 번들 ID를 등록하지 않으면 인증이 실패하는데, 시뮬레이터에서 테스트할 때 이 문제가 반복됐습니다.

### UIViewControllerRepresentable 래퍼 + 폴백

SDK의 엔진 라이프사이클(`prepare → activate → pause → reset`)을 SwiftUI 뷰 라이프사이클에 매핑하는 래퍼를 만들었습니다.

```swift
struct KakaoMapContainerView: View {
    let latitude: Double?
    let longitude: Double?
    let height: CGFloat
    let markerTitle: String?
    @State private var showFallback = false

    var body: some View {
        Group {
            if showFallback || !hasValidCoordinates {
                MapFallbackView(placeName: markerTitle, height: height)
            } else {
                KakaoMapView(
                    latitude: latitude,
                    longitude: longitude,
                    markerTitle: markerTitle,
                    isScrollEnabled: false,
                    onAuthFailure: { showFallback = true }
                )
                .frame(height: height)
            }
        }
    }

    private var hasValidCoordinates: Bool {
        guard let lat = latitude, let lon = longitude else { return false }
        return !(lat == 0 && lon == 0)
    }
}
```

인증 실패 시 빈 화면 대신 폴백 뷰를 보여주는 게 핵심입니다. `onAuthFailure` 콜백으로 인증 실패가 감지되면 `showFallback = true`로 전환합니다. 좌표가 nil이거나 (0, 0)이어도 폴백으로 빠집니다.

```swift
struct KakaoMapView: UIViewControllerRepresentable {
    func updateUIViewController(_ vc: KakaoMapViewController, context: Context) {
        let newLat = latitude ?? KakaoMapViewController.defaultLatitude
        let newLon = longitude ?? KakaoMapViewController.defaultLongitude

        // 좌표가 실제로 변경된 경우에만 카메라+마커 이동
        if abs(newLat - oldLat) > 0.00001 || abs(newLon - oldLon) > 0.00001 {
            vc.updatePosition(latitude: newLat, longitude: newLon, title: markerTitle)
        }
    }

    // 뷰 해제 시 엔진 정리 — 메모리 누수 방지
    static func dismantleUIViewController(_ vc: KakaoMapViewController, coordinator: ()) {
        vc.cleanup()
    }
}
```

`dismantleUIViewController`에서 엔진 리소스를 명시적으로 정리하는 것도 중요했습니다. SwiftUI는 뷰를 자주 재생성하는데, 매번 카카오맵 엔진이 새로 초기화되면서 메모리가 누적되는 문제를 이것으로 막았습니다.

약속 상세(220pt), 장소 검색(160pt), 출발지 검색(120pt 미니맵) 3개 화면에서 `isScrollEnabled = false`로 사용합니다. ScrollView 안에 지도가 있으면 드래그 제스처가 충돌하기 때문입니다.

---

## 카운트다운이 1분마다 뚝뚝 바뀐다

### 임박한 상황에서 긴박감이 없다

"출발까지 2분" → 60초 동안 아무 변화 없음 → "출발까지 1분". 임박한 상황에서 초 단위 긴박감이 전혀 전달되지 않았습니다.

```swift
// Before — 60초 고정 간격
.onReceive(Timer.publish(every: 60, on: .main, in: .common).autoconnect()) { _ in
    secondsRemaining = Int(targetDate.timeIntervalSince(Date()))
}
```

### 자기 재호출 패턴으로 동적 간격

30분 이내면 1초 간격, 그 이후면 60초 간격으로 동적 전환하는 패턴을 구현했습니다.

```swift
private func updateAndReschedule() {
    timer?.invalidate()
    let seconds = Int(targetDate.timeIntervalSince(Date()))
    secondsRemaining = max(0, seconds)
    guard secondsRemaining > 0 else { return }  // 0이면 타이머 중지

    // 30분(1800초) 이내 → 1초, 이외 → 60초
    let interval: TimeInterval = secondsRemaining <= 1800 ? 1 : 60

    let newTimer = Timer(timeInterval: interval, repeats: false) { _ in
        updateAndReschedule()  // 자기 재호출 — 매번 간격 재계산
    }
    timer = newTimer
    RunLoop.main.add(newTimer, forMode: .common)
}
```

`Timer.publish(every:)`는 고정 간격만 지원하니까, `repeats: false`로 1회 타이머를 만들고 매번 자기 재호출하면서 간격을 다시 계산합니다. `RunLoop`의 `.common` 모드로 등록해야 스크롤 중에도 타이머가 멈추지 않고, `guard secondsRemaining > 0`으로 0에 도달하면 타이머를 자동 중지합니다.

---

## 길찾기 버튼을 눌렀는데 아무 일도 안 일어난다

### 특정 앱에 의존하면 미설치 시 실패

약속 상세의 "길찾기" 버튼은 카카오맵 앱으로 연결하도록 구현되어 있었는데, 카카오맵을 설치하지 않은 사용자는 버튼을 눌러도 아무 반응이 없었습니다.

### 3단 폴백 체인

```swift
func openNavigation() {
    let lat = appointment.latitude
    let lng = appointment.longitude
    let name = appointment.placeName
        .addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""

    let candidates: [(scheme: String, urlString: String)] = [
        ("kakaomap://", "kakaomap://route?ep=\(lat),\(lng)&by=CAR"),
        ("nmap://", "nmap://route/car?dlat=\(lat)&dlng=\(lng)&dname=\(name)&appname=com.odiya"),
        ("maps://", "maps://?daddr=\(lat),\(lng)&dirflg=d")
    ]

    for candidate in candidates {
        guard let schemeURL = URL(string: candidate.scheme),
              let targetURL = URL(string: candidate.urlString) else { continue }

        if candidate.scheme == "maps://" || UIApplication.shared.canOpenURL(schemeURL) {
            UIApplication.shared.open(targetURL)
            return
        }
    }
}
```

`canOpenURL`로 앱 설치 여부를 확인하고, 설치된 첫 번째 앱으로 연결합니다. 애플맵은 iOS에 기본 내장이라 `canOpenURL` 체크를 생략합니다. 카카오맵과 네이버맵 둘 다 없어도 애플맵이 최종 폴백이니까, 어떤 환경에서든 길찾기가 동작합니다.

---

## DateFormatter를 매번 만들고 있었다

### 보이지 않는 성능 문제

iOS에서 `DateFormatter` 인스턴스 생성은 비용이 큰 작업입니다. Apple 공식 문서에서도 캐싱을 권장합니다. 약속 목록의 각 카드마다, 카운트다운 타이머가 매초 갱신될 때마다 새 `DateFormatter`가 생성되고 있었습니다.

```swift
// Before — 매번 새로 생성
var koreanFormatted: String {
    let formatter = DateFormatter()
    formatter.locale = Locale(identifier: "ko_KR")
    formatter.dateFormat = "M월 d일 (E) a h:mm"
    return formatter.string(from: self)
}
```

### static let으로 타입 레벨 캐싱

```swift
extension Date {
    private static let koreanFullFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "ko_KR")
        formatter.dateFormat = "M월 d일 (E) a h:mm"
        return formatter
    }()

    private static let koreanDateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "ko_KR")
        formatter.dateFormat = "yyyy년 M월 d일 (E)"
        return formatter
    }()

    private static let koreanTimeFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "ko_KR")
        formatter.dateFormat = "a h:mm"
        return formatter
    }()

    var koreanFormatted: String { Self.koreanFullFormatter.string(from: self) }
    var koreanDateFormatted: String { Self.koreanDateFormatter.string(from: self) }
    var koreanTimeFormatted: String { Self.koreanTimeFormatter.string(from: self) }
}
```

`static let` + 클로저 초기화는 앱 라이프타임 동안 딱 한 번만 실행됩니다. 이후로는 같은 인스턴스를 재사용합니다. 카운트다운이 매초 갱신되는 상황에서 formatter를 매번 만드는 것과 캐싱된 것을 쓰는 건 체감 차이가 큽니다.

---

## 데이터가 바뀌었는데 화면이 안 바뀐다

### Pull-to-Refresh가 없다

친구를 추가했는데 목록에 안 보이고, 약속을 만들었는데 목록이 안 바뀌고. 화면을 나갔다 다시 들어와야만 새 데이터가 보였습니다. iOS 사용자라면 당연히 기대하는 "당겨서 새로고침"이 어디에도 없었습니다.

### SwiftUI `.refreshable` 일괄 적용

```swift
// MyAppointmentsView.swift
List {
    // ...
}
.onAppear {
    Task { await viewModel.loadAppointments() }
}
.refreshable {
    await viewModel.loadAppointments()
}
```

4개 주요 화면에 동일한 패턴을 적용했습니다.

| 화면 | 호출 |
|------|------|
| 약속 목록 | `viewModel.loadAppointments()` |
| 친구 목록 | `viewModel.loadFriends()` |
| 알림 목록 | `viewModel.loadNotifications()` |
| 캘린더 | `viewModel.loadCalendarData()` |

SwiftUI의 `.refreshable`이 좋은 점은, async 함수가 완료되면 refresh indicator가 자동으로 사라진다는 겁니다. 별도의 `isRefreshing` 상태 변수를 관리할 필요가 없습니다.

빈 상태(약속이 하나도 없는 화면)에서도 `.refreshable`을 걸었습니다. 빈 화면에서 당겨서 새로고침을 시도하는 건 자연스러운 사용자 행동이니까요.

---

## 프로필 이미지 없으면 전부 같은 아이콘이다

### 누가 누구인지 구분이 안 된다

카카오 프로필을 설정하지 않은 사용자는 친구 목록, 약속 참여자, 알림 모든 곳에서 동일한 회색 아이콘으로 표시됐습니다.

```swift
// Before — 이미지 없으면 동일한 기본 아이콘
if let url = imageUrl {
    AsyncImage(url: URL(string: url)) { ... }
} else {
    Image(systemName: "person.circle.fill")
        .foregroundStyle(.gray)
}
```

### 그라데이션 이니셜 아바타

닉네임의 첫 글자를 보라 그라데이션 배경 위에 표시하는 이니셜 아바타 시스템을 만들었습니다.

```swift
struct ProfileImageView: View {
    let imageUrl: String?
    var size: CGFloat = 40
    var nickname: String? = nil  // 기존 호출부 하위 호환

    @ViewBuilder
    private var placeholderView: some View {
        if let nickname, let initial = nickname.first {
            ZStack {
                OdiyaColors.initialAvatarGradient
                Text(String(initial))
                    .font(.system(size: size * 0.4, weight: .bold))
                    .foregroundStyle(.white)
            }
            .clipShape(Circle())
        } else {
            Image(systemName: "person.circle.fill")
                .resizable()
                .foregroundStyle(OdiyaColors.odiya300)
        }
    }
}
```

`nickname` 파라미터를 `nil` 기본값으로 추가해서 기존 호출부를 수정하지 않아도 됩니다. 기존에 `ProfileImageView(imageUrl: url, size: 40)`으로 호출하던 코드가 그대로 동작합니다. 닉네임을 전달하는 곳만 `ProfileImageView(imageUrl: url, size: 40, nickname: "민지")`로 바꾸면 됩니다.

"민"이라는 글자 하나가 보라색 원 위에 표시되는 것만으로도, 회색 아이콘이 6개 나열된 것보다 훨씬 구분이 잘 됩니다.

---

## SPM으로는 카카오 로그인이 안 된다

### Bundle ID가 없다

iOS 프로젝트를 처음 만들 때 SPM(Swift Package Manager)의 `.executableTarget`으로 구성했습니다. 빌드는 되는데 카카오 로그인 콜백이 동작하지 않았습니다. 원인은 SPM 빌드가 iOS 앱 번들을 정상적으로 생성하지 못해서 `CFBundleExecutable`이 누락되고, URL Scheme도 설정할 수 없기 때문이었습니다.

### xcodegen으로 전환

`project.yml` 스펙 파일을 작성하고 xcodegen으로 `.xcodeproj`를 생성했습니다. Info.plist에 카카오 URL Scheme(`kakao{앱키}`), `LSApplicationQueriesSchemes`, `UILaunchScreen`을 설정하니 카카오 로그인이 시뮬레이터에서 정상 동작했습니다.

같은 PR에서 iOS 최소 타겟을 v16에서 v17로 올렸습니다. `foregroundStyle`, `onChange(of:initial:)`, `#Preview` 매크로 등 iOS 17+ API를 이미 여러 곳에서 사용하고 있었는데, 빌드 에러가 6건이나 쌓여 있었습니다. 타겟을 올리니 일괄 해결됐습니다.

---

## Pencil 프로토타입에서 코드까지

### 디자인 없이 만든 화면의 한계

처음에는 SwiftUI 기본 컴포넌트로만 화면을 구성했습니다. 동작은 하지만, 단색 아바타, 그림자 없는 카드, 단조로운 알림 목록. "앱처럼 보이지 않는" 상태였습니다.

### Pencil MCP로 프로토타입 → iOS 적용

Pencil MCP를 활용해서 12개 화면의 프로토타입을 제작하고, 거기서 도출된 개선사항을 실제 코드에 적용했습니다.

| 개선 | Before | After |
|------|--------|-------|
| 아바타 | 단색 원 | 보라 그라데이션 + 한글 이니셜 |
| 카드 | 테두리만 있는 평면 | 소프트 섀도우 + 보라 그라데이션 뱃지 |
| 카운트다운 | 텍스트만 | 핑크 그라데이션 배너 |
| 알림 목록 | 단순 리스트 | 날짜 그룹핑 + 유형별 색상 아이콘 |
| 설정 | 메뉴만 | 이니셜 프로필 헤더 + 서브텍스트 + 버전 표시 |

466줄 추가, 20개 파일 변경. 같은 기능인데 확실히 "앱다워" 보였습니다.

`OdiyaColors`에 그라데이션 속성을 추가하고, 앱 전체에서 일관된 오디야 보라(#5B3A8C → #8B5FBF) 팔레트를 사용하도록 통일한 것이 가장 효과가 컸습니다. 색상 하나만 바꿔도 앱이 "브랜드가 있는 앱"처럼 보입니다.

---

## 되돌아보며

iOS 개발에서 가장 많이 배운 건 **"동작하는 것"과 "쓸 수 있는 것" 사이의 거리**입니다.

MockData로 동작하는 화면은 데모용이지 앱이 아닙니다. Pull-to-Refresh 없는 목록, 프로필 이미지 없는 아바타 6개, 먹통인 길찾기 버튼. 기능 명세에는 안 적혀 있지만, 없으면 사용자가 바로 알아챕니다.

같은 API를 호출해도 `async let`으로 병렬 로드하느냐에 따라 체감이 달라지고, 같은 데이터를 보여줘도 그라데이션 아바타 하나로 앱의 인상이 바뀝니다.

사용자가 쓰는 건 API가 아니라 화면입니다. 거기에 시간을 쓴 건 아깝지 않았습니다.
