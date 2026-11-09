# 47. 서버 통신 중 버튼 중복 클릭 방지

## 문제
서버 호출이 30개 파일 67곳에 `api.xxx().enqueue(new Callback…)`로 흩어져 있고, 응답(타임아웃 15초)이
올 때까지 버튼이 눌린다. 로그인·가입·약속 생성 같은 쓰기 요청이 중복으로 나간다.

## 설계
- `util/BusyCallback<T>`: 넘겨받은 버튼들을 생성 시점에 비활성화하고, `onResponse`/`onFailure`
  어느 쪽이든 되돌린 뒤 원래 콜백에 위임한다. 해제를 빠뜨리는 실수가 구조적으로 없다.
- `BaseActivity.busy(callback, buttons...)`: 위 래퍼를 만드는 헬퍼. 호출부는
  `enqueue(busy(new Callback…{…}, btn))`.
- 서버와 통신하는 버튼만 대상이다. 화면 전체 오버레이는 쓰지 않는다(아이디 찾기·회원가입 이동 같은
  통신 없는 버튼까지 막히기 때문). 카카오 로그인은 기존 `overlay_loading`이 이미 전체를 막는다.

## 택하지 않은 대안
- 전체 터치 차단(dispatchTouchEvent 가로채기): 통신 없는 버튼도 막혀 요구와 어긋남.
- 버튼별 `setEnabled`를 호출부마다 직접: 응답 경로(성공/실패/예외)마다 해제를 빠뜨리기 쉬움.

## 진행 상태
- 적용: `Login.startLogin()`(로그인 버튼 + 카카오 버튼).
- 남음: 나머지 29개 파일의 쓰기·조회 버튼. 어댑터(`NotificationMultiViewAdapter`)와 Fragment는
  `busy`가 `BaseActivity` 메서드라 Fragment에서는 `new BusyCallback<>(…)`를 직접 쓴다.
