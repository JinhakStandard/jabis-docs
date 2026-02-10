# jabis-common

> 공유 UI 라이브러리 (@jabis/ui, @jabis/layout, @jabis/auth, @jabis/core, @jabis/menu)

---

## 기술 스택
- pnpm monorepo
- React 18.2.0
- Radix UI
- Tailwind CSS
- TypeScript (일부)
- JavaScript

## 프로젝트 구조
```
jabis-common/
├── ui/                 # @jabis/ui (30+ shadcn/ui 컴포넌트)
│   ├── src/components/
│   ├── src/styles/globals.css
│   ├── src/index.js
│   └── tailwind.preset.js
├── layout/             # @jabis/layout (Header, Sidebar)
├── auth/               # @jabis/auth (OAuth, session)
│   ├── authConfig.js
│   ├── authService.js
│   ├── SessionTimeoutProvider.jsx
│   └── LoginPage.jsx
├── core/               # @jabis/core (Theme, providers)
│   ├── ThemeProvider.jsx
│   └── ThemeToggle.jsx
├── menu/               # @jabis/menu (공통 메뉴, 역할 데이터, 빌더)
│   └── src/
│       ├── commonGroups.js   # createCommonGroups() — 5개 공통 메뉴 그룹
│       ├── roles.js          # ROLES, roleLabels, defaultUserNames
│       ├── buildMenu.js      # buildMenu(), staticMenu() 유틸리티
│       └── index.js
├── package.json
└── pnpm-workspace.yaml
```

## 주요 명령어
없음 (워크스페이스/라이브러리 전용)

## 포트
N/A (라이브러리)

## 환경변수 (Dockerfile)
N/A

## Dockerfile
없음 (컨테이너화 안 됨)

## Bitbucket Pipeline
없음

## Skills
없음

## @jabis/menu 사용법

```js
import { buildMenu, staticMenu, roleLabels, defaultUserNames } from '@jabis/menu'

// 공통 그룹 + 역할 전용 메뉴 조합
export const myMenu = buildMenu('hr', (common, { extend }) => [
  { icon: 'LayoutDashboard', label: '대시보드', path: '/hr' },
  common.work,                    // 공통 그룹 그대로 사용
  extend(common.work, [           // 공통 그룹에 아이템 추가 (뒤에)
    { icon: 'FolderKanban', label: '프로젝트', path: '/hr/projects' },
  ]),
  extend(common.organization, {   // 앞뒤로 아이템 추가
    prepend: [{ icon: 'Users', label: '팀 관리', path: '/hr/teams' }],
    append: [{ icon: 'BarChart3', label: '리포트', path: '/hr/reports' }],
  }),
  common.communication,
  common.life,
  common.etc,
])

// 공통 그룹 없이 정적 메뉴
export const portalMenu = staticMenu([
  { icon: 'LayoutDashboard', label: '홈', path: '/portal' },
])
```

## @jabis/layout 역할 전환 기능

Header 컴포넌트에서 역할 배지 클릭 시 역할 전환 드롭다운을 표시할 수 있다.

```jsx
<DashboardLayout
  // ...기존 props...
  availableRoles={[
    { id: 'developer', label: '개발자' },
    { id: 'operation', label: '운영자' },
    // ...
  ]}
  onRoleChange={(roleId) => {
    localStorage.setItem('jabis_role', roleId)
    navigate(`/${roleId}`)
  }}
>
```

- `availableRoles`: `[{ id, label }]` 배열. `@jabis/menu`의 `ROLES`와 `roleLabels`로 생성 가능.
- `onRoleChange`: 역할 변경 시 호출. localStorage 업데이트 + 페이지 이동 처리.
- 두 prop이 모두 없으면 기존 정적 Badge로 폴백 (하위 호환).

## 프로젝트 고유 규칙
- pnpm 필수 (npm 사용 금지)
- 모든 UI 컴포넌트는 @jabis/ui로 export
- 메뉴 공통 그룹/역할 데이터는 @jabis/menu로 export
- tailwind.preset.js로 디자인 토큰 통일
- Git submodule로 jabis, jabis-template, jabis-design-system에서 참조
- 부서 리포에서 수정 금지 — import만 허용
