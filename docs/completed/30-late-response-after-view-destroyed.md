# 30. 탭을 옮긴 뒤 늦게 도착한 응답이 앱을 죽이던 문제

## 문제

채팅 탭에서 `ChatRoomListFragment$3.onResponse`가 NPE로 앱이 종료됐다.

```
Attempt to read from field '...FragmentChatRoomListBinding.tvEmpty' on a null object reference
  at ChatRoomListFragment$3.onResponse(ChatRoomListFragment.java:178)
```

제보는 "친구 요청을 받으면 앱이 팅긴다"였지만, 친구 요청 수락 경로(`FriendController`/`FriendService`,
`FriendListFragment`)는 채팅 목록을 건드리지 않고 백엔드도 친구 요청에 푸시·채팅방 생성을 하지 않는다.
**실제 트리거(어느 화면에서 어느 탭을 눌렀는지)는 확인하지 못했다.** 스택트레이스가 보여주는 것은
"채팅 목록 응답이 도착했을 때 View가 이미 없었다"는 사실뿐이다.

## 원인

`ChatRoomListFragment.onResume()`은 탭에 들어올 때마다 `loadChatRooms()`로 요청을 보낸다. 하단 탭은
`NavigationUI`로 전환되므로 다른 탭을 누르면 채팅 Fragment의 View가 파괴되고 `onDestroyView()`가
`binding = null`로 만든다. Retrofit 콜백은 Fragment 생명주기와 무관하게 응답이 오면 실행되므로, 응답이
탭 전환보다 늦으면 `binding.tvEmpty`에서 `binding`이 null이라 터진다. 서버가 느릴수록 재현이 쉽다.

같은 구조가 `binding`/`requireContext()`를 비동기 콜백에서 쓰는 다른 Fragment에도 있었다.

| Fragment | 증상 |
|---|---|
| `ChatRoomListFragment` | NPE (`binding.tvEmpty`), `requireContext()` 토스트 |
| `FriendListFragment` | NPE (`bindFriendList`, `updateRequestCount`의 `binding`), 토스트 |
| `HomeFragment` | NPE — `onDestroyView()`가 `adapter`와 `binding`을 함께 null로 만들어서 `setupRecyclerView()`가 새 어댑터를 만들며 `binding.rvMedication`을 읽음 |
| `YaksokListFragment` | NPE (`fetchSharedUserList` → `binding.tvSharedOwnerLabel`), 나머지는 `requireContext()` 토스트로 `IllegalStateException` |
| `SettingsFragment` | `onNotificationToggle` 성공 경로의 `requireContext()` → `IllegalStateException` |

## 판단

- **콜백 첫 줄에서 `if (binding == null) return;`.** `onDestroyView()`와 Retrofit 콜백은 둘 다 메인
  스레드라 검사와 사용 사이에 `binding`이 바뀔 틈이 없다. 가드는 반드시 첫 줄이어야 한다 — 그 아래의
  `else` 분기·`onFailure`의 `requireContext()` 토스트도 분리된 뒤에는 예외를 던지기 때문이다.
  버려진 응답의 데이터는 다음 `onResume()`의 재조회로 채워지므로 잃는 것이 없다.
- **`SettingsFragment`는 가드를 쓰지 않고 `requireContext()`를 `button.getContext()`로 교체.**
  이 콜백은 서버에 바뀐 알림 설정을 `SprefsManager.setUserInfo`로 로컬에 저장하는 일을 하는데, 화면이
  없어도 반드시 실행돼야 한다. 가드로 건너뛰면 서버와 로컬 캐시가 어긋나 다음에 설정 화면을 열 때 스위치가
  서버 상태와 달라진다. `button`은 콜백이 이미 캡처하고 있고 분리된 뒤에도 `getContext()`가 null이 아니다.
- `HomeFragment`에서 가드가 건너뛰는 `SprefsManager.setNotifications`는 통신 실패 시 대체용 캐시일 뿐이라
  다음 `onResume()`의 재조회로 다시 채워진다. 건너뛰어도 되는 쪽이다.

### 택하지 않은 대안

- **`onDestroyView()`에서 진행 중인 `Call.cancel()`.** 불필요한 응답 처리가 아예 없어져 더 깔끔하지만
  `Call`을 필드로 들고 있어야 하고 취소 시 `onFailure`가 호출되는 것도 처리해야 한다. 한 Fragment에 요청이
  여러 개(친구 6개, 약속 목록 4개)라 필드가 늘어난다.
- **`SettingsFragment`에 `binding == null` 가드.** 위에서 적은 대로 로컬 캐시가 어긋난다.
- **`LiveData`/`viewLifecycleOwner` 기반으로 재구성.** 생명주기를 자동으로 따라가 가장 안전하지만 콜백
  기반 코드를 한 번에 뜯어야 하는 규모라 이번 범위 밖으로 뒀다.

## 변경 (프론트)

- `ChatRoomListFragment`: `loadChatRooms`, `deleteChatRoom`의 `onResponse`/`onFailure` 4곳에 가드.
- `FriendListFragment`: `fetchFriendList`, `deleteFriend`, `updateRequestCount`의 `onResponse`/`onFailure` 6곳에 가드.
- `HomeFragment`: `loadNotificationList`의 `onResponse`/`onFailure` 2곳에 가드.
- `YaksokListFragment`: `fetchYaksokList`(2), `fetchSharedUserList`(`onResponse`만, `onFailure`는 Log뿐),
  `fetchSharedYaksokBySender`(2), `removeSharedYaksok`(2) 7곳에 가드.
- `SettingsFragment`: `onNotificationToggle` 성공 경로의 `requireContext()` → `button.getContext()`.

`BottomSheetAddYaksok`/`SignUpTypeBottomSheet`은 네트워크 호출이 없어 대상이 아니었다.

## 남은 것

- **Activity는 점검하지 않았다.** Activity는 `binding`을 null로 만들지 않아 NPE는 드물지만, `finish()`된
  Activity에서 `Dialog`/`AppDialog`를 띄우면 `BadTokenException`, 사라진 화면의 콜백이 `startActivity`/
  `finish`를 부르면 엉뚱한 화면 전환이 난다. `tech-debt.md`에 항목으로 남겼다.
- **친구 요청 제보의 실제 재현 경로는 미확인.** 위 Fragment들은 "탭을 옮겼을 때 늦은 응답"이라는 같은
  원인을 막았을 뿐, 제보 당시의 정확한 조작을 재현해 확인한 것은 아니다.

## 상태

빌드와 실기기 확인 완료.
