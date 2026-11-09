# 12. 프론트엔드 Activity/Fragment 패키지 재배치

**상태:** 완료

## 배경

`ui/<feature>/` 패키지 구조(login, signup, main, myinfo, setting, yaksok, sharedyaksok,
chattingroom, finddrugstore)는 이미 일부 도입돼 있었고, 소유자가 일부 화면을 직접 옮겨둔
상태였다. 나머지 화면들이 여전히 루트 패키지(`com.example.medication`)에 남아 있어 이번에
정리했다.

## 이동한 파일과 판단 근거

기준: **Activity 또는 Fragment(화면)만 이동**, Dialog/View/Service/Receiver/Application 같은
비-화면 클래스는 제외. 각 파일이 실제로 어느 기능에서 쓰이는지 Intent/import 참조를 추적해
소속을 정했다:

- `ui/yaksok/` — `ModifyYaksok`, `CreateDirectSchedule`, `CreatePaperEnvelope`,
  `CreatePrescription`, `ShowAddMedicationList`(약속 추가 방식 선택 바텀시트). 전부 약속
  생성/수정 흐름이고 `YaksokList`/`YaksokDetail`에서만 참조됨.
- `ui/finddrugstore/` — `DrugStoreList`, `DrugStoreDetail`. 기존 `FindDrugStore`와 같은
  약국 검색 기능.
- `ui/sharedyaksok/` — `SharingPage`(공유 메뉴 진입 화면). 기존 `ShareYaksokList`/
  `ShareYaksokDetail`과 같은 기능.
- `ui/signup/` — `SignUpTypeBottomSheet`(회원 유형 선택 바텀시트, `Login`에서 호출).
  `AddressSearch`(카카오 주소 검색 웹뷰)도 여기 포함시켰는데, **현재 어느 화면에서도 호출하지
  않는 죽은 코드**였다 — 매니페스트에만 등록돼 있고 Java 어디에서도 `AddressSearch.class`를
  참조하지 않음. 용도상 약국 회원가입 시 주소 입력에 쓰일 것으로 추정해 signup 밑에 뒀지만,
  실제로 연결할 때 다른 위치가 더 맞을 수도 있음.
- `ui/friend/`(신규 패키지) — `FriendList`. 기존에 친구 기능 전용 `ui` 패키지가 없어서 새로
  만듦.
- `ui/medicine/`(신규 패키지) — `MedicineSearchActivity`. 약국 검색과는 별개 기능(낱알/의약품
  검색)이라 `finddrugstore`에 합치지 않고 분리.
- `ui/common/`(신규 패키지) — `WipActivity`. 특정 기능에 속하지 않는 "미구현 화면" 플레이스홀더라
  공용 패키지로 뺌.

## 이동하지 않은 것

`LoadingDialog`(Dialog), `InputView`/`VerificationCodeView`(커스텀 View), `NotificationActionReceiver`
(BroadcastReceiver), `MyFirebaseMessagingService`(Service), `MedicationApp`(Application),
`LastCharVisibleTransformation`(PasswordTransformationMethod)은 여러 기능에서 공유되는
비-화면 컴포넌트라 루트 패키지에 남겼다. 이들도 옮기려면 `view/`(커스텀 위젯은 이미
`MapTouchContainer`가 있는 이 패키지가 자연스러움) 같은 타입 기반 패키지가 필요한데, 이번
요청 범위(Activity 파일)를 벗어나 보류함.

## 부수 변경

- 루트 패키지 밖으로 옮긴 파일들은 더 이상 `R`(리소스) 클래스와 같은 패키지가 아니게 되므로
  전부 `import com.example.medication.R;`을 추가해야 했다. 일부는 `InputView`/`LoadingDialog`도
  같은 이유로 import 추가.
- 서로 다른 새 패키지로 흩어진 클래스끼리 참조하는 곳(`ShowAddMedicationList` → 각
  `CreateXxx`, `ModifyYaksok`/`CreateXxx` → `MedicineSearchActivity`, `FriendList`/`SharingPage`
  → `WipActivity`, `SharingPage` → `FriendList`)은 새 import를 추가했고, 같은 패키지로 함께
  옮겨진 것들(`YaksokDetail`↔`ModifyYaksok`, `YaksokList`↔`ShowAddMedicationList`)은 기존
  import를 제거했다.
- `AndroidManifest.xml`의 `<activity android:name=".Xxx">` 항목 11개를 새 패키지 경로로 갱신.
  `ShowAddMedicationList`/`SignUpTypeBottomSheet`는 Fragment라 매니페스트에 등록되지 않으므로
  손대지 않음.
- `MainActivity`, `ChatRoomList`, `Settings`, `Login`, `YaksokList`, `YaksokDetail`의 import를
  새 경로로 갱신.

`./gradlew assembleDebug`로 컴파일 확인 완료.
