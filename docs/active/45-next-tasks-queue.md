# 45. 다음 작업 큐 (쉬운 순)

**상태:** 2026-11-09 현황 — 45.1 코드 적용(복구 남음)·45.2 완료·45.4 완료·45.5 완료·45.6 완료, **45.3·45.7 미착수**.
소유자가 직접 구현한다(코드는 대신 쓰지 않고 설명만 — 루트 `CLAUDE.md`).
각 항목은 독립 세션에서 진행할 수 있게 출발점만 적는다. 끝난 항목은 `completed/`로 옮기고
이 파일에서 지운다. 설계 결정이 필요한 곳은 **결정할 것**으로 표시했다 — 착수 전에 먼저 묻는다.

## 45.1 이메일 인증 유효시간 1분으로 (시연용) — 코드 적용됨, 시연 뒤 5분 복구 남음

- 위치: `auth/EmailVerificationStore.java:16` `TTL_MINUTES = 5` → `1`.
  만료는 발급 시 `plusMinutes(TTL_MINUTES)`, 검증 시 `expiresAt.isBefore(now)`로 판정.
- 목적은 "만료되면 막힌다"를 시연하는 것. 시연 후 5로 되돌린다 — `main` push는 EC2로 즉시 배포된다.
- 앱에 남은 시간 표시가 있으면 그 숫자도 맞춘다(`ui/login/FindId`, `FindPassword`, `strings.xml` 확인).
- 별개: `PasswordResetTokenStore`의 3분은 그대로 둔다.
- 택하지 않은 대안: `application.yaml` 설정값 분리 — 시연 한 번에는 과함.

## 45.2 회원가입 이메일 인증 — 구현됨

`completed/46-signup-email-verification.md`로 옮김. 실기기 확인은 완료했고, 시연 후 TTL 5분 복구가 남았다.

## 45.3 회원정보 변경 사항 알려주기 (예: 닉네임 수정)

- 의미 확인 필요 — **결정할 것:** "누구에게" 알리나. (a) 같은 계정의 다른 기기 화면 갱신
  (b) 친구/채팅 상대 화면에 바뀐 닉네임 반영.
- 출발점: 이미 있는 `REFRESH` FCM 타입 + `PushSender` + 앱 `YaksokEventBus`(3장 자동 갱신 패턴,
  `completed/03-auto-refresh.md`). 다기기 전송은 `user_devices` 기준(`completed/02`).
- 닉네임 유니크는 33장에서 제거됐다 — 중복 검사 없음 전제.
- 채팅 메시지의 닉네임은 조회 시점 매핑이라(`findByEmailIn`) 저장분은 따로 안 바꿔도 되는지 확인.

## 45.4 친구 요청 알림 + 화면 실시간 갱신

- 요청 생성 지점에서 상대에게 푸시. 받은 요청 다이얼로그/친구 목록이 열려 있으면 `REFRESH`류 신호로
  재조회. 새 `FcmMessageType` 추가가 필요할 수 있다(새 필드/타입 → 착수 전 질문).
- 설정의 알림 off(`User.notificationEnabled`)는 `PushSender` 단일 지점에서 걸러지므로 그 경로를 탄다.
- 앱: `MyFirebaseMessagingService`의 채널 분기(`resolveChannelId`)에 채널을 추가할지 결정.
- **발송 방식: 이벤트 + `AFTER_COMMIT` 리스너로 간다**(`FriendService`가 `@Transactional`이라 서비스에서
  바로 보내면 커밋 전에 푸시가 나감 — `completed/03` 3.3과 같은 이유). 채팅(45.5)을 먼저 하고 같은 구조로 붙인다.
- **`createFriendRequest` 결과가 4갈래라 이벤트 설계가 필요하다:** ① 새 요청(상대: 받은 요청 목록)
  ② 수락 이력 있고 관계만 없어 관계 재생성(상대: 친구 목록) ③ 상대가 먼저 보낸 대기 요청이 있어 **자동 수락**
  (양쪽: 친구 목록) ④ 거절 이력 재시작(상대: 받은 요청 목록). `acceptFriendRequest`/`rejectFriendRequest`/
  `deleteFriend`도 상대 화면에 영향.
- **알림 vs 조용한 갱신 구분:** 요청 받음·친구 성립은 알림+갱신, 거절·삭제는 알림 없이 갱신만
  (거절을 푸시로 알리면 불쾌). 이벤트를 의미별로 나눌지(A) `kind`를 실은 하나로 묶을지(B)는 미정 —
  필요한 것만 만드는 A를 잠정 선호.
- **미정:** 앱 신호를 화면 종류별 별도 버스로 둘지 `YaksokEventBus`를 확장할지. 채팅 목록 갱신(45.5)과
  같은 결정이라 채팅 구현 때 정한 방식을 그대로 따른다.

### 45.4 결정 (코드 확인 후 위 분석을 좁힘) — 서버·앱 코드 적용됨, 실기기 확인 완료

- **구현 위치:** `user/event/FriendshipChangedEvent`·`FriendshipChangedListener`, `FriendService`의 `publish()` 헬퍼
  (createFriendRequest ①③④, acceptFriendRequest에서 호출), 앱 `util/FriendEventBus`, `MyFirebaseMessagingService`의
  `FRIENDSHIP` 분기·채널·switch case, `FriendListFragment` 구독(onResume/onPause).
- **프로필 다이얼로그(친구 목록 탭 / QR)는 액션 성공 시 닫는다.** 요청 보내기·수락·채팅하기 모두. 예전엔
  다이얼로그를 둔 채 `refreshProfileDialog`로 버튼만 바꿨는데(채팅하기는 채팅방에서 돌아오면 다이얼로그가 남아 있었음),
  토스트로 결과를 알리고 닫는 쪽으로 바꾸며 `refreshProfileDialog`와 `code` 인자를 제거했다. 실패 시에는 열어 둔다.
- **친구 요청 알림에 수락/거절 버튼.** 버튼이 `PATCH /api/friend/request/{requestId}/answer`를 부르려면 요청 id가
  필요해, `FriendshipChangedEvent`에 `requestId`(REQUESTED일 때만, ACCEPTED는 null)를 추가하고 리스너가 FCM data에
  `requestId`로 싣는다(그래서 `sendMessage` 대신 `sendData`). 앱은 **`requestId`가 있을 때만** 버튼을 붙인다 —
  "친구가 되었어요" 알림엔 버튼이 없다. 별도 `kind` 필드를 싣는 대안은 버튼에 꼭 필요한 값의 유무로 충분해서 택하지 않음.
  - 버튼 생성은 `showNotification`의 FRIENDSHIP case → `addFriendRequestActions`(같은 파일의 private 메서드. 버튼 있는
    타입이 2개가 되어 case 안에 직접 쓰지 않고 메서드로 뺌. 파일 분리는 타입별 정책이 커질 때).
  - `PendingIntent`: 수락/거절은 action(`ACTION_FRIEND_ACCEPT`/`REJECT`)으로 구분, requestCode는 요청 id —
    같은 requestCode·같은 action이면 FLAG_UPDATE_CURRENT로 다른 요청의 extras가 덮인다.
  - 처리는 `NotificationActionReceiver.answerFriendRequest`(복약 완료와 같은 goAsync 패턴). 성공 시 `FriendEventBus` 발행.
    알림은 API 결과 전에 지워지므로 실패(이미 앱에서 처리한 요청 등)하면 토스트로 앱에서 확인하라고 안내.
- **다음(미착수): 알림 탭 → 화면 이동.** 복약 → 메인(홈), 친구 요청·수락 → 친구 목록, 공유 약속 → 약속 탭의 공유
  약속 화면. `MainActivity.EXTRA_TARGET_TAB` + `onNewIntent`가 이미 있다. 앱이 꺼진 상태에서 눌렀을 때 시작 화면이
  `Login`이라 로그인 확인 흐름과 이어붙이는 게 관건.
- **알림 id tag 분리도 함께 적용:** `notify(safeType, id, ...)`, 복약 완료 버튼 intent에 `notificationTag`를 실어
  `NotificationActionReceiver`가 `cancel(tag, id)`. 업데이트 전에 뜬 알림은 tag가 null이라 `cancel(id)`와 같게 동작.

- **상대 화면이 실제로 바뀌는 건 2가지뿐이다:** "요청 받음"(①새 요청, ④거절 이력 재시작 → 상대의 받은 요청
  배지)과 "친구 성립"(③자동 수락, `acceptFriendRequest` → 상대의 친구 목록). 나머지는 상대에게 보낼 게 없다.
  - ② 관계 재생성: `deleteFriend`가 **내 쪽 관계만** 지우므로(역방향이 남아 있으면 요청 기록도 유지) 상대 목록엔
    애초에 내가 남아 있다. 내 쪽 한 줄만 다시 생긴다.
  - 거절: 보낸 사람에게 "보낸 요청 목록" 화면이 없다(조회 API도 없음). 갱신할 화면이 없다.
  - 삭제: 위와 같은 이유로 상대 화면이 바뀌지 않는다.
- **이벤트는 하나 + kind:** `FriendshipChangedEvent(targetUserId, actorUserId, actorNickname, kind=REQUESTED/ACCEPTED)`.
  두 경우 모두 "한 명에게 알림+갱신"으로 하는 일이 같고 문구만 달라서. 택하지 않은 대안: 의미별 이벤트 2개(이름은
  분명하지만 리스너가 거의 같아 파일만 늘어남). 발송은 채팅과 같은 `AFTER_COMMIT` 리스너, `PushSender.sendMessage`.
- **FCM 타입 `FRIENDSHIP` 하나**, 알림 id는 `actorUserId`(같은 사람의 요청→수락 알림은 한 줄로 덮어씀).
- **앱:** `FriendEventBus`(ChatEventBus 복사). 택하지 않은 대안: topic 문자열을 싣는 버스 하나로 합치기(중복은
  사라지지만 약속 4화면·채팅 화면까지 수정 범위가 넓어짐). `FriendListFragment`가 구독해 친구 목록과 받은 요청
  배지를 재조회. 알림 채널은 `yaksok_friend_channel`로 분리(채팅 알림을 끌 때 친구 알림이 같이 꺼지지 않게).
- **알려진 한계:** 받은 요청 다이얼로그가 열려 있는 동안 새 요청이 와도 다이얼로그 목록은 그대로다(뒤의 배지만 갱신).
- **같이 발견한 문제:** 알림 id 공간이 타입끼리 공유된다. 복약 `notificationId`, 채팅 `roomId`, 친구 `actorUserId`가
  같은 숫자면 서로의 알림을 덮어쓴다. `notify(tag, id)`의 tag를 type으로 주면 분리된다(복약 완료 버튼의
  `cancel`도 같은 tag로 맞춰야 함).

## 45.5 채팅 알림 + 화면 실시간 갱신

- 채팅은 STOMP. 푸시는 앱이 백그라운드일 때, 화면 갱신은 STOMP 구독으로 이미 되는지 먼저 확인.
- **얽힌 부채:** STOMP 메시지 경로에 메시지 단위 인증이 없고 `sender`가 이메일(`tech-debt.md`,
  `completed/09-chat-room-split.md`). 푸시 수신자 결정 시 참여자(`ChatParticipant`: user/drugStore)
  기준으로 보내야 하고, 보낸 사람 본인과 **현재 그 방을 보고 있는 사용자는 제외**해야 중복 알림이 없다.

### 45.5 구현 (서버·앱 코드 적용됨, 실기기 확인 완료)

- **STOMP 대신 FCM 데이터 메시지로 목록 갱신 + 알림을 한 번에.** 목록 화면에는 STOMP 연결이 없어 B안
  (사용자별 토픽)은 연결 위치·생명주기·재연결에 더해 **STOMP 메시지 인증 부채 때문에 남의 토픽을 구독할 수
  있는 보안 문제**가 있고, 놓친 메시지는 결국 목록 API 재조회로 메워야 한다. 채팅방 안은 기존 STOMP 유지.
- **발송은 이벤트 + `AFTER_COMMIT`.** `ChatService.saveMessage`(`@Transactional`)가 `ChatMessageSavedEvent`
  (roomId, senderParticipantId, senderName, preview — 엔티티 아님)를 발행, `ChatMessageSavedListener`가
  커밋 후 수신자를 계산해 `PushSender.sendData`. 택하지 않은 대안: 컨트롤러에서 `saveMessage` 반환 후
  직접 호출(파일은 적지만 약속 쪽과 구조가 갈라짐), 서비스 안에서 바로 발송(커밋 전 푸시 문제).
- **수신자:** `findAllWithRefsByRoomId`(user·drugStore fetch join — 리스너는 영속성 컨텍스트 밖)로 방 참여자를
  가져와 보낸 사람 제외, 약국 참여자는 `findAllByMyDrugStore_Hpid`로 사장 유저들로 변환. 리스너는 예외를
  삼킨다(`AFTER_COMMIT` 예외가 호출자로 전파되면 이미 저장된 메시지의 STOMP 브로드캐스트가 막힐 수 있어서).
- **새 `FcmMessageType.CHAT`.** `REFRESH`는 앱이 알림 없이 return 하므로 쓸 수 없다. `notificationId`를 roomId로
  줘서 같은 방 알림은 한 줄로 갱신된다.
- **앱:** `MyFirebaseMessagingService`가 `CHAT`에서 `ChatEventBus`를 발행하고, `ChattingRoom.currentRoomId`와
  푸시 roomId가 같으면 알림만 생략. `showNotification`의 `CHAT` case가 없으면 default로 빠져 "약속 완료" 버튼이
  붙는다. 채널은 복약 알림과 분리(`yaksok_chat_channel`) — 안 그러면 복약 알림을 끌 때 채팅도 같이 꺼진다.
- **`currentRoomId`는 `onStart`/`onStop`에서 관리.** `onStop`은 내 방일 때만 비운다(방 A→B 이동 시 B.onStart가
  A.onStop보다 먼저 와서 무조건 비우면 B 값이 지워짐). `String` 타입인 이유: 푸시 roomId가 문자열이라
  `Long`과 `equals`하면 컴파일은 되지만 항상 false.
- **`ChatEventBus`는 `YaksokEventBus`와 별도.** 채팅이 올 때마다 약속 화면까지 재조회하지 않으려고. 인자를
  싣지 않는 3장 결정은 동일. 친구 요청(45.4)도 이 방식을 따를지는 거기서 결정.
- **알려진 한계:** 알림을 눌러도 채팅방으로 이동하지 않는다(`TAKEN_ALERT`도 `contentIntent` 없음). 알림 off
  사용자는 목록 갱신 신호도 못 받는다(`REFRESH`와 동일).

- 읽음 커서는 `active/42-chat-unread.md` 참고(목록 배지와 연동).
- **결정할 것:** "방을 보고 있음"을 서버가 어떻게 아는가(구독 상태 vs 앱이 신호 전송 vs 푸시는 항상 보내고
  앱이 포그라운드 방이면 무시).

## 45.6 카카오 로그인 — 기존 회원 통합 — 구현됨(33.14)

> 2026-11-09: 아래는 착수 전 메모다. 기존 회원 연동·연동 해제는 33.14에서 서버·앱 모두 끝났고, 이 큐에 남는 일은 없다.
> 계정 복구(아이디 찾기·비밀번호 재설정)의 카카오 회원 처리만 33.10에 남아 있다.

- `active/33-kakao-login.md`의 **2차**(기존 회원 연동, 계정 복구 정리). 1차는 이메일 충돌을 차단하고
  연동 안내만 한다. 이 문서의 설계를 정본으로 따르고, 스키마 변경(연동 컬럼 등)은 착수 전 질문.

## 45.7 AI 파싱 결과 캐시 (마지막)

- 이미 조회한 약은 캐시. 현재 처방전/약봉투 AI 분석은 앱이 Gemini를 직접 호출하고
  서버 경유가 없다(`completed/37-ai-analysis-in-app.md`) — 그래서 서버 캐시는 저장 지점이 없다.
- **결정할 것:** 캐시 대상이 (a) Gemini 결과(앱 로컬 캐시) 인지 (b) 약 이름→공공데이터 조회 결과
  (`medicine/` 서버 프록시, 서버 캐시) 인지. 37장이 "저장할 게 없어서 서버 제외"로 정했으므로,
  (b)가 자연스럽다. 새 의존성(Caffeine 등)은 착수 전 질문.
