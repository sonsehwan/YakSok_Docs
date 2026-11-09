# 4. 회원 탈퇴 오류 수정 & 개발용 목 데이터 (2026-09-02)

**상태:** `@OnDelete` 추가와 목 데이터 시더는 코드 반영 완료.
운영 DB 스키마 반영(4.3의 절차)은 배포 시 수행.

## 4.1 증상

앱에서 회원 탈퇴 시 **"탈퇴 실패 (코드: 400)"** 토스트만 뜨고 탈퇴되지 않음.
`MyInfo.deleteUserFromServer()` → `DELETE /api/users/{id}` → `UserService.deleteUser()`.

## 4.2 원인 — 연관 데이터 미삭제로 FK 제약 위반

`UserService.deleteUser()`는 `userRepository.delete(user)` 한 줄이고,
`User` 엔티티는 `yaksoks`만 `cascade = ALL`이라 약속/약/알림만 함께 지운다.
나머지 자식 테이블은 아무도 정리하지 않는다. MySQL FK 기본값이 `ON DELETE RESTRICT`이므로
자식 행이 하나라도 있으면 `Cannot delete or update a parent row`가 나고,
`@Transactional` 롤백 → 컨트롤러 `catch (Exception)` → 400 `FAIL_DELETE_USER`.

삭제를 막던 테이블 (수정 전):

| 테이블                                                   | 참조 컬럼                     | `@OnDelete`   |
| -------------------------------------------------------- | ----------------------------- | ------------- |
| `friend_relation`                                        | `user_id`, `friend_id`        | ❌ 둘 다 없음 |
| `friend_request`                                         | `requester_id`, `receiver_id` | ❌ 둘 다 없음 |
| `chat_participant`                                       | `user_id`                     | ❌ 없음       |
| `share_yaksok`, `user_devices`, `yaksok`, `notification` | —                             | ✅ 있음       |

`friend_relation.friend_id`, `friend_request.receiver_id` 때문에
**다른 사용자가 나를 친구로 갖고만 있어도** 탈퇴가 막힌다.

### `@OnDelete`가 붙어 있어도 안 먹히는 함정

`ddl-auto: update`는 **이미 존재하는 FK 제약을 수정하지 않는다.**
어노테이션을 나중에 붙여도, 그 전에 생성된 테이블이면 DB 제약은 여전히 `RESTRICT`다.
`@OnDelete`는 **테이블을 새로 만들 때만** FK를 `... ON DELETE CASCADE`로 생성한다.

## 4.3 택한 해결책 — DB 레벨 CASCADE

`FriendRelation`(`user`, `friend`), `FriendRequest`(`requester`, `receiver`),
`ChatParticipant`(`user`)에 `@OnDelete(action = OnDeleteAction.CASCADE)` 추가.
탈퇴는 `userRepository.delete(user)` 그대로 두고 DB가 자식 행을 지우게 한다.

**서비스 코드에서 자식 행을 먼저 삭제하는 방식을 택하지 않은 이유:**
`friend_relation`은 `@EmbeddedId` + `@MapsId`라 벌크 삭제 쿼리를 직접 써야 하고,
참조 테이블이 6개라 삭제 순서를 손으로 관리해야 한다. 새 자식 테이블이 생길 때마다
`deleteUser`를 고쳐야 하는 것도 부담. DB CASCADE는 FK 정의에 한 번만 선언하면
이후 자동으로 따라온다.

**적용 절차 (스키마 재생성 필요 — `update`로는 FK가 안 바뀜):**

1. `application.yaml` → `ddl-auto: create`
2. 서버 1회 실행 (DB 전체 재생성)
3. 종료 → `ddl-auto: update`로 되돌림
4. 재실행
5. `SHOW CREATE TABLE friend_relation / friend_request / chat_participant`로
   각 FK에 `ON DELETE CASCADE` 확인

운영 DB에 실사용자가 있으면 `create` 대신
`ALTER TABLE ... DROP FOREIGN KEY ... / ADD CONSTRAINT ... ON DELETE CASCADE`로 FK만 교체.

## 4.4 탈퇴 시 상대방에게 보이는 것

| 데이터                                                     | 결과                                            |
| ---------------------------------------------------------- | ----------------------------------------------- |
| 친구 관계 (양방향 2행)                                     | 삭제 → **상대 친구 목록에서도 사라짐**          |
| 친구 요청, 채팅 참여자, 약속/약/알림, 공유 약속, 기기 토큰 | 삭제                                            |
| **채팅방 (`chatting_room`)**                               | **유지** (user 참조 FK 없음)                    |
| **채팅 메시지 (`chat_message`)**                           | **유지** (`roomId`, `sender`가 문자열, FK 아님) |

채팅방이 남아서 상대 화면에는 방과 지난 대화가 그대로 보인다.
`resolveRoomName()`이 "나 외 참여자" 닉네임으로 방 이름을 만드는데 탈퇴로 그 참여자가
사라지면 `room.getRoomName()`으로 fallback → 친구 채팅방은 `roomName`이 `null`이라
방 이름이 빈 값이 될 수 있다. 채팅방까지 정리하려면 별도 처리 필요(4.7).

DB에서는 즉시 삭제되지만 상대 앱 화면은 목록을 다시 불러올 때 반영된다.

## 4.5 개발용 목 데이터 시더

`config/DataInitializer.java` — `ApplicationRunner` 구현체.
`create`로 DB를 초기화할 때마다 수동 재가입하는 번거로움을 없앤다.

서버 시작 시 `users` 테이블이 비어 있으면:

- 일반 유저 4명: `test1~test4@yaksok.com` / 비밀번호 `1234` / 닉네임 `테스트유저1~4` / role `NORMAL`
- 4명 전원 상호 친구 (수락된 `friend_request` 6행 + `friend_relation` 12행)

`userRepository.count() > 0`이면 스킵하므로 `create`/`update` 어느 쪽이든 안전하다.
`@ConditionalOnProperty(matchIfMissing = true)`라 기본 활성. 운영 배포 시 `app.seed.enabled: false`로 끈다.

## 4.6 변경 범위

**백엔드**

- 수정: `user/entity/FriendRelation`(`@OnDelete` ×2), `user/entity/FriendRequest`(`@OnDelete` ×2),
  `chat/entity/ChatParticipant`(`@OnDelete` ×1)
- 신규: `config/DataInitializer`

## 4.7 남은 것 (기술 부채)

- 탈퇴한 유저의 채팅방·메시지가 상대 쪽에 남는다. 참여자 0명이면 방+메시지 삭제,
  아니면 시스템 메시지("상대방이 탈퇴했습니다") 후 참여자 행만 제거하는 처리 필요.
