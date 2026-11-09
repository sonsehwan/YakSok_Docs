# 2. FCM 토큰 다기기 지원 (2026-08-31)

**상태:** 구현 완료 — 아래 내용은 모두 코드에 반영되어 있다.

## 2.1 발단

원래 하려던 것은 **"알림에서 복약 완료를 누르면 그 데이터를 보여주는 화면이 즉시 갱신되게"** 하는 기능이었다.
설계를 검토하다가 **선행 조건이 성립하지 않는다**는 것을 발견해서, 그 기반 작업을 먼저 하게 됐다.

> "완료를 누르면 각 기기의 화면이 갱신되게 하자"
> → 그런데 서버가 **다른 기기에 신호를 보낼 방법 자체가 없었다.** 토큰을 하나만 저장하고 있었기 때문.

## 2.2 발견한 문제

### 문제 1 — 다기기 로그인 시 알림이 조용히 사라짐 (가장 심각)

`User` 엔티티가 `fcmToken`을 **단일 String 컬럼**으로 갖고 있었고, `updateToken()`이 그걸 덮어썼다.

```
기기 A 로그인 → users[7].fcm_token = "A토큰"
기기 B 로그인 → users[7].fcm_token = "B토큰"   (A토큰은 DB에서 사라짐)
→ 기기 A는 이제 아무 푸시도 못 받음
```

다기기 로그인을 막고 있지 않은데도 나중에 로그인한 기기가 토큰을 덮어써서,
**먼저 로그인해둔 기기는 복약 리마인더조차 받지 못했다.** 사용자는 알 방법이 없다.
복약 관리 앱에서 알림이 소리 없이 누락되는 것은 가장 위험한 실패 모드.

### 문제 2 — 로그아웃 실패 시 두 유저가 같은 토큰을 소유

로그아웃 시 서버에 `fcmToken = null`을 PATCH해서 컬럼을 비우는 방식이었는데,
이 요청은 `enqueue()`로 던지고 실패해도 로그만 찍는 fire-and-forget이었다.
게다가 직후 `finishAffinity()`로 앱이 종료된다.

```
A 로그아웃 → 요청 유실 → users[A].fcm_token 은 "T" 그대로
같은 폰에서 B 로그인 → users[B].fcm_token = "T"
→ A와 B가 같은 토큰을 소유
→ A의 복약 알림이 B가 쓰는 폰으로 날아감
```

### 문제 3 — 로그아웃 버튼이 확인 전에 실행됨

```java
ivLogout.setOnClickListener(v -> {
    showLogOutDialog();   // 다이얼로그를 띄우기만 하고 즉시 리턴
    deleteToken();        // 사용자가 답하기 전에 실행됨
});
```

**로그아웃 아이콘을 눌렀다가 "취소"를 눌러도 서버 토큰이 이미 삭제돼서**,
다음 로그인 전까지 알림을 받지 못하는 상태가 됐다.

## 2.3 검토했지만 택하지 않은 방향

### "그냥 동시 로그인을 막으면 안 되나?"

- **진짜 인증 기반 차단은 더 어렵다.** `SecurityConfig`가 `permitAll()`이고 JWT도 세션도 없어서,
  서버가 기기 A의 요청과 B의 요청을 **구분할 수단이 아예 없다.** 인증 계층부터 만들어야 한다.
- **싼 대안은 있었다.** 로그인 시 옛 토큰으로 `FORCE_LOGOUT` 푸시를 쏴서 이전 기기를 튕겨내는 방식.
  스키마 변경 없이 가능하고 1:N보다 싸다.
- 폰+태블릿 동시 사용을 실제 기능으로 남기고 싶어서 **다기기 지원을 택했다.**

## 2.4 해결 설계

### 토큰 저장을 1:N으로

`user_devices` 테이블 신설. `user_id : token = 1:N`.

```java
@Table(name = "user_devices",
       uniqueConstraints = @UniqueConstraint(name = "uk_user_devices_token", columnNames = "token"))
```

`token`에 **unique 제약**을 건 이유: FCM 토큰은 "기기 하나 + 앱 설치본 하나"에 발급되므로
전역에서 유일해야 한다. 서로 다른 두 폰이 같은 토큰을 갖는 일은 없다.

참고로 **토큰은 로그인할 때마다 새로 발급되지 않는다.** `FirebaseMessaging.getToken()`은
캐시된 값을 돌려주는 getter이고, 앱 재설치·데이터 삭제·기기 복원 때만 바뀐다.
그래서 같은 폰에서 A가 로그아웃하고 B가 로그인해도 **토큰은 동일하다.**

### 로그인 시 upsert + 재배정 (설계의 핵심)

```java
userDeviceRepository.findByToken(token).ifPresentOrElse(
        device -> {
            // 이미 등록된 토큰이면 새로 만들지 말고 현재 로그인한 유저에게 재배정
            device.setUser(user);
            device.setUpdatedAt(LocalDateTime.now());
        },
        () -> userDeviceRepository.save(UserDevice.builder().user(user).token(token).build())
);
```

**왜 create만으로는 안 되는가:**
로그아웃 시 행을 지우니까 create만 있으면 될 것 같지만, 로그아웃 삭제는 best-effort다
(요청 유실, 로그아웃 없이 앱 삭제, 기기 초기화 등). 행이 남은 상태에서 create를 시도하면
unique 위반 예외가 나고, 클라이언트는 로그만 찍으므로 **그 기기는 영영 등록되지 않는다.**
게다가 옛 소유자의 행이 살아 있어 그 사람의 알림이 이 폰으로 계속 온다. 복구 방법이 없다.

**불변식:** _이 토큰(=이 기기)은 가장 최근에 여기서 로그인한 유저의 것이다._
로그인이라는 행위 자체가 "지금 이 기기를 쓰는 사람은 나"라는 증거이므로,
재배정 한 번으로 이전 유저의 정리와 새 유저의 등록이 동시에 끝난다.

**일반 원칙:** 프로세스가 죽는 순간에 나가는 best-effort 네트워크 호출에 정합성을 의존하면 안 된다.
대신 **"다음번 성공하는 작업이 상태를 고친다"**로 설계한다.

단, 재배정의 복구 시점은 **"그 기기에서 다음 로그인이 일어날 때"**다.
아무도 다시 로그인하지 않으면 복구되지 않으므로, 로그아웃 시 삭제는 여전히 필요하다.

### 기기 단위 로그아웃

`null`은 "어느 기기를 뺄지"를 말해주지 못하므로 별도 엔드포인트로 분리했다.

- `DELETE /api/users/{id}/fcm-token` — body에 자기 토큰을 실어 보냄
- `deleteByUserIdAndToken(userId, token)` — 조건에 `userId`를 함께 넣어서,
  이미 다른 유저에게 재배정된 행을 **뒤늦게 도착한 로그아웃 요청이 지우지 않도록** 함
- Retrofit의 `@DELETE`는 `@Body`와 함께 쓸 수 없어서 `@HTTP(method="DELETE", hasBody=true)` 사용.
  쿼리스트링에 실으면 액세스 로그에 토큰이 남기 때문에 body를 택함

### 다기기 발송

`FcmService`를 `sendEachForMulticast()` 기반으로 전환. 응답이 `UNREGISTERED` / `INVALID_ARGUMENT`인
토큰은 행을 삭제해 죽은 토큰이 쌓이지 않게 했다. (일시적 서버 오류로 지우면 그 기기가
알림을 영영 못 받으므로, 이 두 코드일 때만 정리)

## 2.5 변경 범위

**백엔드**

- 신규: `UserDevice`, `UserDeviceRepository`
- 수정: `UserService`(upsert/삭제), `UserController`(DELETE 엔드포인트), `FcmService`(멀티캐스트),
  `NotificationScheduler`(리마인더 다기기 발송), `YaksokService.notifyWatchers()`(공유자 알림 다기기 발송)
- 제거: `User.fcmToken`, `UserRequest.fcmToken`(회원가입 경로에서 값이 채워진 적 없는 죽은 필드였음)

**프론트엔드**

- `UserApi.deleteFcmToken()` 추가
- `Settings.deleteToken()` — `getToken()`으로 자기 토큰 조회 후 DELETE 전송,
  호출 위치를 다이얼로그 확인 콜백으로 이동
