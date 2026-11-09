# 9. 채팅방 구조 분리 — 약국 상담 채팅 ↔ 유저 사적 채팅

**상태:** 완료. 백엔드·프론트엔드 커밋(양쪽 `main`), 스키마 `create` 재생성 후 로컬 실행 검증
통과(약국 상담 고객/사장 양쪽, 친구 채팅, 재입장, 탈퇴 후 방 정리, 약속 공유).

## 9.1 문제 정의

지금은 **약국과의 상담 채팅**과 **유저끼리의 사적 채팅**이 같은 채팅방으로 합쳐진다.

- 약국 = `myDrugStore`가 달린 `User` 계정 하나. 개인 정체성과 약국 정체성이 한 계정에 공존한다.
- 유저 B가 약국 P의 상담 버튼을 눌러도(`DrugStoreDetail` → `POST /api/chat/room`, hpid),
  유저 B가 P의 사장과 친구가 되어 친구 채팅을 걸어도(`POST /api/chat/room/friend`, friendId),
  둘 다 `(clientUser, pharmacyUser)` 유저 쌍으로 방을 찾고 만든다.
- 방 조회 키가 [`findExistingRoom(user1, user2)`](https://github.com/sonsehwan/YakSok_Backend/blob/main/src/main/java/com/sehwan/YakSok/chat/repository/ChattingRoomRepository.java#L16)
  **유저 쌍 하나뿐**이라, B가 P 사장과 친구이기도 하면 상담방과 친구방이 **같은 방으로 병합**된다.
- 방 이름은 `resolveRoomName()`이 "나 외 참여자"에 `myDrugStore`가 있으면 약국명으로 내주지만,
  **말풍선의 발신자 닉네임**(`saveMessage` / `getChatMessages`가 `userRepository.findByEmail(sender).nickname`으로
  전역 조회)은 사장의 개인 닉네임으로 뜬다. 방 이름은 "○○약국"인데 대화 상대 이름은 "홍길동"이다.

## 9.2 원인

**참여자를 항상 `User`로만 모델링**한 것. `ChatParticipant`는 `user` FK만 채우고,
방은 `List<ChatParticipant>`의 유저 조합으로만 식별된다. "이 사람이 어떤 자격(개인 / 약국)으로
이 방에 있는가"라는 정보가 스키마에 없다.

원래 설계 의도는 있었던 것으로 보인다 — `chat/entity/ParticipantType`(`USER` / `DRUGSTORE`) enum이
**만들어져 있지만 아무 데서도 안 쓰이고**, `ChatParticipant.user`는 `nullable = true`에
`// 약국이 참여자일 경우 이 필드는 DB에 null로 저장됨` 주석까지 달려 있다. 절반만 구현되고 멈췄다.

## 9.3 택한 방향 — B안: 약국을 독립 참여자로

`ChatParticipant`가 `user` **또는** `drugStore` 중 하나를 가리키는 다형 참여자로 만든다.

| 방 종류        | 참여자 구성                          |
| -------------- | ------------------------------------ |
| 약국 상담방    | `[User(고객), DrugStore(약국)]`      |
| 유저 사적 채팅 | `[User(A), User(B)]`                 |

B가 약국 P의 사장 A와 친구여도 상담방은 `[B, P]`, 친구방은 `[A, B]`로 **참여자 집합이 달라
자연히 별개 방**이 된다. 방에 타입 딱지를 붙여 억지로 구분할 필요가 없다.
발신자 이름도 참여자가 `drugStore`면 약국명, `user`면 닉네임으로 깔끔하게 갈린다.

### 검토했지만 택하지 않은 대안

- **A안 — 참여자는 계속 항상 `User`, `ChattingRoom.roomType`(상담 / 친구) 딱지로만 구분.**
  `findExistingRoom`에 `roomType` 조건을 추가해 같은 유저 쌍이라도 방을 나눈다. DB 변경이 가장 작다.
  택하지 않은 이유: 발신자 이름 문제가 안 풀린다. 약국 상담방에서 사장이 보낸 메시지의 `sender`가
  여전히 개인 계정이라, "이 방에서는 약국 이름으로 보여라"를 매번 방 맥락에 기대어 우회해야 한다.
  사장의 **개인 채팅 목록과 상담 목록도 같은 유저 행 하나에 엉킨 채**로 남아, 9.6의 목록 분리
  쿼리를 짤 때 "이 방이 상담인가"를 참여자가 아니라 roomType으로만 판단하게 된다 — 참여자 구조가
  거짓말을 하는 상태.
- **약국마다 별도의 "약국 페르소나" `User` 행을 하나 더 만든다.** 상담방은 그 페르소나 계정과
  연결. 택하지 않은 이유: 계정이 2배가 되고, 로그인·JWT·기기 토큰·비밀번호가 페르소나 계정에도
  달려야 하는지 등 인증(8장) 전반이 복잡해진다. `DrugStore` 엔티티가 이미 있는데 유저를 하나 더
  파생시킬 이유가 없다.
- **`roomType`을 아예 두지 않고 참여자 집합만으로 구분.** B안이면 기술적으로 가능하다(상담방엔
  `drugStore` 참여자가 있고 친구방엔 없다). 9.4에서 `roomType`을 그래도 두기로 한 판단과 근거는
  아래에 따로 적었다.

## 9.4 엔티티 설계

### `ChatParticipant`

- `user` — `@ManyToOne`, nullable (이미 그렇다)
- `drugStore` — `@ManyToOne(DrugStore)`, nullable 신규 추가. `@OnDelete(CASCADE)`
  (4장에서 `chat_participant.user_id`에 이미 걸어둔 것과 동일 정책)
- `participantType` — `ParticipantType`(`USER` / `DRUGSTORE`) enum, `@Enumerated(STRING)`, NOT NULL.
  **이미 있는 enum을 이제 실제로 채운다.**
- **불변식:** `participantType == USER`면 `user != null && drugStore == null`,
  `participantType == DRUGSTORE`면 그 반대. 편의 생성자를 `ChatParticipant(User)` /
  `ChatParticipant(DrugStore)` 둘로 나눠 호출부에서 잘못 조합할 수 없게 한다.
- `hidden`, `joinedAt`는 참여자 단위 그대로 유지. 약국 참여자의 `hidden`은 "사장의 상담 목록에서
  이 방을 숨김"을 뜻하게 된다.

### `ChattingRoom.roomType` — 둔다 (판단)

`RoomType`(`PHARMACY_CONSULT` / `DIRECT`) enum을 추가한다.

참여자 집합만으로도 구분은 되지만, **방 목록 조회·필터가 매번 참여자를 들여다봐야 하는 것**을
피하려는 것이다. 이 프로젝트는 이런 판별을 명시적 타입으로 두는 쪽으로 이미 통일돼 있다
(`FcmMessageType.REFRESH` 3.3, JWT 화이트리스트 8.8 등 — "실수했을 때 안전한 쪽으로"). 
다만 이건 A안처럼 **roomType이 유일한 구분자**인 것과는 다르다. B안에서 roomType은
참여자 구조가 이미 말하고 있는 사실의 **캐시**이고, 진실의 원천은 참여자다.

### 리포지토리 변경

- `findExistingRoom(user1, user2)` → 두 개로 분리
  - `findDirectRoom(userId1, userId2)` — 친구 채팅용. 두 참여자가 모두 `USER`
  - `findConsultRoom(customerUserId, drugStoreHpid)` — 상담용. 한쪽은 `USER`, 한쪽은 해당 `DRUGSTORE`
- `findAllWithParticipantsByUserId(userId)` — 지금은 `cp.user.id = :userId`. 그대로 두되
  용도가 "**내가 유저 자격으로 들어간 방**"으로 좁혀진다(친구방 + 내가 고객인 상담방).
- 신규 `findConsultRoomsByDrugStoreHpid(hpid)` — 약국 사장의 상담함용. 참여자에 그 약국이 있는 방.
  fetch join도 `LEFT JOIN FETCH p.drugStore` 추가.

## 9.5 메시지 발신자(`chat_message.sender`) — 두 방식 검토

약국 상담방에서 **약국 쪽이 보낸 메시지**는 개인이 아니라 약국으로 귀속돼야 한다.
그런데 `chat_message.sender`는 지금 **email 문자열**이고(User PK가 email이던 시절 잔재, 5장·6.1),
6.1에 "email→id 전환은 JWT/WebSocket 인증 sweep에 묶어서 한다"고 이미 적혀 있다.
이 작업이 그 지점을 건드리므로 두 방식을 다 검토한다.

### 방식 1 — `sender` 컬럼 유지, 렌더링 시점에 자격 판별

- `chat_message.sender`에 계속 사람의 email을 저장(고객 email 또는 약국 사장 email).
- 이름을 만들 때(`saveMessage`, `getChatMessages`, `resolveRoomName`),
  `sender` email → `User` → 그 방의 `ChatParticipant`를 찾아:
  - 그 유저가 이 방의 **약국 참여자의 사장**이면 → `drugStore.dutyName`
  - 아니면 → `user.nickname`
- 장점: `ChatMessage` 엔티티·`ChatMessageDto`·프론트 전송 payload·`ChattingRoomAdapter`의
  내/상대 판별(`msg.getSender().equals(myEmail)`)을 **안 건드린다.** 스키마 변경이 참여자
  테이블에만 국한된다.
- 단점:
  - "이 email이 이 방 약국 참여자의 사장인가"가 `message.sender → user → user.myDrugStore ==
    room의 drugStore` **3단 조인**이다. `saveMessage`는 지금 방 맥락 없이 `roomId`와 `sender`만
    갖고 있어서, 참여자 조회 로직을 새로 붙여야 한다.
  - `user.myDrugStore`가 바뀌면(약국 변경·해제) 과거 메시지의 귀속이 흔들린다.
  - 6.1의 email→id 문제(사칭 가능, 재가입 시 재귀속, email이 STOMP payload로 상대에게 전달)는
    그대로 남는다. 나중에 또 이 파일들을 건드린다.

### 방식 2 — `sender`를 참여자 기준으로 전환

- `chat_message`에 `senderType`(`USER` / `DRUGSTORE`) + `senderUserId`(nullable) +
  `senderDrugStoreHpid`(nullable), 또는 `senderParticipantId`(FK to `chat_participant`) 하나.
- 발신자 = "이 방에서 나를 대표하는 참여자". 클라이언트가 방에 들어올 때 서버가 내려준
  `myParticipantId`(또는 `myParticipantType`)를 payload에 실어 보낸다.
- 이름 매핑은 참여자 → (`user.nickname` or `drugStore.dutyName`)로 **1단**. 방 맥락이 곧 발신자.
- 장점: 귀속이 사장·닉네임 변경에 안 흔들린다. 6.1의 email→id 목표를 이 작업이 함께 달성한다.
  email이 브로드캐스트 payload에서 사라진다.
- 단점: 범위가 넓다.
  - 백엔드: `ChatMessage`, `ChatMessageDto`(`toEntity`/`from`), `saveMessage`,
    `getChatMessages`의 닉네임 매핑(`findByEmailIn` → 참여자 기반), `findLastMessagesByRoomIds`
    (roomId만 쓰므로 영향 없음), `resolveRoomName`.
  - 프론트: `model/ChatMessage`, `ChattingRoom`의 전송부(`new ChatMessage(roomId, myEmail, …)`),
    `ChattingRoomAdapter`의 `myEmail` 비교 → `myParticipantId` 비교, `ChatRoomResponse`에
    `myParticipantId` 추가.
  - `ShareYaksokService`의 약속 공유 검증(9.7)도 같이 본다.
  - **WebSocket 인증은 8.12에서 계속 제외 중**이다. 즉 이 방식을 지금 하면, 서버가 클라이언트가
    보낸 `senderParticipantId`를 (지금 email을 믿는 것과 같은 수준으로) **일단 믿는다.**
    진짜 사칭 방지는 `StompChannelInterceptor` 도입 때 완성된다.

### 중간안 (검토 대상)

방식 2의 **저장 모델만 지금 도입**하되 WebSocket 인증은 기다린다. 클라이언트가
`senderParticipantId`를 보내고 서버는 당분간 신뢰(현행 email과 동일한 신뢰 수준). "발신자
데이터 모델"과 "WebSocket 인증"을 분리해서, 깨끗한 모델을 먼저 얻고 사칭 방지는 8.12에
남긴다. 기술 부채 표(5장)의 "채팅 sender가 email" 항목이 이걸로 해소되고, "STOMP 참여자 검증
없음" 항목만 남는다.

### 추천

**중간안.** 어차피 B안이 `ChatMessage` 주변을 다 건드리고, 6.1이 "두 번 하지 않는다"고 못박아
뒀다. 방식 1로 갔다가 인증 sweep 때 같은 파일을 다시 여는 게 6.1의 취지에 어긋난다.
단 최종 결정은 착수 시 9.9로 남긴다.

## 9.6 약국 사장의 채팅 목록 — 탭/화면 분리

사장 계정(`role == DRUGSTORE`)은 **개인 친구 채팅**과 **약국 상담 채팅**을 둘 다 갖는다.
두 목록은 참여자 축이 달라 **쿼리가 서로소**다.

| 목록          | 조회 기준                                  | 사장 계정이 참여자인가            |
| ------------- | ------------------------------------------ | -------------------------------- |
| 개인 채팅     | `ChatParticipant.user.id = 사장 userId`    | O (USER 자격)                    |
| 약국 상담     | `ChatParticipant.drugStore.hpid = 내 약국` | X (참여자는 `DrugStore`)         |

- 일반 유저(`role == NORMAL`): 탭 없음. 자기가 고객인 상담방은 `user.id` 조회에 그대로 잡히므로
  기존 단일 목록에 섞여 나온다(약국명으로 표시).
- 사장: `ChatRoomList`에 `TabLayout` [상담 | 개인] 추가.
  - `GET /api/chat/room` — 개인(기존 그대로)
  - `GET /api/chat/room/consult` — 내 약국의 상담방 (신규). 서버가 토큰에서 사장 userId → 그
    유저의 `myDrugStore.hpid` → 상담방 목록. 약국 미등록이면 빈 목록.
  - 엔드포인트를 나눌지(`/consult`) 쿼리 파라미터(`?type=consult`)로 할지는 9.9.

## 9.7 곁가지로 같이 볼 것

- **`resolveRoomName()` 재작성.** "나 외 참여자"를 찾아 그게 `drugStore`면 `dutyName`,
  `user`면 `nickname`. 지금은 `participant.getUser().getMyDrugStore()`를 보는데, 참여자
  자체가 `drugStore`일 수 있게 바뀌므로 분기 기준이 참여자 타입으로 바뀐다.
- **약속 공유 검증(`ShareYaksokService.validateSharedToMe`)** — `findRoomIdByUserId(userId)`로
  "내가 참여한 방"을 구한다. 고객은 상담방에서도 `USER` 참여자라 그대로 동작한다. 하지만
  **약국 사장이 상담방에 약속을 공유**하는 경우, 사장은 그 방의 `user` 참여자가 아니라서
  `findRoomIdByUserId`에 안 잡힌다. 약국이 약속을 공유하는 기능이 있다면
  `findRoomIdByDrugStoreOwner(userId)` 같은 경로가 필요하다. (약국→고객 약속 공유가 실제
  시나리오인지 먼저 확인 — 9.9)
- **`getOrCreateFriendRoom` 친구 관계 미검증(8.12 / 5장 기술 부채).** 이 메서드를 어차피
  다시 쓰므로, 친구 검증(`friendRelationRepository`로 실제 친구인지 확인)을 이때 함께 넣는
  것을 권장. 지금은 `friendId`에 아무 값이나 넣으면 남과도 방이 생긴다.
- **탈퇴 시 채팅방 미정리(4.7 / 5장).** 이번 범위는 아니지만 참여자 모델이 바뀌므로,
  4.7에서 예고한 "참여자 0명이면 방 삭제" 처리를 넣을 거면 스키마가 흔들리는 지금이 같이
  하기 좋은 시점이다. 별도 판단.

## 9.8 변경 예정 범위

**백엔드**

| 구분 | 대상 |
| --- | --- |
| 엔티티 | `ChatParticipant`(`drugStore` FK, `participantType`, 편의 생성자 분리, 불변식), `ChattingRoom`(`roomType`) |
| enum | `RoomType` 신규. `ParticipantType`은 이미 있음 — 사용 시작 |
| 리포지토리 | `findExistingRoom` → `findDirectRoom` / `findConsultRoom` 분리, `findConsultRoomsByDrugStoreHpid` 신규, fetch join에 `drugStore` 추가 |
| 서비스 | `ChatService.getOrCreateRoom`(상담방 = `[고객 User, DrugStore]`), `getOrCreateFriendRoom`(친구방 + 친구 검증), `getMyChatRooms`(개인), `getConsultRooms` 신규, `resolveRoomName` 재작성, `saveMessage`/`getChatMessages` 발신자 처리(9.5 결정에 따라) |
| 컨트롤러 | `ChatRoomController`에 상담 목록 엔드포인트 추가, `ChatRoomResponse`에 `myParticipantId`(9.5 방식 2/중간안 채택 시) |
| DTO | `ChatMessage`/`ChatMessageDto`(9.5 방식에 따라), `ChatRoomListDto`에 `roomType` 노출 여부 |

**프론트엔드**

| 구분 | 대상 |
| --- | --- |
| 목록 | `ChatRoomList` — `role == DRUGSTORE`면 `TabLayout` [상담 / 개인], 탭별 조회. `ChatApi`에 상담 목록 메서드 추가 |
| 채팅방 | `ChattingRoom` — 내/상대 판별을 `myEmail` → `myParticipantId`로(9.5 방식에 따라), 방 이름/헤더 |
| 어댑터 | `ChattingRoomAdapter` 생성자 인자(`myEmail` → `myParticipantId`), `ChatRoomListAdapter`에 배지/구분 표시 여부 |
| 모델 | `model/ChatMessage`, `model/response/ChatRoomResponse`, `ChatRoomListDto` |
| 진입점 | `DrugStoreDetail`(상담, 변경 거의 없음 — 서버가 방 종류를 결정), 친구 채팅 진입점(변경 없음) |

**데이터**

- 기존 채팅방·메시지는 **버린다.** `ddl-auto: create`로 스키마 재생성(4.3과 동일 절차).
  마이그레이션 스크립트 없음.

## 9.9 확정된 결정

1. **발신자 저장 방식 → 중간안 + 이름 스냅샷(하이브리드).** `chat_message`에
   `sender_participant_id`(발신자 참여자 id) + `sender_name`(보낼 당시 이름 스냅샷) 두 컬럼.
   `sender`(email) 컬럼 제거. `sender_participant_id`에 하드 FK를 걸지 않는다 — 참여자가 방을
   나가도(행 삭제) 지난 메시지는 남아야 하므로.
   - 원래 중간안은 "조회 시 `sender_participant_id` → 참여자 → displayName 매핑". 여기에
     이름 스냅샷을 더한 이유: 소프트-나가기(`hidden=true`, 행 유지)는 이름이 계속 뜨지만
     **회원 탈퇴로 참여자 행이 삭제되면** 매핑이 깨져 "알 수 없음"이 된다. 이 프로젝트는
     "상대가 나가도 내가 메시지를 보내면 다시 들어온다"는 유연한 참여를 이미 구현해 뒀는데,
     탈퇴 케이스에서 이름이 사라지는 건 그 방향과 안 맞는다. 스냅샷이면 탈퇴·닉네임 변경·
     약국명 변경 어느 경우에도 과거 말풍선 이름이 안 흔들린다(카톡과 동일). 부수 효과로
     조회 경로에서 참여자 배치 조회가 사라져 더 단순해진다.
   - 검토했지만 택하지 않음: **방식 1(sender=email 유지, 렌더링 시 자격 판별)** — 6.1의
     "email→id는 두 번 안 건드린다"에 어긋남. **방식 2 풀버전(StompChannelInterceptor까지)**
     — WebSocket 인증은 독립적으로 뗄 수 있어 범위 과대.
2. **상담 목록 엔드포인트 → 별도 경로 `GET /api/chat/room/consult`.** 권한·쿼리 축이 개인
   목록과 완전히 다르고 프론트 Retrofit 메서드도 어차피 분리됨. 쿼리 파라미터(`?type=consult`)
   불채택.
3. **약국→고객 약속 공유 → 지원 안 함.** 따라서 9.7의 `findRoomIdByDrugStoreOwner` 경로
   불필요. `ShareYaksokService` 변경 없음(`findRoomIdByUserId` = 내가 USER 참여자인 방, 고객은
   상담방에서도 USER라 그대로 동작).
4. **`getOrCreateFriendRoom` 친구 검증 → 포함.** `friendRelationRepository.existsById(
   new FriendRelationId(me, friend))`로 실제 친구일 때만 방 생성. (기술부채 5장 "친구 미검증"
   해소)
5. **`roomType`을 프론트 목록 DTO로 안 내린다.** 사장은 탭 `[상담|개인]`으로 이미 갈리고,
   일반 유저는 방 이름(닉네임 vs 약국명)이 이미 구분 신호를 줌. 배지 추가는 정보량 대비
   레이아웃 비용만 큼. 단 `ChatRoomListDto`에 `myParticipantId`는 추가한다(아래 참고) —
   배지가 아니라 목록에서 방을 열 때 메시지 전송에 필요한 데이터라서 결정 5의 범위 밖.
6. **탈퇴 시 방 정리 → 포함.** `UserService.deleteUser`가 유저 삭제 직전
   `ChatService.handleUserWithdrawal(userId)` 호출: 내 참여자 행 삭제 → 그 방의 남은 참여자가
   0명이면 방 + 메시지(`chat_message` 벌크 delete)까지 삭제, 아니면 방 유지. (4.7 예고분 반영)

## 9.9.1 실제 반영된 변경 (백엔드)

| 파일 | 변경 |
| --- | --- |
| `chat/entity/RoomType` | **신규.** `PHARMACY_CONSULT` / `DIRECT` |
| `chat/entity/ParticipantType` | 기존 enum, 이제 실제 사용 |
| `chat/entity/ChatParticipant` | `drugStore` FK(nullable, `@OnDelete CASCADE`) 추가, `participantType` NOT NULL 추가, 편의 생성자 `(User,room)` / `(DrugStore,room)` 분리, `isUser()`/`isDrugStore()`/`displayName()` 추가 |
| `chat/entity/ChattingRoom` | `roomType` NOT NULL 추가 |
| `chat/entity/ChatMessage` | `sender`(String email) 제거 → `senderParticipantId`(Long) + `senderName`(String 스냅샷) |
| `chat/dto/ChatMessageDto` | `sender` → `senderParticipantId`. `from()`이 인자 1개(엔티티)로, 이름은 `senderName` 스냅샷 사용. `senderNickname` 필드명은 프론트 wire 호환 위해 유지 |
| `chat/dto/response/ChatRoomResponse` | `myParticipantId` 추가 (방 생성/입장 시 클라에 전달) |
| `chat/dto/response/ChatRoomListDto` | `myParticipantId` 추가 — 목록에서 방을 바로 열 때(재입장·사장 상담방) 전송에 쓸 내 참여자 id. `DrugStoreDetail`/친구 진입은 `ChatRoomResponse`로 받지만 목록 진입은 이게 유일한 경로 |
| `chat/repository/ChatParticipantRepository` | `findWithRefsByIdAndRoomId`(발신자 검증+이름 스냅샷, user/drugStore fetch), `findByChattingRoomIdAndDrugStore_Hpid`(사장이 상담방 열 때 — 참여자가 약국), `findByUserId`, `countByChattingRoomId` 추가 |
| `chat/repository/ChatRepository` | `deleteAllByRoomId`(`@Modifying` 벌크) 추가 |
| `chat/repository/ChattingRoomRepository` | `findExistingRoom` 제거 → `findDirectRoom` / `findConsultRoom` 분리. `findAllWithParticipantsByUserId` → `findAllAsUserByUserId`(fetch에 `p.drugStore` 추가, `u.myDrugStore` fetch는 불필요해져 제거). `findAllConsultByDrugStoreHpid` 신규. `findRoomIdByUserId` 유지 |
| `chat/service/ChatService` | `getOrCreateRoom`(상담방 = `[User, DrugStore]`, 자기약국 거부를 hpid 비교로), `getOrCreateFriendRoom`(+친구 검증), `getConsultRooms` 신규, `saveMessage`(참여자 검증 + 이름 스냅샷), `getChatMessages`(스냅샷 그대로 사용 + `resolveMyParticipant`로 사장도 상담방 조회 가능), `resolveRoomName`(참여자 타입 분기, myUserId/myHpid로 '나' 판별), `resolveMyParticipant`/`myParticipantOf`/`toRoomList` 헬퍼, `handleUserWithdrawal` 신규 |
| — 사장의 상담방 메시지 조회 | `getChatMessages`가 `findByChattingRoomIdAndUserId`만 쓰면 사장은 상담방의 USER 참여자가 아니라 "참여 안 함"으로 막힌다. `resolveMyParticipant`가 USER→(실패 시)DRUGSTORE 순으로 찾아 이 구멍을 막음 |
| `chat/controller/ChatRoomController` | `GET /consult` 엔드포인트 추가 |
| `chat/controller/ChatController` | 변경 없음 (STOMP 흐름 동일, DTO 필드만 교체됨) |
| `user/service/UserService` | `ChatService` 주입, `deleteUser`가 `handleUserWithdrawal` 호출 |

- `SecurityConfig` 변경 없음 — `/api/chat/room/consult`는 `anyRequest().authenticated()`에
  자동 포함, `/ws-stomp/**`는 이미 `permitAll`.
- `ShareYaksokService` 변경 없음 (결정 3).

## 9.9.2 실제 반영된 변경 (프론트엔드)

| 파일 | 변경 |
| --- | --- |
| `model/ChatMessage` | `sender`(email String) 제거 → `senderParticipantId`(Long). 생성자 인자 교체. `getSenderNickname()`은 비면 빈 문자열(서버가 스냅샷으로 항상 채움) |
| `model/response/ChatRoomResponse` | `myParticipantId` getter 추가 |
| `model/response/ChatRoomListDto` | `myParticipantId` getter 추가 |
| `model/request/ChatRoomRequest` | 레거시 `userId` 필드 제거 (백엔드가 토큰에서 신원 확인 — 8장 원칙). 생성자 `(hpid)` |
| `model/request/FriendChatRoomRequest` | 레거시 `userId` 필드 제거. 생성자 `(friendId)` |
| `network/ChatApi` | `getConsultRooms()` (`GET /api/chat/room/consult`) 추가 |
| `adapter/ChattingRoomAdapter` | 생성자 `String myEmail` → `Long myParticipantId`. 내/상대 판별 `getSender().equals(myEmail)` → `Objects.equals(getSenderParticipantId(), myParticipantId)` |
| `ui/chattingroom/ChattingRoom` | `myEmail`(Sprefs) → `myParticipantId`(Intent extra `"myParticipantId"`). 없으면 잘못된 접근 처리. STOMP 전송 페이로드에 participantId 사용. `chatReady` 가드(STOMP 연결/구독 전 전송 차단, 실패 시 Toast+입력 복구) — 첫 진입 전송 실패 증상이 있었으나 근본 원인 미규명, 이 가드는 완화용 |
| `ui/chattingroom/ChatRoomList` | `role == "DRUGSTORE"`면 [친구 \| 상담] 탭 노출, **친구 기본**. 선택에 따라 `getMyChatRooms()` / `getConsultRooms()` 조회. `openChatRoom`이 `myParticipantId`·`roomName`도 Intent에 실음 |
| `DrugStoreDetail` | `new ChatRoomRequest(userId, hpid)` → `(hpid)`. 응답의 `myParticipantId`를 ChattingRoom Intent에 전달 |
| `ui/yaksok/YaksokDetail` | `new FriendChatRoomRequest(myUserId, friendId)` → `(friendId)`. 응답의 `myParticipantId`를 Intent에 전달 |
| `res/layout/activity_chat_room_list.xml` | `rg_chat_type` `RadioGroup` + `RadioButton` 2개(기본 `gone`, `button="@null"`, 배경은 `state_checked` 셀렉터). RecyclerView top 제약을 그 아래로. Material `TabLayout`은 앱 디자인과 안 맞아 불채택 (→ 디자인이 세그먼트로 바뀐 뒤 22장에서 `TabLayout`으로 교체) |
| `res/drawable/selector_chat_tab_friend.xml`, `selector_chat_tab_consult.xml` | 신규. `state_checked`면 `bg_touch_my_yaksok_list` / `bg_touch_shared_yaksok_list`(위·바깥 테두리), 아니면 `bg_black_border`(밑줄). YaksokList의 내 약속/공유 약속 폴더 탭과 동일 디자인을 RadioGroup에 입힘 — `RadioGroup`이 상호배타 선택을 처리하니 자바 배경 스왑 코드가 없다 |

**참여자 payload 일관성** (9.10 함정) — 과거 메시지 로드(REST)와 실시간 수신(STOMP)이 둘 다
`senderParticipantId`를 담는다. 서버가 저장 후 브로드캐스트하는 DTO(`saveMessage` →
`ChatMessageDto.from`)와 조회 DTO가 같은 필드라 어댑터 판별이 일관된다.

## 9.9.3 검증 및 후속

- `ddl-auto: create`로 스키마 재생성(기존 채팅 데이터 버림) 후 로컬 실행 검증 통과.
- `tech-debt.md` 갱신: "채팅 sender가 email", "탈퇴 시 채팅방 미정리",
  "`getOrCreateFriendRoom` 친구 미검증" → 해소. "STOMP 메시지에 참여자 검증 없음" → 부분 완화
  (`saveMessage`가 방 참여 여부는 확인, 요청자 신원 확인은 여전히 없음 — `senderParticipantId`를
  클라가 고름. `StompChannelInterceptor` 도입 시 완성).
- **남은 관찰 항목** — 첫 채팅방 진입 후 전송이 한 번 실패하는 증상이 개발 중 있었다. `chatReady`
  가드(연결/구독 전 전송 차단 + 실패 Toast)를 넣었으나 근본 원인은 미규명. 재현되면 Logcat
  (`ChattingRoom` 태그) + 백엔드 `saveMessage` 로그로 확인. 로컬 에코(내 메시지를 서버
  브로드캐스트 기다리지 않고 즉시 표시)로 바꾸는 게 정석 수정.

## 9.10 예상 함정

- **`ddl-auto: update`는 FK·컬럼을 못 바꾼다(4.2에서 이미 밟음).** `ChatParticipant`에
  `drug_store_id` 컬럼과 FK를 추가하는 것, `participant_type`을 NOT NULL로 만드는 것 모두
  기존 테이블에는 `create` 재생성이 있어야 반영된다. 어차피 기존 데이터를 버리기로 했으니
  4.3 절차를 그대로 탄다.
- **`@ManyToOne` 두 개(`user`, `drugStore`)가 다 nullable**이면 JPA는 불변식을 강제하지
  않는다. 편의 생성자 분리 + 서비스 레이어 검증으로 막아야 한다. DB `CHECK` 제약은 이
  프로젝트가 안 쓰는 방식.
- **`resolveRoomName`이 참여자에서 `user`를 꺼내 `getMyDrugStore()`를 부르던 코드가 곳곳에
  전제**돼 있다. 참여자가 `user == null`(약국 참여자)일 수 있게 되면 NPE. `Objects::nonNull`
  필터가 이미 일부 있지만(현재 `resolveRoomName`) 전수 확인 필요.
- **프론트 `ChattingRoomAdapter`는 `myEmail` 문자열 비교로 내/상대를 가른다.** 9.5 방식 2/중간안이면
  이 기준이 바뀌는데, 과거 메시지 로드(`getPreviousMessages`)와 실시간 수신(STOMP) **양쪽 payload가
  같은 필드를 담아야** 어댑터가 일관되게 판별한다.
- **`ChatRoomListDto`의 `roomId` 타입 불일치(5장 기술 부채)** — 백엔드 `String`, 프론트 `Long`.
  이 작업에서 목록·DTO를 건드리므로 같이 정리할 기회지만, 범위를 키우지 않으려면 그대로 두고
  5장에 남긴다.
- **사장이 자기 약국 상담을 시작하려는 경우** 지금은 `getOrCreateRoom`이 `clientUser.getId()
  .equals(pharmacyUser.getId())`로 막는다. B안에서는 약국 참여자가 유저가 아니므로 이 비교가
  성립하지 않는다. "고객 userId의 `myDrugStore.hpid == 요청 hpid`이면 거부"로 조건을 바꿔야 한다.
