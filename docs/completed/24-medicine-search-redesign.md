# 24. 약 검색(MedicineSearchActivity) 새 디자인 적용 + 상단 겹침 수정

## 문제
- 화면이 아직 옛 스타일(노란 버튼, 그림자 카드 아이템)이었다. `yaksok_design.pen`의 `MedicineSearchActivity` 프레임과 불일치.
- 헤더가 상태바 밑으로 들어가 겹침. 원인: `onCreate`의 `EdgeToEdge.enable(this)`로 상태바 뒤까지 그리는데 레이아웃 루트에 inset 처리가 없었다. `BaseActivity`는 세로 고정·상태바 아이콘 색·401 처리만 하고 inset은 다루지 않는다.

## 변경
- `activity_medicine_search.xml`: 루트를 세로 LinearLayout + `fitsSystemWindows="true"`로 재작성(겹침 해결). AppBar 56dp(chevron-left / 가운데 제목 18sp / 오른쪽 40dp 여백), 헤더 구분선 제거(디자인에 없음), 검색 행 padding 16/8/16/12, 버튼은 p600 + 흰 글자.
- `item_simple_medicine_info.xml`: 카드 → 플랫 행. 56dp 썸네일(`bg_icon_box_neutral` g50 r12, `clipToOutline`) + 이름 15sp bold, 하단 g100 1dp 구분선. 좌우 16dp는 RecyclerView `paddingHorizontal`이 담당.
- 새 drawable `bg_search_field`(흰 배경 + g100 테두리, 포커스 시 p600 테두리).
- `MedicineAdapter`: Glide placeholder/error를 안드로이드 기본 아이콘에서 `ic_pill_placeholder_vec`(회색 알약)로 교체.
- view id 전부 유지 → 액티비티 로직 변경 없음. 액티비티의 `EdgeToEdge.enable`은 그대로 두고 레이아웃 쪽에서 inset 처리.

## 택하지 않은 대안
- `EdgeToEdge.enable` 제거: 다른 화면과 처리 방식이 달라지고, 상태바 색이 시스템 기본으로 바뀔 수 있음.
- 썸네일을 `centerCrop`: 약 사진이 가로로 긴 경우가 많아 잘림. `centerInside`로 전체 표시.

## 후속: 투약 횟수/시간대 그룹 좌우 여백
- 약속 등록 화면에서 "투약 횟수"(`ll_dasage`)·"투약 시간대"(`rg_dosage_time`) 칩 그룹이 다른 입력칸보다 안쪽으로 들어가 보였다. 입력칸은 `marginHorizontal=15dp`인데 그룹은 18dp + 그룹 `padding=2dp` + 칩 `layout_margin=2dp`라 칩 가장자리가 22dp에 놓였기 때문.
- 그룹 margin을 11dp로 바꿔(11+2+2=15) 칩 바깥 가장자리를 입력칸에 맞춤. 칩 사이 간격(4dp)은 유지. 같은 패턴이던 `activity_create_prescription`, `activity_modify_yaksok`, `activity_yaksok_detail` 3곳에 적용.
- 택하지 않은 대안: 그룹 padding·칩 margin을 0으로 — 칩 사이 간격까지 사라짐.

## 후속: 투약 횟수 칩 글자 잘림
- `activity_create_prescription.xml`의 `cb_lunch`/`cb_dinner`에 `padding=10dp`가 남아 있었다. 칩 높이 60dp에서 위아래 padding 20 + 아이콘 24 + 간격 2 + 글자 12sp(약 16)가 합쳐져 콘텐츠 영역이 부족해 글자 아래가 잘림. `cb_morning`은 이미 padding이 빠져 있었음. 두 칩의 padding 제거(높이는 그대로 60dp). 수정/상세 화면(52dp)은 처음부터 padding이 없어 해당 없음.
- 위 수정(padding 제거)만으로는 `drawableTop` 아이콘이 칩 윗변에 붙었다. TextView는 compound drawable을 항상 padding 안쪽 맨 위에 그리고, `gravity`로는 글자만 남은 공간에서 가운데 정렬하기 때문(아이콘+글자 묶음이 중앙 정렬되지 않음). 칩에 `paddingVertical`을 대칭으로 줘서 위(아이콘)·아래(글자) 여백을 맞춤: 60dp 칩 8dp. 수정/상세 화면 칩도 52dp → 60dp, 8dp로 등록 화면과 통일. 글자 높이(≈16dp)가 남는 공간 안에 들어가도록 계산한 값이라 더 키우면 다시 잘린다.
- 택하지 않은 대안: 칩을 `LinearLayout`(아이콘 `ImageView` + `TextView`, `gravity=center`)으로 교체 — 정확히 중앙 정렬되지만 CheckBox/RadioButton의 체크 상태·그룹 동작을 직접 구현해야 해서 과함.
- "약 추가" 버튼(`btn_add_pill`)의 불필요한 `FrameLayout` 래퍼 제거 → 점선 배경·클릭 속성을 가진 `TextView` 하나로 통합(`gravity=center` + `drawableStart`는 아이콘·글자가 한 덩어리로 중앙 정렬됨). `activity_create_prescription`, `activity_modify_yaksok`. id·클릭 리스너 사용은 그대로.

## 정정: "약 추가" 버튼 TextView 통합은 잘못이었음 (FrameLayout 복원)
- `TextView`에 `gravity=center` + `drawableStart`를 주면 아이콘이 글자 옆이 아니라 뷰의 왼쪽 끝에 붙고 글자만 가운데 놓인다(compound drawable은 padding 안쪽 가장자리에 그려짐). 그래서 `FrameLayout` 안에서 `wrap_content` TextView를 `layout_gravity=center`로 두어 "아이콘+글자" 덩어리를 가운데 정렬하던 기존 구조가 필요한 구조였다. `activity_create_prescription`, `activity_modify_yaksok` 모두 FrameLayout 구조로 되돌림.

## 약 아이템(`item_add_medication_setting.xml`) 수정
- 1회 투약량 단위 "개"가 잘리던 문제: `EditText`(weight 3)와 단위 `TextView`(weight 1, paddingHorizontal 14dp)가 따로 있어 단위 칸이 좁아 글자가 잘렸다. .pen `AmountBox`처럼 박스 하나(`bg_pill_field`) 안에 `EditText`(배경 없음, weight 1) + 단위 `TextView`(wrap_content)로 재구성. `et_dosage_value` id 유지.
- 상세 화면(읽기 전용)에서 횟수 증감 버튼을 `INVISIBLE` → `GONE`(`DetailYaksokMedicationAdapter`): 투명한 자리가 남아 횟수 박스가 오른쪽으로 치우쳐 보였다. .pen에서도 상세 인스턴스는 Minus/Plus를 비활성화하고 CountBox가 전체 폭을 쓴다. 등록/수정 화면은 버튼 유지.
- 상세 화면 단위는 .pen대로 "정": 단위 TextView에 `tv_dosage_unit` id를 주고 `DetailYaksokMedicationAdapter`에서만 "정"으로 설정(등록/수정은 "개" 유지). 횟수/투약량 폭은 weight 1:1 + 6dp 마진으로 .pen(gap 12)과 같고, 버튼이 GONE이라 횟수 박스가 전체 폭을 쓴다.
- 제품 검색 입력창 테두리: .pen이 주황(#FD7E14)이라 `bg_search_field`의 포커스 selector를 없애고 항상 p600 테두리로 고정(이전: 평상시 g100, 포커스 시 p600).

## 단일 자식 FrameLayout 제거
- 자식이 하나뿐인 `FrameLayout`(배경 + 가운데 아이콘/텍스트)을 전부 찾아 제거. 아이콘 박스 6곳(`activity_find_id`, `activity_login`, `bottom_sheet_sign_up_type`×2, `item_drugstore`, `item_simple_drugstore`)은 `ImageView` 하나로: 바깥 크기 + `background` + `padding=(바깥−아이콘)/2`로 아이콘을 가운데에 둠.
- "약 추가" 버튼(`activity_create_prescription`, `activity_modify_yaksok`)은 `TextView`로는 아이콘이 왼쪽 끝에 붙어 불가 → `MaterialButton`(`app:icon` + `iconGravity=textStart`, 흰 배경·p600 테두리 1.5dp·r12)으로 교체. 아이콘+글자가 한 덩어리로 가운데 정렬됨. 쓸 곳이 없어진 `bg_add_pill_button` drawable은 삭제.

## .pen Flow 03 ↔ 앱 정합 (후속)
- `.pen` 쪽을 앱에 맞춤: "약 추가" 테두리 p600, 투약 횟수 칩(60 높이, 시간 텍스트 없음, 선택 시 시간 입력줄), 투약 시간대 칩(r12·48 높이·선택 p100/p600), 1회 투약량 값 가운데 정렬 + 단위 우측, "약 선택" 18 bold, 그 위 구분선(g100).
- 앱 `activity_share_yaksok_detail.xml`은 옛 스타일(검정 구분선, 노란 테두리 칩)이라 `activity_yaksok_detail.xml` 구조로 재작성(id 전부 유지). 상단 공유자 줄은 `tv_owner`를 유지한 채 `bg_sender_card`(g20 r16) + 사람 아이콘 `drawableStart`, 버튼은 p600 "내 목록에 저장". 구분선 색은 다른 약속 화면과 같은 g100.
- 택하지 않은 대안: `tv_owner`를 LinearLayout(흰 원 아바타+텍스트)으로 — ViewBinding 타입(TextView)과 `setText` 호출이 깨져서 보류. 아바타 흰 원은 생략됨.
- 투약 시작 날짜/투약일 수를 .pen `Date Row`처럼 한 줄(가로 2단, weight 1:1·간격 12)로 배치: `activity_share_yaksok_detail`, `activity_yaksok_detail`(등록/수정 화면은 이미 가로). id 유지.
