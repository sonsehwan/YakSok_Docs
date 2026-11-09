# 41. 엔티티 id 타입 `Long` 통일

## 문제

자체 발급 id는 `Long`인데 두 곳만 달랐다.

- `ChatMessage.roomId` — `String`. `ChattingRoom.id`(`Long`)를 가리키면서도 문자열이라
  `ChatService`/`ShareYaksokService`에서 `Long.valueOf` / `String.valueOf`를 오가야 했고,
  숫자가 아닌 값이 들어오면 `NumberFormatException`, `varchar` 비교라 사전순·인덱스 불리.
  왜 `String`이었는지 기록은 없다(08장 기술 부채에 "기존 설계"로만 남아 있음). STOMP로 시작하며
  JSON/토픽 경로의 값을 그대로 받은 흔적으로 추정.
- `Notification.id` — `int`. 나머지 PK는 전부 `Long`.

## 변경

| 위치 | 변경 |
|---|---|
| `ChatMessage.roomId`, `ChatMessageDto.roomId` | `String` → `Long` |
| `ChatRepository` | `findByRoomId…`, `findLastMessagesByRoomIds`, `countSharedInRooms`, `deleteAllByRoomId`의 파라미터를 `Long`/`List<Long>`으로 |
| `ChatService` | `saveMessage`/`getChatMessages`/`handleUserWithdrawal`/`toRoomList`의 변환 제거, 마지막 메시지 맵 키 `Long` |
| `ShareYaksokService.validateSharedToMe` | `String` 변환 없이 `findRoomIdByUserId` 결과를 그대로 전달 |
| `Notification.id`, `NotificationStatusChangedEvent.notificationId` | `int` → `Long` |

## 택하지 않은 것

- **`DrugStore.hpid`, `User.loginId`, `User.socialId`** — 이름에 id가 붙었을 뿐 외부(공공데이터포털·카카오)나
  사용자가 정하는 문자열이라 `String` 유지.
- **`FriendRequestId`(`String userId/friendId`)** — 어디서도 참조하지 않는 이메일 PK 시절 잔재(죽은 코드).
  타입을 바꾸지 않고 삭제 대상으로 남겨둠(아직 미삭제).
- **프론트 `model/ChatMessage.roomId`(`String`)** — 이번에 손대지 않음. 서버가 숫자로 내리면 Gson이 문자열 필드로
  읽고, 앱이 `"10"`을 보내면 Jackson이 `Long`으로 변환하므로 구버전 앱과 호환된다.

## 배포 시 주의 (DB)

`chat_message.room_id`(varchar → bigint)와 `notification.id`(int → bigint) 컬럼 타입 변경이 필요하다.
운영 DB에 숫자가 아닌 `room_id`가 있으면 `MODIFY`가 실패하므로 먼저 확인할 것.
`notification`을 참조하는 FK가 있으면 같이 넓혀야 한다. 배포는 main push 즉시라 DDL을 먼저 적용한 뒤 jar를 올린다.

## 검증

`./gradlew test`: `YakSokApplicationTests.contextLoads`만 실패(로컬 MySQL 없음, 이번 변경과 무관). 나머지 통과.
