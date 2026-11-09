# 22. 채팅 목록 [친구 | 상담] 토글 RadioGroup → TabLayout

## 배경

약국 사장 계정(`role == DRUGSTORE`)의 채팅 목록(`ChatRoomListFragment`)은 [친구 | 상담]
토글로 조회 API(`getMyChatRooms` / `getConsultRooms`)를 바꾼다. 18장(Flow 02) 리스킨 때 이
토글을 `RadioGroup` + `RadioButton` 2개로 세그먼트 컨트롤처럼 만들었다(회색 캡슐 배경 안에서
선택된 쪽만 흰 알약). 소유자 요청으로 **디자인은 그대로 두고 컴포넌트만 Material `TabLayout`으로
교체**했다.

## 9장 판단을 뒤집은 이유

9장은 "Material `TabLayout`은 앱 디자인과 안 맞아 불채택"하고 `RadioGroup`을 택했다. 당시
디자인은 YaksokList 폴더 탭(위·바깥 테두리 + 밑줄)이라 TabLayout 기본 밑줄 인디케이터로는
재현이 어려웠다. 지금 디자인은 "캡슐 안에서 흰 알약이 선택 탭으로 이동"하는 세그먼트라
`tabIndicatorGravity="stretch"` + 커스텀 인디케이터 드로어블로 그대로 표현되고, 선택 전환
슬라이드 애니메이션도 기본으로 따라온다(RadioGroup은 즉시 스냅).

## 변경

| 파일 | 내용 |
|---|---|
| `res/layout/fragment_chat_room_list.xml` | `rg_chat_type` RadioGroup → `tab_chat_type` TabLayout + `TabItem` 2개. 배경 `bg_chat_tab_group` 유지, 패딩 없음(아래 "함정" 참고). `tabIndicator=bg_chat_tab_indicator`, `tabIndicatorColor=white`, `tabIndicatorFullWidth=true`, `tabIndicatorGravity=stretch`, `tabIndicatorAnimationMode=elastic`, 리플 투명, 선택/비선택 글자색 `g900`/`g500` |
| `res/drawable/bg_chat_tab_indicator.xml` | 신규. 흰 배경 + radius 10dp 알약을 `<inset android:inset="4dp">`로 감쌈 — 캡슐 안쪽 4dp 여백 담당 |
| `res/drawable/selector_chat_tab_segment.xml` | 삭제. TabLayout 인디케이터는 선택된 탭에만 그려지므로 `state_checked` 분기가 필요 없어 알약 shape만 위 파일로 남김 |
| `res/color/selector_chat_tab_segment_text.xml` | 삭제. `tabSelectedTextColor`/`tabTextColor` 속성이 대신함 |
| `res/values/type.xml` | `TextAppearance.App.ChatTab` 추가 — Pretendard 14sp bold, `textAllCaps=false`. 기존 RadioButton 글자 스타일 유지용 |
| `ui/chattingroom/ChatRoomListFragment.java` | `setupChatTypeToggle()`: `rgChatType` → `tabChatType`, `check(R.id.rb_chat_friend)` → `getTabAt(0).select()`, `OnCheckedChangeListener` → `OnTabSelectedListener`(position 1이면 `showConsult = true`). 조회 로직은 그대로 |

## 검토했지만 택하지 않은 대안

- **`ViewPager2` + `TabLayoutMediator`로 친구/상담을 하위 프래그먼트 2개로 분리.** TabLayout의
  정석 사용법이고 스와이프도 생기지만, 지금 구조는 RecyclerView 하나를 공유하며 API 호출만 바꾸는
  것이라 하위 프래그먼트 + 어댑터 신설은 "컴포넌트 교체" 요청 범위를 넘는다.
- **RadioGroup 유지 + 알약 슬라이드 애니메이션만 직접 구현.** 요청(TabLayout 전환) 자체를
  만족하지 못하고, TabLayout이 기본 제공하는 걸 손으로 다시 만드는 셈.
- **4dp 여백을 바깥 `FrameLayout`(배경 + `padding="4dp"`)으로 감싸서 주기.** 원래 RadioGroup과
  픽셀 단위로 같다(알약 폭 `(전체폭 − 8dp) / 2`). 하지만 뷰가 한 겹 늘고 숨김/표시 대상도 컨테이너로
  옮겨야 해서, 소유자 결정으로 inset 방식을 택했다. inset은 각 탭 안에서 사방 4dp를 빼므로 알약이
  가운데 경계 쪽으로 4dp 좁다 — 알약이 한 번에 하나만 보여 육안 차이는 거의 없다.

## 함정 (첫 적용 때 실기기에서 디자인이 깨졌던 원인)

`tabIndicator`에 흰 드로어블을 줘도 그것만으로는 목업처럼 안 나온다. 아래 세 속성이 모두 필요하다.

- **`tabIndicatorColor="@android:color/white"`** — Material3 TabLayout은 인디케이터 드로어블을
  `tabIndicatorColor`(기본값 `colorPrimary` = 보라)로 **틴트**한다. 드로어블 자체 색은 무시된다.
- **`tabIndicatorFullWidth="true"`** — Material3 기본값이 false라 알약이 글자 폭만큼만 그려진다.
- **4dp 여백은 인디케이터 드로어블의 `inset`으로 준다. TabLayout에 `padding`을 주면 안 된다.**
  fixed 모드 TabLayout은 `onMeasure`에서 탭 줄(자식) 폭을 패딩을 빼지 않은 자기 전체 폭으로 다시
  재서, 탭 줄이 시작 패딩만큼 밀리고 끝 4dp가 잘린다(오른쪽 "상담" 알약의 둥근 모서리가 사라지는
  증상). `stretch` 인디케이터는 탭 영역을 꽉 채우므로, inset이 곧 캡슐 안쪽 여백이 된다.

또, 레이아웃 안에서 옛 id(`rg_chat_type`)를 참조하는 제약(RecyclerView `constraintTop_toBottomOf`)이
남아 있으면 리소스 링킹이 실패한다. `compileDebugJavaWithJavac`로는 이 실패가 안 잡혔으므로,
리소스를 바꿨으면 `assembleDebug`로 확인할 것.
