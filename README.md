# YakSok 설계 문서

복약 관리 앱 **YakSok**을 1인으로 기획·개발하며 남긴 설계·의사결정 기록입니다.
작업마다 **문제 정의 → 원인 분석 → 택한 방법 → 검토했지만 택하지 않은 대안과 그 이유**를 적었습니다.

- 백엔드 (Spring Boot · Java 21 · MySQL · AWS EC2): [sonsehwan/YakSok_Backend](https://github.com/sonsehwan/YakSok_Backend)
- 안드로이드 앱 (Java): [sonsehwan/YakSok_Frontend](https://github.com/sonsehwan/YakSok_Frontend)

## 먼저 읽으면 좋은 문서 3개

| 문서 | 한 줄 요약 |
|---|---|
| [JWT 인증 도입](docs/completed/08-jwt-auth.md) | 요청 본문의 userId를 믿던 API를 토큰 기반으로 전환. 겉으론 정상이던 "Bearer" 공백 버그(매 요청 401 → 재발급 → 재시도)를 헤더 비교로 찾은 과정 포함 |
| [FCM 다기기 지원](docs/completed/02-fcm-multi-device.md) | 두 번째 기기 로그인 시 첫 기기 푸시 토큰이 덮어써지던 구조를 1:N으로 재설계. create-only가 왜 안전하지 않은지, 로그아웃 삭제에 정합성을 맡기지 않은 이유 |
| [홈 알림이 사라지던 버그](docs/completed/19-notification-adapter-reattach-fix.md) | 처음 세운 '캐시 오염' 가설을 로그로 반증하고 Fragment 생명주기에서 진짜 원인을 찾은 과정 |

JWT 도입 과정은 [블로그 글 4편](blog/JWT/)으로 더 길게 정리했습니다.

## 구조

| 위치 | 내용 |
|---|---|
| [docs/INDEX.md](docs/INDEX.md) | 전체 목차 (문서마다 한 줄 요약) |
| [docs/completed/](docs/completed/) | 끝난 작업의 설계 판단 기록 |
| [docs/active/](docs/active/) | 진행 중이거나 계획 단계인 작업 |
| [docs/spec.md](docs/spec.md) · [docs/screens.md](docs/screens.md) | 기능 명세 · 화면 인벤토리 |
| [docs/tech-debt.md](docs/tech-debt.md) · [docs/todo.md](docs/todo.md) | 알려진 기술 부채 · 앞으로 할 일 |
| [blog/](blog/) | 특정 작업을 길게 풀어 쓴 글 |

## 기록 원칙

- **남은 일도 숨기지 않고, 미룬 이유와 함께 적습니다.** `active/`, `tech-debt.md`, `todo.md`의 항목은 의도적으로 범위를 나눈 결과입니다.
- **택하지 않은 대안을 꼭 남깁니다.** 같은 고민을 다시 하지 않기 위해서입니다.
- AI 코딩 도구(Claude Code)를 함께 사용했습니다. 세션이 바뀌어도 작업을 이어갈 수 있도록 `INDEX.md`를 진입점으로 두고, 코드가 바뀌었는데 문서가 갱신되지 않으면 작업을 끝낼 수 없도록 훅으로 강제했습니다. `INDEX.md` 상단의 작성 규칙이 그 흔적입니다.
