# 문서 인덱스

작업이 여러 세션에 걸쳐 이어지므로, **새 세션은 이 파일부터 읽는다.** 여기 실린 한 줄 요약과
상태만 보고, 지금 작업과 관련된 파일 1~2개만 열어본다. `completed/`는 이미 끝난 작업의 설계
판단 기록이라 **관련 기능을 다시 만질 때만** 열어보면 되고, 매 세션 읽을 필요는 없다.

> **이 문서 세트의 목적**
>
> **이 인덱스 + 관련 파일만 읽으면 중단된 지점부터 바로 이어갈 수 있도록** 프로젝트를
> 정리해둔다. 새 작업을 시작하기 전에 관련 파일을 먼저 읽고, 작업이 끝나면 반영한다.
>
> 작업 방식(언제 질문할지, 코드를 직접 수정할지)은 워크스페이스 루트 `CLAUDE.md`에 있다 —
> 여기서 반복하지 않는다. **이 블록은 문서를 어떻게 쓸지에 대한 정본이다.**
>
> **적을 것** — 문제 정의, 원인 분석, 설계 판단과 그 근거, 검토했지만 택하지 않은 대안,
> 변경된 클래스·메서드, 각 작업의 진행 상태, 알려진 기술 부채.
>
> **적지 않을 것** — PR 번호와 링크, 검증·테스트 수행 절차, 브랜치·커밋 전략,
> 저장소 공개 여부 같은 운영 정보. 금방 낡아서 문서의 밀도만 떨어뜨린다.
>
> 핵심은 **"무엇을 어떻게 구현했고 왜 그렇게 판단했는가"**를 남기는 것이다.
> 특히 **택하지 않은 대안과 그 이유**는 나중에 같은 고민을 반복하지 않게 해주므로 꼭 남긴다.
>
> **새 작업을 끝냈을 때 어디에 적을지:**
> - 완료됐다 → `completed/`에 새 파일(`NN-slug.md`)로. 이 인덱스에 한 줄 추가.
> - 계획 단계이거나 여러 세션에 걸쳐 진행 중이다 → `active/`에 새 파일로. 끝나면 `completed/`로 옮기고 인덱스를 갱신.
> - 앱의 안정적인 기능 정의가 바뀌었다 → `spec.md`.
> - 새로 발견한 이슈나 부채다 → `tech-debt.md`.
> - 앞으로 할 일이 늘었다 → `todo.md`.

## 현재 상태 (현재 상태 파일 — 자주 갱신됨)

- [spec.md](spec.md) — 앱 전체 기능 명세(인증·약속·알림·공유·친구·채팅·약국·기술 스택). 2026-11-09 코드 기준으로 작성, 안정적이라 거의 안 바뀜
- [todo.md](todo.md) — 구현 예정 작업 목록. 끝난 항목은 해당 장으로 옮겨 표시했고, 6.6에 진행 중인 작업을 모았다
- [tech-debt.md](tech-debt.md) — 알려진 이슈 / 기술 부채(해결된 항목은 "해결됨"으로 남김)

## 진행 중 / 계획 (지금 관련 작업을 할 때만 열어볼 것)

- [active/47-busy-buttons.md](active/47-busy-buttons.md) — 서버 통신 중 버튼 중복 클릭 방지(`BusyCallback`, `BaseActivity.busy`). Login만 적용, 나머지 29개 파일 남음

- [active/45-next-tasks-queue.md](active/45-next-tasks-queue.md) — 다음 작업 큐. 45.1 이메일 인증 TTL 1분(시연용, 복구 남음)·45.2 회원가입 이메일 인증(46장)·45.4 친구 요청 푸시+실시간 갱신·45.5 채팅 푸시+실시간 갱신·45.6 카카오 기존 회원 연동은 완료, **45.3 회원정보 변경 알림·45.7 AI 캐시는 미착수**. 항목별 출발점과 착수 전 결정할 것, 구현 결정 기록

- [active/42-chat-unread.md](active/42-chat-unread.md) — 채팅 안 읽은 메시지(목록 배지 + 방 안 구분선). 참여자별 `lastReadMessageId` 커서, `POST /read`가 옮기기 전 값을 돌려줘 진입 경로와 무관하게 구분선 계산. 서버·앱 구현, 실기기 확인 완료. 운영 DB 채우기 남음

- [active/33-kakao-login.md](active/33-kakao-login.md) — 카카오 로그인. OAuth + `app_id` 검증, User에 `social_type`/`social_id` 컬럼 + 로그인 수단 CHECK, 신규 가입은 카카오 토큰 재검증(가입용 토큰 없음)으로 역할·닉네임 선택, 이메일 충돌은 차단 + 연동 안내, 닉네임 유니크 제거. 기존 회원 연동(흐름 C)·연동 해제·`UserResponse.hasPassword`·로그인 화면 원형 심볼 버튼까지 서버·앱 구현, 실기기 확인 완료. 남음: 아이디 찾기·비밀번호 재설정의 카카오 회원 처리, 탈퇴 시 카카오 연결 끊기(보류)

## 완료된 작업 (히스토리 — **관련 기능을 다시 만질 때만 열어볼 것**)

- [completed/47-auth-service-split.md](completed/47-auth-service-split.md) — 가입·로그인·토큰 발급을 `UserService`에서 `auth/service/AuthService`로 이동, 경로도 `/api/users/{signup,login}` → `/api/auth/{signup,login}`(`SecurityConfig`·앱 `UserApi` 함께 수정). 구버전 앱과 비호환 — 서버·앱 동시 배포
- [completed/46-signup-email-verification.md](completed/46-signup-email-verification.md) — 회원가입 이메일 소유 확인. 코드 검증 성공 시 이메일에 묶인 1회용 토큰(`SignUpTokenStore`)을 발급하고 `signUp()`의 `save` 직전에 소비·대조. 앱은 `VerificationCode`를 가입 화면 2개에 재사용(이메일을 고쳐도 인증을 바로 버리지 않고 새 코드 수신 시 폐기, 가입 시 인증 이메일과 대조). 시연 후 TTL 5분 복구 남음

- [completed/44-consult-room-delete.md](completed/44-consult-room-delete.md) — 사장이 상담방을 지우면 "채팅방을 지우지 못했습니다."로 실패하던 문제. 삭제가 USER 참여자만 찾아 DRUGSTORE 참여자(사장)는 400이었음. `findMyParticipant`로 교체하고 반환도 참여자 타입별 목록으로 분기, API·앱 변경 없음. 실기기 확인 완료

- [completed/43-camera-or-album.md](completed/43-camera-or-album.md) — 처방전·약봉투 진입 시 앨범 직행 대신 "카메라 / 앨범" 바텀시트. `ImageSourceBottomSheet`가 갤러리·카메라·권한 런처와 FileProvider 임시 파일을 모두 맡고 Fragment Result로 Uri만 넘김. 실기기 확인 완료

- [completed/41-id-type-unify.md](completed/41-id-type-unify.md) — `ChatMessage.roomId`(String)와 `Notification.id`(int)를 `Long`으로 통일, 서비스의 `valueOf` 변환 제거. 운영 DB 컬럼 타입 변경(DDL) 필요. `hpid`/`loginId`/`socialId`는 외부·사용자 문자열이라 유지, `FriendRequestId`는 죽은 코드

- [completed/40-unify-corner-radius-8.md](completed/40-unify-corner-radius-8.md) — 버튼·카드·입력창 radius(10~16)를 카카오 로그인 버튼 기준 8로 통일(.pen 245노드 + XML). 원형/알약/시트/말풍선은 유지

- [completed/39-share-yaksok-service-tests.md](completed/39-share-yaksok-service-tests.md) — `ShareYaksokService` 단위 테스트(Mockito, 공유 약속 저장/상세/목록/삭제, 공유 관계 검증 분기). 발견된 버그 없음

- [completed/38-yaksok-service-tests.md](completed/38-yaksok-service-tests.md) — `YaksokService` 단위 테스트(Mockito, 저장/수정/알림 처리/삭제). 수정 시 시간이 null이면 일정 변경으로 오판될 수 있는 점을 발견(미수정)

- [completed/37-ai-analysis-in-app.md](completed/37-ai-analysis-in-app.md) — 처방전·약봉투 AI 분석을 서버 경유 없이 앱이 Gemini를 직접 호출(저장할 게 없어 트래픽 절감). 처방전만 ML Kit OCR 병행, 약봉투는 OCR 결과가 나빠 Gemini 단독과 차이가 없어 제외

- [completed/36-failed-pills-inline-notice.md](completed/36-failed-pills-inline-notice.md) — AI 약속 만들기에서 조회 실패한 약 이름을 토스트(금방 사라지고 길면 잘림) 대신 "약 선택" 위 빨간 TextView로 상시 표시. `CreatePaperEnvelope`는 아직 실패 안내 없음

- [completed/35-app-icon.md](completed/35-app-icon.md) — 기본 템플릿(초록 로봇) 앱 아이콘을 디자인의 로고 마크(주황 `p600` + 흰 `pill`)로 교체. 적응형 아이콘 벡터 2개만 수정, monochrome 공용. 알약+체크 배지안은 미채택

- [completed/32-drawer-from-right.md](completed/32-drawer-from-right.md) — 사이드 메뉴 drawer를 오른쪽에서 열리게 변경(햄버거 버튼이 이미 오른쪽 상단). `layout_gravity`와 `GravityCompat` 6곳을 `START→END`, 패널 배경의 둥근 모서리를 왼쪽으로

- [completed/31-friend-tap-profile-dialog.md](completed/31-friend-tap-profile-dialog.md) — 친구 목록 항목 탭이 `WipActivity`(작업 중 화면)로 가던 막다른 길을 기존 프로필 다이얼로그로 교체. 새 화면·API 없이 `UserProfileDto.fromFriend`로 FRIEND 프로필을 만들어 재사용, 친구 목록에서 "채팅하기" 진입 가능. `code=null` 결합과 미사용 `WipActivity` 잔존

- [completed/30-late-response-after-view-destroyed.md](completed/30-late-response-after-view-destroyed.md) — 탭을 옮긴 뒤 늦게 도착한 응답이 `onDestroyView()`로 비워진 `binding`/`requireContext()`를 건드려 죽던 문제. 채팅·친구·홈·약속 목록 Fragment 콜백 첫 줄에 `binding == null` 가드, 설정 화면은 로컬 캐시 저장을 건너뛰면 안 되어 `button.getContext()`로 교체. Activity는 미점검

- [completed/29-light-only-status-bar-icons.md](completed/29-light-only-status-bar-icons.md) — 다크모드 폰에서 상태바 아이콘이 흰색이던 문제. 원인은 `EdgeToEdge.enable`(auto)이 `uiMode`를 보고 `BaseActivity`의 값을 덮어쓴 것. `MODE_NIGHT_NO`로 라이트 고정 + `EdgeToEdge.enable`을 `BaseActivity` 한 곳으로 이동

- [completed/28-text-glitch-fixes.md](completed/28-text-glitch-fixes.md) — 복약 완료 푸시 본문의 짝 없는 `'` 제거, 홈 진행률 문구 "N개 중 M개 완료"의 숫자 순서 수정

- [completed/27-friend-dialog-close-button.md](completed/27-friend-dialog-close-button.md) — 친구 다이얼로그 4종(내 QR·프로필·친구 추가·받은 요청)에 .pen의 "닫기" 버튼 반영, 프로필 X 아이콘 제거, SELF 프로필 높이 유지

- [completed/26-design-dialogs-and-popup-menu.md](completed/26-design-dialogs-and-popup-menu.md) — 시스템 `AlertDialog` 11곳을 공통 `AppDialog`(확인/삭제 액션)로, 약속 상세 `PopupMenu`를 `PopupWindow`로 교체해 .pen 디자인 반영

- [completed/25-share-message-title-only.md](completed/25-share-message-title-only.md) — 약속 공유 메시지 본문을 약속명만 저장(옛 "OO님이 ...\n약속명" 잔재 제거), 채팅 목록 미리보기는 서버가 "약속을 공유했어요"로 대체

- [completed/24-medicine-search-redesign.md](completed/24-medicine-search-redesign.md) — 약 검색 화면·아이템을 .pen 디자인으로 교체, 상단바 겹침(`EdgeToEdge` + inset 미처리) 수정

- [completed/23-pill-card-name-on-top.md](completed/23-pill-card-name-on-top.md) — 약 아이템 카드에서 긴 이름을 이미지 위 한 줄 전체로 이동, 횟수/투약량 가로 2단 (.pen `PillCard` + `item_add_medication_setting.xml`)

- [completed/22-chat-tab-tablayout.md](completed/22-chat-tab-tablayout.md) — 채팅 목록 사장
  계정 [친구 | 상담] 토글을 RadioGroup → `TabLayout`으로 교체(세그먼트 디자인 유지, stretch
  인디케이터). 9장의 "TabLayout 불채택"을 뒤집은 이유, M3 인디케이터 틴트/폭 함정 포함

- [completed/21-yaksok-list-percent-and-shared-all-filter.md](completed/21-yaksok-list-percent-and-shared-all-filter.md) —
  약속 목록 아이템에 완료율 % 표시 복원(홈처럼 텍스트만, 프로그레스 바 없이), 공유받은 약속
  탭 공유자 목록에 "전체" 필터 추가(기본 선택, `senderId=null`로 전체 조회)

- [completed/20-shared-yaksok-detail-fix.md](completed/20-shared-yaksok-detail-fix.md) — 채팅방
  공유 약속 "확인하기"가 "본인의 약속만 조회할 수 있습니다"로 막히던 버그. JWT 도입(8장) 때
  `GET /api/yaksok/{yaksokId}`에 소유자 검증을 추가하면서, 이 엔드포인트의 유일한 실사용처가
  공유 약속 미리보기였다는 걸 고려 못 해 생긴 회귀. 새 엔드포인트를 추가하는 대신 기존 것을
  `ShareYaksokController`/`Service`로 옮기고 `getSharedYaksokDetail`로 개명, 검증을 실제 공유
  관계 확인으로 교체

- [completed/19-notification-adapter-reattach-fix.md](completed/19-notification-adapter-reattach-fix.md) —
  홈 화면 알림이 탭 전환/날짜 변경 후 사라지던 버그. 원인은 캐시 오염이 아니라 `HomeFragment`가
  `onDestroyView()`에서 `adapter`를 null로 안 만들어 재생성된 RecyclerView에 재부착이 안 된 것.
  같이 발견한 SprefsManager 캐시 덮어쓰기 잠재 버그와 `SaveYaksokResponse.notifications` 필드
  제거도 포함

- [completed/18-design-system-migration.md](completed/18-design-system-migration.md) — 프론트엔드
  전체를 `yaksok_design.pen` 새 디자인으로 리스킨 (Flow 01~05 전부 완료: 로그인/가입, 메인 5탭,
  약속 등록/상세, 채팅/약국/내 정보, 친구 추가 QR. 실기기 확인 완료)

- [completed/17-qr-friend-add.md](completed/17-qr-friend-add.md) — QR 코드로 친구 추가
  (백엔드 발급/조회 API, `FriendQrCode`의 `@MapsId` 프레임워크 버그 우회, 친구 목록 QR
  진입점·내 QR 다이얼로그·ZXing 스캔·상대 프로필 다이얼로그(FRIEND는 "채팅하기"))

- [completed/10-find-id-password.md](completed/10-find-id-password.md) — 아이디 찾기 / 비밀번호
  재설정 (`loginId` 도입, 이메일 인증코드, 재설정 토큰 방식으로 새 비번 직접 설정. 운영 DB
  마이그레이션·Gmail 발송까지 EC2에서 직접 검증)
- [completed/02-fcm-multi-device.md](completed/02-fcm-multi-device.md) — FCM 토큰 다기기 지원
  (`user_devices` 1:N, upsert-and-reassign)
- [completed/03-auto-refresh.md](completed/03-auto-refresh.md) — 복약 완료 시 화면 자동 갱신
  (`REFRESH` FCM 타입, `YaksokEventBus`, `PushSender`)
- [completed/04-withdraw-fix.md](completed/04-withdraw-fix.md) — 회원 탈퇴 오류 수정(FK CASCADE)
  & 개발용 목 데이터 시더
- [completed/07-pretendard-font.md](completed/07-pretendard-font.md) — 앱 기본 폰트 Pretendard
  적용 (ViewPump 런타임 주입)
- [completed/08-jwt-auth.md](completed/08-jwt-auth.md) — JWT 인증 도입 (액세스/리프레시 토큰,
  화이트리스트 `.authenticated()`, 프론트 자동 재발급)
- [completed/09-chat-room-split.md](completed/09-chat-room-split.md) — 채팅방 구조 분리 (약국
  상담 ↔ 친구): `ChatParticipant` 다형화(user/drugStore), `senderParticipantId`+스냅샷, 사장
  상담 탭, 탈퇴 시 빈 방 정리
- [completed/11-password-last-char.md](completed/11-password-last-char.md) — 비밀번호 마지막
  글자 2초 노출 (마스크 인스턴스 2개 교체, 포커스 아웃·타이머로 숨김, 삭제 시 노출 안 함)
- [completed/12-frontend-package-reorg.md](completed/12-frontend-package-reorg.md) — 프론트엔드
  루트 패키지에 남아있던 Activity/Fragment를 `ui/<feature>/`로 이동 (friend/medicine/common
  패키지 신설, AddressSearch가 죽은 코드였음을 발견)
- [completed/13-insets-overlap-fix.md](completed/13-insets-overlap-fix.md) — 액티비티 6개
  상태바/내비바 겹침 수정 (A `fitsSystemWindows` / B `InsetsUtil` / C 직접 리스너 세 방식 정리, 14장 Fragment 전환 후 구조 반영, 실기기 확인 완료)
- [completed/14-bottom-nav-fragment-migration.md](completed/14-bottom-nav-fragment-migration.md) —
  하단 네비 5개 탭 Activity → 호스트 1개 + Jetpack Navigation/Fragment 전환 (백스택 누적 버그
  해소, 호스트 밖 화면 복귀용 `EXTRA_TARGET_TAB` 패턴, 실기기 확인 완료)
- [completed/15-notification-permission-and-opt-out.md](completed/15-notification-permission-and-opt-out.md) —
  `POST_NOTIFICATIONS` 런타임 요청(MainActivity 진입 시, 프리프롬프트 없이) + 설정 화면 알림
  on/off 토글(`User.notificationEnabled`, `PushSender` 단일 지점 필터링)
- [completed/16-viewbinding-migration.md](completed/16-viewbinding-migration.md) — Activity/Fragment
  24개 + 어댑터 14개 ViewBinding 전환 (커스텀 뷰 3개는 범위 제외, 진행 중 `MedicationAdapter`의
  존재하지 않는 id 참조 버그 발견·수정)

## 연혁

2026-09-09 이전까지는 레포 루트의 `PROJECT_NOTES.md` 한 파일에 1~10장이 완료 여부와 무관하게
전부 누적되고 있었다. 그 시점에 내용을 전부 이 `docs/` 구조로 옮기고 원본 파일은 삭제했다.
