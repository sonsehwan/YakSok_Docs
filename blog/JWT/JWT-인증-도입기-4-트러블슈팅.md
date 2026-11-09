# JWT 인증 도입기 (4) — 삽질 총정리

[1편](./JWT-인증-도입기-1-배경과-백엔드-설계.md), [2편](./JWT-인증-도입기-2-필터와-컨트롤러-전환.md),
[3편](./JWT-인증-도입기-3-안드로이드-프론트엔드-연동.md)에서 설계와 구현을 정리했다면, 이번
글은 그 과정에서 실제로 만났던 버그들을 모은 것이다. 다들 사소해 보이는데, 하나하나가
"왜 안 되는지 한참 찾아야 했던" 것들이라 기록해둔다.

## 1. `JwtProvider` 생성자 타입 미스매치

가장 처음 만든 코드는 이랬다.

```java
public JwtProvider(@Value("${jwt.secret}") SecretKey key, ...) { ... }
```

`String` → `SecretKey`로 변환해주는 Spring 컨버터는 없다. 게다가 `application.yaml`에
`jwt.secret` 자체가 존재하지도 않았다. 이 상태로는 플레이스홀더 해석에 실패해서 **서버가
부팅조차 안 된다.**

```java
// 수정 후
public JwtProvider(@Value("${jwt.secret}") String secret, ...) {
    this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
}
```

`String`으로 받아서 직접 `SecretKey`로 변환하고, yaml에 `jwt.secret`/`jwt.access-token-validity`를
추가하고 나서야 부팅이 됐다.

## 2. `builder()`를 호출했는데 `@Builder`가 없었다

`RefreshToken` 엔티티 예시 코드를 줄 때 컬럼 구조에만 집중하다가 Lombok 어노테이션 조합을
빠뜨렸다. 그런데 서비스 코드에서는 `RefreshToken.builder()...build()`를 쓰고 있었으니
당연히 컴파일이 안 됐다. 이미 있던 `UserDevice` 엔티티와 같은 조합
(`@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`)으로 맞추고 나서
해결했다.

## 3. 유니크 제약을 두 번 선언

```java
@Column(nullable = false, length = 255, unique = true)  // 여기도 unique
private String token;
```

그리고 클래스에는 `@Table(uniqueConstraints = @UniqueConstraint(...))`도 같이 있었다.
같은 제약을 이름만 다르게 중복 생성하는 꼴이다. 이 프로젝트는 이미 `@Table`에 이름을 붙여
제약을 선언하는 방식으로 통일돼 있었어서(`UserDevice` 참고), `@Column`에는
`nullable`/`length`만 남기고 제약은 `@Table` 한 곳에 몰아넣었다.

## 4. 리프레시 토큰 만료 계산 — 세 번 틀렸다

[1편](./JWT-인증-도입기-1-배경과-백엔드-설계.md)에서도 다뤘지만, "슬라이딩이면서 상한을
두고 싶다"는 요구를 `ChronoUnit.DAYS.between()`으로 풀려던 시도가 세 가지 형태로 전부
틀렸다.

- 뺄셈 부호가 반대 → 계산된 만료일이 과거가 되어 즉시 만료
- 순서를 바꾸면 사실상 원래 만료일과 같아짐 → 슬라이딩이 아니라 그냥 고정 만료와 동일
- 이전 만료일에 계속 더하는 방식 → 재발급마다 무한정 증가

**교훈:** 계산의 기준점은 "아예 안 건드림"이나 "지금 시각" 둘 중 하나여야 한다. "이전
계산 결과"를 기준점으로 다시 계산하는 방식은 아무리 정교해도 답이 안 나온다. 최종 코드는
`saved.setExpiresAt(LocalDateTime.now().plusWeeks(2))` 한 줄이었다.

## 5. `modifyPassword()`가 엉뚱한 값을 인코딩

```java
// 버그
user.setPassword(passwordEncoder.encode(dto.getCurrentPassword()));  // 현재 비밀번호를 인코딩?

// 수정
user.setPassword(passwordEncoder.encode(dto.getNewPassword()));
```

비밀번호 변경 API를 호출해도 조용히 아무 효과가 없어지는 버그였다. 응답은 200이 오니
프론트에서는 "성공"으로 보이는데, 실제로는 비밀번호가 그대로였다.

## 6. `TokenResponse` 필드명 오타

```java
@SerializedName("acessToken")   // c가 하나 빠짐
private String accessToken;
```

백엔드가 실제로 보내는 JSON 키는 `accessToken`이다. Gson은 매칭되는 키를 못 찾으면 예외를
던지지 않고 그냥 `null`을 넣는다. 그러면:

```java
.header("Authorization", "Bearer " + newTokens.getAccessToken())
// → "Authorization: Bearer null"
```

재발급 자체는 `isBusinessSuccess() == true`로 "성공"처럼 보이는데, 실제로는 `null` 토큰으로
재시도해서 다시 401을 받는 — 겉만 보면 원인을 짐작하기 어려운 버그가 될 뻔했다.

## 7. Lombok 설정을 잘못 짚은 사건

안드로이드 모듈에 Lombok이 설정돼 있는지 확인하려고 `app/build.gradle`을 열어봤는데
`compileOnly("org.projectlombok:...")` 같은 게 안 보였다. "이 프로젝트엔 Lombok이 없다"고
결론 내리고 `@Getter` 대신 명시적 getter를 쓰라고 안내했었다.

문제는 파일을 잘못 짚었다는 것이었다 — 이 프로젝트는 Kotlin DSL을 쓰고 있어서 실제 파일은
`app/build.gradle.kts`였다. 그 파일을 열어보니 `compileOnly`/`annotationProcessor`가
정확하게 설정되어 있었다. 확인할 때 파일 확장자(Groovy `.gradle` vs Kotlin `.gradle.kts`)를
먼저 확인하지 않으면 이런 실수가 생긴다.

## 8. `Login.java`에 엉뚱하게 붙은 `@Getter`

```java
@Getter
public class Login extends BaseActivity {
```

`import lombok.Getter;`도 없이 클래스 선언에 `@Getter`가 붙어 있었다. 컴파일 에러가 나는
상태였고, 애초에 `Login`(Activity)의 필드들(뷰 위젯)에 getter를 자동 생성해줄 이유도
없었다. 다른 파일 작업하다가 실수로 잘못 붙인 것으로 보여서 그냥 지웠다.

## 9. `clearAuthInfo()`로는 부족했다

강제 로그아웃 로직에서 처음엔 이렇게 썼다.

```java
private void forceLogout() {
    SprefsManager.clearAuthInfo(MedicationApp.getContext());  // 토큰만 지움
    AuthEventBus.get().publish();
}
```

`clearAuthInfo()`는 액세스/리프레시 토큰만 지우는 메서드다. 그런데 `Login` 화면은 로그인
여부를 토큰이 아니라 `is_logged_in`이라는 별도 플래그로 판단한다.

```java
if (SprefsManager.isLoggedIn(this)) {
    startActivity(new Intent(this, MainActivity.class));
    finish();
    return;
}
```

토큰만 지우고 이 플래그를 안 지우면, 강제로 `Login` 화면에 보내도 `isLoggedIn()`이
여전히 `true`라서 **곧바로 `MainActivity`로 다시 튕겨버린다.** `MainActivity`는 다시 인증이
필요한 API를 부르고, 토큰이 없으니 또 401, 리프레시 토큰도 없으니 이번엔 강제 로그아웃도
안 걸리고 그냥 조용히 실패한다 — 로그인 화면에 영원히 도달하지 못하는 상태.

`clearUserInfo()`(내부적으로 `clearAuthInfo()`까지 호출하도록 이미 연결돼 있었다)로 바꿔서
해결했다.

## 10. 제일 재밌었던 버그: 공백 하나

```java
.header("Authorization", "Bearer" + accessToken)   // "Bearer"와 토큰 사이에 공백이 없다
```

백엔드는 이렇게 검사한다.

```java
if (header != null && header.startsWith("Bearer ")) {   // "Bearer " — 공백 포함
```

공백이 없으니 이 조건은 **항상 거짓**이다. 즉 액세스 토큰이 방금 정상 발급된 완벽하게
유효한 토큰이어도 `SecurityContext`는 절대 채워지지 않는다.

여기서 흥미로운 지점은, **이 버그가 있어도 앱은 "정상 동작"한다는 것**이었다. 왜냐하면:

1. `AuthInterceptor`가 공백 없는 헤더로 요청 → 백엔드가 인식 못 함 → 401
2. `TokenAuthenticator.authenticate()`가 자동 실행 → 액세스 토큰은 멀쩡했으니 리프레시도
   당연히 성공 → 리프레시 토큰도 로테이션됨
3. 재시도 요청은 `TokenAuthenticator` 자신이 직접 만드는데, 그 코드는
   `"Bearer " + newTokens.getAccessToken()`으로 **공백이 제대로 들어가 있었다** → 이번엔
   정상 인식 → 200 성공

즉 **모든 API 호출이 결국 성공**하기 때문에, 화면을 눌러보면서 하는 기능 테스트로는 절대
드러나지 않는다. 실제로는:

- 모든 요청이 항상 2번 왕복한다 (1차는 무조건 실패, 2차는 재발급을 거쳐서 성공)
- 모든 요청마다 **리프레시 토큰이 매번 로테이션**된다 — DB에 불필요한 UPDATE가 API 호출
  횟수만큼 발생하고, "2주 슬라이딩 만료"로 설계했던 리프레시 토큰이 사실상 API를 호출할
  때마다 갱신되는 꼴이 되어 설계 의도와 어긋난다

고치는 방법은 공백 하나를 추가하는 것뿐이었다.

```java
.header("Authorization", "Bearer " + accessToken)
```

## 마치며

이번 작업에서 반복해서 얻은 교훈은 결국 하나로 모인다. **"기능이 동작하는 것처럼 보인다"는
것과 "설계한 대로 동작한다"는 것은 다르다.** 10번 사례처럼 겉으로는 완벽하게 성공하는
API 호출 뒤에, 매번 불필요한 왕복과 토큰 로테이션이 숨어 있을 수 있다. 코드 리뷰 때 실제
네트워크 요청/응답 로그를 한 번씩 까보는 습관이 왜 필요한지 다시 느낀 계기였다.
