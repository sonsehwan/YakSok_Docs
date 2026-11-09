# 33. 카카오 로그인 (2026-10-12 설계)

**상태:** 설계 확정. 1차(로그인·가입)의 **백엔드 구현 완료**(컴파일·`KakaoClientTest` 통과, 서버 기동·실제 카카오
호출은 미확인), 카카오 콘솔 설정 완료, 앱(프론트) 구현 중. **기존 회원 연동(흐름 C)은 2차로 구현한다**
(한때 제외했다가 되돌림, 33.7). 닉네임 유니크 제거(33.12)도 같이 진행한다.

## 33.1 문제 정의

가입·로그인이 아이디/비밀번호뿐이다. 카카오 계정으로 가입·로그인할 수 있게 한다.
회원이 일반(`NORMAL`)과 약국(`DRUGSTORE`) 두 종류라, 소셜로 들어온 사람의 역할을 언제 정할지가
핵심 질문이었다. 약국 회원도 카카오로 가입 가능하게 한다.

## 33.2 흐름

세 흐름을 구분한다. 액세스 토큰은 항상 **우리 DB의 userId**로 만든다(kakaoId 아님).

**A. 카카오 로그인 (이미 카카오로 가입한 회원)** — 비로그인 상태
1. 앱: 카카오 SDK 로그인 → 카카오 access token
2. `POST /api/auth/kakao` (본문: 카카오 토큰)
3. 서버: 카카오 토큰 검증(33.3) → kakaoId → `findBySocialTypeAndSocialId`
4. 있음 → 기존 로그인과 같은 `LoginResponse`(액세스 + 리프레시) → 메인 화면

**B. 카카오 신규 가입** — 비로그인 상태
1~3. A와 같음 → 없음
4. 이메일 검사(33.5) 통과 시 "신규 회원" 응답 + 카카오 닉네임(있으면). 액세스 토큰 없음
5. 앱: 가입 완료 화면 — 역할, 닉네임(카카오 닉네임 미리 채움, 수정 가능), 약국이면 약국 선택.
   "이미 아이디로 가입하셨나요? 로그인 후 설정에서 카카오를 연결하세요" 안내 문구
6. `POST /api/auth/kakao/signup` (본문: **카카오 토큰 다시** + role + nickname + drugStore)
7. 서버: 카카오 토큰 재검증 → User 생성(닉네임·약국 검증은 일반 가입과 동일) → `LoginResponse`

6번 전에 이탈(앱 종료 등)하면 서버에 남는 것이 없다(User도 임시 데이터도 없음). 다시 카카오
로그인하면 kakaoId 조회가 또 비어 흐름 B로 돌아오고, 카카오 동의 화면만 생략된다. 입력하다 만
값(역할·약국)은 저장하지 않는다. "역할 미정 유저"가 생기지 않는 것이 즉시 생성안(33.7)을 기각한 이유다.

**C. 카카오 연동 (일반 회원, 2차)** — 로그인 상태. 이 흐름이 없으면 이미 아이디로 가입한 이메일로는
카카오 로그인을 못 하고(B에서 차단), 카카오 이메일을 바꾸라는 요구가 되어 UX가 나쁘다.
1. 설정 "카카오 연결" → 카카오 토큰
2. 헤더에 우리 액세스 토큰, 본문에 카카오 토큰
3. 서버: 액세스 토큰으로 회원, 카카오 토큰으로 kakaoId 확정 → `social_type`/`social_id` 채움
4. 그 kakaoId가 이미 다른 회원에 연결돼 있으면 거절. 이메일은 비교하지 않고 `users.email`은 그대로

A·B 이후 앱은 기존 로그인 후처리(토큰 저장 → FCM 기기 등록)를 그대로 탄다. 기기 등록은 로그인과
별개의 인증 API라 서버 변경이 없다.

## 33.3 카카오 토큰 검증

서버는 앱이 보낸 kakaoId를 믿지 않고 카카오 토큰으로 직접 묻는다(요청 본문 `userId`를 믿지 않는
것과 같은 원칙). 검증 메서드 하나를 A·B·C가 공유한다.

1. `GET /v1/user/access_token_info` → **`app_id`가 우리 앱 ID인지 확인**, 다르면 401
2. `GET /v2/user/me` → kakaoId, `kakao_account.email`, `is_email_valid`, `is_email_verified`,
   `profile.nickname`

1번이 없으면 공격자가 **자기 카카오 앱**으로 받은 피해자 토큰을 우리 서버에 보내 피해자 계정으로
로그인할 수 있다(`/v2/user/me`는 어느 앱의 토큰인지 따지지 않는다).
카카오 API 호출은 기존 `RestClientConfig`를 쓴다(백엔드 의존성 추가 없음).

## 33.4 스키마

- `users`에 `social_type`(VARCHAR, `KAKAO`), `social_id`(VARCHAR) 추가, 둘 다 nullable,
  `UNIQUE(social_type, social_id)`. MySQL UNIQUE는 null 중복을 허용하므로 미연결 회원끼리 충돌 없음
- `login_id`, `password` → nullable. 카카오 가입 회원은 둘 다 null
- `email`은 `NOT NULL` 유지(이메일 필수 동의, 33.5)
- 로그인 수단이 최소 한 쌍 온전히 있음을 DB가 보장. **엔티티에 선언**한다
  (`User`의 `@Table(check = @CheckConstraint(name = "check_users_login_method", ...))`, JPA 3.2 표준):
  ```sql
  CHECK ((login_id IS NOT NULL AND password IS NOT NULL)
      OR (social_type IS NOT NULL AND social_id IS NOT NULL))
  ```
  컬럼명은 필드명(`loginId`)이 아니라 DB 컬럼명(`login_id`)으로 쓴다
- 연동 해제 = `social_type`/`social_id`를 null로. 비밀번호 없는 회원의 해제는 거부(로그인 수단이
  없어짐). CHECK가 DB 차원에서도 막는다

**DB 반영 방법 — 테이블을 `create`로 새로 만든다.** `ddl-auto: update`는 새 컬럼과 UNIQUE만 추가하고, 기존
컬럼의 `NOT NULL` 해제나 CHECK 추가는 하지 않는다. 엔티티만 바꿔 배포하면 카카오 가입 INSERT가 실패한다.
현재 DB에는 목 데이터뿐이라 지워도 되므로, 테이블을 엔티티 기준으로 새로 만든다. 이러면 `login_id`·
`password` nullable, `social_*` 컬럼, UNIQUE, CHECK가 한 번에 반영된다(과거에 `update`가 남긴 잔존 컬럼 —
`users.fcm_token` 등 — 도 같이 사라진다).
- `create`는 **기동할 때마다 테이블을 지운다.** 한 번만 쓰고 반드시 `update`로 돌아와야 한다.
  `application.yaml`을 고쳐 배포하면 되돌리는 배포를 빠뜨릴 위험이 있어, 운영에서는 환경변수
  `SPRING_JPA_HIBERNATE_DDL_AUTO=create`를 준 채 jar를 한 번만 수동 실행하는 방식을 쓴다
  (`deploy.yml`에는 이 변수가 없어 다음 정상 배포부터 자동으로 `update`로 돌아온다)
- 로컬 DB도 같은 방식으로 한 번 만든다. 만든 뒤 `SHOW CREATE TABLE users;`로 CHECK가 들어갔는지 확인한다.
  운영 MySQL은 CHECK를 강제하는 **8.0.16 이상**이어야 한다(낮으면 문법은 통과해도 무시됨)
- **데이터를 보존해야 할 때의 방법(택하지 않음):** 기존 테이블에는 CHECK가 반영되지 않으므로 SQL로
  직접 해야 한다. 순서가 중요하다 — CHECK는 `social_type` 컬럼을 참조하는데 그 컬럼은 새 백엔드가
  처음 기동할 때 Hibernate가 만든다.
  1. 배포 전: `ALTER TABLE users MODIFY login_id VARCHAR(20) NULL, MODIFY password VARCHAR(255) NULL;`
  2. 새 백엔드 배포
  3. 배포 후: `ALTER TABLE users ADD CONSTRAINT check_users_login_method CHECK (...);`
- **택하지 않은 대안:** ① `application.yaml`을 `create`로 바꿔 배포 후 되돌리기(배포 2회, 되돌림 누락 시
  이후 배포마다 데이터 삭제) ② DB를 DROP 후 `update`로 재생성(결과는 같으나 수동 단계가 더 있음)
  ③ CHECK 생략(안전망일 뿐 기능에는 필요 없으나, 코드 실수로 로그인 불가 계정이 저장되는 것을 막으려 유지)

**필수값 검증은 DTO 몫이다.** DB를 nullable로 바꿔도 일반 가입은 `UserRequest`의 `@NotBlank`가
막는다. 카카오 가입은 loginId/password 필드가 없는 별도 DTO를 쓴다.

**카카오 가입 DTO의 `role`은 필수 + `NORMAL|DRUGSTORE`만 허용**(`@NotBlank` + `@Pattern`)한다.
일반 가입 `UserService.signUp`은 role이 없거나 모르는 값이면 일반 회원으로 처리하지만(앱이 항상
값을 보내고 결과가 가장 낮은 권한이라 그대로 둠), 카카오 가입은 역할 선택이 그 화면의 핵심이라
빠진 요청을 기본값으로 받아주면 안 된다. 400으로 거절한다.

## 33.5 이메일

- 카카오 앱을 비즈 앱(개인 개발자)으로 전환하고 이메일을 **필수 동의**로 받는다
- `is_email_valid`·`is_email_verified` 중 하나라도 false면 가입 거절. 안 그러면 남의 이메일을
  인증 없이 단 카카오 계정이 우리 DB의 그 이메일을 선점해 진짜 주인의 가입을 막는다
- **이메일 충돌(흐름 B):** 카카오 이메일이 기존 회원 이메일과 같으면 가입을 막고 "이미 가입된
  이메일입니다. 아이디로 로그인해주세요" 안내(연동 구현 전 임시 문구. 연동을 만들면 "아이디로 로그인 후 설정에서 카카오를 연결하세요"로 되돌린다)
- **반대 방향**(카카오 가입 후 같은 이메일로 일반 가입)은 기존 `existsByEmail`이 이미 막는다.
  문구만 "카카오로 가입된 이메일입니다. 카카오로 로그인해주세요"로 구분
- **이메일이 다른 동일인**이 연동 대신 카카오 로그인을 누르면 별개 계정이 생긴다. 서버는 같은
  사람인지 알 방법이 없다(33.7). 가입 화면 안내 문구로 대응하고, 이미 만들었으면 두 번째 계정
  탈퇴 후 연동

## 33.6 기존 코드 중 카카오 회원 때문에 손볼 곳

| 위치 | 문제 | 처리 | 차수 |
| --- | --- | --- | --- |
| `UserService.login` 102~111줄 | 카카오 로그인도 같은 토큰 발급이 필요 | 액세스·리프레시 발급부를 메서드로 추출해 공유 | 1차 |
| `UserService.modifyPassword` | password null → `matches` 실패, 엉뚱한 에러 | 카카오 회원 거부, 앱에서 메뉴 숨김 | 1차 |
| `UserResponse` | 앱이 카카오 회원인지 알 수 없음 | `socialType` 필드 추가(필드 추가라 기존 앱 호환) | 1차 |
| 앱 내 정보 화면 | loginId null 표시 | 카카오 회원이면 "카카오 로그인" 표시 | 1차 |
| 앱 로그아웃 | 다음 카카오 로그인 때 계정 전환 불가 | 카카오 SDK 로그아웃도 호출 | 1차 |
| `AccountRecoveryService` 아이디 찾기 | null loginId 반환 | "카카오로 가입된 계정입니다" 응답 | 2차 |
| 비밀번호 재설정 | 카카오 회원에게 비밀번호가 생김 | 거부 | 2차 |
| `SecurityConfig` | 새 공개 경로 | `permitAll()`에 `/api/auth/kakao`, `/api/auth/kakao/signup` | 1차 |

회원 탈퇴는 비밀번호 재확인이 없어 그대로 동작한다.

## 33.7 검토했지만 택하지 않은 대안

- **로그인 전에 [일반 | 약국] 선택** — 기존 회원도 매번 골라야 하고, 고른 것과 실제 계정이 다를
  때 처리가 생긴다. 역할은 우리 서비스의 정보라 가입 때 한 번만 묻는다
- **카카오 로그인 즉시 `NORMAL`로 User 생성 후 역할 변경** — 역할 선택 중 이탈하면 "역할 미정인데
  로그인된" 유저가 남고, 약국 hpid 검증이 "역할 변경" 경로에 하나 더 생긴다
- **카카오 로그인을 일반 회원 전용으로** — 범위는 작지만 약국 회원도 받기로 함
- **`social_accounts` 별도 테이블(1:N)** — 한 계정에 여러 소셜 연결이 가능하지만 지금은 카카오
  하나뿐. 조회 한 단계·탈퇴 CASCADE 부담. 두 번째 제공자가 확정되면 두 컬럼을 테이블로 옮긴다
- **OIDC `id_token` 검증** — 인증용 표준이지만 JWKS 조회·캐시·키 교체 대응과 콘솔 OIDC 활성화가
  필요. `app_id` 검증으로 같은 위협을 막을 수 있어 OAuth + `app_id` 검증 채택. OIDC가 기본인
  제공자(구글 등)를 붙일 때 재검토
- **가입용 토큰(JWT, `JwtProvider`로 발급)** — 요청 1과 2 사이에 kakaoId를 서명한 메모로 들고
  다니는 방식. `JwtAuthenticationFilter`가 서명만 맞으면 subject를 userId로 받아들이므로 kakaoId를
  subject로 넣으면 **그 숫자 id의 회원으로 인증**된다. `getUserId`에 type 검사를 넣어야 하는 위험을
  스스로 만드는 구조라 기각. 카카오 토큰 재검증은 신규 가입 시 카카오 호출 한 번이 더 들 뿐
- **가입용 토큰(인메모리 랜덤 문자열)** — 필터와 섞일 위험은 없지만 배포 재시작 때 가입 도중 사용자가
  처음부터 다시 해야 함. 재검증 방식이 저장소 자체가 필요 없음
- **이메일이 같으면 자동 연동** — 현재 일반 가입은 이메일 인증이 없다. 공격자가 피해자 이메일로
  일반 계정을 먼저 만들어 두면, 피해자가 카카오로 들어올 때 공격자 계정에 연결된다(비밀번호를
  공격자가 앎). 차단 + 안내로 결정
- **연동 시 이메일이 다르면 거부** — 회사/개인 메일처럼 정상 사례만 막는다. 본인 확인은 두 토큰으로
  이미 끝났다
- **이메일이 다른 동일인 자동 판별** — 전화번호 비교는 일반 가입이 전화번호를 받지 않아 불가하고,
  새로 받기엔 개인정보 수집 부담이 크다. 같은 기기 로그인 기록은 공용·중고 기기에서 틀리고 남의
  아이디를 노출할 수 있다
- **카카오 닉네임 자동 정리 후 바로 저장** — 우리 규칙(`^[a-zA-Z0-9가-힣]{2,10}$`, unique)과 안 맞는
  경우(공백·이모지·길이·중복)가 흔하다. 어차피 역할 선택 화면을 거치므로 같은 화면에서 미리 채워
  확인받는다. 동의 항목은 실명 "이름"(검수 필요) 대신 "닉네임"(`profile_nickname`, 선택)
- **이메일 nullable** — 이메일 없는 회원은 충돌 검사·메일 기능이 모두 예외가 된다. 비즈 앱 전환으로
  필수화
- **카카오 계정에 아이디/비밀번호 추가 기능** — 수요가 드물다. 필요하면 C와 같은 모양(현재 회원의
  두 컬럼 채우기)으로 붙이고, 그때 "비밀번호 없는 회원의 연동 해제 거부"를 "비밀번호 만든 뒤
  해제"로 풀 수 있다
- **두 계정 병합** — 약속·채팅·친구 데이터 병합 문제라 범위 밖
- **연동(흐름 C)을 빼고 출시** — 한때 제외했으나 되돌림. 이메일 충돌은 B에서 차단하는데 연동이 없으면
  아이디 가입자는 카카오 이메일을 바꾸지 않는 한 카카오를 쓸 수 없다. 출시 범위에 연동을 포함한다

## 33.8 구현 전 준비 (코드 밖)

- Kakao Developers: 개인 개발자 비즈 앱 전환, 동의 항목(이메일 필수, 닉네임 선택), Android
  플랫폼에 디버그·릴리즈 키 해시 모두 등록
- 백엔드 환경변수 `KAKAO_APP_ID` — `application.yaml`, `deploy.yml` 시크릿
- 프론트 의존성: 이미 `com.kakao.sdk:v2-all`(카카오맵용)이 있어 추가 없음
- 33.4의 마이그레이션 SQL

## 33.9 백엔드 1차 구현 메모

- 새 코드: `auth/KakaoClient`(검증 + 프로필 조회), `KakaoLoginRequest`/`KakaoSignUpRequest`/`KakaoLoginResponse`,
  `AuthController`의 `POST /api/auth/kakao`·`/kakao/signup`, `UserService.kakaoLogin`/`kakaoSignUp`
- `UserService` 정리: 로그인 토큰 발급부를 `issueLogin`으로, 가입의 약국 검증부를 `resolveDrugStore`로
  추출해 일반·카카오가 공유(일반 가입 동작은 그대로)
- **`/kakao` 응답:** `{newUser, login, nickname, email}`. 기존 회원이면 `login`에 `LoginResponse`, 신규면 `newUser=true`와
  미리 채울 닉네임. 신규 회원의 이메일 검증(필수·인증·중복)은 이 단계에서 미리 한다
- **실패는 400**(`KAKAO_LOGIN_FAIL`/`KAKAO_SIGNUP_FAIL`). 401을 주면 앱 `TokenAuthenticator`가 토큰 재발급을
  시도하므로 로그인 엔드포인트에서는 쓰지 않는다(기존 `/users/login`도 400)
- **`kakao.app-id`(`KAKAO_APP_ID`)는 기본값을 빈 문자열**로 둔다. 시크릿 등록 전에 배포해도 서버가 죽지 않고
  카카오 로그인만 전부 거절된다(`MAIL_USERNAME`과 같은 방식). `deploy.yml`에 export 추가됨 —
  GitHub Secrets에 `KAKAO_APP_ID` 등록 필요
- **Jackson 3 함정:** Spring Boot 4는 `tools.jackson`을 쓴다. `com.fasterxml.jackson.databind.JsonNode`를
  import하면 컴파일은 되지만 `RestClient`가 응답을 변환하지 못해 런타임에 실패한다(테스트가 잡음)
- `is_default_nickname`이 true면 닉네임을 미리 채우지 않는다(카카오가 운영정책 위반 닉네임을 "닉네임을
  등록해주세요"로 바꿔 내려줌). 카카오 닉네임 길이·문자 규칙은 공식 문서에서 확인하지 못해, 우리 규칙
  (`^[a-zA-Z0-9가-힣]{2,10}$`)을 가입 화면과 `KakaoSignUpRequest`에서 그대로 검증한다. 닉네임 유일성은 유지
- 1차 항목(운영 DB 마이그레이션(33.4), 앱 전체)은 완료했고, 실기기에서 카카오 로그인 정상 동작을 확인함

## 33.10 미룬 것

- **탈퇴 시 카카오 연결 끊기** — `deleteUser`에서 `social_type=KAKAO`면 `POST /v1/user/unlink`
  (Admin 키, `target_id={social_id}`, best-effort)를 호출하는 분기만 추가하면 된다. `social_id`를
  이미 저장하므로 스키마 변경 없음. 미뤄도 재가입은 정상(흐름 B). 대안으로 앱이 탈퇴 직전 SDK
  `unlink()` 호출(Admin 키 불필요, 둘 중 한쪽만 처리될 수 있음)
- 2차 남은 것: 아이디 찾기·비밀번호 재설정의 카카오 회원 처리 (흐름 C·연동 해제는 33.14)

## 33.11 앱: 로그인 화면 버튼 (UI만)

- `activity_login.xml`의 `btn_login` 바로 아래에 `btn_kakao_login`(ImageView 하나, 46dp, `#FEE500`, 모서리 8dp)을
  추가하고 `ll_find_account`의 앵커를 이 버튼으로 옮김. .pen `CTA 카카오 로그인`과 동일. 클릭 리스너는 아직 없음(SDK 연동 미착수)
- 라벨은 카카오 공식 SVG의 경로를 벡터 드로어블(`ic_kakao_login_label`)로 변환 — 로고를 새로 그리지 않음.
  `MaterialButton` 아이콘은 `iconSize`가 정사각형이라 가로로 긴 라벨이 찌그러져 택하지 않음(배경 드로어블 + `scaleType=center`인 ImageView 하나로 처리, 감싸는 레이아웃 없음)

## 33.12 닉네임 유니크 제거

카카오 닉네임은 다른 회원과 겹칠 수 있어, 가입이 "이미 존재하는 닉네임"으로 막힌다. 닉네임을 표시용 이름으로 보고
`users.nickname`의 UNIQUE와 서버의 중복 검사(`existsByNickname`)를 없앤다.
- 영향: `FriendService.searchUserByNickname`이 `findByNickname`(Optional)에 의존한다. 중복이 생기면
  `IncorrectResultSizeDataAccessException`으로 500이 되므로 친구 검색 방식을 함께 바꿔야 한다(결정 대기:
  이메일 검색 / 닉네임 목록 반환). 채팅방 이름·알림 문구 등 나머지 닉네임 사용처는 표시용이라 영향 없음
- 반영은 33.4의 `create` 재생성 때 한 번에(엔티티 `unique = true` 제거)

## 33.13 앱: 카카오 가입 완료 화면 (`SignUp_Kakao`)

- `/api/auth/kakao`가 `newUser=true`를 주면 `Login`이 카카오 토큰·닉네임·이메일을 extra로 넘겨 `SignUp_Kakao`를 연다.
  화면은 .pen `SignUp_Kakao_Normal`/`SignUp_Kakao_DrugStore`와 같다: 회원 유형 선택 → 닉네임(미리 채움, 수정 가능) →
  이메일(`isReadOnly`, 카카오 값) → 약국 회원일 때만 소속 약국(기존 `FindDrugStore` 재사용)
- 회원 유형 선택은 `TextView` 두 개 + `setSelected`(`bg_role_option`, `selector_role_option_text`). `MaterialButtonToggleGroup`은
  체크 상태 색을 따로 선언해야 하고 한쪽을 끌 수 없게 만드는 코드가 필요해 택하지 않음
- **버그 수정:** `KakaoLoginRequest`/`KakaoSignUpRequest`의 필드명이 `acessToken`(오타)이라 서버 DTO(`accessToken`)와
  안 맞아 토큰이 null로 전달되던 문제
- 미구현: 가입 완료 버튼의 `POST /api/auth/kakao/signup` 호출과 성공 후 처리(토큰 저장·FCM 등록·메인 이동). 이 후처리는
  `Login.onLoginSuccess`에 private으로 있어, 호출하기 전에 공용으로 뺄지 결정해야 함

## 33.14 기존 회원 연동 (흐름 C) 구현

**서버** — `POST /api/auth/kakao/interlock` (인증 필요, `permitAll` 밖). 본문 `KakaoLoginRequest` 재사용, 응답은 갱신된
`UserResponse`, 실패 400 `INTERLOCK_FAIL`. `AuthService.kakaoInterlock` 검사 순서: 이미 연동된 회원 거절 → `fetchProfile`
(app_id 검증 공유) → 그 kakaoId가 다른 회원에 있으면 거절 → `social_type`/`social_id` 채움(변경 감지)
- 이메일은 비교하지 않고 `users.email`도 그대로. 본인 확인은 우리 토큰 + 카카오 토큰으로 끝난다
- 연동 후 카카오 로그인은 `findBySocialTypeAndSocialId`가 그 일반 회원을 찾아 흐름 A로 들어간다. `checkKakaoEmail`은 조회가
  비었을 때(신규)만 돌아 연동 회원의 이메일이 카카오와 달라도 영향 없음
- `permitAll`에 넣으면 안 되는 이유: 만료 토큰이면 필터가 인증을 세팅하지 않아 `userId=null` → 400이 되어, 401에서만 도는 앱
  `TokenAuthenticator` 재발급이 안 걸린다

**이미 카카오로 따로 가입한 계정이 있을 때** — 거절("이미 다른 계정에 연결된 카카오입니다"). 사용자는 카카오 계정을 탈퇴한 뒤
연동한다(탈퇴로 `social_id`도 사라짐). 일반 앱들도 대부분 거절 + 해제 안내 방식
- 택하지 않은 대안 ① **카카오만 옮기기** — 카카오 계정이 로그인 수단 없는 계정이 되어 CHECK 위반
- ② **연동하며 카카오 계정 자동 삭제** — 버튼 하나로 되돌릴 수 없는 삭제. 불만이 생기면 확인 다이얼로그 + 기존 탈퇴 로직으로 추가 가능
- ③ **병합** — 서로 친구/공유면 자기 자신과의 관계가 되고, 같은 채팅방이면 참여자·읽음 커서 중복, 역할(일반/약국) 충돌,
  `user_devices` 이전까지 도메인별 규칙이 필요
- 탈퇴 안내를 메시지에 붙이지 않은 이유: 토스트는 길면 잘린다. 필요해지면 에러 `code`를 분리하고 앱에서 다이얼로그로

**`UserResponse.hasPassword` 추가** — 연동하면 일반 회원도 `socialType=KAKAO`가 되어, 앱이 `socialType != null`로 숨기던
비밀번호 변경 메뉴가 연동 회원에게서도 사라진다. 앱은 연동 회원과 카카오 전용 회원을 구분할 값이 없었다
- `loginId`를 내려주는 대안은 앱이 필요 없는 식별자를 노출하고, 앱이 알고 싶은 건 "비밀번호 변경 가능 여부"뿐이라 boolean 채택
- 앱 조건은 `socialType != null && !hasPassword`일 때만 숨김. `hasPassword`가 없던 시절 SharedPreferences에 캐시된 일반 회원은
  `false`로 읽히지만 `socialType`이 null이라 메뉴가 유지된다

**앱** — 카카오 SDK 로그인(카카오톡 → 실패 시 계정 로그인)을 `util/KakaoLoginHelper`로 빼 `Login`과 `MyInfo`가 공유.
`MyInfo`에 계정과 분리된 "소셜 연동" 영역 — 동그란 카카오 아이콘(`btn_link_kakao`)을 누르면 연동, 옆에 "연결하기/연결됨". 카카오 전용 회원에게도 보인다(연결됨). 아이콘은 공식 로그인 리소스에 심볼 단독 파일이 없어, 기존 `ic_kakao_login_label`(공식 SVG 변환)의 말풍선 path를 떼어 `ic_kakao_symbol` + 원형 `#FEE500` 배경으로 구성. 받은 `kakaotalk_sharing_btn_small`은 카카오톡 공유용·34px 비트맵이라 쓰지 않음. 연동 실패 시 SDK 로그아웃을 호출해 다음 시도에서
다른 카카오 계정을 고를 수 있게 한다. 실패 문구는 서버 `message`를 그대로 토스트

**연동 해제** — `DELETE /api/auth/kakao/interlock`(인증 필요), `AuthService.kakaoUnlink`. 연동 안 된 회원·비밀번호 없는(카카오로만
가입한) 회원은 거절(로그인 수단이 없어짐, DB CHECK도 같은 조건). 통과하면 `social_*`를 null로, 응답은 갱신된 `UserResponse`, 실패 400
`UNLINK_FAIL`. 카카오 쪽 연결 끊기(`/v1/user/unlink`)는 하지 않는다 — 같은 카카오로 다시 연동하면 그대로 붙는다
- 앱: 같은 동그란 아이콘이 상태에 따라 연동/해제. 연결됨이면 확인 다이얼로그 후 해제하고 SDK 로그아웃. 카카오로만 가입한 회원은 아이콘 비활성

**로그인 화면** — 가로로 긴 카카오 버튼을 없애고 "아이디 찾기 | 비밀번호 찾기" 아래에 "소셜 로그인" 구분선 + 원형 심볼 버튼
(`btn_kakao_login`, `ic_kakao_symbol` + `bg_kakao_circle`)으로 교체. 제공자가 늘면 같은 크기 원형을 옆에 추가하는 구조.
쓰이지 않게 된 `ic_kakao_login_label`·`bg_kakao_login`은 삭제
