# 44. 약국 사장 상담방 삭제 실패 수정

## 문제

사장 계정이 상담 탭에서 채팅방을 삭제하면 "채팅방을 지우지 못했습니다."가 뜬다. 통신은 성공했고
서버가 400을 돌려준 것이다(앱은 `isSuccessful()` false를 이 토스트로 처리).

## 원인

`ChatService.deleteChattingRoom`이 `findByChattingRoomIdAndUserId`로만 참여자를 찾았다. 이 쿼리는
USER 타입 참여자만 찾는다.

- 친구방: 두 참여자 모두 USER → 찾아져서 삭제(`hidden=true`) 성공.
- 상담방의 사장: DRUGSTORE 타입 참여자(`user_id` 없음, `hpid`만 있음) → 빈 결과 →
  "해당 연결된 채팅방 관계가 없습니다." 예외 → 컨트롤러가 400으로 변환.

회원 탈퇴와는 무관하다. 채팅방 구조 분리(9장) 때부터 사장 쪽 `hidden` 설정 경로가 삭제 메서드에
없었다. 설계(9장)는 "약국 참여자의 `hidden` = 사장의 상담 목록에서 숨김"으로 정의하고 목록
쿼리도 그 전제였지만 숨김을 켜는 코드만 빠져 있었다.

부수 결함: 반환이 항상 `getMyChatRooms(userId)`(친구방 목록)였다. 앱은 삭제 응답의 `data`로 현재
탭의 목록을 통째로 교체하므로, 삭제가 되더라도 상담 탭에 친구방 목록이 채워졌을 것이다.

## 수정

`ChatService.deleteChattingRoom` 한 곳.

- 참여자 조회를 기존 헬퍼 `findMyParticipant(roomId, userId)`로 교체. USER로 먼저 찾고 없으면
  사장의 약국(DRUGSTORE)으로 찾는다.
- 반환을 찾은 참여자 타입으로 분기: DRUGSTORE면 `getConsultRooms`, 아니면 `getMyChatRooms`.
- API 스펙·DTO·앱은 변경 없음. 옛 앱과도 호환된다.

## 택하지 않은 대안

- **요청에 `hpid` 추가**: 서버가 `user.getMyDrugStore()`로 이미 안다. 클라이언트 값은 어차피
  서버 값과 대조해야 해서 중복이고, API와 `ChatRoomListDto`, 앱까지 바뀐다. 한 사장이 약국을
  여러 개 가지게 되면 그때 필요하다.
- **요청에 방 종류 추가**: 같은 이유. 참여자 행의 타입으로 서버가 판단할 수 있고, 클라이언트가
  잘못 보내면 오히려 응답 목록이 틀린다.
- **반환 제거 + 앱이 `loadChatRooms()` 재조회**: 서버는 단순해지지만 통신이 한 번 늘고 응답
  타입이 바뀌어 서버·앱 동시 배포가 필요하다(서버만 먼저 나가면 옛 앱이 `data=null`을 받아 목록이
  빈다).
- **상담용 삭제 엔드포인트 신설**: 같은 일을 하는 `findMyParticipant`가 이미 있고, 앱도 탭별로
  호출을 나눠야 한다.

## 남은 것

- `deleteChattingRoom` 앞부분의 `loginUser` 조회는 쓰이지 않는다(제거 가능).
- 숨김 변경이 같은 트랜잭션 안의 목록 쿼리에 반영되는지(JPQL 자동 flush)는 실기기에서 삭제 직후
  지운 방이 목록에 남지 않는 것으로 확인함.
