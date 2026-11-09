# 42. 채팅 안 읽은 메시지 표시

상태: **서버·앱 구현됨, 실기기 확인 완료 (운영 DB 채우기 남음)**

## 문제

채팅 목록에서 방별 안 읽은 메시지 수, 채팅방 안에서 "여기까지 읽었습니다" 구분선을 보여주고 싶다.

## 설계

참여자별 읽음 커서 하나(`ChatParticipant.lastReadMessageId`)로 두 기능을 모두 처리한다.

- **목록 배지**: 서버가 `id > lastReadMessageId`이고 내가 보낸 것이 아닌 메시지를 방별로 세서
  `ChatRoomListDto.unreadCount`로 내린다 (`ChatRepository.countUnreadByParticipantIds`, 쿼리 1번).
- **구분선**: 앱이 방에 들어오면 `POST /api/chat/room/{roomId}/read`를 부른다. 서버는 커서를 방의
  최신 메시지 id로 옮기고 **옮기기 전 값**을 돌려준다. 앱은 그 값보다 큰 첫 메시지(내 것 제외) 위에
  구분선을 그린다. 메시지 비교를 위해 `ChatMessageDto`에 `id`를 추가했다.
- 앱은 방을 나갈 때(`onStop`)도 `/read`를 불러, 방에 있는 동안 받은 메시지를 읽음 처리한다.
- 방 삭제(숨김) 시 커서를 최신으로 옮긴다. 안 그러면 다시 열린 방에서 옛 메시지가 안 읽음으로 잡힌다.

### 판단 근거와 택하지 않은 대안

- **시각(`lastReadAt`) 대신 메시지 id**: 같은 밀리초 메시지·시계 차이에 안전하고 PK 인덱스를 탄다.
- **커서 값은 서버가 결정**: 앱이 "N까지 읽음"을 보내면 조작·순서 뒤집힘으로 커서가 뒤로 갈 수 있다.
- **구분선 위치를 DTO(`ChatRoomListDto`/`ChatRoomResponse`)에 실어 `putExtra`로 넘기는 안 불채택**:
  채팅방 진입 경로가 4곳(목록·친구·약국·약속 공유)이라 전부 고쳐야 하고, 목록 값은 불러온 시점 값이라 낡을 수 있다.
  `/read`가 이전 값을 돌려주면 진입 경로와 무관하게 채팅방 한 곳에서 끝난다.
- **앱은 `/read`와 과거 메시지 조회를 동시에 보내고 둘 다 도착하면 그린다**: 순서대로 부르면 왕복 2번을 기다린다.
  그 사이 STOMP로 온 메시지는 모아 뒀다가 그린 뒤 붙인다.
- 상대방 읽음 표시("1"), 메시지 수신마다 읽음 전송, 개수 캐싱은 지금 규모에 불필요해 제외.

## 남은 일

- 앱 구현됨: `ChattingRoomAdapter.setLastReadId`(상대 메시지 레이아웃 2종에 `layout_unread_divider` include,
  날짜 칩과 같은 "숨겨 둔 뷰를 켜는" 방식 — 별도 viewType으로 끼우면 위치가 밀려 불채택), 목록 `tv_unread_badge`
  (`bg_unread_badge`, 두 자리 이상 대비 알약 모양, 99+)
- 앱 `ChattingRoom` 구현됨: `markRead()`와 `loadPreviousMessages()`를 동시에 보내고 `tryRender()`가 둘 다 도착했을 때
  한 번 그림(`rendered`). 그리기 전에 STOMP로 온 메시지는 `pendingMessages`에 모았다가 이어 붙임. 과거 메시지 조회가
  실패하거나 data가 null이어도 빈 리스트로 채워 `tryRender`가 영원히 기다리지 않게 함. `onStop`에서 결과를 쓰지 않는
  `markRead`(화면이 사라지는 중이라 UI를 건드리지 않음). 목록은 기존 `onResume`의 `loadChatRooms()`로 배지 갱신
- 과거 메시지 조회와 STOMP 구독이 겹치면 같은 메시지가 두 번 보일 수 있음(`id`로 중복 제거 가능, 드물어 보류)
- 운영 DB: 기존 참여자 커서를 방별 최신 id로 채우기
  (`UPDATE chat_participant p SET last_read_message_id = (SELECT MAX(m.id) FROM chat_message m WHERE m.room_id = p.chatting_room_id)`)
- 앱이 강제 종료되면 `onStop` 읽음이 누락될 수 있음 (다음 입장 시 바로잡힘)
- 한 유저가 같은 방에 USER·DRUGSTORE 참여자로 동시에 있으면(`resolveMyParticipant`가 USER 우선) 사장 상담 배지가 안 지워질 수 있음 → `tech-debt.md`
