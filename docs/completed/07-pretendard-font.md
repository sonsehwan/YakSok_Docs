# 7. 앱 기본 폰트 Pretendard 적용 (2026-09-02)

**상태:** 완료.
- 선언 계층(7.3~7.4, 리소스+테마) + ViewPump 런타임 주입(7.7) 모두 코드 반영.
- 삼성 Galaxy Z Flip6(One UI, 기기 글꼴 = ChocoCooky)에서 검증 완료 —
  로그인·메인/친구·친구추가 다이얼로그·약속목록·TabLayout·BottomNavigation·EditText 입력값까지
  전부 Pretendard 로 렌더링. `BottomSheetDialogFragment`(7.7의 불확실 항목)도 커버됨.
  Toast·시스템 키보드는 예상대로 시스템 글꼴 유지.

## 7.1 문제

기본 폰트를 정한 적이 없어 기기 시스템 폰트로 렌더링된다. 삼성/LG/픽셀이 서로 다른
글꼴로 보이고, 사용자가 시스템 글꼴을 바꾸면 앱 모양도 같이 흔들린다.
착수 시점에 레이아웃·코드 어디에도 `fontFamily` / `setTypeface` 지정이 없어
덮어쓰는 지점 없이 깨끗하게 얹을 수 있었다.

## 7.2 안드로이드에 "앱 폰트 한 줄 설정"이 없는 이유

전역 폰트 스위치가 없다. 세 계층으로 나눠 건다.

1. **폰트 파일 + weight 매핑 XML** (`res/font/`)
2. **테마의 `fontFamily`** — 일반 `TextView`/`EditText`/`Button` 계열에 전파. 전체의 대부분.
3. **`textAppearance*` 오버라이드** — Material3 컴포넌트(`MaterialButton` ×19,
   `BottomNavigationView` 라벨 ×5 등)는 테마 `fontFamily`를 무시하고 자기
   `textAppearance`에 박힌 `sans-serif`를 쓴다. 15종을 전부 다시 정의해야 잡힌다.

## 7.3 설계 판단

### 번들 vs Downloadable Fonts

Pretendard는 Google Fonts 제공자에 없다 → 안드로이드 스튜디오 Downloadable Fonts 불가.
`.otf` 파일을 직접 번들. `minSdk = 26`이라 OTF·XML `font-family`·테마 `fontFamily`가
전부 네이티브로 동작해서 TTF 변환이나 라이브러리가 필요 없다.

### weight는 400/600/700 세 개

`res/font/pretendard.xml`이 `app:fontWeight`로 세 파일을 매핑. `textStyle="bold"`가
가짜 볼드가 아니라 진짜 Pretendard Bold로 간다. Thin~Black 9종을 다 넣으면 APK가
~12MB 늘어 실사용 안 하는 무게는 뺐다. (Regular+SemiBold+Bold ≈ 4.7MB)

**택하지 않은 것 — Variable 폰트 한 파일:** API 26+에서 되지만 무게별로
`android:fontVariationSettings`를 뷰마다 수동 지정해야 해서 정적 3파일보다 손이 많이 간다.

### 크기는 고정하지 않는다 (접근성)

`textAppearance` 오버라이드는 `parent`의 `textSize`를 그대로 상속하고 `fontFamily`만
바꾼다. 시스템 글자 크기 설정(`sp`)을 계속 따라간다 — 6.4에서 예고한 저시력 사용자
충돌을 피한다.

### `textAppearance` 15종 전부 vs 버튼·바텀네비만

당장 눈에 보이는 건 `MaterialButton`·`BottomNavigationView`뿐이지만, 나중에 Material
다이얼로그나 `TextInputLayout`을 도입하면 그때 또 손봐야 한다. 한 번에 15종
(`type.xml`)을 다 걸고 테마에서 매핑했다.

### AppCompat 다이얼로그

다이얼로그는 전부 `androidx.appcompat.app.AlertDialog`. `textAppearance*` 오버라이드로
대부분 따라오지만 AppCompat 다이얼로그 자체 텍스트 스타일에 `sans-serif`가 박힌
경우가 있어, 안전장치로 `alertDialogTheme` → `ThemeOverlay.App.AlertDialog`
(`fontFamily`만 얹은 오버레이)를 걸었다. 안 걸려도 무해한 no-op.

## 7.4 변경 범위

**프론트엔드 (리소스만, 자바 코드 변경 없음)**

- 신규: `res/font/pretendard_regular.otf`, `pretendard_semibold.otf`, `pretendard_bold.otf`
- 신규: `res/font/pretendard.xml` — weight 매핑 (400/600/700)
- 신규: `res/values/type.xml` — `TextAppearance.App.*` 15종 (Material3 텍스트 어피어런스 + Pretendard)
- 수정: `res/values/themes.xml`
  - `Base.Theme.Medication`에 `android:fontFamily` / `fontFamily`
  - `textAppearance{Display,Headline,Title,Body,Label}{Large,Medium,Small}` 15종 매핑
  - `alertDialogTheme` → `ThemeOverlay.App.AlertDialog` 신규 스타일
  - `TextAppearance.SideMenu.Item`에 `android:fontFamily` 추가
    (부모가 `Material3.BodyLarge`라 명시 안 하면 시스템 폰트로 남았음)

## 7.5 실기기 확인 결과 (삼성 Galaxy Z Flip6, One UI, API 34+)

기기 글꼴을 **ChocoCooky**(삼성 제공 "귀여운" FlipFont)로 설정한 상태에서 테스트.
7.3~7.4 만으로는 **앱 전체가 여전히 ChocoCooky 로 렌더링됨.**

기기에서 직접 실험해 확인한 우선순위:

| 방법 | FlipFont 이김? |
| --- | --- |
| 테마 `android:fontFamily` (7.3) | ❌ |
| `textAppearance*` 15종 오버라이드 (7.3) | ❌ |
| 위젯 기본 스타일(`android:textViewStyle` 등) | ❌ |
| 레이아웃에서 뷰에 직접 `android:fontFamily="@font/pretendard"` | ✅ |
| 코드에서 `setTypeface()` (7.7) | ✅ |

## 7.6 왜 선언 계층이 삼성에서 안 먹히나

삼성 One UI 의 FlipFont 는 사용자가 '설정 > 디스플레이 > 글꼴'을 바꾸면
**테마·스타일 단계에서 해석된 글꼴을 시스템이 자기 글꼴로 덮어쓴다.**
`@font/` 리소스를 가리켜도 마찬가지. 단, **뷰 생성 후 코드로 `setTypeface()` 한 것**과
**레이아웃 XML 속성에 직접 박은 `android:fontFamily`** 는 못 건드린다.

- 레이아웃 51개 × 텍스트 위젯 170곳에 속성을 직접 다는 건 유지보수가 안 됨
  (새 레이아웃마다 까먹음).
- → **inflate 되는 모든 뷰를 가로채 `setTypeface()` 하는 런타임 장치**가 정답.

**택하지 않은 것:**
- `Typeface` 정적 필드 리플렉션 오버라이드 — 최신 One UI 에서 잘 안 먹고 OS 버전마다 깨짐.
- `Application` + `onActivityPreCreated` 로 Factory 설치 — API 29+ 전용.
  이 앱은 고령 사용자 → 구형 폰 호환이 중요해서 전 API(26~) 동작이 필요, 그래서 제외.

## 7.7 해결: ViewPump 로 런타임 강제 주입

`io.github.inflationx:viewpump` — 레이아웃 XML 이 View 로 inflate 되는 순간을 가로채는
라이브러리(API 14+, 전 구간 커버). Calligraphy 없이 커스텀 인터셉터만 사용.

동작 순서:

```
MedicationApp.onCreate()      : ViewPump.init(PretendardInterceptor)
BaseActivity.attachBaseContext: context 를 ViewPumpContextWrapper 로 wrap
setContentView / RecyclerView 어댑터 inflate
  → PretendardInterceptor.intercept(): TextView 면 setTypeface(Pretendard, 기존 style 유지)
```

- 어댑터가 `LayoutInflater.from(parent.getContext())` 를 쓰므로(전 어댑터 확인) 리스트
  아이템도 자동 커버. `AlertDialog.Builder(this)`, 커스텀 `InputView` 도 액티비티 context
  경유라 커버.
- 폰트는 인터셉터에 정적 캐시(`Typeface` 는 불변). `getApplicationContext()` 로 읽어
  액티비티 누수 방지. 기존 `textStyle="bold"` 는 `getTypeface().getStyle()` 로 읽어 유지.

**변경 파일:**
- 신규: `MedicationApp`(Application), `util/PretendardInterceptor`, `ui/base/BaseActivity`
- 수정: `AndroidManifest.xml`(`android:name=".MedicationApp"`),
  `app/build.gradle.kts`(viewpump 의존성),
  액티비티 25개 (`extends AppCompatActivity` → `extends BaseActivity`, import 교체)
- `BottomSheetDialogFragment` 2개(`ShowAddMedicationList`, `SignUpTypeBottomSheet`)는
  `BottomSheetDialog` 자체 context 라 커버 여부 불확실 — 빌드 후 확인, 안 되면 개별 처리.

**7.3~7.4 선언 계층은 유지한다.** ViewPump 가 런타임을 책임지지만, 테마 `fontFamily` 는
Android Studio 레이아웃 프리뷰에서 폰트가 보이게 하고(ViewPump 는 프리뷰에서 안 돎)
비삼성/기본글꼴 경로의 정공법이기도 하다. 중복이지 충돌 아님.

**한계:** `Toast` 는 시스템이 그려서 못 바꿈. 삼성 FlipFont 사용자는 Toast 만 시스템
글꼴로 남는다 — 6.4 "중요 메시지는 다이얼로그로" 와 맞물림.

## 7.8 검증 결과 & 남은 것

**검증(2026-09-02, Galaxy Z Flip6 / One UI / 기기글꼴 ChocoCooky):**
로그인 화면 전체, 메인/친구 탭, "친구 추가" 다이얼로그, 약속목록 + `내 약속`/`공유 약속`
탭, 하단 네비게이션 라벨, `EditText` 입력 텍스트 — **전부 ChocoCooky → Pretendard 전환 확인.**
`BottomSheetDialogFragment` 도 호스트 액티비티 context 경유라 자동 커버됐다.
`Toast` 와 삼성 키보드는 시스템이 그려서 ChocoCooky 유지(범위 밖, 정상).

**남은 것 (선택):**
- `ThemeOverlay.App.AlertDialog` / `alertDialogTheme`(7.3) — ViewPump 가 다이얼로그도
  잡으므로 이제 불필요. 지우면 themes.xml 이 조금 깔끔해짐. 무해하니 급하진 않음.
- `type.xml` 15종 + `textAppearance*` 매핑 — ViewPump 가 런타임을 책임지므로 실기기엔
  영향 없지만, **Android Studio 레이아웃 프리뷰**에서 Pretendard 로 보이려면 필요.
  프리뷰를 신경 안 쓰면 정리 대상, 쓰면 유지. (현재는 유지)
- 굵은 글씨/희귀 한글: 육안 확인 시 문제 없었음. 대량 데이터로 재확인은 여력 될 때.
