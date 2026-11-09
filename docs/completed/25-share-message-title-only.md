# 25. 약속 공유 메시지 본문을 약속명만으로

## 문제

채팅방의 약속 공유 카드(`SHARE_YAKSOK`)는 "약속 공유" 태그 + 약속명 + "자세히 보기" 버튼으로
디자인이 바뀌었는데, 저장되는 메시지 본문은 옛 말풍선 시절의 `"OO님이 약속을 공유했습니다.\n약속명"`
그대로였다. 카드는 마지막 줄만 필요해서 `ChattingRoomAdapter.shareTitle`이 `\n` 뒤를 잘라 썼다.
문자열 형식에 기대는 파싱이고, 앞부분(보낸 사람 이름 문장)은 카드에서 쓰이지 않는 데이터였다.

## 판단

- **본문을 약속명만 저장한다.** `YaksokDetail.buildShareMessage`가 `yaksok.getTitle()`만 반환.
  보낸 사람 이름은 어차피 `senderName` 스냅샷과 카드 위 이름 라벨로 보인다.
- **채팅 목록의 마지막 메시지 미리보기는 서버가 만든다.** 목록(`ChatRoomListDto.lastMessage`)은
  마지막 메시지의 `message`를 그대로 보여주는데, 본문이 약속명뿐이면 "감기약"만 떠서 공유인지
  알 수 없다. `ChatService.previewOf`가 타입이 `SHARE_YAKSOK`이면 "약속을 공유했어요"를, 아니면
  `message`를 돌려준다. 옛 형식으로 저장된 공유 메시지도 같은 문구로 나온다.
- **`shareTitle`은 지웠다.** 처음에는 옛 메시지(`\n` 앞 문장 포함)도 카드에 제대로 나오도록 남겨뒀지만,
  옛 데이터는 신경 쓰지 않기로 해서 제거했다. 카드가 `message`를 그대로 제목으로 쓰므로 DB에 남은 옛
  공유 메시지는 카드 제목에 두 줄이 다 나온다. 채팅 목록 미리보기는 서버 `previewOf`가 타입만 보므로
  옛 메시지도 영향이 없다.

## 택하지 않은 대안

- **`ChatRoomListDto`에 마지막 메시지 타입 필드 추가:** 문구를 앱이 정할 수 있지만 응답 필드와 앱
  분기가 함께 늘어난다. 서버가 문구를 정하면 앱 수정 없이 한 곳으로 끝난다.
- **저장값을 그대로 두고 카드에서만 파싱:** 코드 변경은 없지만 문자열 형식 의존과 이름 문장 저장이
  계속 남는다.

## 변경

- 백엔드 `ChatService`: `previewOf(ChatMessage)` 추가, 채팅 목록 조회가 `lastMessage`에 사용.
- 프론트 `YaksokDetail.buildShareMessage`: 약속명만 반환.
- 프론트 `ChattingRoomAdapter`: `shareTitle` 삭제, 공유 카드 제목은 `msg.getMessage()`를 그대로 사용.
