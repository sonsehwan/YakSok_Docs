# 35. 앱 아이콘

## 문제
`ic_launcher_background/foreground.xml`이 Android Studio 기본 템플릿(초록 배경 + 격자, 안드로이드 로봇)
그대로였다. 앱 디자인(`yaksok_design.pen`)과 전혀 맞지 않는 상태.

## 결정
디자인의 로고 마크(`Login`의 `Logo Mark`: `$p600` 주황 바탕 + 흰색 lucide `pill`)를 그대로 키워
아이콘으로 썼다. 앱 안의 로고와 홈 화면 아이콘이 같은 모양이 되도록.

- `ic_launcher_background.xml` — 단색 `#FD7E14`(`p600`)
- `ic_launcher_foreground.xml` — lucide `pill` 경로를 `group` 스케일(24→약 50dp)로 108dp 뷰포트 중앙에
  배치. 적응형 아이콘 안전 영역(중앙 66dp) 안에 들어가 원형 마스크에서도 잘리지 않는다.
  `monochrome`(테마 아이콘)도 같은 foreground 파일을 쓰므로 별도 작업 없음.
- `ic_launcher.xml`/`ic_launcher_round.xml`은 변경 없음. minSdk 26이라 `mipmap-*dpi/*.webp`
  구형 폴백은 쓰이지 않아 그대로 뒀다(남아 있는 기본 아이콘 파일일 뿐).

## 검토했지만 택하지 않은 대안
- **알약 + 체크 배지(약속의 의미를 담은 안)**: 의미는 더 분명했지만 소유자가 첫 안(알약만)을 선택.
  작은 크기에서는 단순한 형태가 더 잘 읽히기도 한다.
- **PNG/WebP 래스터 제작**: 디자인 도구의 내보내기가 자식 노드를 그리지 못해(스크린샷 제약) 벡터로 직접 옮김.
