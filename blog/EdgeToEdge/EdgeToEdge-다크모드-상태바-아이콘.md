# 라이트 테마인데 상태바 글자가 흰색? EdgeToEdge가 테마가 아니라 uiMode를 본다

## 증상

안드로이드 앱의 테마를 `Theme.Material3.Light`로 쓰고 있는데, 폰이 다크모드일 때 일부 화면에서
상태바 글자(시간, 배터리)가 **흰색**으로 나왔다. 배경이 흰색이라 글자가 거의 안 보였다.

이상한 점은 화면마다 달랐다는 것이다. 로그인·회원가입 화면은 멀쩡했고, 메인·채팅·약국·약 검색
화면에서만 흰색이었다.

## 먼저 의심한 것: 테마

테마는 이미 라이트였다.

```xml
<!-- values/themes.xml -->
<style name="Base.Theme.Medication" parent="Theme.Material3.Light.NoActionBar">
    <item name="android:windowLightStatusBar">true</item>
    ...
</style>
```

`windowLightStatusBar`도 `true`(= 밝은 상태바 → 어두운 아이콘)였고, `values-night/themes.xml`도 같은 값으로
맞춰져 있었다. 게다가 `BaseActivity`가 코드로 한 번 더 지정하고 있었다.

```java
// BaseActivity
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    lockPortrait();
    applyLightSystemBarIcons();   // setAppearanceLightStatusBars(true) ...
}
```

테마도, 베이스 클래스도 "어두운 아이콘"을 요청하는데 결과는 흰색이었다. 누군가 뒤에서 덮어쓰고 있다는 뜻이다.

## 범인: EdgeToEdge.enable(this)

흰색이던 화면들의 공통점을 찾아보니 전부 `onCreate`에서 `EdgeToEdge.enable(this)`를 호출하고 있었다.

```java
public class MainActivity extends BaseActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);   // BaseActivity: 어두운 아이콘 지정
        EdgeToEdge.enable(this);              // ← 여기서 다시 덮어쓴다
        ...
```

호출 순서가 `BaseActivity`의 `true` → 하위 클래스의 `EdgeToEdge.enable`이라서 뒤쪽이 이긴다.
`EdgeToEdge`를 쓰지 않던 로그인·회원가입은 `BaseActivity`의 값이 그대로 남아 정상이었다.

## EdgeToEdge는 무엇을 하나

`EdgeToEdge.enable(activity)`는 두 가지 일을 한다.

1. **화면을 시스템바 뒤까지 확장한다.** 콘텐츠가 상태바와 내비게이션 바 밑까지 그려진다.
   (바 밑으로 들어가지 않게 패딩을 주는 건 앱 몫이다. `fitsSystemWindows`나 인셋 리스너가 필요하다.)
2. **시스템바의 배경색과 아이콘 색을 정한다.** 인자를 안 주면 기본 스타일은 `SystemBarStyle.auto(...)`다.

| | 낮 | 밤 |
|---|---|---|
| 상태바 배경 | 투명 | 투명 |
| 상태바 아이콘 | 어두운 색 | 흰색 |
| 내비바 배경 | 3버튼 방식이면 밝은 반투명 스크림, 제스처 방식이면 투명 | 어두운 반투명 스크림, 제스처 방식이면 투명 |
| 내비바 아이콘 | 어두운 색 | 흰색 |

핵심은 `auto`가 낮/밤을 **무엇으로 판단하느냐**다. 테마가 아니라 앱 리소스의
`Configuration.uiMode`의 night 플래그를 본다. 이 값은 앱이 따로 지정하지 않으면 폰의 다크모드 설정을
그대로 따른다.

정리하면 이렇다.

- 테마(`Theme.Material3.Light`)는 **앱이 그리는 색**을 라이트로 고정한다.
- `uiMode`는 여전히 **night**다. 앱이 라이트를 강제하지 않았기 때문이다.
- `EdgeToEdge(auto)`는 `uiMode`만 보고 "밤이니까 흰 아이콘"을 고른다.

"앱 UI는 라이트인데 시스템 쪽 판단은 다크"인 상태가 만들어진 것이다.

## 해결

### 1. 앱을 라이트 전용으로 못 박는다

```java
public class MedicationApp extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO);
        ...
    }
}
```

`MODE_NIGHT_NO`는 폰이 다크모드여도 이 앱에서는 항상 낮 모드로 취급하라는 설정이다.
앱이 받는 `uiMode`가 낮이 되므로 `EdgeToEdge(auto)`가 어두운 아이콘을 고른다.

`BaseActivity.onCreate`가 아니라 `Application.onCreate`에 둔 이유가 있다. 액티비티 안에서 바꾸면
이미 night로 만들어진 액티비티를 AppCompat이 다시 만들면서 깜빡일 수 있다. `Application.onCreate`는
어떤 액티비티보다 먼저 실행된다.

앱 안에서만 적용되고 폰 전체나 다른 앱에는 영향이 없다. 사용자가 앱 안에서 다크모드를 고르는 기능도
사라지는데, 어차피 다크 테마가 없는 앱이라 문제가 되지 않았다.

### 2. EdgeToEdge 호출을 BaseActivity 한 곳으로 모은다

각 화면이 따로 호출하던 `EdgeToEdge.enable(this)` 9곳을 지우고 베이스 클래스에 한 번만 둔다.

```java
// BaseActivity
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    lockPortrait();
    EdgeToEdge.enable(this);
}
```

그러면 `applyLightSystemBarIcons()`는 필요가 없어진다. `EdgeToEdge`의 기본 스타일이 상태바와 내비바 아이콘을
이미 "낮이면 어두운 색"으로 정해주기 때문이다. 메서드와 `WindowCompat`,
`WindowInsetsControllerCompat` import를 지웠다.

이렇게 모으면 `BaseActivity`를 상속하는 모든 화면이 한 번에 같은 규칙을 적용받는다. 나중에 새 화면에서
호출을 빼먹어서 같은 버그가 다시 생기는 일도 막을 수 있다.

## 고민했던 것들

**9곳에 스타일을 명시하는 방법.** `EdgeToEdge.enable(this, SystemBarStyle.light(TRANSPARENT, TRANSPARENT), ...)`
처럼 각 호출에 라이트 스타일을 직접 넘길 수도 있었다. 하지만 같은 수정을 9번 하고, 새 화면에서
인자를 빼먹으면 버그가 되살아난다. "이 앱은 라이트 전용"이라는 의도를 한 곳에 적는 쪽이 낫다고 판단했다.

**`applyLightSystemBarIcons()`를 남기는 방법.** 하위 액티비티의 `EdgeToEdge.enable`이 그 뒤에서
덮어쓰는 구조에서는 의미가 없고, `BaseActivity`가 `EdgeToEdge`를 직접 호출하게 되면 중복이다.

**EdgeToEdge를 안 쓰던 14개 화면은 괜찮은가.** 이제 이 화면들도 `EdgeToEdge`를 받는다. 상태바가 투명해지고
콘텐츠가 바 뒤까지 확장되므로 겹침이 걱정됐다. 확인해 보니 이 화면들은 이미 인셋을 처리하고 있었다. 대부분
루트 레이아웃에 `fitsSystemWindows="true"`가 있었고, 웹뷰를 쓰는 화면 하나는 직접 리스너를 달아 둔 상태였다.
Android 15 이상은 어차피 edge-to-edge가 강제되므로, 실제로 달라지는 건 Android 14 이하의 상태바
투명화 정도다.

## 정리

- 테마를 Light로 해도 `Configuration.uiMode`는 폰의 다크모드를 따른다.
- `EdgeToEdge.enable()`의 기본 스타일(`auto`)은 **테마가 아니라 `uiMode`** 로 시스템바 아이콘 색을 정한다.
- 라이트 전용 앱이라면 `AppCompatDelegate.setDefaultNightMode(MODE_NIGHT_NO)`로 `uiMode` 자체를 낮으로 고정해야 한다.
- 베이스 액티비티와 하위 액티비티가 같은 시스템바 설정을 각자 건드리면 호출 순서에 따라 결과가 뒤집힌다.
  한 곳에서만 설정한다.

나중에 다크모드를 지원하게 되면 `MODE_NIGHT_NO`를 `FOLLOW_SYSTEM`으로 바꾸면 된다. 그러면 `EdgeToEdge`도
시스템 모드를 따라 아이콘 색이 자동으로 바뀐다. 단 그때는 `values-night` 테마, 색 리소스, 다이얼로그가
다크에서도 읽히는지 먼저 챙겨야 한다.
