# 32. 사이드 메뉴(drawer)를 오른쪽에서 열리게

## 문제

햄버거 버튼(`iv_menu`)이 홈·약속 목록 두 화면 모두 헤더 **오른쪽 끝**에 있는데, drawer는 왼쪽에서
열렸다. 버튼 위치와 패널이 나오는 방향이 반대라 어색했다.

## 판단

`DrawerLayout`의 방향은 패널의 `layout_gravity`와 코드의 `GravityCompat` 인자가 **서로 맞아야**
동작한다. 한쪽만 바꾸면 `openDrawer(START)`가 "해당 gravity의 drawer 없음"으로 예외를 던진다.
그래서 XML 1곳 + Java 5곳을 한 번에 `START → END`로 바꿨다.

- `start`/`end`를 썼다(`left`/`right` 아님). RTL 로케일에서 햄버거 버튼도 같이 뒤집히므로
  버튼 위치와 drawer 방향이 계속 짝을 이룬다.
- 패널 배경 `bg_nav_drawer_panel`은 "오른쪽 모서리만 둥글게"였다. 오른쪽에서 열리면 화면 가장자리에
  붙는 쪽이 오른쪽이라, 둥근 모서리를 **왼쪽(안쪽)** 으로 옮겼다.
- 햄버거 버튼을 왼쪽으로 옮겨 기존 방향을 유지하는 안은 택하지 않았다. 다른 화면(약속 상세 등)의
  `iv_menu`도 오른쪽에 있어 헤더 패턴이 깨진다.

## 변경

- `activity_main.xml`: `side_nav_view`의 `layout_gravity` `start` → `end`.
- `bg_nav_drawer_panel.xml`: 둥근 모서리 오른쪽 → 왼쪽.
- `MainActivity`: `closeDrawer`/`isDrawerOpen` 3곳 `GravityCompat.END`.
- `HomeFragment`, `YaksokListFragment`: `openDrawer(GravityCompat.END)`.
