# 3. 복약 완료 시 화면 자동 갱신

**상태:** 완료 — 백엔드(3.6)·프론트엔드(3.7) 모두 코드에 반영되어 있고, 다기기에서 동작을 확인했다.

## 3.1 목표

알림의 "복약 완료" 버튼을 누르면, 그 데이터를 보여주는 화면(MainActivity 진행률, YaksokList,
ShareYaksokList)이 `onResume`을 기다리지 않고 **즉시 갱신**되게 한다.
**화면을 켜고 있는 액티비티만** 갱신되면 된다.

## 3.2 착수 후 발견한 것

설계를 코드에 맞춰보다가 3가지를 발견했고, 이것들이 3.3의 결정을 바꿨다.

### 소유자 본인의 다른 기기로 보내는 경로가 아예 없다

`notifyWatchers()`는 `shareYaksokRepository.findAllByYaksokId()`로 **공유받은 사람만** 순회한다.
소유자 본인의 다른 기기에는 아무것도 안 보낸다. "재활용하면 메시지 1건으로 끝난다"는
원래 판단의 전제가 성립하지 않는다 — 소유자용 발송은 어차피 새로 만들어야 한다.

### 복용 취소(`isTaken=false`)일 때 아무 신호도 안 나간다

```java
if(isTaken){ notifyWatchers(notification); }
```

진행률이 **줄어드는** 것도 화면 갱신 대상이다. 그런데 "복약 완료했어요" 문구를 가진
`TAKEN_ALERT`를 취소 시에 보낼 수는 없다.

### 알림 id가 REMINDER와 TAKEN_ALERT 사이에서 충돌한다

`showNotification()`이 서버 `Notification.id`를 안드로이드 알림 id로 그대로 쓰는데,
두 타입이 같은 id를 공유한다. 그래서 "다른 기기 트레이 정리"를 `cancel(notificationId)`로
구현하면 **공유자 기기에서 방금 띄운 TAKEN_ALERT를 지워버린다.** 도착 순서가 보장되지
않으므로 가끔만 재현되는 형태로 나타난다.

## 3.3 확정된 설계

### 갱신 신호는 `REFRESH` 타입을 신설한다

`title`/`body` 없이 `type`만 실은 data-only 메시지. 폰은 알림을 띄우지 않고 재조회만 한다.

**`TAKEN_ALERT` 재활용을 택하지 않은 이유:** 3.2의 앞 두 항목 때문에 결국
"문구 없는 TAKEN_ALERT"를 소유자용·취소용으로 따로 보내게 된다. 그건 이름만 TAKEN_ALERT지
실제로는 REFRESH다. 타입 이름과 실제 의미가 어긋나느니 분리하는 편이 낫다.

책임이 이렇게 갈린다.

| 타입          | 대상            | 눈에 보이는가 | 언제           |
| ------------- | --------------- | ------------- | -------------- |
| `REMINDER`    | 소유자          | O (버튼 2개)  | 복약 시간      |
| `TAKEN_ALERT` | 공유자          | O (버튼 없음) | 완료했을 때만  |
| `REFRESH`     | 소유자 + 공유자 | X             | 완료·취소 양쪽 |

### 발신 기기를 발송 대상에서 제외하지 않는다

완료를 누른 기기도 자기가 유발한 REFRESH를 받지만, 재조회 1회가 더 도는 것뿐이라 무해하다.
제외하려면 PATCH 요청에 자기 FCM 토큰을 실어야 해서 API와 클라이언트 양쪽이 바뀌고,
알림 버튼 경로에서는 `getToken()`을 비동기로 먼저 받아야 해서 완료 요청이 느려진다.
오히려 로컬 이벤트로 먼저 갱신된 뒤 서버 값으로 한 번 더 덮어쓰므로 정합성에는 유리하다.

### 커밋 후 발송은 `@TransactionalEventListener(AFTER_COMMIT)`

`YaksokService`는 이벤트만 발행하고, 발송은 별도 리스너가 커밋 이후에 한다.
서비스에서 FCM 의존성이 빠지고, 앞으로 다른 갱신 신호도 같은 방식으로 붙일 수 있다.
"커밋 전에 FCM이 나간다" 문제가 이걸로 함께 해결된다.

`TransactionSynchronizationManager`를 직접 등록하는 방법도 있었지만 서비스 코드에
트랜잭션 인프라 코드가 섞이고, 컨트롤러에서 발송하는 방법은 발송 대상 조회 로직이
컨트롤러로 새어나가서 택하지 않았다.

### 이벤트 버스는 인자를 싣지 않는다

`onYaksokDataChanged()` 하나만 두고, 받은 화면은 무조건 전체를 재조회한다.
3.5의 **원칙 1**을 끝까지 민 결과다.

세 화면 모두 인자가 필요 없다는 것을 확인했다.

| 화면            | 재조회 메서드            | 인자                               |
| --------------- | ------------------------ | ---------------------------------- |
| MainActivity    | `loadNotificationList()` | 전체를 받아 선택된 날짜로 필터링   |
| YaksokList      | `fetchYaksokList()`      | 유저의 약속 전체                   |
| ShareYaksokList | `loadSharedYaksokList()` | `senderId`를 필드로 이미 갖고 있음 |

**인자를 싣지 않은 결정적 이유:** 완료를 누른 기기가 즉시 갱신하려면
`NotificationActionReceiver`에서 로컬 이벤트를 발행해야 하는데, 이 리시버가 아는 것은
`notificationId` 하나뿐이다. `yaksokId`나 `date`를 채우려면 REMINDER FCM payload에
두 값을 추가하고 → PendingIntent extras로 넘기고 → 리시버가 꺼내는 경로를 전부
만들어야 한다. 아직 하지도 않을 최적화를 위해 세 군데를 더 건드리게 된다.

### 구독 화면은 3개

MainActivity, YaksokList, ShareYaksokList.

`ShareYaksokDetail`은 제외했다. 제목·기간·복용 시간·약 목록만 보여주고
**진행률이나 복약 상태를 표시하지 않아서** 갱신할 것이 없다.

### 다른 기기의 알림 트레이 정리는 하지 않는다

3.2에서 발견한 id 충돌 때문에 그대로 구현하면 버그가 된다. 그런데 검토해보니
**애초에 필요한 작업이 아니다.**

- 스케줄러는 `findByDateAndTimeAndIsTakenFalse`로 **아직 안 먹은 것만** 보낸다.
  이미 완료된 알림이 새로 발송되는 일은 없다.
- 낡은 알림을 눌러도 `updateNotificationStatus`의 `if(isTaken == notification.isTaken()) return`이
  중복 처리를 막는다. **정합성 문제가 아니라 "알림창이 지저분하다"는 것뿐이다.**

이번 작업의 본체는 화면 갱신이므로 범위에서 뺐다. 5장 기술 부채로 옮긴다.

## 3.4 FCM 코드를 `fcm/` 패키지로 분리

`FcmService`와 `FcmMessageType`이 `yaksok/service/`에 있었는데, 이 둘은
`List<String>`과 `Map`만 받고 약속 도메인을 전혀 모르는 순수 기술 코드다.
그 자리에 두면 나중에 채팅 푸시를 붙일 때 `chat/`이 `yaksok/`를 import하게 된다.

```
com/sehwan/YakSok/        ← 앱 루트 (@SpringBootApplication의 스캔 기준점)
├── chat/  config/  user/  yaksok/
└── fcm/                  ← yaksok의 하위가 아니라 형제
    ├── FcmService.java       토큰 리스트 + Map 만 안다
    ├── FcmMessageType.java
    └── PushSender.java       유저 id를 안다
```

**`notification/`이 아니라 `fcm/`인 이유:** `Notification` 엔티티(복약 알림 1건)가 이미 있어서
같은 단어가 두 뜻으로 쓰이면 계속 헷갈린다.

**루트 패키지 밖(`com.sehwan.fcm`)에 두면 안 되는 이유:** `@SpringBootApplication`이
자기 패키지 아래만 스캔하므로 빈으로 등록되지 않는다.

### `PushSender`를 만든 이유

이 3단계 절차가 `NotificationScheduler`와 `YaksokService.notifyWatchers()`에 똑같이 있었고,
새 리스너가 세 번째 복사본이 될 참이었다.

```java
List<String> tokens = userDeviceRepository.findAllByUserId(userId)   // ① 토큰 조회
        .stream().map(UserDevice::getToken).toList();
List<String> invalid = fcmService.sendMessage(tokens, ...);          // ② 발송
invalid.forEach(userDeviceRepository::deleteByToken);                // ③ 죽은 토큰 정리
```

③은 잊기 쉽고, 잊으면 죽은 토큰이 계속 쌓인다. `PushSender`가 유저 id 목록을 받아
①②③을 전부 처리하고, 호출부는 유저 id만 넘긴다.

**결합도는 오히려 낮아진다.** `UserDeviceRepository`를 아는 클래스가 3개에서 1개로 준다.
`FcmService`는 여전히 유저를 모르고, 약속 도메인 지식(누가 공유받았나, 문구를 뭐라고 쓰나)은
`yaksok/event/` 리스너 안에만 남는다.

여러 유저의 기기를 한 번에 가져오려고 `UserDeviceRepository.findAllByUserIdIn()`을 추가했다.
기존 `findAllByUserId()` 반복문은 공유자가 N명이면 쿼리를 N번 날린다.

### `PushSender.sendData()`가 통째로 트랜잭션인 이유

이상적으로는 마지막 삭제만 트랜잭션이어야 FCM 네트워크 대기 동안 DB 커넥션을 안 잡는다.
그런데 같은 클래스 안에서 메서드를 나누면 **자기 호출은 프록시를 안 거치므로
`@Transactional`이 먹지 않는다.** 제대로 쪼개려면 클래스를 하나 더 만들어야 한다.
지금 규모에서는 문제가 되지 않아 단순한 쪽을 택했다.

## 3.5 클라이언트 구조 — 이벤트 버스

`MyFirebaseMessagingService`는 Service라 액티비티 참조가 없다. FCM을 받은 곳에서 화면까지
신호를 전달할 지점이 필요해서 앱 전역 싱글턴 버스를 뒀다.
(`LocalBroadcastManager`는 deprecated)

```
서버 REFRESH ──▶ MyFirebaseMessagingService ──┐
                                              ├──▶ YaksokEventBus ──▶ 떠 있는 화면
NotificationActionReceiver (서버 응답 성공) ────┘
```

### `util/YaksokEventBus`

- `static final INSTANCE` + `private` 생성자. lazy 초기화를 쓰지 않은 이유는 두 스레드가
  동시에 `get()`을 부르면 버스가 2개 만들어질 수 있기 때문. 그러면 **구독한 버스와 발행한
  버스가 달라서 신호가 안 가고, 에러도 안 난다.** `static final`은 JVM이 클래스 초기화를
  한 번만·스레드 안전하게 보장한다.
- `Handler(Looper.getMainLooper())`로 콜백을 메인 스레드에 넘긴다.
  `onMessageReceived`는 백그라운드 스레드다.
- 구독자 목록은 `CopyOnWriteArrayList`. 발행(백그라운드)과 구독/해제(메인)가 다른
  스레드에서 일어나고, 콜백 도중 해제가 일어나도 안전해야 한다.

### 리스너를 단일 변수가 아니라 리스트로 둔 이유

"화면에 보이는 액티비티는 하나"라는 전제가 성립하지 않는다. 안드로이드는 A→B 전환 시
**`B.onStart()`를 `A.onStop()`보다 먼저** 실행한다.

```
A.onStart   → listener = A
B.onStart   → listener = B        (단일 변수면 A를 덮어씀)
A.onStop    → listener = null     (B를 지워버림)
```

`unsubscribe`는 "누가 나가는지" 파라미터를 받지만 **단일 변수에는 지울 대상을 지정할 수
없다.** 그래서 A가 나가면서 B의 구독을 날린다. 결과는 "전환 직후 보이는 화면이 구독되지
않은 상태"이고, 크래시도 로그도 없다. 화면을 옮길 때마다 재현된다.

`listeners.remove(listener)`는 그 객체만 뺀다. 멀티윈도우(분할 화면)에서 두 액티비티가
동시에 보이는 경우도 리스트라야 둘 다 갱신된다.

### `onStart` 구독 / `onStop` 해제

생명주기 짝(`onCreate↔onDestroy`, `onStart↔onStop`, `onResume↔onPause`)을 맞춰야 횟수가
맞는다. `onCreate`에서 구독하면 `onCreate`는 한 번뿐인데 `onStop`은 여러 번이라,
**두 번째로 화면에 들어온 순간부터 구독이 없다.**

`onResume`/`onPause`가 아닌 이유는 다이얼로그가 위에 뜨면 `onPause`만 불려서 뒤에 보이는
화면의 구독이 끊기기 때문. `onStart`/`onStop`은 "보이느냐" 기준이다.

이 짝 덕분에 **"떠 있는 화면만 갱신된다"가 판단 코드 없이 저절로 나온다.**

### 원칙 1 — 푸시 payload를 화면에 직접 반영하지 말 것

푸시는 "다시 불러와라"는 **트리거로만** 쓰고 실제 데이터는 서버에서 재조회한다.

payload를 적용한다는 것은 **과거의 모든 메시지가 정확히 한 번씩, 순서대로 도착했다고
가정**하는 것이다. FCM은 그중 무엇도 보장하지 않는다.

- **순서 뒤바뀜** — 같은 알림의 완료→취소가 역순으로 도착하면 화면에 "완료"가 남는다
- **유실** — +1을 놓치면 이후 신호가 아무리 정확해도 계속 1만큼 어긋난다

둘 다 **틀린 값이 화면에 눌러앉고 스스로 복구하지 못한다.** 재조회는 과거를 가정하지 않고
"지금 서버 상태"만 묻기 때문에, 순서가 꼬이거나 하나 유실돼도 다음 신호 하나에 맞춰진다.
대가는 요청 1회다.

그래서 `publish()`에 인자가 없고 서버 REFRESH도 `type` 하나만 보낸다.

### 원칙 2 — FCM data 메시지는 보장 배송이 아니다

앱 강제종료, Doze, 토큰 만료 시 안 온다. 푸시 갱신은 보조 수단이고
`onResume` 재조회가 최종 안전망이어야 한다.

### 로컬 즉시 발행은 알림 버튼 경로에만

`NotificationMultiViewAdapter`(체크박스)는 이미 낙관적 갱신 + 롤백을 하고, 그 코드가 도는
시점에 보이는 화면은 자기 자신뿐이다. 여기서 `publish()`를 부르면 자기 화면을 두 번 그린다.

`NotificationActionReceiver`는 액티비티 밖(BroadcastReceiver)이라 화면을 갱신할 방법이
버스밖에 없다. 알림창이 `MainActivity` 위에 떠 있을 수 있으므로 필요하다.

## 3.6 백엔드 (완료)

| 단계 | 내용                                                                                                  |
| ---- | ----------------------------------------------------------------------------------------------------- |
| 1    | `FcmMessageType`에 `REFRESH` 추가                                                                     |
| 2    | `FcmService`에 `sendData(tokens, Map)` 추출, `sendMessage`는 그걸 호출하는 껍데기로                   |
| 3    | `fcm/` 패키지로 이동 + `PushSender` 신설 + `findAllByUserIdIn` 추가 + `NotificationScheduler` 전환    |
| 4    | `yaksok/event/NotificationStatusChangedEvent` record 신설                                             |
| 5    | `YaksokService.updateNotificationStatus` — 이벤트 발행으로 교체, `notifyWatchers` 삭제, 필드 3개 정리 |
| 6    | `yaksok/event/NotificationStatusChangedListener` 신설                                                 |

### 밟기 쉬운 함정

**이벤트에 엔티티를 담으면 안 된다.** `AFTER_COMMIT` 시점에는 영속성 컨텍스트가 닫혀 있어서
`notification.getYaksok().getUser().getNickname()` 같은 lazy 접근이
`LazyInitializationException`으로 터진다. `spring.jpa.open-in-view` 기본값 때문에 웹 요청
안에서는 우연히 동작할 수 있으나 의존하면 안 된다. 그래서 record에 `ownerNickname`,
`yaksokTitle`, `timeCategory`, `date`를 **트랜잭션 안에서 미리 꺼낸 문자열로** 담는다.

**`AFTER_COMMIT`에는 트랜잭션이 없다.** 죽은 토큰 삭제는 쓰기라서
`@Transactional(propagation = REQUIRES_NEW)`가 필요하다. `REQUIRED`로는 이미 커밋된
트랜잭션에 얹혀 가서 **예외도 로그도 없이 쓰기가 버려진다.** `PushSender`가 이걸 짊어진다.

**같은 클래스 안에서 자기 메서드를 부르면 `@Transactional`이 안 먹는다.** 프록시를 안
거치기 때문. `PushSender.sendData()`를 "조회 / 발송 / 삭제"로 쪼개려다 이 함정에 걸린다.
쪼개려면 클래스를 하나 더 만들어야 해서, 지금은 메서드 전체를 트랜잭션으로 뒀다.
(`sendMessage → sendData` 자기 호출은 바깥에 이미 트랜잭션이 있어 무해)

**`@Transactional`은 Spring 것을 써야 한다.** `jakarta.transaction.Transactional`에는
`Propagation`이 없다.

**`@RequiredArgsConstructor`는 `final`인 필드만 생성자에 넣는다.** `final`을 빠뜨리면
컴파일은 되고 그 메서드를 부를 때만 NPE가 난다.

**`Map.of()`와 `putData()`에 null을 넣으면 안 된다.** `MulticastMessage.build()`의
`ImmutableMap.copyOf()`가 NPE를 던진다. REFRESH를 "title/body를 null로 넘기는" 방식이 아니라
"아예 안 넣는" 방식으로 만든 이유다.

**`sendEachForMulticast()`의 토큰 한도는 한 번에 500개다.** 지금 규모에선 문제없다.

## 3.7 프론트엔드 (완료)

| 단계 | 내용                                                                               |
| ---- | ---------------------------------------------------------------------------------- |
| 1    | `util/YaksokEventBus` 신설                                                         |
| 2    | `MyFirebaseMessagingService`에 REFRESH 분기 (알림 안 띄우고 `publish()` 후 return) |
| 3    | `NotificationActionReceiver` 성공 콜백에서 `publish()`                             |
| 4    | `MainActivity` / `YaksokList` / `ShareYaksokList` 구독 + 어댑터 재사용             |

### 어댑터를 재사용하게 바꾼 이유 (계획에 없던 것)

`MainActivity.setupRecyclerView()`가 호출될 때마다 **새 어댑터를 만들어
`setAdapter()`로 갈아끼우고 있었다.** 그러면 갱신할 때마다

- 스크롤 위치가 맨 위로 리셋된다
- 접어둔 섹션(`collapsedCategories`가 어댑터 필드)이 다시 펼쳐진다

발신 기기를 제외하지 않기로 했으므로 **체크박스를 누른 그 기기도 잠시 뒤 REFRESH를 받는다.**
즉 이 문제는 구독을 붙이는 순간 반드시 드러난다.

`NotificationMultiViewAdapter`에 `updateData()`를 추가하고 `MainActivity`는 어댑터가
`null`일 때만 생성하도록 바꿨다. 이 프로젝트의 다른 어댑터(`YaksokListAdapter`,
`ShareYaksokListAdapter`, `SharedUserAdapter`)는 이미 `updateData()` 패턴을 쓰고 있어서
방식도 통일됐다.

### `YaksokList`에 탭 상태 필드를 추가한 이유

`currentSenderId == null`이 "내 약속 탭"과 "공유 탭인데 공유자 미선택"을 둘 다 뜻해서
무엇을 다시 불러올지 정할 수 없었다. `isMyYaksokTab` 필드를 추가하고 `refreshCurrentTab()`이
세 경우(내 약속 / 공유자 선택됨 / 공유자 미선택)를 나눠 처리한다.

`onResume()`에도 `refreshCurrentTab()`을 넣어 노트 3.8의 안전망 문제를 해결했다.
앱 실행 시 `onCreate`의 `selectMyYaksokTab()`과 겹쳐 요청이 한 번 중복되지만,
실행할 때 한 번뿐이라 그대로 뒀다.

### 알림 버튼 경로를 `goAsync()`로 보호 (계획에 없던 것)

`BroadcastReceiver.onReceive()`는 즉시 리턴해야 하는 메서드고, **리턴하는 순간 시스템이
프로세스 보호를 거둔다.** 그런데 완료 요청은 `enqueue()` 비동기라 응답은 그 뒤에 온다.
앱이 백그라운드면 **응답 콜백이 아예 안 불릴 수 있다.**

이 앱에서 특히 나빴던 이유는 `onReceive` 앞부분이 **서버 결과와 무관하게 알림창을 먼저
지우기** 때문이다.

```
[약속 완료] 누름 → 알림창 사라짐 → 프로세스 정리 → 서버 반영 안 됨
                                              → Toast도 publish()도 안 됨
```

사용자는 알림이 사라졌으니 됐다고 생각하는데 **복약 기록이 조용히 누락되고, 알림이
사라져서 다시 누를 수도 없다.** 2장의 "알림이 소리 없이 사라지는" 실패 모드와 같은 성질이다.
그리고 **알림을 누르는 상황은 대부분 앱이 백그라운드일 때**라, 가장 흔한 경로가 가장
위험한 경로였다.

`goAsync()`로 `PendingResult`를 받아두고 응답 콜백에서 `finish()`를 부르면, 그때까지
프로세스가 높은 우선순위를 유지한다. `finish()`는 성공·실패 어느 쪽으로 빠지든 반드시
불려야 하므로 **`try { ... } finally { pendingResult.finish(); }`** 로 감쌌다.

- `ACTION_SNOOZE`에는 붙이지 않았다. 비동기 작업이 없어 `onReceive` 안에서 끝난다.
- 예산은 일반 브로드캐스트 기준 약 60초. `NetworkClient`의 타임아웃이 15초라 여유가 있다.
- `PendingResult`는 `BroadcastReceiver`의 내부 클래스라 import가 필요 없다.

**더 확실한 방법은 `WorkManager`**다. 요청을 저장해뒀다가 실패 시 재시도하고 재부팅도
견딘다. 지금 규모에는 과하다고 판단해 `goAsync()`를 택했다. "완료 요청이 반드시 서버에
도달해야 한다"가 요구사항이 되면 그때 올린다.

### 패키지 구조 정리

액티비티들을 `ui/` 하위 화면별 패키지로 옮겼다
(`ui/main`, `ui/yaksok`, `ui/sharedYaksok`, `ui/setting` 등).

## 3.8 남은 것

- ~~`YaksokList.onResume()`이 데이터를 재조회하지 않는다~~ → 3.7에서 해결.
- ~~`notifyWatchers()`가 `@Transactional` 안에서 발송한다~~ → 3.3의 `AFTER_COMMIT` 결정으로 해결.
- ~~다른 기기의 알림 트레이 정리~~ → 3.3에서 범위 제외. 5장으로 옮김.
