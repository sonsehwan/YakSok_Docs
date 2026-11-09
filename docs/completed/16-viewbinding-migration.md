# 16. 프론트엔드 ViewBinding 전환

## 무엇을

`YakSok_Frontend`의 Activity/Fragment(24개)와 RecyclerView 어댑터(14개), 총 38개 파일을
수동 `findViewById`/`LayoutInflater.inflate` 방식에서 안드로이드 Gradle 플러그인이 생성하는
ViewBinding(`com.example.medication.databinding.*`) 방식으로 전환했다.

`buildFeatures.viewBinding = true`는 10장(아이디 찾기/비번 재설정) 때 이미 켜져 있었고,
그때 만든 6개 파일(`FindId`, `FindPassword`, `ResetPassword`, `FindDrugStore`,
`DrugStoreDetail`, `MedicineSearchActivity`)이 기준 스타일이었다. 이번 작업은 나머지 전체로
확장한 것.

## 패턴

- **Activity:** `private ActivityXxxBinding binding;` 필드 → `onCreate`에서
  `binding = ActivityXxxBinding.inflate(getLayoutInflater()); setContentView(binding.getRoot());`
- **Fragment:** `onCreateView`에서 `binding = FragmentXxxBinding.inflate(inflater, container, false); return binding.getRoot();`,
  `onDestroyView`에서 `binding = null;` (뷰 생명주기가 Fragment 생명주기보다 짧아 메모리 누수 방지)
- **RecyclerView 어댑터:** `ViewHolder`가 개별 뷰 필드 대신 `ItemXxxBinding binding` 하나를 들고,
  `onCreateViewHolder`에서 `ItemXxxBinding.inflate(...)`로 만들어 넘긴다.
  `holder.itemView`는 `holder.binding.getRoot()`와 동일하므로 그대로 유지.
- id → 필드명 변환은 실제 생성 규칙(언더스코어 제거 후 camelCase)을 레이아웃 XML과 대조해서
  확인했다 — 기존 Java 필드명이 실제 id와 다르게 지어진 곳(`inputNickName` vs 실제 id
  `input_nickname` → `inputNickname`, `btnFindDrugStore` vs `btnFindDrugstore` 등)이 여러 군데
  있었는데, 전부 XML 기준으로 바로잡았다.

## 범위에서 제외한 것

`InputView`, `VerificationCodeView`, `LoadingDialog` 세 커스텀 뷰/다이얼로그는 이번 전환에서
**제외했다** — 자체 `LayoutInflater.from(context).inflate(...)` 호출이 있어 같은 방식으로
바꿀 수 있지만, 이번 요청 범위를 Activity/Fragment/어댑터로 한정했기 때문. 다음에 손댈 때
같은 패턴으로 전환하면 된다.

## 진행 중 발견한 버그 — `MedicationAdapter`

`MedicationAdapter`(어댑터, `item_notification.xml` 사용)의 기존 코드가
`itemView.findViewById(R.id.tv_pill_name)`로 제목 뷰를 찾고 있었는데, `tv_pill_name`은
`item_notification.xml`에 없는 id였다(다른 레이아웃 `item_add_medication_setting.xml`에만
존재 — `R.id`가 전역 네임스페이스라 컴파일은 통과했었다). 즉 이 어댑터가 바인딩될 때마다
`holder.tvName`이 `null`이 되어 `setText()` 호출 시 NPE가 났을 것으로 보인다.

ViewBinding은 존재하지 않는 id에 대응하는 필드를 아예 생성하지 않으므로 이 상태로는 컴파일이
안 됐다. `item_notification.xml`에 실제로 있고 의미상 제목에 해당하는 `tv_notification_name`
(`todo.md`의 "알림 목록 제목 말줄임" 항목에서도 `MedicationAdapter`와
`NotificationMultiViewAdapter`가 같은 레이아웃의 이 id를 공유한다고 확인됨)으로 매핑해서
고쳤다.

**확인 결과 — `MedicationAdapter`는 죽은 코드였다.** 홈 화면 알림 목록(`HomeFragment`,
`rv_medication`)은 나중에 만든 `NotificationMultiViewAdapter`가 맡고 있고, `MedicationAdapter`는
그 전 버전으로 코드베이스 어디에서도 `new MedicationAdapter(...)`로 생성되지 않았다
(소유자 확인, 2026-09-15). `tv_pill_name` 버그가 지금까지 드러나지 않은 것도 이 때문 — 실행
경로를 안 타서 크래시로 나타날 일이 없었다. 이번에 파일 자체를 삭제했다
(`app/src/main/java/com/example/medication/adapter/MedicationAdapter.java`, 다른 곳에서의
참조 없음 확인 후 삭제, `./gradlew compileDebugJavaWithJavac` 재확인).

## 검증

`./gradlew compileDebugJavaWithJavac` BUILD SUCCESSFUL로 컴파일 확인. 이후 실기기
동작 확인까지 완료.
