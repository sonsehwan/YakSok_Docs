# 38. YaksokService 단위 테스트

## 범위

`YaksokServiceTest` (`src/test/.../yaksok/service/`) — Mockito만 사용, Spring 컨텍스트/DB 없음
(`FriendServiceQrProfileTest`와 같은 방식). 리포지토리 4개를 목으로 두고 엔티티는 빌더로 직접 만든다.

검증하는 동작:

- **saveYaksok** — 시간 비어 있으면 끼니별 기본값(08:00/12:00/18:00), 복약일 × 끼니 수만큼 알림 생성 +
  `totalNotifications` 반영, 월말을 넘기는 날짜 증가, 약-약속 연결, 없는 사용자 예외
- **updateYaksok** — 타인/없는 약속 예외, 제목만 변경 시 알림 제목만 갱신·진행도 유지, 일정 변경 시
  알림 재생성·진행도 0·`TAKING` 복귀, 약 목록 통째 교체
- **updateNotificationStatus** — 타인 알림 예외(이벤트 없음), 같은 상태면 무동작, 체크 시 카운트 증가 +
  `NotificationStatusChangedEvent` 발행, 마지막 체크 시 `COMPLETED`, 해제 시 `TAKING` 복귀,
  카운트가 0 미만/total 초과로 안 넘어감
- **getYaksokListById / deleteYaksok** — 없는 사용자 예외, 타인 약속 삭제 거부, 본인 삭제

## 판단

- **`@DataJpaTest` 불채택** — 로직의 대부분이 상태 전이(진행도, 상태)와 소유자 검증이라 DB가 없어도
  전부 검증된다. 메서드가 `save` 반환값을 쓰지 않는 구조라 목으로 충분하다.
- 소유자 검증 실패 테스트는 예외뿐 아니라 "값이 바뀌지 않았다/이벤트가 없다"까지 확인한다 — 검증
  위치가 변경 앞에 있다는 보장이 이 테스트의 핵심.

## 테스트 중 확인한 점 (고치지 않음)

`updateYaksok`의 `isScheduleChanged`는 요청의 `timeMorning/Lunch/Dinner`를 저장된 값과 그대로
비교한다. 그런데 저장 시에는 비어 있으면 기본값("오전 08:00" 등)으로 채워지므로, 시간을 null/빈 값으로
보내는 클라이언트가 수정을 요청하면 "일정 변경"으로 오판해 알림이 재생성되고 진행도가 초기화될 수 있다.
앱이 수정 시 항상 시간을 채워 보내는지는 확인하지 않았다 → `tech-debt.md` 참고.
