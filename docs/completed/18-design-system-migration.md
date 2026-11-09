# 18. 디자인 시스템 마이그레이션 (yaksok_design.pen)

## 배경

`yaksok_design.pen`(pen.dev 파일, 루트에 있음)에 앱 전체 화면을 새 디자인으로 다시 그린 목업이
있고, 그 목업을 프론트엔드 실제 레이아웃에 화면 단위로 반영하는 여러 세션에 걸친 작업이다.
목업은 "Flow 01" ~ "Flow 06" 헤더로 화면 묶음을 구분해 정리돼 있다(`pencil` MCP로 열람).

디자인 토큰(`res/values/colors.xml`): `g20/g50/g100/g300/g400/g500/g700/g900`(회색조),
`p100/p600/p700`(브랜드 오렌지), `status_done/pending/missed`. 기존 앱의 `brand_*`/`amber_*`/
`neutral_*` 토큰과는 별도 계열이며, 화면을 옮길 때마다 필요한 만큼만 추가한다.
아이콘은 lucide 벡터(`ic_lc_*.xml`)로 통일하며, 원본 SVG는 `design/icons/`에 있다(라이선스:
`design/icons/LICENSE-lucide.txt`). 필요한 아이콘이 없으면 그 SVG를 손으로 벡터 드로어블로
변환해 추가한다(패턴: 여러 `<path>`를 공백으로 이어 하나의 `pathData`로 합치고,
`circle`은 `M(cx-r),cy a r,r 0 1,0 2r,0 a r,r 0 1,0 -2r,0` 형태로 변환).

## 진행 상태

- **Flow 01 (로그인/회원가입)** — 완료 (커밋 `1f09c0b`, `4bfe8d4` 등)
- **Flow 02 (메인 탭 5개: Home/YaksokList/ChatRoomList/FriendList/Settings)** — 완료.
  Home/MainActivity는 커밋됨(`d92c075`). 나머지 4개(YaksokList, ChatRoomList, FriendList,
  Settings)는 작업 트리에 미커밋 상태로 반영되어 있음(다른 세션에서 작업, 커밋은 아직 안 함).
- **Flow 03 (약속 등록/상세)** — 이번 세션에서 완료. 아래 "Flow 03 세부" 참고.
- **Flow 04 (채팅/약국/내 정보)** — 이번 세션에서 완료. 아래 "Flow 04 세부" 참고.
- **Flow 05 (친구 추가 QR)** — 이번 세션에서 완료. 아래 "Flow 05 세부" 참고.

## Flow 03 세부 (약속 등록/상세) — 이번 세션에서 반영

**대상 6개 화면**: `ShowAddMedicationList`(바텀시트), `CreatePrescription`/`CreatePaperEnvelope`/
`CreateDirectSchedule`(**셋 다 `activity_create_prescription.xml` 레이아웃 공유** — 하나 고치면
셋 다 바뀐다), `YaksokDetail`(`activity_yaksok_detail.xml`), `ModifyYaksok`
(`activity_modify_yaksok.xml`).

**판단한 것들:**

- **구조는 그대로, 스킨만 교체.** pen 목업은 투약 시간(아침/점심/저녁)·복용 방법(식전/식후/직후)을
  아이콘+라벨+시간이 있는 카드형 컴포넌트로 그려뒀지만, 실제 코드는 `CheckBox`/`RadioButton` +
  `binding.cbMorning.isChecked()` 등으로 상태를 읽는다. 뷰 타입과 id를 그대로 두고 배경/색상만
  칩 스타일로 바꿨다 — 목업을 픽셀 단위로 그대로 옮기려면 Java 쪽 상태 읽기 로직까지 다시 짜야
  하는데, 이번 범위는 "UI 변경"(리스킨)이라 판단해 구조 변경은 하지 않음. 다만 체크박스에
  `drawableTop`으로 sunrise/sun/moon 아이콘만 얹어 목업의 의도(시간대 아이콘)는 살렸다.
- **`bg_yellow_border.xml`/`bg_error_border.xml`은 파일명을 그대로 두고 내용만 바꿨다.**
  `CreatePrescription.validateInput()`이 `setBackgroundResource(R.drawable.bg_yellow_border /
  bg_error_border)`로 유효성 상태를 토글하는데, 이 두 리소스 id를 Java가 직접 참조하므로 이름을
  바꾸면 Java도 고쳐야 한다. 대신 "평상시" 배경은 투명(칩들이 각자 테두리를 가지므로 굳이 바깥
  박스가 필요 없음), "오류" 배경만 `status_missed` 빨간 테두리로 남겨 의미를 유지했다.
- **`selector_square_check_box.xml`도 파일명 유지**, 내용을 노란 사각형 → p100/p600 칩 스타일로
  교체(체크/라디오 공용, `색상 selector_square_check_box_text.xml` 신설).
- **`updateRegisterButtonState()`의 하드코딩 색을 고치면서 뒤바뀐 버그도 같이 잡음.** 기존 코드는
  "약을 하나도 안 골랐을 때(비활성)" 배경 `0xFFE0E0E0`(회색) + 글자 `0xFFFFFFFF`(흰색)이라 회색
  배경에 흰 글자가 거의 안 보이는 상태였다. `p600`(활성)/`g100`+`g400`(비활성)으로 바꾸면서 이
  조합도 정상화했다 — 같은 줄을 어차피 고치는 김에 반영, 별도 범위 확장 아님.
  `CreatePrescription`/`CreatePaperEnvelope`/`CreateDirectSchedule`/`ModifyYaksok` 4개 파일 전부
  동일하게 고침(`ModifyYaksok`, `CreateDirectSchedule`은 `ContextCompat` import가 없어서 추가함).
- **`ic_arrow_back`은 그대로 둠, `ic_lc_chevron_left`로 바꾸지 않음.** Flow 01에서 이미 마이그레이션된
  화면들(`activity_find_id.xml` 등)이 전부 `ic_arrow_back` + `app:tint="@color/g900"` 조합을 뒤로가기
  아이콘으로 쓰고 있어서, 이게 이미 정착된 컨벤션이라고 판단해 따랐다.
- **`activity_yaksok_detail.xml`의 `iv_menu`는 PNG(`ic_menu.png`)를 쓰고 있었는데 이 화면
  한 곳에서만 쓰여서 `ic_lc_menu.xml`(벡터, g900 틴트)로 교체함.** 다른 화면에 영향 없음(사용처
  1곳 확인 후 교체).
- **새로 추가한 lucide 아이콘**: `ic_lc_file_text`(처방전), `ic_lc_package`(약봉투),
  `ic_lc_pencil_line`(직접입력), `ic_lc_sunrise`/`ic_lc_sun`/`ic_lc_moon`(아침/점심/저녁),
  `ic_lc_plus`(약 추가). `design/icons/`에 SVG 원본이 이미 있어서 좌표를 그대로 벡터로 옮김.
- **`item_add_medication_setting.xml`**(약 카드, 등록/수정/상세 3곳 공유)도 같은 세션에서 리스킨.
  `-`/`+` 버튼 보라색(#7C4DFF) → `p600`, 삭제 아이콘 `@android:drawable/ic_menu_close_clear_cancel`
  → `ic_lc_x`, 나머지 하드코딩 회색 계열 → `g` 토큰.

**확인 방법**: `./gradlew :app:compileDebugJavaWithJavac`로 리소스 링킹 + 컴파일 확인함(성공).
이후 실기기 육안 확인까지 완료.

## Flow 04 세부 (채팅 / 약국 / 내 정보) — 이번 세션에서 반영

**대상**: `ChattingRoom`(+ 메시지 아이템 4종), `FindDrugStore`(+ `item_simple_drugstore`),
`DrugStoreList`(+ `item_drugstore`), `DrugStoreDetail`, `MyInfo`. (`ChatRoomListFragment`는
이미 다른 세션에서 완료돼 있었음 — 손대지 않음.)

**판단한 것들:**

- **회귀 발견 및 수정.** Flow 03에서 `bg_yellow_border.xml`을 "약속 투약 시간 칩 그룹의 평상시
  배경(투명)"으로 재활용했는데, 이 drawable이 `activity_find_drugstore.xml`/`activity_my_info.xml`
  /`activity_drug_store_detail.xml`/`activity_share_yaksok_detail.xml`에서 "입력 박스 테두리"
  용도로도 쓰이고 있었다. 앞의 3개(이번 Flow 04 대상)는 리스킨하면서 `bg_card_outlined`(흰 배경 +
  g100 테두리, 이미 존재하던 신규 컴포넌트)로 교체해 해결했다. `activity_share_yaksok_detail.xml`
  (Flow 03/04 어느 쪽에도 포함되지 않은 화면)은 확인해보니 오히려 그 화면도 `selector_square_check_box`
  를 쓰는 동일한 체크박스/라디오 그룹이라 Flow 03 변경이 그대로 맞아떨어졌다 — 별도 조치 불필요.
- **채팅방(`ChattingRoom`) 헤더를 기존 `brand_surface`(호박색 단색 배경) 대신 흰 배경 + `g900`
  텍스트로 통일.** 다른 화면들의 AppBar 컨벤션(흰/`g20` 배경 + 굵은 제목)과 맞췄다. 상태바 색을
  Java에서 `getWindow().setStatusBarColor(...)`로 직접 지정하는 부분도 같이 흰색으로 바꿨다 —
  `BaseActivity.applyLightSystemBarIcons()`가 모든 화면에서 상태바 아이콘을 이미 짙은 색으로
  고정해두므로(엣지투엣지 대응) 흰 상태바에 짙은 아이콘 조합이 앱 전체와 일관됨.
  전송 버튼은 텍스트 버튼("전송")에서 원형 아이콘 버튼(`ic_lc_send`, `p600` 배경)으로 바꿈.
- **채팅 말풍선 색은 기존 "내 메시지=흰색, 상대 메시지=호박색(카카오톡 톤)" 구성을 유지**하고
  호박색만 `p100`(연한 브랜드 틴트)으로 교체했다. 메시지 소유자별 배경을 서로 바꾸는 건(예: 내
  메시지를 강조색으로) 동작 변경에 가까워 범위 밖으로 판단.
- **약국 카드 목록(`item_drugstore`, `item_simple_drugstore`)의 "왼쪽 색상 스트라이프 + 카드
  엘리베이션" 스타일을 걷어내고, 다른 화면에서 이미 쓰는 "아이콘 박스 + 텍스트 + 화살표"
  패턴(`bottom_sheet_add_yaksok`에서 먼저 쓴 것과 동일)으로 통일.** 앱 안에 카드 스타일이
  두 갈래(엘리베이션 vs 아웃라인)로 있던 것을 아웃라인 쪽으로 맞춘 것 — `item_yaksok.xml` 등
  기존 리스트 카드들이 전부 아웃라인 스타일이었음.
- **지도 핀 아이콘이 lucide 세트에 없어서 `design/icons/ic_lc_map_pin.svg`를 새로 벡터로 추가.**
  기존 `ic_map`(구 아이콘)을 대체.
- **`DrugStoreDetail` 화면에 없던 헤더 타이틀("약국 상세")을 추가.** 원래 뒤로가기 아이콘만
  있었는데, 다른 모든 화면이 "뒤로가기 + 제목" 조합을 쓰고 있어 통일했다. 정적 텍스트라 Java
  변경은 없음.
- **`MyInfo`의 "회원 탈퇴" 버튼을 단색 빨강 채움 → 흰 배경 + 빨강 테두리(outline)로 변경.**
  바로 아래 "수정하기"(단색 `p600`)와 나란히 있어서 위험한 동작 쪽을 시각적으로 더 약하게(보조
  버튼처럼) 두는 게 낫다고 판단. 위험 표시 자체(`status_missed` 톤)는 유지.

## Flow 05 세부 (친구 추가 QR) — 이번 세션에서 반영

기능은 이미 구현돼 있었고(17장), 다이얼로그 5개(`dialog_my_qr`, `dialog_add_friend`,
`dialog_select_friend`, `dialog_profile`, `dialog_received_request`)와 `item_received_request`가
디자인 토큰이 아닌 그때그때 지정한 hex 값(`#1A1A1A`, `#9A9A9A`, `#7A6100` 등)과 구 토큰
(`brand_surface`, `bg_avatar_circle`)을 쓰고 있어서 `g900`/`g500`/`g400`/`p600`/
`bg_avatar_circle_neutral`(Flow 02 작업에서 이미 만들어진 이니셜 아바타 배경)로 맞췄다. QR
스캔 화면(`PortraitCaptureActivity`)은 ZXing 라이브러리가 그리는 카메라 오버레이라 커스텀
레이아웃이 없음 — 손댈 것 없음.

## 화면 배경색 결정 — 전부 흰색

처음에는 Foundations의 Elevation이 "Level 0 · Page · G20"이라 앱 XML 대부분이 루트 배경을
`g20`으로 하고 있었는데, `.pen` 화면 프레임은 흰색·`g20`이 섞여 있었다(디자인 시스템 정의와
목업이 어긋난 상태). 실제로 화면을 보니 회색 페이지가 별로라서 **화면 배경은 전부 흰색
(`@color/white`)으로 통일**하기로 했다.

- `.pen`: 360폭 화면 프레임 75개를 `$white`로, Foundations의 Level 0 카드와 문구를
  "Page · White"로 바꿨다. `g20`(Secondary bg) 토큰과 팔레트 견본은 남겨뒀다 — 화면 배경이
  아니라 보조 영역 용도.
- 앱: 화면 루트 배경이 `g20`/`page_bg`(앰버)였던 레이아웃 13개(`activity_main`,
  `activity_medicine_search`, 채팅방·약국·내 정보·약속 수정/상세, 메인 탭 프래그먼트 5개 등)를
  `@color/white`로. 죽은 코드인 `activity_share_yaksok_list.xml`은 그대로.
- **검토했지만 택하지 않은 것**: Foundations 정의(G20 페이지)에 맞춰 `.pen` 흰색 프레임을
  `g20`으로 고치는 안 — 실제로 한 번 적용했다가 결과가 마음에 들지 않아 되돌렸다.
- 흰 배경이 되면 카드/입력창은 1px `g100` 테두리로만 구분된다. `activity_chatting_room.xml`
  안쪽의 `g20` 영역(채팅 목록 배경)은 루트가 아니라서 남겼다.

## 다음에 할 것

Flow 01~05 전부 완료. 다음 세션에서 다시 볼 것:

- `docs/todo.md`의 "공통 상단바 컴포넌트" 항목 — 화면마다 반복되는 뒤로가기+제목 헤더를 하나로
  묶을 시점이 됐다(이번 세션에서도 같은 마크업을 화면마다 또 복붙함).
- `input.xml`(공용 `InputView`)의 `tv_error` 텍스트 색(`#FF0000`)은 이번 범위 밖이라 손대지
  않음 — `status_missed` 토큰으로 바꿀 후보.
