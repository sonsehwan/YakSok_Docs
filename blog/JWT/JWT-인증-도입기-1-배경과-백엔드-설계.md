# JWT 인증 도입기 (1) — 왜 다시 만들었나, 그리고 백엔드 설계

복약 관리 앱 **YakSok**을 만들면서 가장 늦게, 그리고 가장 크게 후회한 부분이 인증이었다.
초기 버전은 로그인은 있었지만 "로그인 이후"가 없었다 — 모든 API가 클라이언트가 요청 본문에
실어 보내는 `userId`를 그냥 믿고 썼다. 이 글에서는 이 문제를 왜 JWT로 풀었는지, 그리고
백엔드 설계에서 어떤 선택을 했고 무엇을 버렸는지를 정리한다.

## 문제: `userId`를 클라이언트가 자기 입으로 말하고 있었다

```java
@PostMapping
public ApiResponse<Void> saveYaksok(@RequestBody YaksokRequest request) {
    yaksokService.saveYaksok(request); // request 안에 userId 필드가 있었다
}
```

이 구조의 문제는 단순하다. **요청 바디의 `userId`를 다른 사람의 id로 바꿔서 보내면, 서버는
아무 의심 없이 그 사람 행세를 해준다.** 인증서버가 "너가 누구인지"를 검증하는 게 아니라
"네가 자기 입으로 말한 걸" 그대로 믿는 구조였던 셈이다. 친구 목록, 채팅방, 약속 데이터 —
전부 같은 패턴이었다.

이걸 고치려면 결국 "이 요청을 보낸 사람이 정말 누구인지"를 서버가 스스로 판단할 수 있는
장치가 있어야 했다. 그게 JWT였다.

## 왜 액세스 토큰은 JWT이고 리프레시 토큰은 아닌가

가장 먼저 정한 것은 **토큰을 두 종류로 나눈다**는 것이었다 — 액세스 토큰(JWT)과 리프레시
토큰(DB에 저장하는 불투명 문자열).

이 둘을 다르게 만든 이유는 명확하다. JWT의 장점은 "서명만 검증하면 DB를 안 타도 된다"는
것이다. 매 API 요청마다 이 이점을 누리려면 액세스 토큰은 JWT가 맞다. 그런데 리프레시
토큰은 애초에 **재발급할 때마다 DB 조회로 유효성(폐기 여부)을 확인해야 한다.** 그럴 거면
서명까지 넣어서 자체완결적으로 만들 이유가 없다. 그냥 `UUID.randomUUID()`로 충분하고, 오히려
DB에 넣고 빼는 게 자유로워서 다루기 쉽다.

```java
@Component
public class JwtProvider {
    private final SecretKey key;
    private final long accessTokenValidityMs;

    public JwtProvider(@Value("${jwt.secret}") String secret,
                        @Value("${jwt.access-token-validity}") long accessTokenValidityMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenValidityMs = accessTokenValidityMs;
    }

    public String createAccessToken(Long userId) {
        return Jwts.builder()
                .subject(String.valueOf(userId))
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + accessTokenValidityMs))
                .signWith(key)
                .compact();
    }

    public Long getUserId(String token) {
        return Long.valueOf(Jwts.parser().verifyWith(key).build()
                .parseSignedClaims(token).getPayload().getSubject());
    }
}
```

`JwtProvider`는 `Long userId`만 다루고 `User` 엔티티를 아예 모른다. `fcm/` 패키지를 만들 때
세웠던 기준과 같다 — 유저 도메인을 몰라도 되는 순수 기술 코드는 `user/` 패키지 밖에 둔다.

## 리프레시 토큰: 로테이션(1회용) + 슬라이딩 만료

리프레시 토큰 설계에서 정한 두 가지 규칙이 있다.

**로테이션.** 재발급할 때마다 DB에 저장된 토큰 값을 새 UUID로 갈아치운다. 이때 행을
지우고 새로 만드는 게 아니라 **같은 행을 재사용**한다(더티 체킹으로 UPDATE).

```java
@Transactional
public TokenResponse reissueAccessToken(String refreshToken) {
    RefreshToken saved = refreshTokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new IllegalArgumentException("유효하지 않는 리프레쉬 토큰입니다."));

    if (saved.getExpiresAt().isBefore(LocalDateTime.now())) {
        refreshTokenRepository.deleteByToken(refreshToken);
        throw new IllegalArgumentException("만료된 리프레쉬 토큰입니다.");
    }

    String newAccessToken = jwtProvider.createAccessToken(saved.getUser().getId());
    String newRefreshToken = UUID.randomUUID().toString();

    saved.setToken(newRefreshToken);
    saved.setExpiresAt(LocalDateTime.now().plusWeeks(2));

    return new TokenResponse(newAccessToken, newRefreshToken);
}
```

옛 토큰 문자열은 재발급되는 그 순간 DB 어디에도 존재하지 않는 값이 된다. 만약 리프레시
토큰이 유출되더라도, 유효 기간이 "최대 2주"가 아니라 **"다음 재발급 전까지"**로 확 좁혀진다.

**슬라이딩 만료.** 재발급할 때마다 만료 시각을 `now + 2주`로 다시 잡는다. "활동이 없으면
2주 뒤 자동 로그아웃, 계속 쓰는 한 세션은 유지된다"는 뜻이다. 카카오톡이나 인스타그램 같은
앱들이 쓰는 방식과 같다.

이 설계를 잡을 때 "1시간마다 재발급되면 결국 무한정 로그인 상태가 유지되는 거 아니냐"는
걱정이 있었는데, 실은 그렇지 않다. 매 시점의 **남은 유효기간이 항상 최대 14일로 캡핑**되는
것이지, 기간이 누적되는 게 아니다. 1시간마다 갱신해도 "지금부터 14일"일 뿐 한 달치가 쌓이지
않는다.

### 여기서 실제로 여러 번 틀렸던 계산

처음엔 "슬라이딩이지만 무한정 늘어나면 안 되니 상한을 두자"는 생각으로
`ChronoUnit.DAYS.between()`으로 남은 기간을 계산해서 더하는 방식을 시도했다. 결과적으로
전부 버그였다.

- 기준을 잘못 잡아 뺄셈 부호가 반대로 나오면, 계산된 만료일이 **과거**가 되어 토큰이
  발급되자마자 만료됨
- 순서를 바꿔서 계산하면 사실상 원래 만료일과 거의 같은 값이 나와서, 결국 "슬라이딩"이
  아니라 "고정 만료"와 동일해짐 (아무 의미 없는 계산)
- `saved.getExpiresAt().plusDays(계산값)`처럼 **이전 만료일**을 기준으로 더하면, 재발급할
  때마다 만료일이 무한정 늘어남

세 시도를 겪고 나서 내린 결론은 이거였다: **기준점은 "아예 안 건드림"(고정 만료) 아니면
"지금 시각(`now`)"(슬라이딩) 둘 중 하나여야 한다. "이전 만료일"을 기준으로 삼는 계산은
아무리 정교해도 둘 중 어느 것도 되지 않는다.** 최종 코드는 계산 자체가 필요 없는
`now.plusWeeks(2)` 한 줄이었다.

## userId는 오직 토큰에서만 나온다

인증 미들웨어를 다 만들고 나면, 컨트롤러가 신원을 알아내는 방법은 딱 하나여야 한다.

```java
@AuthenticationPrincipal Long userId
```

body, query parameter, path variable 그 어디에도 클라이언트가 "나는 몇 번이다"라고 주장할
경로가 남아있으면 안 된다. 이게 애초에 문제였던 것을 그대로 반복하는 것이기 때문이다.
액세스 토큰도 body가 아니라 `Authorization` 헤더로만 전달한다 — GET 요청은 애초에 body가
없는 경우가 많고, "누가"(인증)와 "무엇을"(요청 데이터)을 같은 층에 섞으면 모든 DTO마다
토큰 필드를 중복 선언하는 꼴이 되기 때문이다.

## 점진적으로 적용한다

`SecurityConfig`의 `permitAll()`을 마지막까지 유지한 채로 단계를 밟았다.

1. 비밀번호 해시화(BCrypt)
2. 토큰 발급 인프라 구축(로그인 시 발급, 재발급 API)
3. 검증 필터 설치 — **아직 아무것도 막지 않는다**, 토큰이 유효하면 신원만 채워둔다
4. 컨트롤러를 `@AuthenticationPrincipal`로 전환
5. 마지막에 경로별로 `.authenticated()` 강제 적용

이렇게 나눈 이유는 단순하다. 한 번에 다 바꾸면(빅뱅 배포) 어디서 깨졌는지 구분이 안 된다.
필터를 먼저 넣어두고 로그만 찍어보면서 "유효한 토큰일 때 신원이 제대로 채워지는지"를 먼저
확인한 다음에, 그 신원을 실제로 쓰는 컨트롤러 전환으로 넘어가고, 마지막에서야 "토큰 없으면
막는다"는 스위치를 켰다. 중간 단계 어디서든 서버는 계속 정상 동작한다.

---

다음 편에서는 3번(검증 필터)과 4번(컨트롤러 전환) 단계에서 실제로 무슨 코드를 어떻게
바꿨는지, 그리고 그 과정에서 발견한 — 지금 생각해도 아찔한 — 소유권 검증 누락 사례를 다룬다.

**다음 글:** [JWT 인증 도입기 (2) — 필터, 컨트롤러 전환, 그리고 숨어있던 구멍](./JWT-인증-도입기-2-필터와-컨트롤러-전환.md)
