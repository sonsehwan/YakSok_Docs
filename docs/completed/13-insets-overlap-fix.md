# 13. 액티비티 상태바/내비게이션 바 겹침 정리 (2026-09-13)

**상태:** 발견한 버그 6개 화면 전부 코드 반영 완료. 컴파일 확인 후
**실기기 육안 확인까지 완료.** 이후 하단 네비 5개 탭이 Fragment로 전환되면서(14장) 방식 B의
동작 원리가 달라졌다 — 13.2는 **현재 구조 기준**으로 갱신했고, 전환 전 설명은 13.2.1에 남겼다.
`FriendList`를 되돌릴지 하는 문제는 Fragment 전환으로 해소됐다(13.6 끝 참고).

## 13.1 발단

전체 액티비티를 대상으로 상태바 겹침 여부를 조사해달라는 요청에서 시작. `targetSdk 36`
(Android 15+)은 모든 액티비티에 edge-to-edge를 강제하므로, `WindowInsets`를 받아 직접 패딩으로
처리하지 않는 화면은 무조건 상태바/내비게이션 바 밑에 콘텐츠가 깔린다.

## 13.2 이 앱이 쓰는 세 가지 처리 방식

| 방식 | 동작 | 적용 대상 |
|---|---|---|
| A. `android:fitsSystemWindows="true"` | 리스너 없이도 `View`의 기본 구현이 인셋을 4면 다 패딩으로 넣고 **소비**한다. 소비되면 그 아래 자식은 `dispatchApplyWindowInsets` 호출 자체를 못 받는다 | 하단 네비 호스트 밖의 일반 Activity 화면 (레이아웃 19개 — 로그인/가입, 약국 상세·목록, 공유약속 목록·상세, 약 검색 등) |
| B. `InsetsUtil.applySystemBarPadding()` | 직접 리스너를 걸어 top/left/right만 패딩, bottom은 0으로 두고 **소비하지 않고 반환** | 하단 네비 호스트 안의 탭 Fragment 5개(`HomeFragment`, `YaksokListFragment`, `FriendListFragment`, `SettingsFragment`, `ChatRoomListFragment`) |
| C. 화면 안에서 직접 `ViewCompat.setOnApplyWindowInsetsListener` 작성 | B와 API는 같고 헬퍼 없이 그 자리에서 필요한 방향·대상 뷰만 처리 | `MainActivity`(`bottomNavigation`에 bottom만), `WipActivity`(4면), `AddressSearch`(`android.R.id.content`에 4면, 12장에서 밝혀졌듯 현재 어디서도 호출 안 하는 죽은 코드), `PortraitCaptureActivity`(ZXing `statusView`에만 bottom) |

B는 C의 "위·좌·우 패딩 + bottom 0" 패턴 하나에 이름을 붙여 묶은 것이다. 공용 헬퍼가 이 패턴
하나뿐인 이유는 이 패턴만 5곳에서 반복되기 때문이고, 나머지는 한두 곳이라 묶을 이유가 없다.

### 13.2.1 B의 bottom=0이 안전한 이유 — 구조에 따라 다르다

**현재 구조 (14장 이후).** 하단 네비는 탭 Fragment의 자식이 아니라 **형제**다:

```
DrawerLayout
 └ host_root (ConstraintLayout — 리스너/fitsSystemWindows 없음, 인셋을 소비 안 하고 통과)
     ├ nav_host_fragment   ← 탭 Fragment (루트에 방식 B)
     ├ bottom_navigation_divider
     └ bottom_navigation   ← MainActivity.java에서 방식 C로 bottom 인셋을 직접 적용
```

Fragment 영역은 하단 네비 위에서 끝나 화면 맨 아래에 닿지 않는다. 여기에 내비바 높이만큼
bottom 패딩을 또 주면 같은 높이를 Fragment 안쪽(빈 공간)과 하단 네비(자체 패딩)에서 두 번
비우게 된다. 그래서 Fragment는 bottom=0, 하단 네비만 bottom 인셋을 가져간다.

**14장 이전 구조 (Activity 시절).** `BottomNavigationView`가 각 Activity 루트의 **자식**이었다.
B가 bottom을 0으로 두고 소비하지 않은 채 반환하면, 자식인 `BottomNavigationView`가 Material
내장 리스너로 같은 인셋 객체를 받아 자기 bottom 패딩을 처리했다. 코드(`InsetsUtil`)는 바뀐
게 없고, bottom=0이 안전한 이유만 "자식이 처리"에서 "Fragment가 바닥에 안 닿음"으로 바뀌었다.

`MainActivity`의 방식 C 리스너가 `BottomNavigationView`의 Material 내장 리스너를 덮어쓰는지는
확인하지 않았다(추정만 함).

## 13.3 발견한 버그

### `DrugStoreDetail` — 상단바 겹침

세 방식 중 아무것도 안 쓰고 있었다. `EdgeToEdge.enable(this)`는 호출하면서도 루트
`LinearLayout`에 `fitsSystemWindows`도 없고 인셋 리스너도 전혀 없어서, 뒤로가기 버튼과
콘텐츠가 상태바 밑에 깔렸다.

### `ShareYaksokList` / `ShareYaksokDetail` / `DrugStoreList` / `MedicineSearchActivity` — 하단 겹침

(당시 구조 기준 서술.) 방식 B(`InsetsUtil`, bottom=0)를 쓰고 있었는데, 이 네 화면 레이아웃에는
`BottomNavigationView`가 **없다.** bottom 인셋을 "자식이 알아서 처리하겠지" 하고 넘겼는데 받아줄 자식이 없어서, 아무도
bottom을 처리하지 않는 상태였다. `ShareYaksokDetail`은 "저장하기" 버튼이 `parent` 바닥에
`layout_constraintBottom_toBottomOf="parent"`로 붙어 있어 내비게이션 바에 가려지는 게 가장
확실한 사례였고, 나머지 셋은 `RecyclerView`에 하드코딩된 `paddingBottom="20dp"`(인셋과 무관한
고정값)가 우연히 어느 정도 가려줬을 뿐이라 내비 바가 20dp보다 크면 마지막 항목이 가려질 수 있는
상태였다.

## 13.4 검토했지만 택하지 않은 대안

### `BaseActivity`에 인셋 처리를 공통으로 넣기

모든 액티비티가 `BaseActivity`를 상속하니 거기서 한 번에 처리하면 어떤지 검토했다. 기각한 이유
두 가지:

1. **bottom 처리 주체가 화면마다 다르다.** `BottomNavigationView`가 있는 화면은 bottom을 그
   자식에게 맡겨야 하고(방식 B), 없는 화면은 root가 bottom까지 다 처리해야 한다(방식 A).
   `BaseActivity`는 서브클래스가 어떤 레이아웃을 쓸지, 그 안에 `BottomNavigationView`가
   있는지 알 방법이 없어서 이 둘을 구분해 줄 수 없다.
2. **`android.R.id.content`(각 화면의 루트보다 한 단계 위)에 패딩을 걸면 edge-to-edge 배경이
   깨진다.** 각 화면 루트는 배경을 화면 끝까지 채우고 그 안의 자식만 인셋만큼 밀어내는 구조인데,
   패딩을 한 단계 위에서 걸면 루트 자체가 줄어들어 배경도 인셋 안쪽에서 끊긴다.

## 13.5 변경 범위

- `ui/finddrugstore/DrugStoreDetail.java` 대응 레이아웃(`activity_drug_store_detail.xml`) —
  루트에 `fitsSystemWindows="true"` 추가
- `ui/sharedyaksok/ShareYaksokList.java`, `activity_share_yaksok_list.xml` — `InsetsUtil` 호출
  제거, 루트에 `fitsSystemWindows="true"` 추가
- `ui/sharedyaksok/ShareYaksokDetail.java`, `activity_share_yaksok_detail.xml` — 동일
- `ui/finddrugstore/DrugStoreList.java`, `activity_drug_store_list.xml` — 동일
- `ui/medicine/MedicineSearchActivity.java`, `activity_medicine_search.xml` — Java의 `InsetsUtil`
  호출만 제거했고 **XML에는 `fitsSystemWindows`를 추가하지 않아** 이 화면은 상·하단 모두 인셋
  처리가 없는 상태로 남았다. 24장(약 검색 화면 리디자인)에서 레이아웃을 재작성하며 루트에
  `fitsSystemWindows="true"`를 넣어 해소됐다.
- `ui/friend/FriendList.java`, `activity_friend_list.xml` — 반대 방향: `fitsSystemWindows` 제거,
  `InsetsUtil.applySystemBarPadding()` 호출 추가 (13.6 참고 — 이 화면은 진단이 한 번 뒤집혔다)

## 13.6 진단이 뒤집힌 기록 — `FriendList`

`FriendList`는 `fitsSystemWindows="true"`와 `BottomNavigationView`를 동시에 갖고 있었다.
처음에는 "이중 패딩"(root가 한 번, `BottomNavigationView` 내장 리스너가 또 한 번)이라고
진단하고 방식 B로 바꿨는데, 사용자가 "다른 `BottomNavigationView` 화면들과 눈으로 봤을 때
패딩·높이가 똑같았다"고 지적하면서 재검토했다.

**정정된 결론:** `fitsSystemWindows`는 처리와 동시에 인셋을 **소비**하고, 소비되면 그 아래
자식(`BottomNavigationView`)은 `dispatchApplyWindowInsets` 호출 자체를 못 받는다
([Android 공식 문서](https://developer.android.com/develop/ui/views/layout/edge-to-edge) —
"consuming insets in a parent prevents children from receiving them"). 즉 `FriendList`는
실제로는 **한 겹만** 적용되고 있었고, 방식 B로 바꿔도 결과적으로 똑같이 한 겹이라 육안으로
차이가 없었던 것 — "이중 패딩" 진단은 틀렸다.

여기서 근거로 들었던 "2026-07-28 커밋(`d3b8c36`)의 옛날 버그가 테마+XML+위젯, 삼중으로
쌓여서 아이콘이 잘렸다"는 설명도 같은 이유로 **근거 없이 지어낸 것이었다** — `InsetsUtil.java`의
설계 의도 주석("건드리면 이중으로 적용돼서 잘린다")을 옛 버그의 실제 원인인 것처럼 갖다 붙였을
뿐, 검증한 적이 없었다. 소비 규칙을 그대로 적용하면 옛 버그는 오히려 "제일 위(`content`)에서
먼저 소비해버려서 그 아래 전부 인셋을 못 받고 그냥 겹친 것"에 더 가깝다 — 이쪽이 커밋 메시지
("겹치지 안게 수정")와도 더 맞는다.

`FriendList`를 방식 B로 바꾼 것 자체는 틀린 수정은 아니다(다른 `BottomNavigationView` 화면들과
처리 방식이 통일된다는 이점은 있음). 다만 "버그를 고쳤다"는 근거는 성립하지 않았다.

**후속 (14장 이후):** `FriendList`는 `FriendListFragment`가 되어 하단 네비 호스트 안으로
들어갔다. 이제 하단 네비가 형제이므로 Fragment 루트에 `fitsSystemWindows`(4면 패딩)를 두면
bottom이 이중으로 비워진다 — 방식 B가 필수가 되어 "되돌릴지" 문제는 의미가 없어졌다.

## 13.7 남은 것

- 6개 화면 실기기 육안 확인 완료(`ShareYaksokDetail`의 저장 버튼, `FriendList`의 하단 네비 포함).
- 14장의 Fragment 전환 후 구조(13.2.1)도 육안 확인 전이다 — 특히 탭 Fragment 상단 여백과 하단
  네비 하단 여백.
- 방식 A와 C의 차이는 소비 여부뿐인 화면(`WipActivity`, `AddressSearch`)을 A로 통일할지는
  정하지 않았다. 요청 전에는 건드리지 않는다.
- 이 작업과 무관하게 세션 시작 전부터 있던 미커밋 변경 두 개는 건드리지 않았다:
  `util/InsetsUtil.java`의 주석 3줄 삭제, `activity_address_search.xml` staged 삭제.
