# 39. ShareYaksokService 단위 테스트

## 범위

`ShareYaksokServiceTest` (`src/test/.../yaksok/service/`) — Mockito만 사용, Spring 컨텍스트/DB 없음
(38장 `YaksokServiceTest`와 같은 방식). 리포지토리 5개를 목으로 두고 서비스는 생성자로 직접 만든다.

검증하는 동작:

- **saveSharedYaksok** — 채팅방에 공유된 약속이면 `ShareYaksok(receiver, yaksok)` 저장, 자기 약속은
  저장 거부(`save` 호출 없음)
- **getSharedYaksokDetail** — 이미 저장한 약속은 채팅방 조회 없이 통과(소유자 닉네임 포함 DTO),
  저장도 안 했고 채팅으로 받은 적도 없으면 거부
- **getSharedUserList / getSharedYaksokList** — 공유자 목록 매핑, `senderId` 유무에 따라 조회 쿼리 분기
- **deleteSharedYaksok** — 본인이 저장한 행 삭제

## 판단

- 공유 여부 검증(`validateSharedToMe`)은 private이라 저장/상세 테스트를 통해서만 건드린다.
  "방이 없음" 쪽 분기만 edge로 확인했고, "방은 있으나 공유 메시지 없음"(`countSharedInRooms == 0`)은
  같은 예외라 별도 테스트하지 않았다.
- edge는 3개(자기 약속 저장, 공유받지 않은 약속 상세, 위 방 없음 분기)로 제한. 이미 저장한 약속
  중복 저장과 저장하지 않은 약속 삭제 예외는 미검증.

## 테스트 중 확인한 점

버그로 볼 만한 것은 발견하지 못했다.
