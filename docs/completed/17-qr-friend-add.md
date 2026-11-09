# 17. QR 코드로 친구 추가

**상태:** 완료. 백엔드 QR 코드 발급/조회 API, 친구 목록 QR 진입점(팝업 메뉴), 내 QR 이미지
다이얼로그, ZXing 기반 QR 스캔, 스캔한 상대 프로필을 친구 목록 위 다이얼로그로 표시(FRIEND 상태는
"채팅하기")까지 전부 반영됨. 화면 설계는 `design/yaksok_design.pen`의 Flow 05.
코드 변경 목록은 워크스페이스 루트 `QR_FRIEND_ADD_CHANGELOG.md` 참고.

## 무엇을 구현했나

QR 코드를 스캔해 얻은 값으로 상대 프로필(닉네임/이메일)과 나와의 관계 상태를 보여주고,
상태에 맞는 버튼 하나(요청 보내기 / 수락하기 / 채팅하기, 요청 보냄은 비활성)로 바로 다음
행동을 할 수 있게 하는 프로필과 그 뒤를 받치는 백엔드 API.

- 백엔드
  - [`FriendQrCode`](https://github.com/sonsehwan/YakSok_Backend/blob/main/src/main/java/com/sehwan/YakSok/user/entity/FriendQrCode.java) —
    유저당 한 행, `User`와 1:1로 PK(`user_id`) 공유(방식은 아래 "프레임워크 버그로 @MapsId
    제거" 참고). `code`(UUID)와 `expiresAt`을 갖고, `reissue()`로 매번 새 코드+만료시각(5분)을 부여.
  - `FriendService.issueQrCode(userId)` — "내 QR 보여주기" 화면을 열 때마다 호출해 재발급.
  - `FriendService.getUserProfileByQrCode(code, loginUserId)` — 코드 유효성/만료 확인 후,
    대상 유저와 나의 관계를 SELF/FRIEND/SENT/RECEIVED/NONE 중 하나로 판단
    (`FriendRelation`·`FriendRequest` 조회는 기존 로직 그대로 재사용).
  - [`FriendController`](https://github.com/sonsehwan/YakSok_Backend/blob/main/src/main/java/com/sehwan/YakSok/user/controller/FriendController.java)에
    `POST /api/friend/qr-code`(발급), `GET /api/friend/profile/{code}`(조회) 추가.
  - [`UserProfileDto`](https://github.com/sonsehwan/YakSok_Backend/blob/main/src/main/java/com/sehwan/YakSok/user/dto/response/UserProfileDto.java) —
    `userId`, `nickname`, `email`, `relation` enum(SELF/FRIEND/SENT/RECEIVED/NONE) +
    `requestId`(RECEIVED일 때만, 수락에 필요).
- 프론트엔드 (현재 코드 — 상세는 `QR_FRIEND_ADD_CHANGELOG.md`)
  - `FriendApi`에 `issueQrCode()`, `getUserProfile(code)` 추가. `UserProfileDto`/`FriendQrCodeDto`
    프론트 미러링.
  - [`FriendListFragment`](https://github.com/sonsehwan/YakSok_Frontend/blob/main/app/src/main/java/com/example/medication/ui/friend/FriendListFragment.java) —
    헤더에 `iv_qr`(`ic_lc_qr_code`)/`iv_add_friend`(`ic_lc_user_plus`) 아이콘 두 개. `iv_qr` 클릭 시
    `PopupMenu`로 "내 프로필"/"코드 스캔":
    - **내 프로필** — `issueQrCode()` 호출 후 `dialog_my_qr.xml`에 Glide로 QR 이미지(`qrCodeAPI_URL`
      + 코드)를 그려 다이얼로그로 표시.
    - **코드 스캔** — `CAMERA` 런타임 권한 확인(`DrugStoreList`의 위치 권한 요청과 같은
      `ActivityResultContracts.RequestPermission` 패턴) 후 ZXing(`zxing-android-embedded`)의
      `ScanContract`(+`PortraitCaptureActivity`)로 스캔 화면을 열고, 인식된 문자열로 `showProfileDialog(code)` 호출.
    - `showProfileDialog`가 `getUserProfile(code)`로 상대 프로필+관계를 받아 `dialog_profile.xml`
      다이얼로그(닫기 버튼 `btn_close`)를 채운다. relation별 액션 버튼: SELF 숨김, FRIEND는
      "채팅하기"(`enterFriendChatRoom` → `ChattingRoom`, `YaksokDetail.openChatRoomAndShare`와 같은
      패턴), SENT는 비활성 "요청 보냄", RECEIVED는 "친구 요청 수락하기", NONE은 "친구 요청 보내기".
      액션 성공 시 같은 다이얼로그를 새로 조회해 버튼만 갱신.
  - 프로필은 별도 Activity(`Profile`, 구 `FriendProfile`)였다가 위 다이얼로그로 흡수되며
    `Profile.java`/`activity_profile.xml`은 삭제됨.

## 어떻게 구현했나 / 왜 그렇게 판단했나

**QR에 무엇을 담을지 — TTL 있는 별도 코드 테이블.** 처음엔 QR에 `userId`를 그대로 담는 안을
검토했는데, 로그인한 아무나 id를 1씩 늘려가며 전체 유저의 닉네임·이메일을 순회 조회할 수 있어
폐기. `User`에 고정 `friendCode`(UUID) 컬럼을 추가하는 안도 검토했지만, 코드가 안 바뀌므로
QR 스크린샷 하나가 유출되면 그 사람은 영구적으로 그 코드로 요청을 보낼 수 있고, 이미 운영 DB에
있는 기존 유저들에게 컬럼을 백필해야 하는 부담도 있어 폐기. 최종적으로 `FriendQrCode`를
`User`와 분리된 테이블로 두고 코드에 **유효시간**(현재 5분)을 줘서, "내 QR 보여주기" 화면을 열
때마다 새로 발급하는 방식으로 정했다 — 유출돼도 노출 창이 좁고, `User` 엔티티/기존 유저
데이터는 전혀 건드릴 필요가 없다.

**`FriendQrCode`를 `User`와 별도 테이블로 둔 이유.** 이 값은 화면을 열 때마다(자주) 바뀌는
휘발성 데이터라, `User` 행 자체를 매번 갱신하는 것보다 분리해두는 게 책임이 깔끔하다. 1:1이라
`@MapsId`로 PK를 공유시켜 별도 FK 컬럼 없이 `user_id` 하나로 PK와 FK를 겸하게 했다
(`FriendRelation`이 복합키에 `@MapsId`를 쓰는 것과 같은 계열의 패턴, 여기는 연관관계가
하나뿐이라 `@EmbeddedId` 없이 `@Id` 를 연관관계 필드에 바로 붙임).

**프로필은 별도 화면이 아니라 친구 목록 위 다이얼로그 (디자인 확정, 코드 전환 전).** 소유자 결정.
흐름은 "코드 스캔 → 인식 성공 시 스캔 화면을 닫고 친구 목록으로 복귀 → 그 위에 프로필 다이얼로그".
프로필에 담기는 정보가 아바타·닉네임·이메일·버튼 하나뿐이라 전체 화면 전환까지 필요 없고,
친구를 추가한 뒤 사용자가 있어야 할 곳이 친구 목록이라 그 맥락을 유지하는 편이 자연스럽다.
다이얼로그에는 기존 `iv_back` 대신 `btn_close`(닫기)를 둔다. 내 QR 코드 다이얼로그와 같은 모양으로 맞췄다.

**FRIEND 상태는 비활성 안내문 대신 "채팅하기".** 소유자 결정. 이미 친구인 상대를 스캔했을 때 막힌
버튼을 보여주는 대신 바로 할 수 있는 다음 행동을 준다. 백엔드/응답 변경 없이 기존 흐름으로 연결된다 —
`UserProfileDto.userId`를 `FriendChatRoomRequest(friendId)`에 넣어 `ChatApi.enterFriendChatRoom`을
호출하고, 받은 `ChatRoomResponse`의 `roomId`·`myParticipantId`와 닉네임(`roomName`)으로
`ChattingRoom`을 연다(`YaksokDetail.openChatRoomAndShare`와 같은 방식). 백엔드에서 `friendId`는
`FriendRelationId`·`createFriendRequest` 등 전반에서 상대 **유저 id**로 쓰이므로 `userId`를 그대로 넘기면 된다.

## 프레임워크 버그로 PK 공유 방식을 포기함

`FriendQrCode`가 `User`와 PK(`user_id`)를 공유하는 설계를 두 가지 방식으로 시도했는데,
**둘 다 이 프로젝트의 Spring Boot(`4.0.3`, Hibernate 7 계열)에서 서로 다른 방식으로 죽었다.**
그래서 최종적으로는 PK 공유 자체를 포기했다.

**1차 시도 — `@Id @OneToOne @MapsId private User user;` (파생 식별자).**
서버 기동 중 `FriendQrCode` 리포지토리 빈을 초기화하는 단계에서 예외가 나며 죽었다(소유자가
실서버에서 직접 진단). Spring Data JPA가 엔티티 메타모델을 읽을 때 `hasIdClass()`가 잘못
`true`로 판정되어, 실제로는 없는 `@IdClass`를 찾다가 `IllegalArgumentException`을 던진다.

**2차 시도 — 명시적 스칼라 PK + 읽기전용 `@JoinColumn`(같은 물리 컬럼 공유).**
1차 문제를 피하려고 아래처럼 바꿨었다:
```java
@Id
@Column(name = "user_id")
private Long userId;

@OneToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "user_id", insertable = false, updatable = false, ...)
private User user;
```
그런데 실제로 QR 코드를 발급(`issueQrCode` → `save()`)하는 순간 `org.hibernate.AssertionFailure:
null identifier`로 또 죽었다 — `create(User user)`에서 `userId`를 직접 채워 넣었는데도(전엔
`@MapsId`가 자동으로 복사해줬던 것), insert 시점에 Hibernate가 식별자를 null로 판단했다.
정확한 내부 메커니즘까지는 이 환경에서 재현이 안 돼 단정 못 하지만, "PK를 다른 엔티티와
공유"하는 시도 자체가 이 Hibernate 버전에서 서로 다른 경로로 두 번 다 깨졌다는 게 강한 신호였다.

**최종 조치 — PK 공유를 그만두고 독립된 자동증가 PK로 전환.**
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

@OneToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "user_id", nullable = false, unique = true,
        foreignKey = @ForeignKey(name = "fk_friend_qr_code_user"))
private User user;
```
"유저당 한 행"이라는 제약은 더 이상 PK 공유가 아니라 `user_id` 컬럼의 `unique` 제약으로
보장한다. `FriendQrCodeRepository`도 `findById(userId)` 대신 `findByUser_Id(userId)`로
바뀌었다(PK가 이제 `id`라 `userId`로 `findById`를 쓸 수 없음).

**검토했지만 택하지 않은 대안**
- Spring Boot 버전을 내린다 — 이 버그가 몇 버전부터인지 확인이 더 걸리고, 이미 4.0.3 전용
  기능을 쓰고 있으면 되돌리기 부담도 있어 보류.
- `@IdClass`/`@EmbeddedId`를 명시적으로 붙인다 — 진짜 복합키(연관관계 2개 이상)일 때 쓰는
  도구라 PK가 하나뿐인 이 케이스엔 과함(YAGNI).
- 2차 시도(수동 스칼라 PK + 읽기전용 조인)를 더 디버깅해서 살린다 — PK 공유 패턴 자체가 이
  버전에서 두 번째로 깨진 것을 보고, 더 파고들기보다 그 패턴 자체를 포기하는 쪽을 택함.

**원복 조건.** Spring Boot(또는 Hibernate)가 PK 공유(파생 식별자) 관련 버그를 고친 버전으로
올라가면, 다시 `@Id @OneToOne @MapsId` 한 필드로 합치는 걸 고려할 수 있다. 다만 지금 방식(독립
PK + unique FK)도 그 자체로 흠이 없는 평범한 매핑이라, 버전이 올라간다고 꼭 되돌려야 하는 건
아니다 — 그때 가서 굳이 바꿀 이유가 있는지부터 다시 판단.

**로컬 DB 마이그레이션 필요.** 이미 `friend_qr_code` 테이블이 만들어진 적이 있다면(1차/2차
시도 중 하나로 PK가 `user_id`로 잡혀 있는 상태), `ddl-auto: update`는 기존 PK 정의를 자동으로
바꿔주지 않는다. `DROP TABLE friend_qr_code;`로 지우고 서버를 다시 띄워 새 스키마(자동증가
`id` PK + unique `user_id`)로 재생성해야 한다.

## 검토했지만 택하지 않은 대안

- **QR에 `userId`를 그대로 담기** — 구현이 가장 단순하지만 id 순회로 전체 유저 정보가
  노출돼서 폐기.
- **`User`에 고정 `friendCode`(UUID) 컬럼** — 무한 재사용 가능(스크린샷 유출 시 영구 유효)이라
  이번에 지적받아 폐기. 기존 운영 유저 백필도 별도로 필요했음.
- **스캔에 성공하는 순간 코드 자동 재발급(사실상 1회용)** — 유출 문제는 가장 확실히 막히지만,
  같은 QR을 여러 사람에게 보여주고 각자 찍게 하는 흐름(첫 번째 이후엔 전부 만료된 코드를 보게
  됨)을 막아버려서 택하지 않음. TTL 방식은 이 사용성을 유지하면서도 노출 창을 좁힌다.
- **프로필을 별도 Activity로 유지** — 정보량에 비해 화면 전환이 크고, 추가 후 친구 목록으로
  한 번 더 돌아와야 해서 다이얼로그로 바꿈(`Profile.java`/`activity_profile.xml` 삭제,
  `FriendListFragment`의 다이얼로그로 흡수).
- **프로필 다이얼로그를 카메라(스캔) 화면 위에 띄우기** — 스캔 직후라 전환은 가장 적지만, 친구
  추가·채팅하기 뒤 결국 친구 목록이나 채팅으로 가야 해서 친구 목록 위로 정함.
- **FRIEND 상태를 비활성 "이미 친구입니다"로 유지** — 정보만 주고 할 수 있는 게 없어서
  "채팅하기"(`enterFriendChatRoom`)로 바꿈.
- **QR 스캔 라이브러리로 ML Kit Barcode Scanning** — 정확도는 더 강건하지만 CameraX(`camera-core`/
  `camera-camera2`/`camera-lifecycle`/`camera-view`)까지 얹어 카메라 화면을 직접 만들어야 해서
  의존성·코드량이 훨씬 커짐. 이 기능은 앱이 자체 생성한 QR을 화면에 띄워 가까운 거리에서 찍는
  깨끗한 조건이라 강건함의 이점이 크지 않아 택하지 않음. → **ZXing(`zxing-android-embedded`)** 채택 —
  `ScanContract`가 카메라 화면·오버레이까지 통째로 제공해 의존성 1개·코드 몇 줄로 끝남.
  (`IntentIntegrator`는 4.3.0에서 `@Deprecated`라 쓰지 않는다.)

## 남은 작업

(현재 없음 — 이 장은 완료됨)

**구현 시 설계 문서와 달라진 점:** "예: DialogFragment"로 남겨뒀던 부분은 별도 클래스 없이
`FriendListFragment`의 private 메서드(`showProfileDialog` 등)로 흡수했다 — 이 파일이 이미
`showAddFriendDialog`/`showMyQrDialog` 등 똑같은 방식(일반 `Dialog` + ViewBinding)으로 다이얼로그
여러 개를 만들고 있어서, 그 관례를 그대로 따르는 게 새 클래스를 만드는 것보다 일관적이고 코드도
적었다. 스캔 화면이 인식한 코드는 결과 반환 없이, ZXing 결과 콜백에서 바로
`showProfileDialog(code)`를 호출하는 방식으로 처리했다(스캔도 다이얼로그도 전부 같은
`FriendListFragment` 안에서 일어나므로 화면 간 결과 전달이 애초에 필요 없었음).
