# 약속 목록 완료율 표시 + 공유 약속 "전체" 필터

## 무엇을 구현했나

1. `YaksokListFragment`(내 약속 / 공유받은 약속 목록)의 아이템에 전체 복약 알림 중 완료한
   비율을 %로 표시. 디자인 시스템 마이그레이션(18장) 때 빠졌던 완료율 정보를 홈 프래그먼트처럼
   프로그레스 바 없이 텍스트(%)로만 복원.
2. 공유받은 약속 탭의 공유자 목록 맨 앞에 "전체" 항목을 추가. 선택하면 특정 공유자로 필터링하지
   않고 나에게 공유된 모든 약속을 보여준다. 탭 진입 시 기존에는 (의도한 동작이 아니었지만) 첫
   번째 실제 공유자가 자동 선택됐는데, 이제는 "전체"가 기본 선택된다.

## 어떻게 구현했나

- **완료율**: `Yaksok`에 이미 있던 `totalNotifications`/`currentClearNotifications`(백엔드
  `YaksokService.updateYaksokCurrentClearNotifications`가 알림 완료/취소 시마다 갱신하는 값)를
  그대로 사용. `item_yaksok.xml`의 뱃지 줄 오른쪽에 `tv_yaksok_percent` 추가, `YaksokListAdapter`/
  `ShareYaksokListAdapter` 둘 다 `bindPercent()`로 계산해 채움 (두 어댑터가 같은
  `item_yaksok.xml`을 쓰므로 내 약속/공유 약속 탭 둘 다 자동 반영).
- **전체 필터**: 백엔드 `GET /api/shared-yaksok`는 이미 `senderId`가 `null`이면
  `findAllByReceiverId`로 전체를 반환하도록 돼 있어(`ShareYaksokService.getSharedYaksokList`)
  백엔드 변경 없이 프론트에서만 처리 가능했음. `SharedUser`에 클라이언트 전용 생성자를 추가해
  `userId=null, nickName="전체"` 항목을 만들고, `fetchSharedUserList()`에서 서버가 준 공유자
  목록 맨 앞에 붙여 어댑터에 전달. 클릭 시 `user.getUserId()`가 null이면 라벨을 숨기고
  `fetchSharedYaksokBySender(null)`을 호출.
- 탭 재진입 시 공유자 목록을 다시 불러올지 판단하는 기준을 `currentSenderId != null`에서
  `sharedFilterSelected`(boolean)로 교체. 기존에는 `currentSenderId == null`이 "아직 아무것도
  선택 안 함"과 "전체 선택함"을 구분하지 못해서, null을 유효한 필터 값으로 쓸 수 없었음.
- **버그**: "전체" 도입 직후, 공유 약속 삭제(`removeSharedYaksok`) 후 화면이 갱신 안 되는 회귀가
  생겼음. 원인은 삭제 성공 콜백이 여전히 `if (currentSenderId != null) fetchSharedYaksokBySender(...)`
  로 감싸져 있었던 것 — "전체" 선택 상태에서는 `currentSenderId`가 정상적으로 null이라 조건이
  거짓이 되어 재조회 자체가 안 됐음.
- 처음엔 재조회 대신 `ShareYaksokListAdapter`에서 해당 `yaksokId`만 로컬로 지우는 방식으로
  고쳤었으나, 그러면 공유자 칩 목록(`SharedUserAdapter`)은 그대로 남는다는 문제가 있었음 —
  방금 지운 항목이 어떤 공유자의 마지막 공유 약속이었는지는 서버만 알 수 있어서 로컬 리스트
  조작만으로는 "이제 공유 약속이 0개인 공유자" 칩을 없앨 수 없었음. 최종적으로는 삭제 성공 후
  기존 `fetchSharedUserList()`를 재사용해서 공유자 목록을 서버에서 다시 불러오는 방식으로 변경.
- 처음엔 재조회 후 항상 "전체"로 선택을 리셋했으나, 특정 공유자를 보던 중 삭제하면 그 사람에게
  다른 공유 약속이 남아 있어도 선택이 날아가는 게 불편하다는 피드백으로 `selectSharedUser()`를
  추가: 갱신된 목록에서 `currentSenderId`(전체면 null)와 같은 `userId`를 가진 항목을 찾아 그
  위치를 다시 선택하고, 그 대상으로 목록을 재조회한다. 그 공유자가 리스트에서 아예 사라졌으면
  (마지막 공유 약속이 방금 삭제된 경우) 못 찾으므로 자연스럽게 인덱스 0("전체")로 폴백된다.
  탭 최초 진입 시(`selectSharedYaksokTab`에서 `currentSenderId`를 null로 리셋해둔 상태)에도
  같은 함수가 "전체"를 선택하므로 별도 분기 없이 하나의 경로로 처리됨.

## 왜 그렇게 판단했나

- 완료율 데이터소스를 새로 안 만들고 기존 `Yaksok` 필드를 재사용한 이유: 이미 백엔드가 알림
  완료/취소마다 실시간으로 갱신하는 값이라 별도 계산 로직이나 API 호출이 필요 없었음.
- "전체" 필터를 서버 집계 API 신설 없이 처리한 이유: `senderId` 파라미터가 이미
  `@RequestParam(required = false)`로 옵셔널이었고, null일 때 전체 반환하는 서비스 로직도 이미
  있었음 — 프론트에서 `senderId=null`로 호출하는 것만으로 충분했음.

## 검토했지만 택하지 않은 대안

- **각 공유자별로 개별 조회해서 클라이언트에서 병합** — 백엔드가 이미 `senderId=null` 전체 조회를
  지원하므로 N+1 호출을 만들 이유가 없었음.
- **`currentSenderId`에 특수 sentinel 값(예: -1L)을 넣어 "전체"를 표현** — null이 이미 "필터 없음
  = 전체"라는 의미로 API와 자연스럽게 맞아떨어져서, sentinel보다 별도 boolean
  (`sharedFilterSelected`) 하나로 "선택 여부"만 분리하는 쪽이 더 단순했음.
- **삭제 후 로컬 리스트에서만 항목 제거(네트워크 추가 호출 없음)** — 화면에 보이는 약속 목록만
  갱신하면 되는 줄 알았으나, 공유자 칩 목록도 같이 정확해야 한다는 요구사항이 있었고 그 정확성은
  서버 재조회 없이는 알 수 없는 정보라 폐기.
- **삭제 후 항상 "전체"로 리셋** — 구현은 더 단순하지만, 특정 공유자를 보던 중 삭제할 때마다
  필터가 풀리는 게 불편하다는 피드백으로 폐기하고 `selectSharedUser()`로 대체.
