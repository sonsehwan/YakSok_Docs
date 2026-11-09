# 10. 아이디 찾기 / 비밀번호 찾기 (2026-09-07 설계 → 2026-09-15 완료)

**상태:** 완료(2026-09-15). 로컬 검증 단계 없이 EC2 운영 서버에 바로 배포해 확인하는 방식이라,
`login_id` 컬럼 마이그레이션·`MAIL_USERNAME`/`MAIL_PASSWORD`(Gmail 앱 비밀번호) 설정·5경로
(find-id 2, reset-password 3) 전부 운영 환경에서 직접 검증 완료 — 아이디 찾기·비번 재설정
인증코드 수신·확인·새 비번 설정까지 확인됨.
선행이던 `AuthInterceptor` Bearer 공백 버그는 8.11에서 이미 수정됨.

## 10.1 문제 정의

로그인이 email + password다. **아이디라는 개념이 따로 없어서** 아이디 찾기가 성립하지 않고,
비밀번호를 잊으면 복구 경로가 전혀 없다. (6.1에서 "결정할 것"으로 남겨뒀던 항목)

비밀번호는 BCrypt 해시 저장(8.4)이라 원문 "찾기"는 불가능하다 —
**재설정(임시 비밀번호 발급)**만 가능하다.

## 10.2 설계 방향

- `User`에 `loginId` 추가. **로그인은 `loginId`로** 한다.
- `email`은 로그인 자격에서 빠지고 **본인 확인 + 임시 비밀번호 수신** 수단이 된다.
- 아이디 찾기·비밀번호 재설정 **둘 다** 이메일 인증코드(6자리, TTL 5분, 시도 5회)를 통과해야
  진행된다.

### 확정한 선택과 근거

| 항목 | 결정 | 근거 |
| --- | --- | --- |
| 인증코드 저장 | 인메모리 `ConcurrentHashMap` | 수명 5분, 유실돼도 재요청하면 그만, 다른 곳에서 조회 안 함. DB/Redis는 서버 인스턴스가 2대 이상이거나 재시작 중 유지가 필요할 때나 의미. 현재 EC2 1인스턴스 |
| 메일 발송 | Gmail SMTP + 앱 비밀번호 | 발신 전용 계정. 수신 도메인은 무관(naver 등 그대로). 무료, 일 500통 |
| 아이디 찾기 결과 | 앱 화면에 바로 표시 | 코드 인증으로 본인 확인이 끝났으므로 노출해도 됨 |
| 비번 재설정 본인확인 | loginId + email 일치 **+ 이메일 인증코드** | 코드 없이 loginId+email만으로 하면 제3자가 피해자 비번을 임의로 임시값 리셋 가능(탈취는 아님, 임시비번은 메일로만 감 — 성가심/방해). 코드를 요구하면 피해자 메일함이 있어야 완료됨 |
| 재설정 성공 시 | 그 유저의 `RefreshToken` 전부 삭제 | "비번 바뀌면 기존 로그인 끊김"은 자연스러운 동작. 다른 기기는 액세스 토큰 만료(10.6 이후 ≤10분) 후 기존 401→재발급 실패→강제 로그아웃 경로(8.11)로 튕김 |
| loginId 유니크 제약 | `@Column(unique = true, length = 20)` | User.java가 email·nickname에 쓰는 스타일. 8.10의 `@Table(uniqueConstraints)` 컨벤션과 다르나 파일 내 일관성 우선 |
| loginId 형식 검증 | 백엔드에도 `^[a-z0-9]{8,20}$` 추가 (특수문자 제외·8자 하한 — 최초 계획 `^[a-z0-9_-]{5,20}$`에서 축소) | 기존 email·nickname은 프론트 `InputView`만 검증하나, 이참에 백엔드에도 넣어 악성 클라이언트 대비. email·nickname도 함께 검증 추가 |
| 검증 구현 방식 | `jakarta.validation` 애노테이션(`@NotBlank`/`@Pattern`/`@Email`) + 컨트롤러 `@Valid` | 서비스 수동 `matches()`보다 선언적. `starter-validation`이 이미 의존성에 있음. 실패 시 `MethodArgumentNotValidException` → `GlobalExceptionHandler`에 핸들러 신설(첫 필드 에러 메시지를 `ApiResponse.error(400,"SIGNUP_FAIL",…)` 모양으로) — 안 그러면 Spring 기본 에러 바디가 나가 프론트 `contains("아이디")` 분기가 죽음 |

### 검토했지만 택하지 않은 것

- **비번 재설정에 코드 인증 없이 loginId+email 일치만** — 초안이었으나 위 표의 이유로 코드
  인증 추가.
- **인증코드 DB/Redis 저장** — 위 표 참고. 지금 규모에 불필요.
- **강제 로그아웃 시 기기 행(`user_devices`)까지 삭제** — 다른 기기가 즉시 푸시를 못 받게 하려면
  필요하나, `RefreshToken`만 지워도 ≤10분 후 세션이 끊긴다. 즉시성이 필요하면 FCM
  `FORCE_LOGOUT` data 메시지 방식이 더 맞다(10.8). 지금은 `RefreshToken` 삭제까지만.

## 10.3 loginId 도입

**상태:** 완료. 백엔드 코드 반영(2026-09-07), `compileJava` 통과.
운영 DB에 `login_id` 마이그레이션 적용 + 배포 후 검증 완료(로컬 재생성 단계는 건너뜀 — EC2
운영 서버에 바로 배포해 확인하는 워크플로).

**기계적 변경:**

| 파일 | 변경 |
| --- | --- |
| `user/entity/User` | `loginId` (`@Column(nullable = false, unique = true, length = 20)`) — `email` 필드 바로 아래 |
| `user/repository/UserRepository` | `findByLoginId`, `existsByLoginId` |
| `user/dto/request/UserRequest` | `loginId` 필드 + `@NotBlank`/`@Pattern("^[a-z0-9]{8,20}$")` + `toEntity()` 매핑. email(`@Email`)·nickname(`@Pattern("^[a-zA-Z0-9가-힣]{2,10}$")`)도 검증 추가 |
| `user/dto/request/LoginRequest` | `email` → `loginId` (`@Pattern` 안 붙임 — 형식 오류도 "존재하지 않는 사용자"로 수렴) |
| `user/controller/UserController.signUp()` | `@Valid` 추가 (login엔 안 붙임) |
| `common/exception/GlobalExceptionHandler` | `MethodArgumentNotValidException` 핸들러 신설 |
| `user/service/UserService.signUp()` | `existsByLoginId` 중복 검사 (기존 email 검사와 동형) |
| `user/service/UserService.login()` | `findByEmail` → `findByLoginId` |
| `config/DataInitializer` | 목 유저 4명에 `loginId` (`testuser1`~`testuser4` — `user1`은 8자 하한 미달) |

**밟기 쉬운 함정 — DB 마이그레이션:**
`ddl-auto: update`는 기존 행이 있는 `users`에 `NOT NULL UNIQUE` 컬럼을 못 넣는다(4.2에서 이미
밟은 함정과 동류). 로컬은 `ddl-auto: create`로 1회 재생성(데이터 소실) 후 `update`로 복귀.
운영 DB는 배포 시 `ALTER TABLE users ADD COLUMN login_id ...` + 백필 수동.

**로그인 JSON 키 변경:** `LoginRequest`의 `email`→`loginId`는 프론트가 보내는 JSON 키도
바뀌어야 한다. 백엔드만 바꾸면 앱 로그인이 깨지므로 프론트 10.5(3-A)와 함께 진행하거나
중간에 Postman으로 검증한다.

## 10.4 백엔드 — 메일 · 인증코드 · 복구

**상태:** 완료. 코드 반영(2026-09-07), 커밋됨. `MAIL_USERNAME`/`MAIL_PASSWORD`(Gmail 앱
비밀번호) 배포 시크릿으로 주입 완료. 5경로(find-id 2, reset-password 3) 운영 환경 검증 완료 —
인증코드 수신·확인·새 비번 설정까지 확인됨.

**계획과 달라진 점:**
- 로컬 시크릿: `spring.config.import: optional:application-local.yaml` 방식.
  `application-local.yaml`은 `.gitignore`에 추가. IDE 환경변수도 대안.
- 임시 비밀번호: 영문 대/소·숫자·특수(`!@#$%^&*`) **각 1개 이상 보장 + 나머지 랜덤, 총 12자**,
  `Collections.shuffle`로 섞음. `SecureRandom` 사용.
- 메일 발송: **인증코드 2건은 비동기**(`MailService.sendHtmlAsync`, `@Async("mailExecutor")` —
  `config/AsyncConfig`의 전용 스레드풀). `issue()`만 끝나면 즉시 200 반환 → `send-code` 응답
  지연(SMTP 1~3초) 제거. 없는 이메일은 SMTP 전 검사라 여전히 빠르게 400.
  **임시비번 1건은 동기**(`sendHtml`) — 발송 실패 시 트랜잭션 롤백이 필요하므로.
- 메일 본문은 **HTML**(`MimeMessage` + `MimeMessageHelper`, UTF-8). `mail/MailTemplate`가
  브랜드 셸 + 코드/임시비번 강조 박스 생성. code·임시비번은 HTML 특수문자 없어 이스케이프 불필요.
  `sendHtml` 실패는 `IllegalStateException`으로 래핑 → 롤백·400. 비동기는 `log.error`만.
- `application.yaml`: SMTP `connectiontimeout`/`timeout`/`writetimeout` 5000 (멈춘 연결이
  `mail-` 스레드 영구 점유 방지).
- 복구 4개 엔드포인트에 `@Valid` + DTO 검증 애노테이션. 실패는 10.3에서 만든
  공용 `MethodArgumentNotValidException` 핸들러가 처리(코드 `INVALID_INPUT`).
- `verifyResetPw`는 `@Transactional` — 비번 인코딩·저장 → 메일 발송 → `RefreshToken` 삭제 순.
  **메일 발송이 던지면 롤백**되어 비번 변경이 취소됨(사용자가 임시비번을 못 받았는데 기존 비번이
  바뀌는 상황 방지). 단 `verificationStore`는 인메모리라 롤백 안 됨 — 코드 재요청 필요.
- enumeration: 없는 email/loginId에 명확한 에러 유지(10.8과 일치, 실사용 전환 시 통일).

**DTO 위치:** `auth/dto/request/` (`FindIdSendCodeRequest`, `FindIdVerifyRequest`,
`ResetPwSendCodeRequest`, `ResetPwVerifyRequest`), `auth/dto/response/` (`FindIdResponse`).

### 메일 인프라 — `com/sehwan/YakSok/mail/`

fcm/, auth/와 같은 기준 — User 도메인을 모르는 순수 기술 코드는 user/ 밖.

- `MailService.sendHtml(to, subject, html)` 동기 / `sendHtmlAsync(...)` 비동기 — `JavaMailSender`
  (`MimeMessage`) 래핑. `MailTemplate`이 HTML 본문 생성
- `config/AsyncConfig` — `@EnableAsync` + `mailExecutor`(core 2 / max 4 / queue 50)
- `build.gradle`: `spring-boot-starter-mail`
- `application.yaml`: `spring.mail.*`, 계정·비밀번호는 `${MAIL_USERNAME}` / `${MAIL_PASSWORD}`
  (8.9 패턴). 평문 커밋 금지 — 로컬은 `application-local.yaml`(gitignore) 또는 IDE 환경변수
- `deploy.yml`: GitHub Secrets 주입

### 인증코드 저장소 — `auth/EmailVerificationStore`

- `enum Purpose { FIND_ID, RESET_PW }` — 키를 `(Purpose, email)`로 잡아, 같은 이메일이 두 흐름을
  동시에 타도 코드가 안 섞이게
- `Entry(code, expiresAt, attempts, lastSentAt)`
- `issue(purpose, email)` — 6자리 숫자, TTL 5분, **직전 발송 60초 내면 거부**, 호출 시 만료
  엔트리 함께 정리
- `verify(purpose, email, code)` — 만료·불일치·시도 5회 초과 체크, 성공 시 제거

### 복구 서비스·컨트롤러 — `auth/`

- `auth/service/AccountRecoveryService` — `UserRepository` · `EmailVerificationStore` ·
  `MailService` · `PasswordEncoder` · `RefreshTokenRepository`를 순서대로 엮는다.
  `UserService`에 안 넣는 이유: 이미 크고(signup~FCM토큰), 메일·인증코드 의존성이 복구에서만
  쓰인다(8.5의 관심사 분리 방향).
  - `sendFindIdCode(email)` / `verifyFindId(email, code) → loginId`
  - `sendResetPwCode(loginId, email)` — `findByLoginId` 후 email 일치 확인 → `issue` + 메일
  - `verifyResetPw(loginId, email, code)` — loginId+email 재확인 + `verify` 성공 →
    임시 비밀번호(영문+숫자+특수 10자) 생성 → 해시 저장 → 메일 발송 →
    `refreshTokenRepository.deleteByUser_Id(userId)`
- `user/repository/RefreshTokenRepository.deleteByUser_Id(Long)` 신규
- `auth/controller/AccountRecoveryController` (`@RestController`, `/api/auth`) — AuthController
  (토큰 수명 담당)와 분리

| 메서드 | 경로 | body | 응답 |
| --- | --- | --- | --- |
| POST | `/api/auth/find-id/send-code` | `{email}` | 200 |
| POST | `/api/auth/find-id/verify` | `{email, code}` | `{loginId}` |
| POST | `/api/auth/reset-password/send-code` | `{loginId, email}` | 200 |
| POST | `/api/auth/reset-password/verify` | `{loginId, email, code}` | `{resetToken}` (10.4a) |
| POST | `/api/auth/reset-password/confirm` | `{resetToken, newPassword}` | 200 (10.4a) |

- `SecurityConfig` 화이트리스트에 위 4경로 추가 (8.8이 화이트리스트 방식이라 명시 안 하면
  자동 차단됨)

### 10.4a 비밀번호 재설정 흐름 재설계 — 임시 비밀번호 폐기 (2026-09-09)

**바꾼 이유:** 임시 비밀번호를 메일로 보내면 사용자가 그걸 **그대로 계속 쓸 수 있어** "재설정"이
반쪽만 됐다. 코드 인증 통과 후 **바로 새 비밀번호를 직접 설정**하게 한다.

**흐름:** `reset-password/verify` 성공 → 서버가 **1회용 재설정 토큰** 발급(메일 발송 없음) →
앱이 `ResetPassword` 화면으로 이동(토큰 전달) → 새 비밀번호 입력 → `reset-password/confirm
{resetToken, newPassword}` → 서버가 토큰 검증 후 비번 교체 + `RefreshToken` 전부 삭제.

**신규 `auth/PasswordResetTokenStore`** — `EmailVerificationStore`와 같은 인메모리 방식.
`ConcurrentHashMap<token, (userId, expiresAt)>`, `issue(userId)→UUID`, `consume(token)→userId`
(1회용, 만료 시 예외), TTL 3분. "방금 이메일 인증을 통과했다"를 로그인 전 상태에서 짧게
증명하는 용도라 인메모리로 충분(재시작·스케일아웃 한계는 인증코드와 동일 — 10.8).

**엔드포인트 변화:**
- `reset-password/verify` 응답: `200` → `{resetToken}` (`ResetPwVerifyResponse`)
- `reset-password/confirm` 신규 — `{resetToken, newPassword}`. `newPassword`에
  `@Pattern`(`^(?=.*[a-zA-Z])(?=.*\d)(?=.*[!@#$%^&*()_+])...{8,16}$`, `ModifyPassword` 프론트와 동일)
- `SecurityConfig` 화이트리스트에 `/api/auth/reset-password/confirm` 추가

**`AccountRecoveryService` 변화:**
- `verifyResetPw` — 임시비번 생성·해시저장·메일발송·`RefreshToken` 삭제를 전부 제거,
  코드 검증 후 `passwordResetTokenStore.issue(userId)` 반환. `@Transactional` 뗌(DB 쓰기 없음)
- `confirmResetPw(resetToken, newPassword)` 신규 — `@Transactional`, 토큰 consume →
  `user.setPassword(encode)` → `refreshTokenRepository.deleteByUser_Id`
- `generateTempPassword`·`pick`·`TEMP_PW_*`·`SecureRandom` 제거. `MailTemplate.tempPassword`는
  안 쓰이나 파일은 남겨둠(무해)
- 10.4의 "임시비번 1건은 동기 발송, 실패 시 롤백" 판단은 이제 해당 없음 (임시비번 자체가 없음)

**설계 판단**

- **왜 토큰을 쓰나 (confirm에 loginId+email+code를 다시 안 보내고).** `verify`의
  `verificationStore.verify()`가 성공 시 코드를 **소비(제거)**하므로 confirm에서 그 코드를 다시
  못 쓴다. "방금 이메일 인증을 통과했다"는 짧은 증표를 한 번 주고받는 게, 코드 검증을 두 번
  하거나 인증코드의 수명을 늘려 잡는 것보다 깔끔하다.
- **토큰 검증 = 맵 키 조회.** 토큰은 서버가 만든 랜덤 `UUID`라, 앱이 보낸 값과 무언가를
  대조하는 게 아니라 `ConcurrentHashMap.remove(token)`이 저장된 키들과 `hashCode`+`equals`로
  비교해 "서버가 발급했고 + 3분 안 지났나"를 본다. 이점은 값이 **예측 불가·1회용·단기(3분)**
  라는 것. 토큰만으로는 비번 변경만 가능하고 세션/로그인 탈취는 아니다.
- **`verifyResetPw`에 `@Transactional`이 없는 이유.** DB 쓰기가 없다 — `findByLoginId`(읽기 1)
  + `verificationStore.verify`(인메모리) + `passwordResetTokenStore.issue`(인메모리)뿐이라
  커밋/롤백할 게 없다. 붙이면 프록시·트랜잭션 오버헤드만 는다. (옛 버전엔 `@Transactional`이
  있었는데, 그때는 `user.setPassword(임시비번)`이라는 DB 쓰기가 이 메서드에 있었기 때문 —
  그 쓰기를 `confirmResetPw`로 옮기며 애노테이션도 같이 옮겼다.)
- **`confirmResetPw`에 `@Transactional`이 필요한 이유.** ① `user.setPassword()`는 영속 엔티티의
  더티 체킹 UPDATE라 영속성 컨텍스트가 열려 있어야 flush 된다(트랜잭션 없으면 detached라
  DB에 안 감). ② `setPassword`(UPDATE)와 `deleteByUser_Id`(DELETE) 두 쓰기가 원자적이어야
  한다 — 한쪽만 성공하면 "새 비번인데 옛 세션이 살아있는" 상태가 된다.
- **인메모리 스토어는 트랜잭션 밖이라는 한계.** `confirmResetPw`가 `consume()`으로 토큰을 지운
  뒤 DB 단계에서 롤백돼도 토큰은 이미 사라져 되돌릴 수 없다 → 처음부터 다시. `verify`쪽 인증코드도
  같은 성질(10.4). `@Transactional`은 DB 부분만 묶는다.

### 10.5 프론트엔드

### 로그인 (3-A) — 완료(2026-09-07)

- `InputView.java`: `idPattern` `^[a-z0-9_-]{5,20}$` → `^[a-z0-9]{8,20}$`, 에러 메시지도 갱신
  (백엔드 `@Pattern`과 일치). `nicknamePattern`은 이미 백엔드와 같아 유지
- `activity_login.xml`: `input_email` **자리를** `input_login_id`로 교체
  (`validationType="id"`, 힌트/아이콘 `ic_user`, `inputType="text"`)
- `model/request/LoginRequest`: `email` → `loginId`. **Lombok화**(`@Getter @AllArgsConstructor`, setter 제거)
- `ui/login/Login`: `inputEmail`→`inputLoginId` 전면 교체, 에러 분기 `"이메일"`→`"아이디"`
- 로그인 버튼 아래 "아이디 찾기 · 비밀번호 찾기" 링크 2개 → **3-C/3-D와 함께** (액티비티가 그때 생김)

### 회원가입 (3-B) — 완료(2026-09-07)

- `activity_sign_up_normal.xml` · `activity_sign_up_drugstore.xml`: `imageView`와 `input_email`
  사이에 `input_login_id`(`validationType="id"`) 삽입, `input_email` 제약 대상 재연결.
  **email 필드는 유지** (본인확인·임시비번 수신용)
- `model/request/UserRequest`: `loginId` 필드 추가(첫 파라미터). **Lombok화**(`@Getter`, setter 제거).
  죽은 `fcmToken` 필드 제거(양쪽 생성자가 채운 적 없고 읽는 곳도 없음, 백엔드도 2.5에서 제거)
- `ui/signup/SignUp_Normal` · `SignUp_DrugStore`: `inputLoginId` 검증·전송,
  에러 분기에 `message.contains("아이디")` 추가(다른 분기보다 먼저)

### 아이디 찾기 (3-C) / 비밀번호 찾기 (3-D) — 완료(2026-09-07), `assembleDebug` 통과

- `ui/login/FindId` + `activity_find_id.xml`, `ui/login/FindPassword` + `activity_find_password.xml`,
  매니페스트 등록(`exported="false"`)
- 한 화면 단계식: 입력 → [인증코드 발송] → 코드칸·[확인] 노출 → 결과 + [로그인하러 가기].
  코드칸/확인/결과/로그인버튼은 `visibility="gone"`으로 시작
- `AuthApi`(+4): `sendFindIdCode`, `verifyFindId`, `sendResetPasswordCode`, `verifyResetPassword`
- DTO: `model/request/`에 `FindIdSendCodeRequest`, `FindIdVerifyRequest`, `ResetPwSendCodeRequest`,
  `ResetPwVerifyRequest` / `model/response/FindIdResponse` (전부 Lombok)
- `NetworkClient.getAuthApi()` 신설 — `getHttpClient()` 재사용. `AuthInterceptor`는 accessToken null이면
  통과, 4경로 permitAll이라 `TokenAuthenticator`도 안 걸림
- `Login`: 버튼 아래 "아이디 찾기 · 비밀번호 찾기" `TextView` 2개 → 각 액티비티로 `startActivity`
- **코드 형식 검증은 프론트에서 안 함** — `code.isEmpty()`만 체크, 일치 여부는 서버(`EmailVerificationStore.verify`)가 판단
- 재발송 쿨다운: `CountDownTimer` 60초, 버튼 비활성 + "N초 후 재발송", `onDestroy`에서 `cancel()`.
  백엔드 `issue()`의 60초 제한과 짝

**네트워크 계층:** 4개 호출 모두 로그인 전 상태. `AuthInterceptor`는 accessToken이 null이면
그냥 통과시키고(코드 확인함), 엔드포인트가 permitAll이라 401도 안 나므로 `TokenAuthenticator`도
안 걸린다.

### 후속 — 비번 재설정 흐름 변경 + 코드칸 자동 포커스 (2026-09-09, `assembleDebug` 통과)

- **비밀번호 찾기 3-D:** verify 성공 시 "임시비번 보냈어요" 결과 화면(`tv_result`/`btn_go_login`,
  레이아웃에서 제거)을 없애고, 응답의 `resetToken`을 담아 **신규 `ui/login/ResetPassword`**
  액티비티로 이동 → 새 비밀번호 2칸(`InputView` isPassword, `validationType="password"`) 입력 →
  `AuthApi.confirmResetPassword` → 성공 시 Toast + `Login`(CLEAR_TASK). 실패(토큰 만료 등)는
  되돌릴 수 없으므로 그대로 로그인 화면으로. `ResetPassword`도 ViewBinding.
- DTO 추가: `model/request/ResetPwConfirmRequest{resetToken,newPassword}`,
  `model/response/ResetPwVerifyResponse{resetToken}`. `AuthApi.verifyResetPassword` 응답 타입이
  `ApiResponse<Void>` → `ApiResponse<ResetPwVerifyResponse>`.
- **코드칸 자동 포커스:** `InputView.requestInputFocus()` 신설(EditText 포커스 + `showSoftInput`),
  `VerificationCodeView.start()`에서 호출 → 발송/재발송 직후 바로 코드 입력 가능.
- 회전 대응은 앱이 세로 고정(`943768b`)이라 불필요 — 스킵 확정.

### 후속 — `VerificationCodeView` 추출 + 인증 유효시간 표시 (2026-09-08, 완료 · `assembleDebug` 통과)

**착수 전:** 코드 입력 단계가 `FindId`·`FindPassword` 각 액티비티에 **인라인**으로 있었고,
`CountDownTimer` 60초 **재발송 쿨다운만** 있었다. **5분 인증 유효시간은 화면에 안 보였다**
(서버 `EmailVerificationStore`만 5분 후 무효 처리). 두 화면의 코드 단계 UI가 사실상 중복.

**한 것 (요약):** `VerificationCodeView`(루트 패키지, `InputView`와 나란히) +
`res/layout/view_verification_code.xml`(`<merge tools:parentTag="LinearLayout">`) 신설.
`InputView`처럼 생성자 1개 + `init(context)` 분리(커스텀 `declare-styleable` 없어 `attrs` 불필요,
자식 필드는 non-final). 구성: 코드칸(내부 `InputView` 재사용) + 유효시간 `TextView`(`남은 시간
4:32`, 1분 이하 빨강) + 재발송 `OutlinedButton` + 확인 버튼. 노출 API: `start()` / `stop()` /
`getCode()` / `showError()` / `onResendFailed()` / `setOnResendListener` / `setOnVerifyListener`.
네트워크는 뷰가 안 함 — 재발송/확인은 리스너만, 액티비티가 API 호출. `CountDownTimer` 2개는
`onDetachedFromWindow`에서 cancel. `FindId`·`FindPassword`는 인라인 타이머·`btnVerify`·`onDestroy`를
걷어내고 이 뷰로 교체, 첫 발송 성공 시 `btnSendCode`는 `GONE`(뷰의 재발송 버튼이 인계).
`activity_find_id/password.xml`은 `input_code`+`btn_verify` → `VerificationCodeView` 하나로.
`FindId`·`FindPassword`는 이참에 **ViewBinding으로 전환**(`findViewById` 제거,
`ActivityFindIdBinding`/`ActivityFindPasswordBinding` — 프로젝트에 `viewBinding = true` 이미 설정,
`FindDrugStore` 등이 쓰는 패턴). 커스텀 뷰라 `binding.verificationCode`가 `VerificationCodeView`를
바로 반환하고, 뷰 내부(`input_code` 등)는 별도 merge 레이아웃이라 액티비티 바인딩에 안 노출됨(캡슐화 유지).

너비 정렬: `InputView`는 `input.xml`의 `MaterialCardView`에 `layout_margin="8dp"`가 있어 보이는
카드가 8dp씩 안쪽이다. `MaterialButton`은 가로 inset이 0이라 버튼이 입력칸보다 16dp 넓게 보였음
→ 찾기 두 화면의 버튼·결과 텍스트·`tv_expiry`에 `layout_marginHorizontal="8dp"`(expiry는 end 8dp)
를 줘서 카드 가장자리에 맞춤. (로그인·회원가입 화면의 같은 어긋남은 이번 범위 밖)

**남은 것:** 회전 대응(`onSaveInstanceState` — 기존 인라인 버전도 안 했음). 실기기 확인은 완료.

<details><summary>착수 시 계획 (참고)</summary>

**할 것:** 코드 단계를 compound view로 뽑고, 유효시간 카운트를 추가한다. `InputView`를 처음
뽑아낸 것과 같은 이유(반복 UI 추출) — 단 `InputView`에 `code` 타입을 넣는 방식은 아니다.
`InputView`는 "아이콘+입력칸+에러문구, blur 시 정규식 검사"하는 범용 필드라, 타이머 2개
소유·생명주기·만료/재전송 콜백이 들어오면 상태 있는 플로우 로직이 범용 위젯으로 샌다.
재전송 버튼도 입력 필드의 일부가 아니다.

- 신규 `ui/login/VerificationCodeView` + `res/layout/view_verification_code.xml`
- 구성: 코드 입력칸(내부에 평범한 `InputView`/`EditText` — `InputView`는 범용 유지) +
  **인증 유효시간 카운트(`05:00`, 임박 시 빨강)** + **재전송 버튼(대기 60초 "N초 후 재전송")**
- 두 타이머는 코드 발송 성공 시 동시 시작. 60초 → 재전송 활성화, 5분 → 코드 무효 + 만료 안내
  (재전송 안 눌렸으면 이때 활성화). 인증 성공 시 둘 다 취소.
- 노출: `start()`(발송 성공 후 호출, 타이머 리셋) / `getCode()` / `setOnResendListener`(버튼 →
  액티비티가 실제 네트워크 호출 — 아이디 찾기 `sendFindIdCode`, 비번 찾기 `sendResetPasswordCode`라
  뷰가 직접 안 부름) / `setOnExpiredListener` / `stop()`
- 내부: `CountDownTimer` 2개, `onDetachedFromWindow`에서 cancel (지금 `onDestroy` cancel을 뷰로 이동)
- `FindId`·`FindPassword`는 인라인 타이머·코드칸 로직을 걷어내고 이 뷰로 교체
- 서버 TTL(5분)·쿨다운(60초)은 그대로. 화면 숫자는 표시용이고 판정은 서버가 함

</details>

### 후속 — 재발송 성공 시 화면 타이머가 리셋되지 않던 버그 (2026-09-20)

**문제:** `FindId`·`FindPassword`의 `sendCode(email, resend)`가 성공 분기에서
`if (!resend) binding.verificationCode.start()`였다. 서버는 재발송마다 `issue()`가 새 `Entry`로
덮어써 TTL 5분·시도 횟수가 리셋되는데(위 계획도 "start() = 타이머 리셋"이 의도), 화면은 최초
발송 때의 만료 타이머·`expired` 플래그를 그대로 썼다.
- 만료 후 재발송: 새 코드가 나가도 "만료됨"·확인 버튼 비활성이 유지돼 **새 코드를 입력할 수 없음**
  (화면 재진입 필요).
- 만료 전 재발송: 서버 코드는 5분 새로 유효한데 화면은 남은 시간만 세다가 확인 버튼을 꺼버림.
- 재발송 버튼은 클릭 시 `setEnabled(false)` 후 성공 경로에서 다시 켜는 곳이 없어 만료 타이머
  종료 때까지 비활성, 쿨다운 타이머도 재시작 안 됨.

**수정:** 두 액티비티 모두 `if (!resend)` 조건을 제거해 성공 시 항상 `start()` 호출.
`start()`가 입력·에러 초기화, `expired=false`, 확인 버튼 활성화, 만료/쿨다운 타이머 재시작을 이미
다 하므로 재발송에도 그대로 맞다. `resend` 인자 자체는 **실패 분기**(`onResendFailed()`로 재발송
버튼 복구 — 최초 발송 실패는 발송 버튼이 그대로 보여 복구 불필요)에서 여전히 쓰여 유지.

**택하지 않은 것:** 재발송 전용 `onResendSuccess()`/`restartTimers()` 신설(`start()`가 이미 필요한
초기화를 전부 함 — 코드만 늘어남), 타이머 재시작을 뷰가 스스로 하게 하는 구조 변경(호출처가 2곳뿐이라
과함).

**실기기 확인 완료.** 참고로 위 서술의 `VerificationCodeView`는 현재 코드에서 루트 패키지의
`VerificationCode.java`다(추출 후 이름/위치가 바뀐 듯 — 이 문서의 해당 항목은 갱신 안 함).

## 10.6 곁다리 — 액세스 토큰 TTL 축소

`jwt.access-token-validity` 3600000(1시간) → 600000(10분). **8.5의 "1시간" 표기는 이 값으로
대체된다.**

- 강제 로그아웃(10.2)이 다른 기기에 전파되는 창이 ≤1시간 → ≤10분으로 좁아진다. 액세스 토큰은
  서명·만료만 검사하고 DB를 안 보므로(8.2), `RefreshToken`을 지워도 이미 발급된 액세스 토큰은
  만료 전까지 통한다. 그 남은 만료 시간이 곧 강제 로그아웃 지연이다.
- 재발급 빈도는 실사용상 거의 안 는다. 재발급은 "요청 시 액세스 토큰이 만료돼 있음"일 때만
  일어나고, 이 앱은 한 번의 사용이 10분을 잘 안 넘는다. 사용 간격(보통 하루)이 10분·1시간보다
  길어서 어느 쪽이든 앱 열 때 재발급 1회로 같다. 액세스 토큰 수명이 차이를 만드는 건 한 번의
  사용이 토큰 수명보다 길 때뿐.

**주의:** 10분 넘게 연속으로 켜두는 세션은 중간에 재발급을 맞고, 그 순간 네트워크가 끊기면
재발급 실패 → 강제 로그아웃(작업 중이던 것 소실). 1시간짜리보다 이 창이 자주 열린다. 확률이
낮아 감수. 리프레시 토큰 로테이션(8.2)도 그만큼 잦아짐.

## 10.7 진행 순서

1. ~~loginId (10.3) 백엔드 코드~~ 완료 — 로컬 `ddl-auto: create` 재생성 → `update` 복귀 + Postman 검증은 남음
2. ~~메일·인증코드·복구 API (10.4) 코드~~ 완료 — Gmail 앱 비밀번호 준비 → Postman으로 4경로 확인은 남음
3. ~~TTL 축소 (10.6)~~ 완료 — `access-token-validity: 600000`, `deploy.yml`에 `MAIL_USERNAME`/`MAIL_PASSWORD` export 추가
4. ~~프론트 로그인·회원가입 (10.5의 3-A/3-B)~~ 완료 — `compileDebugJavaWithJavac` 통과
5. ~~프론트 찾기 화면 2개 (10.5의 3-C/3-D)~~ 완료 — `assembleDebug` 통과

**전 단계 완료** (`VerificationCodeView` 추출·인증 유효시간 표시 포함, 2026-09-08).
운영 DB `login_id` 마이그레이션, Gmail 앱 비밀번호 배포 시크릿, 운영 서버 통합 검증까지
전부 완료(2026-09-15) — EC2에 바로 배포해 확인하는 워크플로라 로컬 재생성·Postman 단계는
건너뜀.

## 10.8 알려진 한계 (의도적으로 미룸)

| 항목 | 왜 지금 안 하는가 |
| --- | --- |
| 이메일/아이디 enumeration | 없는 email·loginId에 명확한 에러 → 존재 여부 노출. 기존 로그인·회원가입도 동일하게 노출 중이라 찾기만 막아도 의미 없음. 실사용 전환 시 세 경로를 함께 "가입돼 있으면 메일 발송" 형태로 통일 |
| 본격 레이트리미팅 | IP·계정별 정교한 제한은 Redis/Bucket4j 필요, 현재 규모에 과함. `issue()`의 60초 재발송 제한으로 메일·리셋 폭탄의 실질 피해는 대부분 차단 |
| 인증코드 인메모리 | 서버 인스턴스 1개 전제. 스케일아웃 시 Redis |
| 강제 로그아웃 즉시성 | `RefreshToken`만 지우므로 다른 기기는 액세스 토큰 만료(≤10분) 후 튕김. 앱을 켜두고 요청을 0건 보내면 더 지연됨(액세스 토큰 검증은 요청 시에만 일어남). 즉시성이 필요하면 FCM `FORCE_LOGOUT` data 메시지(REFRESH 패턴 재사용, 401 경로는 백스톱으로 유지) — 후속 과제 |
| ~~임시 비밀번호 최초 로그인 강제 변경~~ | 10.4a에서 해소 — 임시 비밀번호 자체를 없애고 `ResetPassword` 화면에서 사용자가 새 비번을 직접 설정 |

## 10.9 변경 예정 범위

**백엔드 신규:** `mail/MailService`, `auth/EmailVerificationStore`,
`auth/service/AccountRecoveryService`, `auth/controller/AccountRecoveryController`, 요청/응답 DTO
**백엔드 수정:** `User`, `UserRepository`, `RefreshTokenRepository`, `UserRequest`,
`LoginRequest`, `UserService`(signUp/login), `DataInitializer`, `SecurityConfig`,
`application.yaml`, `build.gradle`, `deploy.yml`
**프론트 신규:** `FindId` / `FindPassword` 액티비티·레이아웃, DTO 5개, `AuthApi` 메서드 4개,
`NetworkClient.getAuthApi()`, `VerificationCodeView` + 레이아웃(10.5 후속)
**프론트 수정:** `Login`, `SignUp_Normal`, `SignUp_DrugStore`, `LoginRequest`, `UserRequest`,
`activity_login.xml`, 회원가입 레이아웃 2개, `AndroidManifest.xml`

## 10.10 착수 시 정할 것 (열린 결정)

1. ~~임시 비밀번호 생성 규칙~~ → 10.4a에서 폐기(임시 비밀번호 자체가 없어짐). 새 비밀번호는
   `ResetPassword` 화면에서 사용자가 입력, 백엔드 `@Pattern`으로 `ModifyPassword`와 동일 규칙 검증.
2. 인증코드 발송 메일 문구·제목. (현재 `MailTemplate` 기본값 사용 중)
3. ~~`UserResponse`에 `loginId`를 실어 MyInfo 화면에 "아이디"를 표시할지~~ → **안 함.** loginId는 변경 불가 필드라 조회 가치가 낮음. 필요해지면 그때 `User.toDto()`·`UserResponse`에 추가.
