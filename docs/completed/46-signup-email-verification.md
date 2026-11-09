# 46. 회원가입 이메일 인증

**상태:** 구현됨(백엔드·앱 컴파일, 실기기·서버 연동 확인 완료). 카카오 가입은 대상 아님(33장).

## 문제

`UserService.signUp()`이 이메일이 본인 것인지 확인하지 않았다. 남의 이메일로 가입할 수 있고, 그
이메일로 아이디 찾기·비밀번호 재설정이 엉뚱한 계정에 걸린다. 확인하려는 것은 존재 여부가 아니라
**소유**(그 주소의 주인이 지금 가입하는 사람인가)다.

## 설계: 일회용 토큰 방식

가입 API가 "이메일 인증 완료"를 믿는 방법으로 (a) 일회용 토큰을 택했다.

1. `POST /api/auth/signup/send-code` — 이미 가입된 이메일이면 거절, 아니면 코드 발급·메일 발송
2. `POST /api/auth/signup/verify` — 코드 검증 성공 시 **인증된 이메일에 묶인** 토큰 발급
3. 가입 요청(`UserRequest.emailVerifyToken`)에 토큰을 실어 보냄. `signUp()` 맨 앞에서
   `SignUpTokenStore.consume(token, email)`로 소비(검증을 모두 통과한 `save` 직전)하며 토큰의
   이메일과 요청 이메일을 대조

`auth`는 "소유 증명"(코드 발송·검증·토큰 발급)까지만 맡고, 토큰 소비와 가입 완료는 `user`의
`UserService.signUp()`이 맡는다. 두 영역을 잇는 것은 토큰 하나뿐이다.

### 변경된 클래스·메서드

- 백엔드: `EmailVerificationStore.Purpose.SIGN_UP`, `SignUpTokenStore`(`PasswordResetTokenStore`와
  같은 인메모리 1회용, 값이 `userId`가 아니라 **이메일**, TTL 10분), `SignUpVerificationService`
  (`sendCode`/`verify`), `AuthController`의 두 엔드포인트, `SecurityConfig.permitAll()` 두 경로,
  `UserRequest.emailVerifyToken`(`@NotBlank`), `UserService.signUp()`의 `save` 직전
- 앱: `VerificationCode`(아이디/비번 찾기와 공유하는 부품)를 가입 화면 2개(일반·약국)의 이메일 칸
  자리에 끼움. 부품에 `getEmail()`·`reset()`·`markVerified()`·`showEmailError()` 추가.
  `AuthApi.sendSignUpCode`/`verifySignUp`, `UserRequest`에 토큰 인자 추가

### 판단과 택하지 않은 대안

- **(b) 인증된 이메일을 TTL로 기억** — 1회용 소비가 없어 인증 한 번으로 여러 번 가입을 시도할 수
  있다.
- **`PasswordResetTokenStore` 일반화** — 값 타입이 `Long`→`String`으로 바뀌고 만료 메시지도 달라져
  기존 재설정 흐름을 건드린다. 40줄짜리 저장소를 하나 더 두는 쪽을 택했다.
- **Redis** — 새 의존성·서버가 필요하고, 코드·재설정 토큰·가입 토큰을 한꺼번에 옮겨야 일관된다.
  EC2 1대 전제라 인메모리 유지. 스케일아웃 때 세 저장소를 함께 이전(`tech-debt.md`).
- **`UserService`가 `SignUpVerificationService`를 거쳐 소비** — 전달만 하는 메서드가 늘 뿐이라
  `UserService`가 `SignUpTokenStore`를 직접 주입.
- **앱의 재요청 차단(글자를 바꿨다 되돌려 재요청)** — 서버가 이메일별 60초 재발송 제한을 걸고,
  거절 응답은 메일 발송 없이 가벼워서 막지 않음. 대량 요청은 앱이 아닌 서버 레이트리미팅의 영역.
- **토큰을 SharedPreferences에 저장** — 하지 않음. 화면 필드로만 보관해 화면을 나가면 인증을 다시
  받는다(안전한 쪽으로 실패).
- **이메일 입력 잠금(`setEnabled(false)`)** — 오타를 못 고치므로 택하지 않음.
- **이메일 칸을 고치는 즉시 토큰 폐기·화면 초기화** — 처음엔 `TextWatcher`로 구현했으나, 실수로
  한 글자만 건드려도 재인증이 필요해져 버렸다. 대신 **새 코드를 받는 순간**(발송/재발송 성공)
  `emailVerifyToken`을 null로 바꾸고, 가입 시점에 `verificationCode.getEmail()`이 인증받은
  `sentEmail`과 같은지 대조해 다르면 막는다. 인증 후에도 재발송 버튼을 남겨 두어 이메일을 바꾸면
  다시 받을 수 있다. 인증받은 이메일과 다른 주소로 가입되는 일은 이 대조와 서버의 토큰 이메일
  대조가 막는다.

### 알려진 한계

- 토큰 소비는 `signUp()`의 `save` 직전이다. 처음엔 맨 앞에서 소비했으나, 그러면 아이디 중복 같은
  검증 실패에도 토큰이 사라져 이메일 인증을 다시 받아야 해서 옮겼다. `send-code`가 이미 이메일
  가입 여부를 알려주므로 `existsByEmail` 노출을 이유로 앞에 둘 필요가 없다. 대신 토큰 없이도
  아이디 중복 여부는 가입 요청으로 확인된다(원래부터 가입 API가 알려주던 정보). 앱은 메시지에
  "이메일 인증"이 들어간 실패(토큰 만료·불일치)에만 인증 상태를 초기화한다. 소비 후 `save`
  실패(드문 DB 오류)는 토큰이 사라지는 채로 남는다.
- 인증코드 TTL을 시연용 1분(45.1)으로 두었고, 앱 `VerificationCode.EXPIRY_MS`도 1분으로 맞췄다
  (아이디/비번 찾기 화면도 함께 1분). 시연 후 서버 `EmailVerificationStore.TTL_MINUTES`와 앱
  `EXPIRY_MS`를 **함께** 5분으로 되돌려야 한다. 타이머 빨간색 임계는 10초로 낮춤.
- 가입 화면은 `VerificationCode`가 코드 입력칸을 처음부터 보여준다(아이디 찾기와 동일). 발송
  전에는 숨기는 변경은 하지 않았다.
- `send-code`는 레이트리미팅이 없다(`tech-debt.md`의 계정 복구 미결 항목과 같은 부채).
- 서버를 먼저 배포하면 기존 앱의 가입은 토큰이 없어 전부 400이 된다. 앱과 함께 배포.
