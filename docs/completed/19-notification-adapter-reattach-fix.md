# 19. 홈 화면 알림 목록이 화면 전환/날짜 변경 후 사라지는 버그

## 증상

약속을 만든 직후 홈 프래그먼트에 알림이 정상적으로 보이는데, 다른 하단 탭으로 갔다가 홈으로
돌아오거나 날짜를 바꾸면 알림 아이템이 전부 안 보인다.

## 처음 세운 가설(기각)

"전체 알림 목록 조회(`GET /notifications`)가 실패해서 `SprefsManager`의 오염된 로컬 캐시로
폴백되는 것 아니냐"는 가설을 먼저 세웠다 — 실제로 4개 등록/수정 화면(아래 참고)이 캐시를
잘못 덮어쓰고 있었기 때문이다. 하지만 소유자가 "액티비티 시절에도 같은 방식으로 동작했는데
그때는 문제없었다", "날짜별 조회 화면에서도 잘 보였다"며 이 가설에 반복적으로 반박했고,
결국 로그를 찍어 실기기(`R5CT33B9N6D`, 에뮬레이터는 `adb`가 계속 offline이라 사용 불가)로
재현하라는 지시를 받았다.

로그로 확인한 결과 **네트워크 요청은 항상 성공**했다 — 폴백 캐시 가설은 틀렸다.

## 진짜 원인

`HomeFragment`가 `onDestroyView()`에서 `binding = null`은 하면서 **`adapter` 필드는 null로
만들지 않았다.** 이 프래그먼트는 Jetpack Navigation의 `BottomNavigationView` 기본 옵션
(`popUpTo(startDestination){ saveState = true }` + `restoreState = true`)으로 붙어 있어서,
탭을 전환했다 돌아오거나(사실은 날짜 변경도 내부적으로 같은 재조회 경로를 타면서 뷰가 다시
그려짐) 뷰 계층이 파괴/재생성된다 — 단, `HomeFragment` **자바 인스턴스 자체**는 재사용될 수
있다.

`setupRecyclerView()`는 `adapter`가 이미 있으면(`!= null`) 새로 만들지 않고
`adapter.updateData(...)`만 호출하는 분기를 탔다. 뷰가 재생성되면 `RecyclerView`도 새
인스턴스가 되는데, 거기에는 `setAdapter()`가 다시 호출된 적이 없으니 새 `RecyclerView`는
빈 어댑터(`null`) 상태로 남는다. 반면 재사용된 `adapter` 객체 자체는 데이터를 올바르게 들고
있었다 — 그래서 "네트워크는 성공했는데 화면엔 안 보인다"는 증상이 나온 것.

로그로 확인: 재현 시점에 `rvMedication.getAdapter() == null`이면서 재사용된 `adapter`는
`updateData()`로 받은 데이터(`matched=3`)를 정상적으로 갖고 있었다.

## 수정

`HomeFragment.onDestroyView()`에 한 줄 추가:

```java
@Override
public void onDestroyView() {
    super.onDestroyView();
    adapter = null;
    binding = null;
}
```

이러면 뷰가 재생성될 때마다 `setupRecyclerView()`가 항상 새 어댑터를 만들어 새
`RecyclerView`에 `setAdapter()`로 다시 붙인다. 실기기에서 탭 전환 복귀 케이스와 날짜 변경
케이스 둘 다 재현 후 수정 적용해 확인함(로그/스크린샷, 이후 로그는 제거).

`NotificationMultiViewAdapter` 자체의 `updateData()`/`updateVisibleItems()` 로직은 문제
없었다 — 순수히 어댑터 재부착 문제였다.

## 별개로 발견한 잠재 버그 — SprefsManager 캐시 덮어쓰기 (같이 수정)

원인 조사 중 이번 버그와는 무관하지만 독립적으로 존재하던 문제를 발견해 같이 고쳤다:
`CreatePrescription`/`CreateDirectSchedule`/`CreatePaperEnvelope`(셋 다 약속 생성)와
`ModifyYaksok`(약속 수정)이 서버 응답의 `notifications` 필드(그 약속 하나에 대한 알림만
포함된 부분집합)로 `SprefsManager.setNotifications(...)`를 호출해 **로컬 전체 알림 캐시를
그 부분집합으로 덮어쓰고** 있었다.

이번 버그의 원인은 아니었다(재현 중 네트워크 요청이 항상 성공해 캐시 폴백 경로를 타지
않았음을 로그로 확인) — 하지만 `HomeFragment.loadNotificationList()`의 네트워크 요청이
언젠가 실패해 폴백 캐시를 읽는 시점이 오면, 이 4개 화면 중 하나라도 거친 뒤라면 오염된
부분집합이 보였을 것이다.

**수정**: 4개 파일에서 캐시 덮어쓰기 호출(및 그 때문에만 쓰이던 `NotificationYaksok` import)을
제거. 이제 전체 알림 캐시는 `HomeFragment.loadNotificationList()`의 성공 콜백만이 쓴다 —
등록/수정 화면은 자신이 만든 `Yaksok`을 `SprefsManager.addYaksok(...)`(별개의 약속 목록
캐시)에만 반영하고, 알림 캐시는 건드리지 않는다.

## 부수 정리 — 이제 안 쓰는 DTO 필드 제거

클라이언트가 더는 `SaveYaksokResponse.getNotifications()`를 읽지 않으므로, 백엔드/프론트
양쪽 DTO에서 `notifications` 필드를 제거했다(`yaksok` 필드만 남김 — 생성/수정 후 서버가
부여한 ID 등을 클라이언트가 여전히 필요로 하므로 응답 자체를 `void`로 바꾸지는 않음).

## 변경된 파일

- `YakSok_Frontend`: `ui/main/HomeFragment.java`(어댑터 재부착 수정),
  `ui/yaksok/CreatePrescription.java`/`CreateDirectSchedule.java`/`CreatePaperEnvelope.java`/
  `ModifyYaksok.java`(캐시 덮어쓰기 제거), `model/response/SaveYaksokResponse.java`(필드 제거)
  — 커밋 `788ed5d`, `32e59e1`
- `YakSok_Backend`: `yaksok/dto/SaveYaksokResponse.java`(필드 제거),
  `yaksok/service/YaksokService.java`(생성자 호출부 수정) — 커밋 `738536a`

## 검증

실기기(`R5CT33B9N6D`)에서 로그 + 스크린샷으로 수정 전/후 재현·확인(탭 전환 복귀, 날짜 변경
두 케이스 모두). 양쪽 저장소 모두 `./gradlew compileDebugJavaWithJavac` / `compileJava`
컴파일 확인.
