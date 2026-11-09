# 14. 하단 네비게이션 Activity → Jetpack Navigation + Fragment 전환

**상태:** 완료. 빌드(`assembleDebug`) 통과. 실기기 확인 완료.

## 문제 정의

하단 탭(홈/기록/채팅/친구/설정)이 5개의 독립된 최상위 Activity(`MainActivity`, `YaksokList`,
`Settings`, `ChatRoomList`, `FriendList`)로 구현돼 있었다. 탭 클릭마다
`startActivity(new Intent(...))` + `FLAG_ACTIVITY_REORDER_TO_FRONT`로 이동했는데, 이 플래그는
기존 인스턴스를 앞으로 가져올 뿐 위에 쌓인 걸 정리하지 않는다. 5개 리스너 전부 `finish()`를
호출하지 않아서, 탭을 옮길 때마다 이전 탭 Activity가 죽지 않고 뒤로가기 스택에 계속 쌓였다
— 뒤로가기를 누르면 앱이 종료되지 않고 예전에 들렀던 다른 탭으로 튕기는 게 "하단 네비 이동이
이상하다"는 증상의 원인이었다. 그 외에도 `BottomNavigationView`가 레이아웃 5곳에 중복
정의되고, `if-else` 탭 전환 리스너가 5곳에 복붙돼 있었다.

## 어떻게 구현했나

호스트 Activity 하나(`MainActivity`) + `NavHostFragment` + `BottomNavigationView` 하나로
합치고, 나머지 4개 탭을 Fragment로 강등했다.

1. `androidx.navigation:navigation-fragment`/`navigation-ui` 의존성 추가(`libs.versions.toml`)
2. 각 탭의 Activity 로직을 Fragment로 이식 (`onCreate` → `onViewCreated`, `this` → `requireContext()`/`requireActivity()`)
3. `res/navigation/nav_graph.xml`에 5개 destination 등록 — id를 `bottom_nav_menu.xml`의
   메뉴 item id(`nav_home`/`nav_history`/`nav_chat`/`nav_friend`/`nav_settings`)와 **동일하게
   맞춰서** `NavigationUI.setupWithNavController(bottomNav, navController)` 한 줄로 자동 연동
4. `activity_main.xml`을 호스트 레이아웃으로 개조: `FragmentContainerView`
   (`android:name="androidx.navigation.fragment.NavHostFragment"`) + `BottomNavigationView`.
   `DrawerLayout`/`NavigationView`(사이드 메뉴)는 호스트에만 남기고 Fragment로 내리지 않음 —
   각 탭 Fragment의 메뉴 버튼은 `requireActivity().findViewById(R.id.drawer_layout)`로 호스트의
   드로어를 연다
5. 옛 Activity 4개(`YaksokList`/`Settings`/`ChatRoomList`/`FriendList`)와 그 레이아웃,
   `AndroidManifest.xml`의 `<activity>` 선언 삭제

### 호스트 바깥에서 특정 탭으로 복귀하는 경우

`SharingPage`(친구 탭)와 `MyInfo`(설정 탭)처럼 `NavHostFragment` 밖의 별도 Activity에서
"탭 화면으로 돌아가기"가 필요한 곳은 `NavController`에 접근할 수 없으므로(Navigation Component는
NavHost 안에서만 통함), `MainActivity`에 `EXTRA_TARGET_TAB`(int, 메뉴/destination 공용 id)를
받는 통로를 만들었다:

```java
public static final String EXTRA_TARGET_TAB = "target_tab";

@Override
protected void onNewIntent(Intent intent){
    super.onNewIntent(intent);
    setIntent(intent);
    applyTargetTabFromIntent(intent);
}

private void applyTargetTabFromIntent(Intent intent){
    int targetTabId = intent.getIntExtra(EXTRA_TARGET_TAB, -1);
    if (targetTabId != -1) {
        bottomNav.setSelectedItemId(targetTabId);
    }
}
```

호출부(`MyInfo`):

```java
Intent intent = new Intent(MyInfo.this, MainActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);
intent.putExtra(MainActivity.EXTRA_TARGET_TAB, R.id.nav_settings);
startActivity(intent);
```

`CLEAR_TOP | SINGLE_TOP` 조합을 쓴 이유: `MainActivity`의 `launchMode`가 기본값(`standard`)이라
`CLEAR_TOP`만 쓰면 기존 인스턴스를 재사용하지 않고 새 인스턴스를 만들어 `onCreate()`가 다시
실행된다(홈 탭 상태가 초기화됨). `SINGLE_TOP`을 같이 줘야 기존 인스턴스를 재사용하며
`onNewIntent()`로 전달된다.

## 왜 그렇게 판단했나

- **호스트+NavHostFragment 구조를 택한 이유** — 탭 전환이 Activity 재생성이 아니라 같은
  Activity 안에서 View만 바뀌는 것이 되어, `REORDER_TO_FRONT` 같은 백스택 편법 자체가 필요
  없어진다. `BottomNavigationView`/드로어도 호스트에 하나만 있으면 되어 5곳 중복이 사라진다.
- **`NavigationUI` id 매칭 방식을 택한 이유** — 하단 탭처럼 "고정된 목록에서 하나를 고르는"
  상황에 정확히 맞는 표준 헬퍼라, 직접 리스너를 짜는 것보다 코드가 훨씬 짧고 백스택 처리
  (`setLaunchSingleTop`/`setPopUpTo`)까지 대신해준다.
- **호스트 바깥 화면은 Intent+extra로 우회한 이유** — Navigation Component의 `NavController`는
  `NavHostFragment` 안에서만 접근 가능하다. `SharingPage`/`MyInfo`는 호스트 밖의 별도 Activity라
  이 경로를 못 쓴다.

## 검토했지만 택하지 않은 대안

- **`FragmentManager`를 직접 써서 수동으로 `beginTransaction().replace()`만 하는 방법** —
  가능은 하지만 백스택·애니메이션을 전부 손으로 관리해야 해서, 지금 겪는 "이동이 이상함"
  문제를 다른 형태로 재현할 위험이 있다고 판단해 Navigation Component를 택함.
- **`MainActivity`에 `android:launchMode="singleTop"`을 매니페스트에 전역 선언** — 매번 Intent
  플래그를 안 붙여도 되지만, `MainActivity`를 여는 다른 모든 곳(`Login`, `CreateDirectSchedule`,
  `CreatePaperEnvelope`, `CreatePrescription` 등)의 동작에도 영향을 주는 앱 전역 변경이라 범위가
  커진다. 지금 필요한 곳(`SharingPage`, `MyInfo`) 2곳에만 Intent 플래그로 국소 적용.

## 변경 범위

| 구분 | 내용 |
| --- | --- |
| 신규 Fragment | `HomeFragment`, `YaksokListFragment`, `SettingsFragment`, `ChatRoomListFragment`, `FriendListFragment` (+ 각각의 `fragment_*.xml`) |
| 신규 리소스 | `res/navigation/nav_graph.xml`, `strings.xml`에 탭 라벨 5개 |
| 개조 | `MainActivity`(순수 호스트화 + `EXTRA_TARGET_TAB` 처리), `activity_main.xml`(`NavHostFragment`+`BottomNavigationView`+드로어) |
| 삭제 | `YaksokList`/`Settings`/`ChatRoomList`/`FriendList` Activity 4개 + 레이아웃 4개, `AndroidManifest.xml`의 해당 `<activity>` 4개 |
| 호출부 수정 | `SharingPage`(친구 탭 진입), `MyInfo`(설정 탭 복귀) — 옛 Activity 클래스 참조를 `MainActivity` + `EXTRA_TARGET_TAB`으로 교체 |
| 별개 삭제 | `SharingPage`/`activity_sharing_page.xml` — 이번 작업 중 안 쓰는 것으로 확인되어 소유자가 별도로 삭제(이 마이그레이션의 직접 목적은 아님) |

## 알려진 제약 / 후속

- 상세·수정 화면(`YaksokDetail`, `ChattingRoom`, `DrugStoreDetail`, `MyInfo` 등)은 여전히
  `NavHostFragment` 밖의 별도 Activity로 남아있다. 이번 범위는 하단 탭 5개로 한정했고,
  이 화면들까지 Fragment화할지는 별도 판단.
- 호스트 바깥→특정 탭 복귀 패턴(`EXTRA_TARGET_TAB`)은 지금 `SharingPage`/`MyInfo` 2곳에만
  적용돼 있다. 비슷한 요구가 다른 화면에서도 생기면 같은 패턴을 재사용하면 된다.
- 탭 전환 애니메이션을 없애려면(구 Activity 시절의 `overridePendingTransition(0,0)`에 대응)
  `NavigationUI.setupWithNavController` 대신 `bottomNav.setOnItemSelectedListener`를 직접 달고
  `NavOptions`에 `setEnterAnim(0)` 등을 명시해야 한다 — 이번 작업 범위엔 포함 안 함.
