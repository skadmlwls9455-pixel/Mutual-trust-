# Pact — MVP 기술 명세서

> 커플 상호 동의 기반 투명성 앱  
> MVP 범위: 위치 공유 + GPS 스푸핑 감지

---

## 1. 프로젝트 개요

### 핵심 가치
- 양방 동의 없이는 작동 불가 (한쪽만 쓰는 구조 원천 차단)
- 연동 거부 또는 해제 자체가 상대방에게 알림으로 전달됨
- Life360과의 차별점: **GPS 스푸핑 감지**

### MVP 기능 범위
1. 회원가입 / 커플 연동 (초대 코드 방식)
2. 실시간 위치 공유 (지도)
3. GPS 스푸핑 감지 및 알림
4. 연동 해제 시 상대방 알림

---

## 2. 기술 스택

| 레이어 | 기술 | 선택 이유 |
|---|---|---|
| 앱 | Flutter 3.x | iOS/Android 동시 개발, 단일 코드베이스 |
| 백엔드 | Firebase | 실시간 DB, Auth, 무료 플랜으로 MVP 충분 |
| 위치 | Google Maps SDK + Geolocator | Flutter 생태계 표준 |
| 알림 | Firebase Cloud Messaging (FCM) | 무료, Flutter 연동 쉬움 |
| 상태관리 | Riverpod | Flutter 현재 권장 패턴 |

---

## 3. Firebase 구조

```
Firestore
├── users/{uid}
│   ├── displayName: string
│   ├── email: string
│   ├── partnerId: string | null
│   ├── fcmToken: string
│   └── createdAt: timestamp
│
├── couples/{coupleId}
│   ├── userA: string (uid)
│   ├── userB: string (uid)
│   ├── status: "pending" | "active" | "dissolved"
│   ├── dissolvedBy: string | null
│   ├── dissolvedAt: timestamp | null
│   └── createdAt: timestamp
│
├── locations/{uid}
│   ├── lat: number
│   ├── lng: number
│   ├── accuracy: number
│   ├── speed: number
│   ├── altitude: number
│   ├── isSpoofSuspected: boolean
│   └── updatedAt: timestamp
│
└── invites/{code}
    ├── fromUid: string
    ├── used: boolean
    └── createdAt: timestamp
```

---

## 4. GPS 스푸핑 감지 로직

스푸핑을 탐지하는 방법은 단일 신호가 아닌 **복합 이상 징후**를 조합:

```dart
class SpoofDetector {
  // 탐지 기준 1: 비현실적 이동 속도
  // 인간 최고 속도 ~45km/h (달리기), 차량 ~200km/h
  // 그 이상이면 스푸핑 의심
  static bool isSpeedUnrealistic(double speedMps) {
    return speedMps > 60; // 216km/h 이상
  }

  // 탐지 기준 2: 위치 정확도가 너무 완벽
  // 실제 GPS는 항상 오차(accuracy > 0)가 있음
  // accuracy == 0.0 이면 시뮬레이터/스푸핑 의심
  static bool isAccuracyTooGood(double accuracy) {
    return accuracy < 1.0;
  }

  // 탐지 기준 3: 순간 이동
  // 이전 위치에서 현재 위치까지 이동 시간 대비 거리
  static bool isTeleported(
    LatLng prev, LatLng curr,
    Duration elapsed,
  ) {
    final distanceMeters = Geolocator.distanceBetween(
      prev.latitude, prev.longitude,
      curr.latitude, curr.longitude,
    );
    final speedMps = distanceMeters / elapsed.inSeconds;
    return speedMps > 100; // 360km/h 이상
  }

  // 탐지 기준 4: mock location 설정 (Android)
  // Location 패키지의 isMock 플래그 활용
  static bool isMockLocation(bool? isMock) {
    return isMock == true;
  }

  // 최종 판정: 2개 이상 해당 시 스푸핑 의심
  static bool evaluate({
    required double speed,
    required double accuracy,
    required bool? isMock,
    LatLng? prev,
    LatLng? curr,
    Duration? elapsed,
  }) {
    int suspicionScore = 0;
    if (isSpeedUnrealistic(speed)) suspicionScore++;
    if (isAccuracyTooGood(accuracy)) suspicionScore++;
    if (isMockLocation(isMock)) suspicionScore++;
    if (prev != null && curr != null && elapsed != null) {
      if (isTeleported(prev, curr, elapsed)) suspicionScore++;
    }
    return suspicionScore >= 2;
  }
}
```

---

## 5. 핵심 플로우

### 5-1. 커플 연동 플로우
```
A가 초대 코드 생성
  → Firestore invites/{code} 생성
  → B에게 코드 전달 (카카오 공유 등)
  → B가 코드 입력
  → couples/{coupleId} 생성, status: "active"
  → 양쪽 앱에서 위치 공유 시작
```

### 5-2. 연동 해제 플로우
```
A가 연동 해제 시도
  → 경고 팝업: "상대방에게 알림이 전송됩니다"
  → 확인 시 couples/{coupleId}.status = "dissolved"
  → B에게 FCM 푸시: "파트너가 연동을 해제했습니다" 즉시 전송
  → 양쪽 위치 공유 중단
```

### 5-3. 스푸핑 감지 플로우
```
위치 업데이트 수신 (30초마다)
  → SpoofDetector.evaluate() 실행
  → isSpoofSuspected: true 이면
    → Firestore locations/{uid}.isSpoofSuspected = true
    → 파트너에게 FCM 알림: "파트너의 위치 신호가 불안정합니다"
  → 3회 연속 의심 시 강력 경고 알림
```

---

## 6. 화면 구성 (MVP)

```
스플래시
├── 온보딩 (3장)
├── 로그인 / 회원가입
│
메인 (연동 전)
└── 초대 코드 생성 / 입력 화면

메인 (연동 후)
├── 지도 화면 (홈)
│   ├── 내 위치 마커
│   ├── 파트너 위치 마커
│   └── 스푸핑 의심 배너 (조건부)
├── 알림 내역
└── 설정
    ├── 프로필
    ├── 연동 해제
    └── 알림 설정
```

---

## 7. 폴더 구조

```
lib/
├── main.dart
├── app.dart                    # MaterialApp, 라우팅
│
├── core/
│   ├── firebase_options.dart
│   ├── router.dart             # GoRouter
│   └── theme.dart
│
├── features/
│   ├── auth/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── couple/
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   ├── location/
│   │   ├── data/
│   │   ├── domain/             # SpoofDetector 여기
│   │   └── presentation/
│   └── notification/
│
└── shared/
    ├── widgets/
    └── utils/
```

---

## 8. 주요 패키지

```yaml
dependencies:
  flutter:
    sdk: flutter

  # 상태관리
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  # Firebase
  firebase_core: ^3.3.0
  firebase_auth: ^5.1.4
  cloud_firestore: ^5.2.1
  firebase_messaging: ^15.0.4

  # 위치
  geolocator: ^12.0.0
  google_maps_flutter: ^2.6.1

  # 라우팅
  go_router: ^14.2.7

  # 기타
  uuid: ^4.4.2
  intl: ^0.19.0

dev_dependencies:
  build_runner: ^2.4.11
  riverpod_generator: ^2.4.3
  flutter_lints: ^4.0.0
```

---

## 9. 환경 변수 (.env)

```
GOOGLE_MAPS_API_KEY_ANDROID=...
GOOGLE_MAPS_API_KEY_IOS=...
```

Firebase는 `google-services.json` (Android) / `GoogleService-Info.plist` (iOS) 사용

---

## 10. 개발 순서 (권장)

```
Day 1-2   : Flutter 프로젝트 생성, Firebase 연결, Auth 구현
Day 3-4   : 커플 연동 (초대 코드 생성/입력)
Day 5-7   : 실시간 위치 공유 + 지도 표시
Day 8-10  : GPS 스푸핑 감지 로직 구현
Day 11-12 : FCM 알림 (연동 해제 알림, 스푸핑 알림)
Day 13-14 : UI 정리, 버그 수정
Day 15+   : 베타 테스트 (지인 커플 3-5쌍)
```

---

## 11. Claude Code에서 시작하는 방법

```bash
# 1. Flutter 프로젝트 생성
flutter create pact --org com.pact.app --platforms ios,android

# 2. 디렉토리 이동
cd pact

# 3. pubspec.yaml 패키지 추가 후
flutter pub get

# 4. Firebase CLI 설치 및 연결
npm install -g firebase-tools
dart pub global activate flutterfire_cli
firebase login
flutterfire configure
```

---

*이 명세서를 Claude Code에 첨부하고 "이 명세서대로 개발 시작해줘" 하면 됩니다.*
