# 43. 처방전·약봉투 사진: 카메라 촬영 / 앨범 선택

## 문제
`CreatePrescription`·`CreatePaperEnvelope`가 진입하자마자 `GetContent`로 앨범만 열었다. 처방전을
받은 자리에서 바로 찍을 수 없었다. 시스템 선택 화면(`GetContent`, Android 13+ Photo Picker)에는
카톡처럼 카메라 칸을 끼워 넣을 수 없어 별도 구현이 필요했다.

## 구현
- 진입 시 `ImageSourceBottomSheet`(새 클래스, `SignUpTypeBottomSheet`와 같은 행 스타일)를 띄워
  "카메라로 촬영 / 앨범에서 선택"을 고른다.
- **시트가 Uri를 얻는 과정 전체를 책임진다**: 갤러리(`GetContent`), 카메라(`TakePicture`),
  권한(`RequestPermission`) 런처 3개를 모두 시트가 가진다. 최종 Uri는 Fragment Result API
  (`REQUEST_KEY`/`KEY_URI`, null = 취소)로 Activity에 넘긴다. Activity는 리스너 하나만 등록하고,
  그 뒤의 분석 코드(`recognizeTextFromImage` / `analyzeTextWithGemini`)는 그대로다.
- 촬영 파일은 `FileProvider`(`${applicationId}.fileprovider`, `res/xml/file_paths.xml`의
  `cache/images/`)로 노출한다. 파일은 `capture.jpg` 하나를 계속 덮어써서 쌓이지 않는다.
- 시트는 카메라나 갤러리가 끝난 뒤 `deliver()`에서 닫는다. 먼저 닫으면 결과를 받을 Fragment가 없다.
- 바깥 터치나 뒤로가기는 `onCancel`에서 null 결과로 처리하고, Activity는 기존처럼 토스트 후 `finish()`.
- 매니페스트에 `CAMERA`가 이미 선언돼 있어(QR 스캔용), 권한 없이 `ACTION_IMAGE_CAPTURE`를
  보내면 `SecurityException`이 난다. 그래서 런타임 권한을 확인한다.
- `pendingCameraUri`는 `onSaveInstanceState`로 보존한다. 카메라 앱이 떠 있는 동안 프로세스가
  죽어도 사진을 잃지 않게 하기 위해서다.
- Activity는 `savedInstanceState == null`일 때만 시트를 띄운다. 회전 후에는 FragmentManager가
  시트를 복원하므로, 매번 띄우면 시트가 두 개 뜬다.
- 아이콘 `ic_camera`, `ic_image`(Material 벡터)를 새로 추가했다.

## 택하지 않은 대안
- **선택창 UI**
  - 시스템 앱 선택창(`createChooser` + `EXTRA_INITIAL_INTENTS`): 기기마다 모양이 달라 투박하다.
  - 카톡식 자체 사진 그리드: MediaStore 조회, `READ_MEDIA_IMAGES` 권한, 그리드 화면을 직접
    만들어야 해서 한 장 고르는 화면치고 비용이 너무 크다.
  - `AppDialog`: 이미 있는 가입 유형 선택 바텀시트와 맞추는 쪽을 택했다.
- **런처 위치**: Activity마다 런처를 두고 시트는 선택만 돌려주는 방식. 권한·카메라·FileProvider
  코드가 두 Activity에 중복된다.
- **결과 전달**: 생성자로 콜백 인터페이스를 넘기는 방식. 회전으로 Fragment가 다시 만들어지면
  콜백이 사라진다.
- **`TakePicturePreview`**(Bitmap 바로 받음, FileProvider 불필요): 썸네일 해상도라 OCR에 부적합.

## 상태
`assembleDebug` 통과. 실기기 확인(촬영→분석, 권한 거부, 회전, 취소) 완료.
