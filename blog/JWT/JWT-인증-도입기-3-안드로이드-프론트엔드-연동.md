# JWT 인증 도입기 (3) — 안드로이드에서 토큰 자동 갱신하기

[지난 글](./JWT-인증-도입기-2-필터와-컨트롤러-전환.md)까지 백엔드는 끝났다. 이제 안드로이드
앱이 이 토큰을 실제로 써야 한다. 이번 글은 OkHttp의 `Interceptor`와 `Authenticator`로
"토큰 자동 첨부"와 "401 자동 재발급"을 구현한 과정, 그리고 그 과정에서 제일 헷갈렸던
재귀 호출 문제를 다룬다.

## 역할을 둘로 나눈다: Interceptor와 Authenticator

OkHttp에는 이 문제에 정확히 들어맞는 확장점이 두 개 있다.

- **`Interceptor`** — 요청이 나가기 *전에* 무조건 끼어든다
- **`Authenticator`** — 서버가 **401을 응답했을 때만** OkHttp가 자동으로 호출해준다

이 둘의 책임을 섞지 않고 나눴다. "모든 요청에 토큰을 붙이는 일"과 "토큰이 만료됐을 때
재발급받고 재시도하는 일"은 성격이 다른 일이다.

## Interceptor — 모든 요청에 토큰 붙이기

```java
public class AuthInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request original = chain.request();
        String accessToken = SprefsManager.getAccessToken(MedicationApp.getContext());

        if (accessToken == null) {
            return chain.proceed(original);
        }

        Request authorized = original.newBuilder()
                .header("Authorization", "Bearer " + accessToken)
                .build();

        return chain.proceed(authorized);
    }
}
```

여기서 `MedicationApp.getContext()`가 등장하는 이유가 있다. `Interceptor`는 OkHttp
내부에서 호출되는 콜백이라 특정 화면(Activity)에 속하지 않는다. 토큰을 읽으려면
`Context`가 필요한데, 어느 화면에서도 이 값을 넘겨줄 수가 없다. 그래서 `Application`
서브클래스에 앱 전체 수명을 가진 `Context`를 static으로 하나 노출해뒀다.

```java
public class MedicationApp extends Application {
    private static Context context;

    @Override
    public void onCreate() {
        super.onCreate();
        context = getApplicationContext();
    }

    public static Context getContext() {
        return context;
    }
}
```

로그인/회원가입처럼 인증이 필요 없는 요청에도 토큰을 붙일지 굳이 URL로 분기하지 않았다.
백엔드의 `JwtAuthenticationFilter`가 토큰이 없거나 무효해도 예외 없이 그냥 통과시키도록
만들어져 있고, 그런 경로는 어차피 `permitAll`이라 무해하기 때문이다.

## Authenticator — 401을 받으면 재발급하고 재시도

```java
public class TokenAuthenticator implements Authenticator {

    private static final String BASE_URL = "http://...";

    @Override
    public Request authenticate(Route route, Response response) throws IOException {
        if (responseCount(response) >= 2) {
            return null;
        }

        String refreshToken = SprefsManager.getRefreshToken(MedicationApp.getContext());
        if (refreshToken == null) {
            return null;
        }

        AuthApi authApi = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(AuthApi.class);

        retrofit2.Response<ApiResponse<TokenResponse>> result =
                authApi.refresh(new RefreshTokenRequest(refreshToken)).execute();

        if (!result.isSuccessful() || result.body() == null || !result.body().isBusinessSuccess()) {
            forceLogout();
            return null;
        }

        TokenResponse newTokens = result.body().getData();
        SprefsManager.saveTokens(MedicationApp.getContext(),
                newTokens.getAccessToken(), newTokens.getRefreshToken());

        return response.request().newBuilder()
                .header("Authorization", "Bearer " + newTokens.getAccessToken())
                .build();
    }

    private int responseCount(Response response) {
        int count = 1;
        while ((response = response.priorResponse()) != null) count++;
        return count;
    }
}
```

`authenticate()`가 `Request`를 반환하면 OkHttp가 그 요청으로 자동 재시도하고, `null`을
반환하면 포기하고 원래 401을 호출한 쪽에 그대로 넘긴다. 이 구조 덕분에 화면 코드는
"토큰이 만료됐으면 재발급받고 재시도한다"는 로직을 단 한 줄도 알 필요가 없다.

몇 가지 세부 결정에는 이유가 있다.

- **`execute()`를 쓴 이유** — `authenticate()`는 반환값을 즉시 줘야 하는데, `enqueue()`는
  결과가 콜백으로 나중에 온다. 여기서는 결과를 기다려야 해서 동기 호출이 필요하다.
  `authenticate()` 자체가 OkHttp의 백그라운드 스레드에서 실행되니 여기서 블로킹해도
  메인 스레드는 안 막힌다.
- **`responseCount()` 가드** — 재발급받은 새 토큰으로 재시도했는데도 또 401이 나는 극단적인
  경우(예: 재시도 도중 다른 곳에서 이미 로그아웃)를 대비해, 한 번 재시도한 흔적이 있으면
  더 이상 시도하지 않고 포기한다.

## 제일 헷갈렸던 부분: 재발급 요청은 왜 별도 Retrofit으로 보내야 하는가

이 코드에서 가장 이해하기 어려웠던 지점이 여기다. `authApi.refresh(...)`를 부를 때 왜
`NetworkClient`가 이미 갖고 있는 공유 Retrofit(`getApi()` 등)을 안 쓰고, 굳이
`new Retrofit.Builder()`로 매번 새로 만드는가.

이유를 이해하려면 먼저 `NetworkClient`의 구조를 봐야 한다.

```java
private static OkHttpClient getHttpClient() {
    return new OkHttpClient.Builder()
            .addInterceptor(new AuthInterceptor())
            .authenticator(new TokenAuthenticator())
            .build();
}
```

앱의 모든 Retrofit 인터페이스(`UserApi`, `YaksokApi`, `ChatApi`...)는 결국 이 하나의
`OkHttpClient`를 공유한다. 만약 `TokenAuthenticator` 안에서 이 공유 클라이언트로 재발급
요청을 보냈다고 해보자.

```
1. 화면이 YaksokApi.getYaksokList() 호출
2. 서버가 401 응답 (액세스 토큰 만료)
3. OkHttp가 TokenAuthenticator.authenticate()를 자동 실행 (①)
4. ① 안에서 공유 클라이언트로 /api/auth/refresh 호출
5. 그런데 리프레시 토큰마저 무효하다면? → 서버가 이 요청에도 401을 응답
   (실제로 백엔드가 그렇게 설계돼 있다 — 리프레시 실패 시 401 + REFRESH_FAIL)
6. OkHttp: "어? 이 요청도 401이네" → authenticate()를 또 실행 (②)
7. ② 안에서도 또 refresh()를 시도 → 또 401 → 또 authenticate() 호출...
```

재발급을 요청하는 행위 자체가, 재발급이 필요한 401 응답을 만들어내고, 그게 다시 같은
`authenticate()`를 호출하게 만드는 **꼬리를 무는 구조**가 생긴다. 앞서 만든
`responseCount()` 가드는 "같은 요청이 재시도된 횟수"만 세는데, 이 두 번째 호출은
원래 요청(`getYaksokList`)의 재시도가 아니라 **refresh 요청 자신의 재시도**라서 별개의
카운터로 취급된다 — 즉 이 가드로는 못 막는다.

해결은 의외로 간단했다. `Retrofit.Builder()`에 `.client(...)`를 지정하지 않으면, Retrofit이
내부적으로 **아무 설정도 없는 순수한 `new OkHttpClient()`**를 자동으로 만들어서 쓴다.

```java
// Retrofit 내부 동작 (개념적으로)
okhttp3.Call.Factory callFactory = this.callFactory;
if (callFactory == null) {
    callFactory = new OkHttpClient();
}
```

이 클라이언트에는 `AuthInterceptor`도 `TokenAuthenticator`도 붙어있지 않다. 그래서 재발급
요청이 401을 받아도 물어볼 `Authenticator`가 없어서, 그냥 평범한 실패 응답으로 끝난다.
"재발급은 성공 아니면 포기" — 재귀가 발생할 통로 자체가 없어진다.

## 재발급도 실패하면: 강제 로그아웃

리프레시 토�큰까지 무효해서 재발급 자체가 실패하면, 로컬에 남은 죽은 토큰을 지우고
사용자를 로그인 화면으로 돌려보내야 한다. 문제는 `TokenAuthenticator`가 OkHttp 백그라운드
스레드에서 실행되기 때문에 **어떤 화면이 지금 떠 있는지 전혀 모른다**는 점이다.

이 앱에는 이미 같은 문제(백그라운드에서 일어난 일을 지금 떠 있는 화면에 알려서 반응하게
만들기)를 풀어둔 패턴이 있었다 — FCM 갱신 신호를 처리하는 `YaksokEventBus`. 그 패턴을
그대로 가져와 `AuthEventBus`를 만들었다.

```java
public final class AuthEventBus {
    public interface Listener { void onForceLogout(); }

    private static final AuthEventBus INSTANCE = new AuthEventBus();
    public static AuthEventBus get() { return INSTANCE; }

    private final Handler mainHandler = new Handler(Looper.getMainLooper());
    private final List<Listener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(Listener l) { if (l != null && !listeners.contains(l)) listeners.add(l); }
    public void unsubscribe(Listener l) { listeners.remove(l); }

    public void publish() {
        mainHandler.post(() -> { for (Listener l : listeners) l.onForceLogout(); });
    }
}
```

`TokenAuthenticator`는 "세션이 죽었다"는 사실만 발행한다. 실제로 화면을 전환하는 건
모든 화면이 상속하는 `BaseActivity`의 몫이다.

```java
public class BaseActivity extends AppCompatActivity implements AuthEventBus.Listener {

    @Override
    protected void onResume() {
        super.onResume();
        AuthEventBus.get().subscribe(this);
    }

    @Override
    protected void onPause() {
        super.onPause();
        AuthEventBus.get().unsubscribe(this);
    }

    @Override
    public void onForceLogout() {
        if (this instanceof Login) return;

        Toast.makeText(this, "로그인이 만료되었습니다. 다시 로그인해주세요.", Toast.LENGTH_SHORT).show();
        Intent intent = new Intent(this, Login.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);
        finish();
    }
}
```

`BaseActivity`에 한 곳에만 넣으면 앱의 모든 화면에 자동으로 적용된다. `onResume`/`onPause`
짝으로 구독하기 때문에, 안드로이드가 한 시점에 `RESUMED` 상태인 액티비티를 하나만 유지한다는
성질 덕분에 **지금 사용자가 보고 있는 화면 하나만** 반응한다 — 토스트가 중복으로 뜨거나
로그인 화면 이동이 여러 번 겹치는 걸 별도 판단 코드 없이 피할 수 있다.

리프레시 토큰이 애초에 없던 경우(최초 미로그인, 이미 로그아웃, 혹은 동시에 실패한 다른
요청이 이미 처리한 직후)는 이 이벤트를 발행하지 않았다. 끊을 세션 자체가 없는 상태라
발행해봐야 중복 토스트만 늘어나기 때문이다.

## 클라이언트가 자기 신원을 다시 말할 필요가 없어졌다

마지막으로 한 일은, `SprefsManager`에서 로컬 `userId`를 꺼내 API 파라미터로 넘기던 코드를
전부 인자 없는 호출로 바꾸는 작업이었다.

```java
// Before
api.getYaksokList(userId).enqueue(callback);

// After
api.getYaksokList().enqueue(callback);
```

서버가 이제 토큰에서 신원을 직접 판별하니, 클라이언트가 자기 자신을 다시 알려줄 필요가
없다. 오히려 로컬에 저장된 값은 조작 가능해서 애초에 신뢰할 수 없는 값이었다 — 이 작업
전체의 출발점이 그거였으니, 마지막까지 일관되게 정리한 셈이다.

---

여기까지가 전체 설계와 구현이다. 마지막 글에서는 이 과정에서 실제로 만났던 버그들을
모아본다 — 특히 마지막에 발견한, "기능은 동작하는데 매 요청이 몰래 두 번 왕복하고 있던"
버그는 꽤 재밌는 사례였다.

**다음 글:** [JWT 인증 도입기 (4) — 삽질 총정리](./JWT-인증-도입기-4-트러블슈팅.md)
