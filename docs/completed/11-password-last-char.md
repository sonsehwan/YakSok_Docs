# 11. 비밀번호 입력 시 마지막 글자만 잠깐 노출 (2026-09-10)

**상태:** 완료(코드 반영). `./gradlew compileDebugJavaWithJavac` 통과. 기기 검증은 미실시.

`InputView`의 비밀번호 모드에서, 방금 입력한 글자 하나만 **2초간** 보여주고 다시 가린다.
포커스가 빠지면 즉시 가리고, 백스페이스로 지울 때는 노출하지 않는다.

## 11.1 문제

`LastCharVisibleTransformation`(`PasswordTransformationMethod` 상속, 마지막 인덱스만 원문 반환)을
만들어 `setupPasswordMode()`에서 한 번 걸어둔 상태였다.

```java
editText.setTransformationMethod(lastCharMask);   // 이전 구현
```

이러면 마지막 글자가 **입력을 멈춘 뒤에도, 화면을 떠날 때까지 계속** 보인다. 오타 방지라는
목적은 달성하지만 어깨너머 노출을 줄인다는 목적은 달성하지 못한다. 노출이 "상태"가 아니라
"고정 설정"이라 되돌릴 수단 자체가 없는 구조였다.

## 11.2 설계 판단 — 왜 플래그가 아니라 인스턴스 교체인가

가장 자연스러운 접근은 `LastCharVisibleTransformation`에 `boolean reveal` 필드를 두고
`charAt()`에서 분기시키는 것이다. **동작하지 않는다.**

- `TextView`는 transformation 적용 결과를 `mTransformed`에 **캐시**한다. 플래그만 바꿔도
  `getTransformation()`이 다시 불리지 않아 화면이 그대로다.
- `invalidate()`로도 안 된다. 캐시된 `CharSequence`를 다시 그릴 뿐이다.
- 같은 인스턴스로 `setTransformationMethod()`를 다시 호출해도 안 된다. 이 메서드의 첫 줄이
  `if (method == mTransformation) return;` 이라 no-op이다.

→ 갱신을 강제하려면 **서로 다른 인스턴스 두 개를 번갈아** 지정해야 한다. 그래서
전체 마스킹용으로 `PasswordTransformationMethod.getInstance()`(싱글턴)를 필드로 하나 더 둔다.
`LastCharVisibleTransformation`이 `PasswordTransformationMethod`를 상속하고 있어 둘 다
`hasPasswordTransformationMethod()`가 true이므로, 교체해도 접근성·IME 쪽 취급은 달라지지 않는다.

```java
private final LastCharVisibleTransformation lastCharMask = new LastCharVisibleTransformation();
private final TransformationMethod fullMask = PasswordTransformationMethod.getInstance();
```

`LastCharVisibleTransformation` 자체는 손대지 않았다. "마지막 글자를 보여주는 방법"과
"언제 보여줄지"를 분리해 두는 편이 각각 이해하기 쉽다.

### 검토했지만 택하지 않은 대안

- **Transformation 내부 플래그 + `invalidate()`** — 위 캐시 문제로 화면이 갱신되지 않는다.
- **`editText.setText(editText.getText())`로 강제 갱신** — 갱신은 되지만 커서·선택 영역이
  날아가고 TextWatcher가 통째로 다시 도는 부작용이 `setTransformationMethod` 교체보다 크다.
- **`EditText`를 상속한 전용 뷰 신설** — `InputView`가 이미 compound view라 계층만 늘어난다.
  transformation 교체로 충분한 문제에 클래스를 하나 더 만들 이유가 없다.

## 11.3 함정 — `setTransformationMethod()`이 TextWatcher를 재진입시킨다

`TextView.setTransformationMethod()`의 마지막 동작은 `setText(mText)`다. 즉 **마스크를 교체하면
TextWatcher가 다시 돈다.** 이때 넘어오는 값은 전체 교체라 `before == count`가 되고,
삽입/삭제를 `count > before`로 판정하는 로직이 이 호출을 **삭제로 오인**한다.

```
'a' 입력 → onTextChanged(before=0, count=1) → 삽입=true
        → afterTextChanged → applyMask(true) → 내부 setText 로 재진입
           → onTextChanged(before=1, count=1) → 삽입=false        ← 플래그 오염
           → afterTextChanged(중첩) → 삭제로 판단 → 타이머 취소 + 즉시 마스킹
        → 결과: 마지막 글자가 아예 보이지 않음
```

그래서 `isUpdatingMask` 플래그로 마스크 교체 구간을 감싸고, 두 콜백 모두 그 구간의 호출을
무시하게 했다. 삭제 판정을 넣는 순간 필수가 되는 가드다.
(삭제 판정이 없던 이전 버전에서는 조건이 멱등이라 이 재진입이 드러나지 않았다.)

커서 복원도 같은 이유에서 필요하다. `setText`를 다시 타면서 선택 위치가 0으로 초기화되므로,
교체 전후로 `getSelectionStart/End` → `setSelection`을 해주지 않으면 2초마다,
그리고 글자를 칠 때마다 커서가 맨 앞으로 튄다.

## 11.4 노출을 끝내는 두 경로

| 경로 | 처리 |
| --- | --- |
| 포커스 아웃 | 기존 `OnFocusChangeListener`(`validateInput()` 호출부)에서 타이머 취소 + 즉시 전체 마스킹 |
| 시간 경과 | `Handler.postDelayed` 2초. 입력마다 `removeCallbacks`로 리셋하는 디바운스 → "마지막 입력 후 2초" |

`Runnable`은 **필드로 하나만** 둔다. 익명 객체를 매번 `new` 하면 `removeCallbacks`로 취소되지
않아 예약이 쌓인다. 그리고 `onDetachedFromWindow()`에서 `removeCallbacks`를 부른다 —
예약된 `Runnable`이 `InputView` → Activity를 붙잡아 누수되는 것을 막는다.

시간은 2초로 정했다. iOS 기본 동작(약 1.5초)보다 조금 길게 잡았는데, 고령 사용자 비중을
고려한 선택이다(→ `todo.md` 접근성 항목과 같은 맥락).

## 11.5 노출하지 않는 경우

- **삭제(백스페이스)** — `onTextChanged`의 `count > before`로 판정. 지울 때 남은 마지막 글자가
  다시 드러나는 것은 사용자가 기대하지 않는 노출이다. 즉시 전체 마스킹하고 타이머도 취소한다.
  참고: 붙여넣기(`before=0, count=N`)는 삽입으로 취급돼 마지막 글자가 2초 노출된다.
  이것까지 막으려면 `count == before + 1`(한 글자 입력)로 좁히면 된다.
- **빈 문자열** — 보여줄 글자가 없다.
- **포커스가 없을 때의 텍스트 변경** — `setText()` 같은 프로그램적 변경에서 글자가 새지 않도록
  `editText.hasFocus()`로 막는다.
- **눈 아이콘으로 전체 보기 중(`isPasswordVisible`)** — `applyMask()` 첫 줄에서 early return.
  타이머가 뒤늦게 깨어나 사용자가 켜둔 전체 보기를 덮어쓰면 안 된다. 토글 핸들러 쪽도
  `lastCharMask` → `fullMask`로 바꾸고 예약을 취소한다.

## 11.6 변경 범위

`YakSok_Frontend`, `InputView.java` 한 파일. `LastCharVisibleTransformation`은 변경 없음.

- 필드 추가: `fullMask`, `REVEAL_DURATION_MS`(2000L), `revealHandler`, `hideLastCharRunnable`,
  `lastChangeWasInsertion`, `isUpdatingMask`
- `setupPasswordMode()` — 초기 마스크를 `fullMask`로, `TextWatcher` 추가, 토글 핸들러가
  `fullMask`를 쓰도록 수정
- `OnFocusChangeListener` — `!hasFocus` 분기에 타이머 취소 + `applyMask(false)`
- 신설: `applyMask(boolean)`, `scheduleHideLastChar()`, `cancelHideLastChar()`,
  `onDetachedFromWindow()`

## 11.7 기기에서 확인할 것

컴파일만 확인했다. 아래는 실제 동작 확인이 필요한 목록이다.

- 타이핑 → 마지막 글자 2초 노출 → 자동 마스킹
- 연속 타이핑 중에는 계속 노출되고, 멈춘 뒤 2초에 숨는지(타이머 리셋)
- 백스페이스 → 즉시 전체 마스킹
- 다른 필드로 이동 → 즉시 마스킹 + 기존 유효성 검사 동작 유지
- 눈 아이콘 켠 상태에서 타이핑 → 타이머가 끼어들어 다시 가리지 않는지
- 커서를 문자열 중간에 두고 입력 → 커서가 맨 앞으로 튀지 않는지
