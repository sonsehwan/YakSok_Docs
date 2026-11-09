# 36. AI 약속 만들기 — 추가 못한 약 이름을 화면에 상시 표시

## 문제
`CreatePrescription`에서 AI가 뽑은 약 이름을 공공 API로 조회할 때 실패한 약은 `failedPillNames`에
모아 마지막에 `Toast`로만 알렸다. 토스트는 몇 초 뒤 사라지고, 약이 많아 문구가 길면 잘려서
어떤 약이 빠졌는지 확인할 수 없었다.

## 변경
- `activity_create_prescription.xml`: "약 선택 *" 라벨 바로 위에 `tv_failed_pills`(`red500`, 기본 `gone`) 추가.
- `CreatePrescription.onPillSearchFinished()`: 토스트 대신 "추가하지 못한 약: A, B"를 이 TextView에 표시.
- `parseAndDisplayGeminiResult()`: 재스캔 시 시작 지점에서 TextView를 다시 숨겨 이전 결과가 남지 않게 함.

## 택하지 않은 대안
- **다이얼로그(`AppDialog`)**: 확실히 보이지만 매번 닫아야 하고, 닫은 뒤엔 다시 확인할 수 없다.
- **스낵바(`LENGTH_INDEFINITE`)**: 스크롤 위치에 따라 화면 하단에 떠서 "약 선택" 영역과 연결되지 않는다.

## 남은 것
`CreatePaperEnvelope`는 조회 실패 약을 사용자에게 알리지 않는다(로그만). 같은 방식으로 맞출지 미정.

## 후속: 약 목록 일괄 반영 + 로딩 다이얼로그 지연 종료
원인은 앱이 아니라 공공 API가 느린 것이었는데, 약을 응답 오는 대로 하나씩 `add`하다 보니
Gemini 응답 직후 로딩 다이얼로그가 먼저 닫히고 목록은 한참 뒤에야 채워졌다.

- `searchedPills[]`에 처방전 순서(인덱스)대로 성공한 약을 모아두고, 마지막 조회가 끝나는
  `onPillSearchFinished()`에서 `selectedPills`에 한 번에 넣고 실패 안내와 함께 반영한 뒤 다이얼로그를 닫는다.
  응답 도착 순서가 아니라 처방전 순서로 정렬되는 부수 효과도 있다.
- `parseAndDisplayGeminiResult()`가 boolean(조회 시작 여부)을 반환 — false(빈 결과·파싱 실패)면
  호출한 `onSuccess`가 바로 다이얼로그를 닫는다.
- 택하지 않은 대안: 다이얼로그를 유지하면서 하나씩 추가(원래 불만이던 "점진적으로 채워짐"이 그대로).
- `CreatePaperEnvelope`는 이미 다이얼로그를 끝까지 유지하지만 약은 여전히 하나씩 추가한다 — 필요하면 같은 방식으로 맞출 것.

`CreatePaperEnvelope`(진입: `BottomSheetAddYaksok`)도 같은 방식으로 맞춤: `searchedPills[]`에 약봉투 순서대로 모아
`showSearchedPills()`에서 한 번에 반영하고 다이얼로그를 닫는다. 조회 실패 안내는 여전히 없다.
