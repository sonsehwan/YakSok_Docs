# 26. 확인/액션 다이얼로그와 약속 상세 메뉴를 .pen 디자인으로

## 문제

.pen에는 확인 다이얼로그(`Dialog/Confirm`), 길게 누르기 액션 다이얼로그, 약속 상세 팝업 메뉴가
새 디자인으로 있는데, 앱은 시스템 `AlertDialog.Builder`(11곳)와 `PopupMenu`(약속 상세)를 그대로
쓰고 있어 모양이 달랐다. 두 버튼을 나란히 놓는 레이아웃(취소=아웃라인, 확인=주황/빨강)과 아이콘이
붙은 메뉴는 `AlertDialog`/`PopupMenu` 스타일 오버라이드만으로는 만들 수 없다.

## 판단

- **공통 헬퍼 `ui/common/AppDialog`.** 기존 커스텀 다이얼로그(`FriendListFragment`, `YaksokDetail`의
  `Dialog` + 레이아웃 + 투명 배경 + 폭 88%) 방식을 그대로 따른다. `confirm(context, title, message,
  positiveText, danger, onPositive)`와 `deleteAction(context, title, onDelete)` 두 개뿐이다.
  `danger`는 확인 버튼을 `red500`으로 바꾼다(삭제·탈퇴·나가기·빼기처럼 되돌릴 수 없는 동작).
- **교체한 호출 지점 11곳.** 복용 완료 확인(`NotificationMultiViewAdapter`), 약속 삭제
  (`YaksokDetail`), 공유 목록에서 빼기(`YaksokListFragment`, `ShareYaksokList`), 채팅방 길게 누르기와
  나가기(`ChatRoomListFragment`), 약국 선택(`FindDrugStore`), 로그아웃(`SettingsFragment`), 회원
  탈퇴(`MyInfo`), 친구 길게 누르기와 삭제(`FriendListFragment`).
- **액션 다이얼로그는 "삭제" 한 가지로 고정.** 두 화면 모두 항목이 삭제 하나뿐이라 항목 목록을
  받는 일반형으로 만들지 않았다. 항목이 늘면 그때 일반화한다.
- **팝업 메뉴는 `PopupWindow` + 레이아웃.** 약속 상세(`popup_yaksok_menu.xml`, 폭 160)와 친구 목록 QR
  아이콘(`popup_friend_qr_menu.xml`, 폭 168, 내 프로필 / 코드 스캔)이 같은 모양이라 `AppDialog.popup`
  (흰색 r-12 + elevation)과 `AppDialog.showBelow`(앵커 바로 아래, 화면 오른쪽에서 16dp)를 같이 쓴다.
  `PopupMenu`는 아이콘과 항목별 글자색(삭제만 빨강)을 정확히 그리기 어렵다. 위치를 앵커 오른쪽 끝이
  아니라 **화면 오른쪽 끝** 기준으로 잡은 이유는, QR 아이콘이 친구 추가 아이콘 왼쪽에 있어 앵커 기준으로
  맞추면 .pen(오른쪽 끝에서 16dp)과 어긋나기 때문이다.
- **색 토큰 추가:** `g600`, `red500`(`colors.xml`), 아이콘 `ic_lc_trash_2`, `ic_lc_share_2`.

## 택하지 않은 대안

- **`AlertDialog` 테마 오버레이로 모서리·글자색만 맞추기:** 시스템 버튼 줄(텍스트 버튼 오른쪽
  정렬)이 그대로 남아 디자인의 가로 2버튼과 다르다.
- **`DialogFragment`로 만들기:** 회전·프로세스 복원에는 유리하지만 호출 지점마다 프래그먼트 클래스와
  결과 전달이 필요해 지금 구조(람다 콜백)와 맞지 않는다. 화면 회전은 세로 고정이라 당장 필요 없다.
- **`PopupMenu`에 `setForceShowIcon`과 SpannableString으로 색 입히기:** 아이콘 틴트와 항목 높이·여백을
  .pen 값과 맞추기 어렵다.

## 알려진 한계

- 복용 완료 확인은 이미 복용한 항목을 누르면 문구가 "복용 취소 처리하시겠습니까?"로 바뀌지만 .pen에는
  "복용 완료" 문구만 있다. 모양은 같다.
- 실기기에서 확인 완료(팝업 위치, 그림자, 88% 폭).
