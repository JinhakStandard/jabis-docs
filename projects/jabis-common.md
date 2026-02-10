# jabis-common

> 공유 UI 라이브러리 (@jabis/ui, @jabis/layout, @jabis/auth, @jabis/core)

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

## 프로젝트 고유 규칙
- pnpm 필수 (npm 사용 금지)
- 모든 UI 컴포넌트는 @jabis/ui로 export
- tailwind.preset.js로 디자인 토큰 통일
- Git submodule로 jabis, jabis-template, jabis-design-system에서 참조
- 부서 리포에서 수정 금지 — import만 허용
