# 8. JWT 인증 도입 (2026-09-06 ~ 2026-09-07)

**상태:** 백엔드 1~5단계, 프론트엔드 6단계까지 전부 완료. EC2에 배포해 로그인·화이트리스트·
인증 강제까지 실동작 확인했고, 안드로이드 앱도 토큰 자동 첨부·401 자동 재발급·재발급 실패 시
강제 로그아웃까지 전부 코드에 반영됨.

## 8.1 배경

5장 "인증 없음" 문제. 모든 API가 클라이언트가 보낸 `userId`를 그대로 믿고 썼다 —
`userId`만 바꾸면 남의 약속·친구·채팅에 접근/수정 가능했다.

## 8.2 설계 판단

### 액세스 토큰 = JWT, 리프레시 토큰 = DB 저장 opaque 문자열

리프레시 토큰은 재발급 때마다 어차피 DB 조회로 유효성(폐기 여부)을 확인해야 하므로,
서명까지 넣어 자체완결적으로 만들 이유가 없다. 액세스 토큰만 JWT로 만들어서 매 API 요청마다
DB를 안 타고 서명 검증만으로 통과시킨다 — 그게 JWT를 쓰는 이유다. 리프레시 토큰은
`UUID.randomUUID()`로 충분하다.

### 리프레시 토큰: 로테이션(1회용) + 슬라이딩 만료

- **로테이션:** 재발급할 때마다 `RefreshToken` 행의 `token` 값을 새 UUID로 덮어쓴다(행을
  지우고 새로 만들지 않고 같은 행을 재사용 — 더티 체킹으로 UPDATE). 옛 토큰 문자열은 그 순간
  DB 어디에도 없는 값이 되어 즉시 무효화된다. 유출된 토큰의 유효 기간이 "최대 2주"에서
  "다음 재발급 전까지"로 좁혀진다.
- **슬라이딩:** 재발급할 때마다 `expiresAt`을 `now + 2주`로 갱신한다. "활동이 없으면 2주 뒤
  로그아웃, 계속 쓰는 한 세션 유지"가 의도한 동작이다 — 카카오톡/인스타그램류 앱과 동일한 방식.
  "재발급마다 늘어나서 결국 무한정 로그인 유지되는 것 아니냐"는 우려가 있었는데, 실제로는
  각 시점의 **남은 유효기간이 항상 최대 14일로 캡핑**되는 것이지 누적되는 게 아니다
  (1시간마다 재발급해도 그 시점 기준 "지금부터 14일"일 뿐, 한 달치가 쌓이지 않는다).

**검토했지만 택하지 않은 대안:**
- 리프레시 토큰도 JWT로 발급 — 기각. DB 조회가 어차피 필요해서 자체완결성의 이점이 없다.
- 슬라이딩 + 절대 상한(예: 로그인 후 90일은 무조건 만료) 이중 구조 — 기각. `issuedAt` 컬럼을
  추가하고 `min(슬라이딩값, 절대상한)`을 계산해야 해서 지금 규모엔 과함. 필요해지면 추가.
- "남은 기간을 계산해서 옛 만료일에 더한다"는 식의 여러 변형(`ChronoUnit.DAYS.between()`으로
  차이를 구해 `saved.getExpiresAt()`에 더하거나, `now`에 더하거나, 순서를 바꾸거나) — 전부
  버그로 판명되어 폐기. 기준점을 계속 흔들리는 "이전 만료일"로 잡아서, 계산할 때마다 값이
  어긋났다(부호 반대로 잡으면 즉시 만료되는 과거 날짜가 나오고, 옛 만료일에 다시 더하면
  재발급할 때마다 무한정 증가). **"고정"이든 "슬라이딩"이든 기준점은 각각 "아예 안 건드림"과
  "지금 시각(`now`)" 둘 중 하나여야 하며, "이전 만료일"을 기준점으로 삼는 식은 계산이 아무리
  정교해도 둘 중 어느 것도 되지 않는다.** 최종 코드는 계산 없이 `now.plusWeeks(2)` 한 줄.

### userId는 body/param이 아니라 토큰에서만 추출

`@AuthenticationPrincipal Long userId`로, `JwtAuthenticationFilter`가 `SecurityContext`에
심어둔 값만 받는다. 클라이언트가 스스로 "나는 몇 번 유저다"라고 주장할 수 있는 경로(body 필드,
쿼리 파라미터, path 변수)를 전부 제거했다. 액세스 토�큰은 body가 아니라 `Authorization` 헤더로만
전달한다 — GET 요청은 body가 없는 경우가 많고, "누가"(인증)와 "무엇을"(요청 데이터)을 같은
층에 섞으면 모든 DTO마다 토큰 필드를 중복 선언하게 된다.

### 점진적 적용

`SecurityConfig`의 `permitAll()`을 유지한 채: 필터 설치(3단계, 검증만 하고 안 막음) →
컨트롤러 전환(4단계) → 마지막에 경로별로 `.authenticated()` 적용(5단계, 미착수). 컨트롤러
전환 도중에도 서버는 계속 정상 동작한다(빅뱅 배포 아님).

## 8.3 진행 단계

| 단계 | 내용 | 상태 |
| --- | --- | --- |
| 1 | 비밀번호 해시화(BCrypt) | 완료 |
| 2 | JWT 발급 인프라(`JwtProvider`, `RefreshToken`, 로그인/재발급/로그아웃) | 완료 |
| 3 | `JwtAuthenticationFilter`(검증만, 아직 강제 안 함) | 완료 |
| 4 | 컨트롤러 전환(`@AuthenticationPrincipal`) + 소유권 검증 보강 | 완료 |
| 5 | 경로별 `.authenticated()` 강제 적용 | 완료 |
| 6 | 프론트엔드(헤더 자동 첨부, 401 자동 재발급) | 완료 |

## 8.4 1단계 — 비밀번호 해시화

- `SecurityConfig`에 `BCryptPasswordEncoder` 빈 등록
- `UserService.signUp()` — `dto.toEntity()`가 평문 비밀번호를 그대로 엔티티에 옮기므로, 그
  직후 비밀번호만 인코딩된 값으로 덮어쓰는 `buildUserwithEncodedPassword()` 헬퍼 추가.
  `toEntity()` 자체(DTO 계층)에서 인코딩하지 않은 이유: `PasswordEncoder`는 Spring 빈이라
  DTO가 그 존재를 알게 하는 건 계층을 섞는 것.
- `login()`, `modifyPassword()` — `.equals()` 평문 비교를 `passwordEncoder.matches()`로 교체
- `DataInitializer` 목데이터 — `passwordEncoder.encode()`로 저장하도록 수정. 안 하면 목
  유저들이 평문 그대로 저장되어 `matches()`가 항상 실패, 로그인 자체가 깨짐

**밟았던 함정:** `modifyPassword()`에서 새 비밀번호(`dto.getNewPassword()`)가 아니라 현재
비밀번호(`dto.getCurrentPassword()`)를 인코딩해서 저장하는 실수가 있었음(비밀번호 변경 API를
호출해도 조용히 아무 효과가 없어짐). 발견 후 수정.

## 8.5 2단계 — JWT 발급 인프라

새 `auth/` 패키지 신설. `fcm/`과 같은 기준(User 엔티티를 몰라도 되는 순수 기술 코드는 `user/`
밖으로) — `JwtProvider`는 `Long userId`만 다루고 User를 모른다.

- `auth/JwtProvider` — `createAccessToken(userId)`, `getUserId(token)`. 시크릿은
  `application.yaml`의 `jwt.secret`(환경변수 `${JWT_SECRET}` 경유), 만료시간은
  `jwt.access-token-validity`(1시간, ms — 10.6에서 10분으로 축소 예정)
- `user/entity/RefreshToken` — `user_id : token = 1:N`, `token`에 유니크 제약. `user_devices`와
  같은 1:N 형태이지만 별도 테이블로 뗀 이유: `UserDevice`는 "이 기기가 지금 누구 것이냐"(기기
  축)이고 `RefreshToken`은 "이 로그인 세션이 유효하냐"(세션 축)라 성격이 다름 — 같은 기기에서
  로그아웃 후 재로그인하면 리프레시 토큰 행은 새로 생기지만 기기 토큰 행은 그대로 재사용됨
- `user/repository/RefreshTokenRepository` — `findByToken`, `deleteByToken`
- `UserService.login()` — 비밀번호 검증 후 액세스+리프레시 토큰 발급, 리프레시 토큰 DB 저장.
  `LoginResponse(UserResponse, accessToken, refreshToken)` 반환
- `UserService.reissueAccessToken()` / `logout()` — 8.2의 로테이션+슬라이딩, `deleteByToken`
- `auth/controller/AuthController` — `POST /api/auth/refresh`, `POST /api/auth/logout`.
  실패 시 다른 컨트롤러들과 다르게 400이 아니라 **401**을 반환 — "입력값이 틀렸다"가 아니라
  "인증이 안 됐다"는 다른 의미라서 클라이언트가 구분해서 처리(재로그인 유도)할 수 있어야 함

**응답 DTO를 record가 아니라 class로 통일한 이유:** `LoginResponse`를 처음엔 record로
제안했으나, 코드베이스의 DTO를 전수 검색해본 결과 `@Setter`가 붙어 있어도 실제로 setter가
호출된 적이 코드 전체에 단 한 번도 없었음(전부 `@Getter`로만 읽힘). 즉 record와 동작상 차이가
없어서, 이 경우엔 기존 파일들과의 스타일 일관성(class + Lombok)을 우선함.

## 8.6 3단계 — JwtAuthenticationFilter

- `auth/JwtAuthenticationFilter`(`OncePerRequestFilter`) — `Authorization: Bearer` 헤더 파싱
  → `jwtProvider.getUserId()` → 성공 시 `SecurityContextHolder`에
  `UsernamePasswordAuthenticationToken(userId, null, ...)`을 심음. **검증에 실패해도 필터가
  직접 요청을 막지 않고 그냥 다음으로 흘려보낸다** — "막을지 말지"는 뒤의
  `authorizeHttpRequests` 규칙(5단계)이 결정하는 구조
- `SecurityConfig` — 필터 등록(`addFilterBefore(jwtAuthenticationFilter,
  UsernamePasswordAuthenticationFilter.class)`), REST API용 401 JSON 응답을 위한
  `AuthenticationEntryPoint` 빈 추가(기본값은 로그인 페이지로 리다이렉트하려 해서 API 서버엔
  안 맞음)
- **검증 결과:** `permitAll()`을 유지한 채로 — 유효한 토큰을 보내면 로그가 찍히고 통과,
  토큰이 없거나 위조돼도 로그 없이 그대로 통과(의도한 동작). 둘 다 확인 완료.

## 8.7 4단계 — 컨트롤러 전환

`userId`를 body/query param/path variable로 받던 곳을 `@AuthenticationPrincipal Long userId`로
교체. 관련 요청 DTO 9개(`ModifyInfoRequest`, `ModifyPasswordRequest`, `YaksokRequest`,
`ShareYaksokRequest`, `ChatRoomRequest`, `FriendChatRoomRequest`, `DeleteChattingRoomRequest`,
`FriendRequestCreateDto`, `FriendRequestAnswerDto`, `CreateDrugStoreRequest`)에서 "나 자신"을
뜻하던 `userId`/`loginUserId` 필드 제거. "작업 대상이 되는 남"을 뜻하는 필드(`friendId`,
`senderId`, `yaksokId` 등)는 그대로 둠 — 이 둘을 구분하는 게 핵심이었다.

전환 대상: `UserController`(5곳), `YaksokController`(7곳), `ShareYaksokController`(4곳),
`ChatRoomController`(5곳), `FriendController`(6곳), `DrugstoreController`(1곳).
`MedicineController`는 유저 데이터를 안 다뤄서 대상 아님.

### 필드 교체만으로 안 끝나는 곳 — 소유권 검증이 아예 없던 4곳

컨트롤러 전환 중 `YaksokService`의 `getYaksok`, `updateYaksok`, `deleteYaksok`,
`updateNotificationStatus` 네 메서드가 **`userId` 파라미터 자체를 안 받고 있었던 것**을
발견했다. `yaksokId`/`notificationId`만 맞으면 그게 로그인한 사람 것이든 완전히 남의 것이든
구분 없이 조회·수정·삭제·복약완료 처리가 됐다. 특히 `updateNotificationStatus`는 알림의
"복약 완료" 버튼이 호출하는 메서드라, 고치기 전엔 **남의 복약 기록을 아무나 임의로
완료/취소 처리할 수 있는 상태**였다.

네 메서드 모두 `userId` 파라미터를 추가하고 `!resource.getUser().getId().equals(userId)`
검증을 새로 넣었다(단순 소유자 일치 검사 — `updateNotificationStatus`도 공유받은 사람은애초에
"복약 완료" 버튼 자체가 없어서(3.3절, `TAKEN_ALERT`는 버튼 없음) 소유자만 허용하면 됨. 처음에
"공유자도 예외로 허용해야 하나" 검토했으나 잘못된 전제였음).

**대비되는 사례:** `ChatService.getChatMessages()`, `deleteChattingRoom()`은 이미
`chatParticipantRepository.findByChattingRoomIdAndUserId()`로 참여자 검증이 있었다. "userId를
어디서 얻느냐"만 문제였던 곳과 "검증 로직 자체가 없었던" 곳이 컨트롤러마다 섞여 있어서, 하나씩
직접 확인이 필요했다 — 필드 치환 작업만으로 끝난다고 가정하면 안 되는 이유.

### 전체 컨트롤러 소유권 검증 재점검 결과

| 컨트롤러/서비스 | 결과 |
| --- | --- |
| `YaksokService`(4개 메서드) | 검증 없었음 → **수정 완료** |
| `ShareYaksokService.saveSharedYaksok` | 이미 `validateSharedToMe()`로 검증됨 |
| `ChatService`(메시지 조회/방 삭제) | 이미 참여자 검증됨 |
| `FriendService`(요청 수락/거절) | 이미 `isReceivedBy()`로 검증됨 |
| `DrugstoreService.modifyDrugStore` | 이미 `existsByMyDrugStore_HpidAndIdNot()`으로 검증됨 |
| `MedicineController` | 유저 데이터 없음, 해당 없음 |
| **`ChatService.getOrCreateFriendRoom`** | **친구 관계 검증이 없음 — 미수정, 5장 기술 부채로 이동** |

## 8.8 5단계 — 경로별 `.authenticated()` 적용

`SecurityConfig`의 `.requestMatchers("/**").permitAll()`을 **화이트리스트 방식**으로 뒤집었다
— 로그인 없이 호출돼야 하는 경로(`/api/users/signup`, `/api/users/login`, `/api/auth/refresh`,
`/api/auth/logout`, 약/약국 검색, `/ws-stomp/**`, `/error`)만 `permitAll()`로 명시하고 나머지는
`anyRequest().authenticated()`로 묶었다.

**블랙리스트가 아니라 화이트리스트로 짠 이유:** 나중에 컨트롤러가 추가됐는데 규칙 등록을
깜빡하면, 블랙리스트 방식(막을 것만 나열)에서는 그 컨트롤러가 자동으로 뚫린 채로 남고,
화이트리스트 방식에서는 자동으로 막힌 채로 남는다. 실수했을 때 안전한 쪽으로 실패하게 만드는
선택.

**EC2 배포 후 실동작 확인 완료:** 로그인(화이트리스트)은 토큰 없이 성공, 그 외 API는
`Authorization` 헤더 없이 401로 막힘, 유효 토큰으로는 정상 통과 — 확인함.

## 8.9 곁다리로 같이 처리한 것 — 자격증명 환경변수 분리

JWT 시크릿을 `application.yaml`에 넣으려다, DB 비밀번호와 공공데이터 API 키가 이미 평문으로
커밋되고 있던 것을 재확인(5장에 이미 있던 문제). `${DB_PASSWORD}`, `${MEDICINE_API_KEY}`,
`${DRUGSTORE_API_KEY}`, `${JWT_SECRET}` 형태의 환경변수 참조로 전환하고, `deploy.yml`의 서버
실행 단계(`Start New Server`)에 `export`로 GitHub Actions Secrets 값을 주입하도록 수정.

**주의(미해결):** 이미 커밋된 git 히스토리에는 과거의 평문 값이 그대로 남아있다. 저장소가
공개돼 있었다면 그 값들은 이미 노출된 것으로 간주하고, 나중에 실제 값 자체를 교체(rotate)해야
한다.

## 8.10 밟았던 함정들

- **`RefreshToken.builder()`를 호출하는 코드를 줬는데 엔티티에 `@Builder`가 없었던 실수.**
  엔티티 예시를 줄 때 컬럼 구조에만 집중하다 Lombok 어노테이션을 빠뜨림. `UserDevice`와 같은
  조합(`@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`)으로 통일.
- **`@Table(uniqueConstraints=...)`와 `@Column(unique=true)`를 같은 필드에 동시 선언.** 같은
  제약이 이름만 다르게 중복 생성될 수 있음. 이 프로젝트는 `@Table`에 이름을 붙여 선언하는
  방식으로 통일돼 있음(`UserDevice` 참고) — `@Column`엔 `nullable`/`length`만 남기고 제약은
  `@Table`에 몰아둠.
- **`JwtProvider` 생성자가 `@Value("${jwt.secret}") SecretKey key`로 선언되어 있었고,
  `application.yaml`엔 `jwt.secret` 자체가 없었음.** 이 상태로는 플레이스홀더 해석 실패로
  서버가 부팅조차 안 됨. `String`으로 받아 `Keys.hmacShaKeyFor(secret.getBytes(...))`로
  변환하는 방식으로 수정하고 yaml에 `jwt.secret`/`jwt.access-token-validity` 추가.
- **리프레시 토큰 만료 계산에서 `ChronoUnit.DAYS.between()`으로 "남은 기간"을 구해 그걸 다시
  더하는 시도를 여러 차례 반복함.** 8.2에 원인과 최종 결론 정리.

## 8.11 6단계 — 프론트엔드 구현

### 설계 판단

- **`Interceptor`(모든 요청에 토큰 첨부)와 `Authenticator`(401 시 재발급+재시도)를 분리.**
  OkHttp가 제공하는 두 확장점의 책임이 다르다 — 전자는 요청마다 무조건 개입하고, 후자는
  401을 받았을 때만 개입한다.
- **앱 전역 `Context` 필요 — `MedicationApp`에 static `getContext()` 추가.**
  `Interceptor`/`Authenticator`는 OkHttp 내부에서 호출되는 콜백이라 특정 화면(Activity)에
  속하지 않는다. 토큰을 읽으려면 `Context`가 필요한데, 화면이 넘겨주는 방식으로는 해결이 안
  돼서 앱 전체 수명을 가진 `Context`를 노출해야 했다.
- **재발급 호출은 `TokenAuthenticator`가 등록된 것과 별도의, 완전히 새로운 `Retrofit`으로
  보낸다.** 같은 `OkHttpClient`를 공유하면, 재발급 요청 자체가 401을 받는 경우(리프레시
  토큰도 무효한 경우 — 백엔드가 이때 401을 반환하도록 설계됨, 8.5) OkHttp가 또
  `authenticate()`를 재귀적으로 호출하는 경로가 생긴다. `.client(...)`를 지정하지 않으면
  Retrofit이 내부적으로 인터셉터·인증기가 전혀 없는 기본 `OkHttpClient`를 만들어 쓴다는 점을
  이용해 이 경로를 원천 차단했다.
- **재발급 응답의 새 리프레시 토큰을 반드시 즉시 저장한다.** 리프레시 토큰이 1회용
  로테이션(8.2)이라, 저장을 빠뜨리면 다음 재발급 시도가 이미 서버에서 폐기된 옛 토큰으로
  실패하게 된다.
- **재시도 무한루프 방지는 `priorResponse()` 체인 길이(2 이상이면 포기)로 막는다.** 단, 이
  카운터는 "같은 요청이 재시도된 횟수"만 세므로, 재발급 요청 자신이 별도로 401을 받는
  경로까지는 못 잡는다 — 그래서 위 "별도 클라이언트" 판단이 실질적인 방어선이고, 이 카운터는
  보조 안전장치다.
- **재발급 실패 시 강제 로그아웃은 새 `AuthEventBus`(기존 `YaksokEventBus`와 동일 패턴,
  3.5절)로 알린다.** `TokenAuthenticator`는 OkHttp 백그라운드 스레드에서 실행되어 어떤
  화면이 떠 있는지 모르므로, "세션이 끊겼다"는 사실만 발행하고 실제 화면 전환은 구독자
  (`BaseActivity`)가 결정하도록 역할을 나눴다. 리프레시 토큰이 애초에 없던 경우(최초
  미로그인 / 이미 로그아웃 / 동시에 실패한 다른 요청이 먼저 처리한 직후)는 끊을 세션 자체가
  없으므로 이벤트를 발행하지 않는다 — 안 그러면 동시 다발 요청 실패 시 토스트·화면전환이
  중복된다.
- **구독/해제는 `BaseActivity`의 `onResume`/`onPause`에.** 모든 화면이 `BaseActivity`를
  상속하므로 한 곳에만 넣으면 전체 화면에 적용된다. `onResume`/`onPause` 짝 덕분에 항상
  "지금 보이는 화면 하나만" 반응하게 되어 별도 판단 코드 없이 중복이 저절로 없어진다
  (`YaksokEventBus`가 `onStart`/`onStop`을 쓴 것과 다이얼로그 유무 차이는 있으나 원리는 같음).
- **`userId`를 로컬 저장값에서 꺼내 넘기던 모든 API 호출을 인자 없는 호출로 전환.** 서버가
  토큰에서 신원을 직접 판별하므로(8.7), 클라이언트가 "나는 몇 번이다"를 다시 알려줄 필요가
  없어졌다 — 오히려 로컬 값은 조작 가능해서 신뢰할 수 없다는 게 이 작업 전체의 출발점이었다.
  Retrofit 인터페이스 8개 메서드와 DTO 2개(`FriendRequestCreateDto`, `FriendRequestAnswerDto`)
  까지 함께 정리했다.

### 검토했지만 택하지 않은 대안

- **재발급 API를 `NetworkClient.getApi()` 등 기존 공유 `Retrofit`으로 호출** — 위 설계 판단대로
  재귀 경로가 생겨 기각.
- **`TokenAuthenticator` 안에서 직접 `startActivity(Login)`까지 처리** — 네트워크 계층
  클래스가 화면 전환 책임까지 지게 되어 역할이 섞인다. 기존 `YaksokEventBus` 패턴(발행자는
  사실만 알리고, 반응은 구독자가 결정)과도 어긋나 기각.

### 변경 범위

**신규**

| 파일 | 역할 |
| --- | --- |
| `network/AuthInterceptor` | 모든 요청에 `Authorization: Bearer <accessToken>` 첨부 |
| `network/TokenAuthenticator` | 401 시 리프레시 토큰으로 재발급 → 원요청 재시도, 실패 시 강제 로그아웃 |
| `network/AuthApi` | `POST /api/auth/refresh`, `POST /api/auth/logout` |
| `model/request/RefreshTokenRequest`, `model/response/TokenResponse` | 재발급 요청/응답 DTO |
| `util/AuthEventBus` | `YaksokEventBus`와 동일 구조, `onForceLogout()` 이벤트 1종 |

**수정**

- `MedicationApp` — static `getContext()` 추가
- `util/SprefsManager` — 토큰 전용 `SharedPreferences`(`YakSokAuthPrefs`)를 유저 정보 저장소와
  분리, `saveTokens`/`getAccessToken`/`getRefreshToken`/`clearAuthInfo` 추가. `clearUserInfo()`가
  마지막에 `clearAuthInfo()`까지 호출하도록 연결해, 기존 로그아웃 호출부를 안 건드려도 토큰까지
  함께 지워지게 함
- `network/NetworkClient.getHttpClient()` — `AuthInterceptor`, `TokenAuthenticator` 등록.
  6개 Retrofit이 이 메서드를 공유하므로 앱 전체에 일괄 적용됨
- `ui/base/BaseActivity` — `AuthEventBus.Listener` 구현, `onForceLogout()`에서 로그인 화면으로
  이동(자기 자신이 `Login`이면 무시)
- `ui/login/Login` — 로그인 응답에서 토큰 저장 후 유저 정보 저장
- Retrofit 인터페이스에서 `userId`/`loginUserId` 파라미터·경로변수 제거: `UserApi`(deleteUser,
  updateFcmToken, deleteFcmToken), `YaksokApi`(getNotifications, getYaksokList,
  getSharedUserList, getSharedYaksokList, deleteSharedYaksok), `FriendApi`(searchUser,
  getReceivedFriendRequests, getFriendList, deleteFriend), `ChatApi`(getPreviousMessages,
  getMyChatRooms, deleteChatRoom) — 및 이 메서드들을 부르던 화면 코드(`FriendList`,
  `YaksokDetail`, `ChatRoomList`, `ChattingRoom`, `YaksokList` 등)에서 이제 필요 없어진 로컬
  userId 조회/전달 코드 제거
- `model/request/FriendRequestCreateDto`, `FriendRequestAnswerDto` — "나 자신"을 뜻하던
  `userId`/`loginUserId` 필드 제거(8.7에서 백엔드 쪽 동명 DTO는 이미 제거됐었는데 프론트만
  남아있었음)

### 밟았던 함정들

- **`TokenResponse`(프론트)에 `@SerializedName("acessToken")` 오타(c 누락).** 서버가 실제로
  보내는 키(`accessToken`)와 안 맞아 Gson 파싱이 조용히 실패 → 컴파일도 런타임 예외도 없이
  필드에 `null`이 저장됨. 재발급 자체는 매번 "성공"으로 보이지만 실제로는
  `Authorization: Bearer null`로 재시도되어 계속 401이 나는, 원인 추적이 까다로운 형태로
  나타났을 것 — 반영 전 발견해 수정.
- **`Login.java`에 import 없이 `@Getter`가 클래스 선언에 잘못 붙어있던 실수.** Lombok import
  누락으로 컴파일 에러가 나는 상태였고, 애초에 `Login`(Activity)에 getter 자동 생성이 필요한
  것도 아니어서 삭제.
- **안드로이드 모듈의 Lombok 설정 여부를 확인할 때 파일 경로를 `app/build.gradle`(Groovy
  DSL)로 잘못 짚어 "Lombok이 설정 안 돼 있다"고 잘못 판단한 적이 있음.** 이 프로젝트는
  Kotlin DSL(`app/build.gradle.kts`)을 쓰고 있었고, 실제로는 `compileOnly`/`annotationProcessor`
  가 정확히 설정되어 있었다.
- **`TokenAuthenticator.forceLogout()`에서 처음엔 `SprefsManager.clearAuthInfo()`(토큰만
  삭제)를 호출.** `is_logged_in` 플래그가 안 지워져서, 강제로 `Login` 화면으로 보내도
  `Login.onCreate()`의 `isLoggedIn()` 체크가 `true`를 반환해 곧바로 `MainActivity`로 다시
  튕겨버리는 문제가 있었다. `clearUserInfo()`(내부에서 `clearAuthInfo()`까지 호출)로 교체해
  해결.
- **`AuthInterceptor`가 `"Bearer" + accessToken`으로 공백 없이 헤더를 만들고 있었음.**
  백엔드 `JwtAuthenticationFilter`는 `header.startsWith("Bearer ")`(공백 포함)로 검사하므로
  이 조건이 항상 거짓이 되어 `SecurityContext`가 절대 채워지지 않았다 — 즉 **유효한 액세스
  토큰을 보내도 매번 401**이 났다. 그런데 `TokenAuthenticator`가 재시도 요청을 만들 때는
  `"Bearer " + newTokens.getAccessToken()`으로 공백을 제대로 넣고 있어서, "1차 요청은 항상
  실패 → 재발급 → 공백 있는 헤더로 재시도 → 성공"이 매 API 호출마다 반복되는 형태로
  가려져 있었다. 최종 결과는 항상 성공이라 기능 테스트로는 드러나지 않고, 매 요청이
  불필요하게 2왕복하고 리프레시 토큰이 API 호출마다 로테이션되는 형태로만 나타난다.
  `AuthInterceptor`의 헤더 조합에 공백을 추가해 수정.

## 8.12 남은 것

- `ChatService.getOrCreateFriendRoom` 친구 관계 미검증 (8.7 참고)
- STOMP 채팅 인증은 이번 범위에서 계속 제외. `ChatController.sendMessage()`가 클라이언트가
  보낸 `sender`를 그대로 신뢰하고, 그 방의 실제 참여자인지도 확인하지 않는 상태 그대로임
  (6.1의 "채팅 sender email→id 전환" 항목이 이 WebSocket 인증에 묶여 있음)
- 6단계 작업 중 새로 발견: `ChatRoomController`의 `roomId`는 백엔드가 `String`으로 받는데
  프론트(`ChattingRoom`, `ChatApi` 등)는 여전히 `Long`으로 다룬다. 이번 인증 작업과는 무관한
  기존 설계라 손대지 않았음 — 5장 기술 부채에 등록.
- 6.1절의 "저장된 로그인 상태의 서버 검증"(`/me` 호출로 로컬 `isLoggedIn` 대신 실제 서버
  상태 확인)은 8장이 끝나 착수 가능해졌으나 아직 미착수.
