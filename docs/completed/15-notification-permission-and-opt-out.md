# 15. 알림 권한 요청 시점 & 알림 켜기/끄기

## 문제

`POST_NOTIFICATIONS`(Android 13+) 런타임 요청 코드가 어디에도 없었다(매니페스트 선언만
있었음). 언제 물어볼지, 거부/차단 시 서버가 계속 무의미한 푸시를 보내는 걸 어떻게 막을지 결정이
필요했다.

## 결정 1 — 요청 시점: `MainActivity` 진입 시, 설명 없이 바로 시스템 다이얼로그

카메라·위치([DrugStoreList.java](https://github.com/sonsehwan/YakSok_Frontend/blob/main/app/src/main/java/com/example/medication/ui/drugstore/DrugStoreList.java))처럼
"기능을 쓰기 직전"에 요청하는 패턴은 알림에는 구조적으로 적용할 수 없다 — 알림이 실제로 발송되는
시점(서버 푸시 도착)엔 앱이 포그라운드가 아닐 수도 있어 그 순간 권한 다이얼로그를 띄울 방법이
없다. 그래서 "약속 생성 화면 진입 시" 같은 대안도 기각하고, 로그인 후 첫 진입 화면인
`MainActivity.onCreate()`에서 바로 요청하는 쪽을 택했다([MainActivity.java](https://github.com/sonsehwan/YakSok_Frontend/blob/main/app/src/main/java/com/example/medication/ui/main/MainActivity.java)).

**택하지 않은 대안:** 설명 화면(프리프롬프트)을 먼저 보여주고 "알림 받기" 버튼을 누른 사람에게만
시스템 다이얼로그를 띄우는 방식. 평생 두 번뿐인 요청 기회를 맥락 없이 쓰지 않는다는 점에서
그랜트율엔 유리하지만, 이 앱엔 온보딩 화면 자체가 없어 새 화면을 만들어야 하는 비용이 있고 단순함을
우선해 스킵했다. 나중에 거부율이 실제로 문제가 되면 추가할 것.

## 결정 2 — 꺼짐 처리: 서버에 저장, `PushSender` 한 곳에서 필터

권한은 승인됐지만 사용자가 껐을 수도 있는 경우를 클라이언트 로컬 필터(푸시는 오되 표시만 안 함)로
할지, 서버가 아예 안 보내게 할지 갈렸다. 서버 저장 쪽을 택해서 `User`에
`notificationEnabled`(기본 `true`, `columnDefinition = "TINYINT(1) DEFAULT 1"`로 기존 행도
자동 채워지게 함)를 추가했다.

필터링은 개별 발송 지점마다 넣지 않고 모든 푸시가 공통으로 거치는
[`PushSender.sendData`](https://github.com/sonsehwan/YakSok_Backend/blob/main/src/main/java/com/sehwan/YakSok/fcm/PushSender.java)
한 곳에서만 한다 — `userRepository.findAllById(userIds)`로 조회해
`notificationEnabled == false`인 사용자를 토큰 조회 전에 걸러낸다. 리마인더 스케줄러, 채팅, 공유,
회원 관련 알림 등 `PushSender`를 부르는 모든 호출자가 코드 수정 없이 자동으로 적용받는다.

**택하지 않은 대안:** 클라이언트 로컬 필터(서버 스키마 변경 없이 가장 쌈). 기각 이유 — 꺼놓은
사용자에게도 서버가 계속 FCM 요청을 보내는 게 낭비이고, 다기기 상황에서 기기마다 로컬 설정이
따로 놀 수 있음.

## 변경 지점

- 백엔드: `User`(`notificationEnabled` 필드), `UserResponse`(필드 추가, `toDto()`/생성자 양쪽),
  `NotificationSettingRequest`(신규), `UserService.updateNotificationSetting`,
  `UserController`의 `PATCH /api/users/notification-setting`, `PushSender.sendData`(필터링)
- 프론트: `UserResponse` 모델(`notificationEnabled` 필드), `NotificationSettingRequest`(신규),
  `UserApi.updateNotificationSetting`, `MainActivity.requestNotificationPermission()`(API 33+
  에서만 요청), `SettingsFragment`(알림 토글 추가, 변경 시 PATCH 호출 후 실패하면 스위치 롤백),
  `fragment_settings.xml`(알림 행 UI)

## 알려진 제약

- 토글 실패 시 로컬 캐시(`SprefsManager`)는 서버 응답이 성공한 경우에만 갱신한다. 네트워크가
  끊긴 채로 반복 토글하면 스위치는 매번 원상복구되므로 UX상 로딩 인디케이터는 없음 — 필요해지면
  추가.
- OS 레벨에서 알림을 차단(설정 > 알림 꺼짐)한 경우는 이 플래그와 별개다. 앱은 여전히
  `notificationEnabled = true`로 보내려 시도하고 OS가 걸러낸다 — 이 둘을 동기화하는 로직은 없음.
