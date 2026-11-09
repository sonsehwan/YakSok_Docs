# 29. 다크모드 폰에서 상단바 아이콘이 흰색으로 나오던 문제

## 문제

폰이 다크모드일 때 `MainActivity`·채팅·약국·약 검색 등 일부 화면에서 상태바 글자(시간·배터리)가
흰색으로 나와 흰 배경 위에서 안 보였다. 앱 UI 자체는 `Theme.Material3.Light`라 라이트다.

## 원인

테마가 Light여도 앱이 받는 `Configuration.uiMode`는 폰 설정(night)을 그대로 따른다. 앱에 라이트를
강제하는 코드(`AppCompatDelegate.setDefaultNightMode`)가 없었기 때문이다.

`EdgeToEdge.enable(this)`를 인자 없이 부르면 상태바 스타일이 `SystemBarStyle.auto(...)`가 되는데,
auto는 테마를 보지 않고 `uiMode`가 night인지만 본다. night면 `isAppearanceLightStatusBars = false`
(흰 아이콘)로 세팅한다. 호출 순서가 `BaseActivity.onCreate`의 `setAppearanceLightStatusBars(true)` →
하위 액티비티의 `EdgeToEdge.enable(this)`였으므로 뒤쪽이 이겨서 `BaseActivity`의 값이 덮였다.

`EdgeToEdge`를 안 쓰던 14개 화면(Login·Signup 등)은 `BaseActivity` 값이 그대로 남아 정상이었다.
그래서 증상이 화면에 따라 갈렸다.

## 판단

- **라이트 고정: `MedicationApp.onCreate`에 `MODE_NIGHT_NO`.** 앱이 받는 `uiMode`를 낮으로 만들면
  `EdgeToEdge`의 auto가 스스로 어두운 아이콘을 고른다. `Application.onCreate`에 둔 이유는 어떤
  액티비티보다 먼저 실행되어 이미 night로 만들어진 액티비티를 AppCompat이 재생성(깜빡임)하지 않기 때문.
- **`EdgeToEdge.enable(this)`를 `BaseActivity.onCreate`로 한 곳에 모으고 개별 9곳은 삭제.**
  `BaseActivity`를 상속하는 모든 화면이 한 번에 적용받는다. `applyLightSystemBarIcons()`는 삭제 —
  `EdgeToEdge`의 기본 스타일이 상태바와 내비바 아이콘을 같은 방향으로 정하므로 중복이다.
- **겹침 위험은 낮다고 봤다.** 새로 `EdgeToEdge`를 받는 14개 화면은 모두 인셋을 처리하고 있다
  (13개는 루트 `fitsSystemWindows="true"`, `AddressSearch`는 직접 리스너,
  `CreatePaperEnvelope`는 `activity_create_prescription` 레이아웃 재사용). Android 15+는 어차피
  edge-to-edge가 강제돼 달라지는 건 Android 14 이하의 상태바 투명화뿐이다.

### 택하지 않은 대안

- **9곳에서 `SystemBarStyle.light(...)`를 명시.** 같은 수정을 9번 하고, 새 화면에서 인자를 빼먹으면
  버그가 되살아난다. 라이트 전용이라는 의도를 한 곳에 적는 쪽이 낫다고 판단.
- **14개 화면에 각각 `EdgeToEdge.enable` 추가.** `BaseActivity` 한 곳이면 끝나는 일이라 기각.
- **`applyLightSystemBarIcons()`를 남겨두기.** 하위 액티비티의 `EdgeToEdge.enable`이 뒤에서 덮는 구조라
  의미가 없고, `BaseActivity`가 `EdgeToEdge`를 직접 호출하게 되면 중복이다.

## 변경

- 프론트 `MedicationApp.onCreate`: `AppCompatDelegate.setDefaultNightMode(MODE_NIGHT_NO)`.
- 프론트 `BaseActivity.onCreate`: `applyLightSystemBarIcons()` 호출 → `EdgeToEdge.enable(this)`.
  메서드와 `WindowCompat`·`WindowInsetsControllerCompat` import 삭제.
- 프론트 `MainActivity`·`ChattingRoom`·`WipActivity`·`ModifyPassword`·`MedicineSearchActivity`·
  `DrugStoreList`·`DrugStoreDetail`·`ShareYaksokDetail`·`ShareYaksokList`: 개별 `EdgeToEdge.enable(this)`와
  import 삭제.

## 주의 (나중에 다크모드를 지원할 때)

상태바 아이콘 색은 이제 **`MedicationApp`의 `MODE_NIGHT_NO` + `BaseActivity`의 `EdgeToEdge.enable`**
두 곳이 같이 정한다. 다크모드를 지원하려고 `MODE_NIGHT_NO`를 지우거나 `FOLLOW_SYSTEM`으로 바꾸면
`EdgeToEdge`의 auto가 night를 따라 아이콘이 시스템 모드에 맞게 바뀌는데, 이는 의도된 동작이지만
`values-night/themes.xml`·색 리소스·다이얼로그 등이 다크를 제대로 지원한다는 전제가 필요하다.

`PortraitCaptureActivity`(QR 스캔, ZXing `CaptureActivity` 상속)는 `BaseActivity`를 상속하지 않아
원래부터 이 처리의 대상이 아니었다.

## 상태

컴파일 통과. 실기기(다크모드 폰) 확인 완료.
