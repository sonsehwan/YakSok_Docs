# 23. 약 아이템(PillCard) 이름을 이미지 위로

## 문제
약 이름이 길어(예: "타이레놀정 500mg", 성분명 포함) 이미지 오른쪽 좁은 영역에서 줄바꿈/잘림이 생김.

## 변경
- `yaksok_design.pen` `PillCard`: `Top`을 세로 배치로 바꾸고 `Header`(Name fill_container + Remove) → `Thumb` 순서로 재구성. Name이 카드 전체 폭을 쓴다.
- `item_add_medication_setting.xml`: 같은 구조로 재작성. 이름 행(이름 weight 1 + X) → 이미지(96x56) → 횟수/투약량 가로 2단.
  - 기존엔 `tv_pill_name`이 RelativeLayout 밖 LinearLayout 안에 있어 `layout_toEndOf`가 무시되던 상태였음.
  - 새 drawable `bg_pill_field`(g20, r12), `bg_pill_stepper_btn`(white + g100 stroke, r12). 기존 `bg_setting_item`은 다른 화면이 쓸 수 있어 건드리지 않음.
- 횟수 텍스트를 "3 번" → "3번"으로 (디자인 일치). `AddMedicationSettingAdapter`, `DetailYaksokMedicationAdapter`.
- view id는 전부 유지 → 어댑터 로직 변경 없음.

## 택하지 않은 대안
- 이름 `maxLines=1` + ellipsize: 긴 이름이 어차피 안 보여 약 구분이 안 됨.
- 이미지를 카드 폭 전체 배너로: 이미지가 아이콘 placeholder 수준이라 공간 낭비.

## 후속 수정
- 벡터 아이콘 `ic_lc_x`/`ic_lc_pill`: 여러 선을 한 경로로 합치며 상대 `m`이 어긋나 X가 "/"로 보이던 것을 절대좌표 `M`으로 수정. 이미지 없을 때 회색 알약 `ic_pill_placeholder`(Glide placeholder/error/fallback).
- 횟수 +/- 시 깜빡임: `notifyItemChanged`가 change 애니메이션(크로스페이드) + 전체 재바인딩(Glide 재로딩)을 일으켜서, 클릭 핸들러에서 `tvFreqValue` 텍스트만 직접 갱신하도록 변경. `setSupportsChangeAnimations(false)`는 재바인딩이 남고 화면마다 설정해야 해서 택하지 않음.
