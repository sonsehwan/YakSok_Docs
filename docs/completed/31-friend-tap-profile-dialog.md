# 31. 친구 항목을 탭하면 프로필 다이얼로그

## 문제

친구 목록 항목을 탭하면 `WipActivity`(작업 중 화면)로 이동했다. "약속 공유 기능 완성 후 연결
예정"이라는 주석이 붙은 임시 코드가 그대로 남아 있던 것이다. 롱클릭은 "삭제" 하나뿐이라 친구
목록 화면에서는 친구에게 할 수 있는 일이 사실상 없었다.

친구와의 채팅을 시작할 수 있는 경로도 ① 약속 공유(`YaksokDetail.openChatRoomAndShare`)
② 이미 친구인 상대의 QR 재스캔(프로필 다이얼로그의 "채팅하기") 둘뿐이었다. 친구 목록에서는
채팅을 시작할 방법이 없었다.

## 판단

- **새 화면·API 없이 기존 프로필 다이얼로그를 재사용한다.** 백엔드 `FriendResponseDto`가
  `friendId`(친구의 유저 id)·`nickname`·`email`을 이미 내려주고, `openChatWithFriend`가 쓰는
  값은 userId와 닉네임뿐이다. 그래서 백엔드 변경이 없다. todo 6.2의 "프로필 화면 vs 바로 기능
  다이얼로그"는 17장에서 다이얼로그가 구현된 시점에 사실상 정해져 있었다.
- **변환은 `UserProfileDto.fromFriend(FriendResponseDto)` 정적 팩토리로.** 프론트
  `UserProfileDto`는 `@Getter`만 있고 필드가 private이라 밖에서 만들 수 없다. 같은 클래스
  안에서 필드를 직접 채우고 `relation = FRIEND`, `requestId = null`로 둔다.
- **탭 리스너는 `renderProfileDialog(null, UserProfileDto.fromFriend(friend))`.** `code`는
  액션 성공 뒤 새로고침(`refreshProfileDialog`)에만 쓰이는데, FRIEND 분기는 새로고침을 쓰지
  않아 `null`을 넘겨도 된다.
- **롱클릭 삭제는 그대로 둔다.**
- **탭의 의미를 "약속 공유"에서 "프로필"로 바꿨다.** 원래 주석의 의도는 "친구를 탭해서 약속
  공유"였다. 프로필 다이얼로그는 그 액션을 나중에 얹을 자리도 된다.

## 택하지 않은 대안

- **탭하면 바로 채팅방 진입:** 탭이 한 번 줄지만 "약속 공유"나 "삭제"를 둘 자리가 없고, 삭제가
  롱클릭에만 있어 찾기 어려운 상태가 이어진다. 방 생성은 `getOrCreate`라 잘못 눌러도 부작용은
  없으므로, 결정 근거는 오탭 방지가 아니라 확장 자리다.
- **프로필 전용 Activity 신설:** 보여줄 정보가 닉네임과 이메일뿐인데 화면과 API만 늘어난다.
- **`UserProfileDto`에 `@AllArgsConstructor`:** 호출부가 `(id, 닉네임, 이메일, FRIEND, null)`처럼
  위치 인자 5개가 되어 읽기 어렵다. 팩토리는 변환 규칙이 한 곳에 모인다.

## 변경

- `UserProfileDto.fromFriend(FriendResponseDto)` 추가 (프론트).
- `FriendListFragment.setRecyclerView()`의 탭 리스너를 `WipActivity` 이동에서 프로필 다이얼로그
  표시로 교체, `WipActivity` import 제거.

## 알려진 제약

- **`code == null` 결합.** `refreshProfileDialog`는 `getUserProfile(code)`를 부르는데,
  `@Path("code")`에 `null`이 들어가면 Retrofit이 호출을 만드는 시점에 `IllegalArgumentException`을
  던진다(`onFailure`로 떨어지지 않고 앱이 바로 죽는다). 지금은 도달할 수 없다 — 새로고침을 부르는
  액션이 NONE("친구 요청 보내기")·RECEIVED("수락하기") 상태에만 있고, 친구 목록에서 연
  다이얼로그는 항상 FRIEND라서다. 다만 FRIEND 상태에 성공 후 새로고침하는 액션(예: 프로필에서
  친구 삭제 → NONE으로 갱신)을 추가하면 터진다. 그때는 ① 새로고침 대신 `dismiss()` + 목록 갱신,
  또는 ② `userId` 기반 프로필 조회 API 추가(스펙에 없는 엔드포인트라 먼저 상의 필요) 중에 고른다.
  ①이 훨씬 가볍다.
- **"채팅하기" 후 돌아오면 다이얼로그가 그대로 떠 있다.** `openChatWithFriend`가 다이얼로그를
  닫지 않는다. `bindProfileDialog`가 `Dialog` 참조를 받지 않아 닫을 수단이 없다. QR 스캔 경로도
  같은 동작이다.
- **`WipActivity`는 이제 아무도 쓰지 않는다.** 클래스, `wip.xml`, 매니페스트 등록이 남아 있다.
- **프로필에 "약속 공유" 액션은 아직 없다.** 탭의 원래 의도였던 기능이고, 이 다이얼로그가
  들어갈 자리다.
