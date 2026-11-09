# 47. 가입·로그인을 auth로 이동 (2026-11-04)

**상태:** 완료(백엔드 컴파일 통과, 앱 빌드·실기기 확인 완료).

## 문제

가입·로그인·토큰 발급이 `UserService`/`UserController`(`/api/users`)에, 카카오 로그인·리프레시·로그아웃은
`AuthController`(`/api/auth`)에 나뉘어 있었다. `AuthController`가 `UserService`를 부르는 구조라 인증 흐름을 보려면
두 도메인을 오가야 했고, 카카오 연동(33장 흐름 C)을 붙이기 전에 인증 로직을 한곳에 모은다.

## 변경

- 새 `auth/service/AuthService`: `signUp`, `login`, `kakaoLogin`, `kakaoSignUp`, `reissueAccessToken`, `logout`과
  내부 헬퍼 `issueLogin`·`resolveDrugStore`·`buildUserwithEncodedPassword`·`checkKakaoEmail`. 코드는 그대로 옮김(동작 변경 없음)
- `UserService`에는 회원정보 수정·비밀번호 변경·탈퇴·FCM 기기·알림 설정만 남음
- **API 경로 변경:** `POST /api/users/signup` → `/api/auth/signup`, `POST /api/users/login` → `/api/auth/login`.
  `SecurityConfig` `permitAll()`과 앱 `UserApi`의 경로도 같이 바꿈
- 앱의 `signUp`/`login` 메서드는 `UserApi`에 그대로 둠(경로만 변경). `AuthApi`로 옮기는 건 호출처 전체 수정이라 생략

## 판단

- **URL까지 옮김(소유자 결정).** 대안은 URL을 유지하고 서비스만 옮기는 것 — 앱 호환에 영향이 없지만 `/api/users`와
  `/api/auth`에 인증 경로가 계속 섞여 남는다
- **호환성:** 구버전 앱은 `/api/users/login`을 호출해 새 서버에서 401이 된다(`permitAll` 밖 → 인증 필요). 서버 배포와 앱
  업데이트를 같이 해야 한다. 구 경로를 한동안 함께 열어두는 방법(컨트롤러에 두 경로 매핑)은 이번에 하지 않음 — 구버전
  앱 사용자가 생기면 그때 추가
