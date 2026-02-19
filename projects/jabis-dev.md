# jabis-dev

> 개발부서 전용 앱 — jabis.jinhakapply.com/dev 경로로 서빙되는 jabis 하위 프로젝트

---

## 기술 스택
- React 18.2 (JSX)
- Vite 5
- pnpm monorepo
- Zustand
- React Router v6
- Tailwind CSS
- @jabis/ui

## 프로젝트 구조
```
jabis-dev/
├── pnpm-workspace.yaml
├── packages/
│   ├── ui/                # @jabis/ui (jabis-common에서 clone)
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   └── core/              # @jabis/core
├── apps/
│   └── dev/
│       ├── src/
│       └── tailwind.config.js
└── public/
    └── serve.json         # /dev/ 경로 rewrite 규칙
```

## 주요 명령어
```bash
pnpm install              # 의존성 설치
pnpm dev                  # 개발 서버 (--filter dev)
pnpm build                # 프로덕션 빌드
pnpm preview              # 빌드 결과 미리보기
```

## 포트
- 3000 (serve)

## 환경변수 (Dockerfile)
- NODE_OPTIONS (--max-old-space-size=8192)
- VITE_BUILD_MINIFY (esbuild)

## Dockerfile
- node:20-alpine multi-stage pnpm 빌드
- jabis-common git clone → packages 빌드 → dev 앱 빌드
- /dev/ 경로 하에서 서빙 (serve.json rewrite)

## Bitbucket Pipeline
- Pattern A
- prod only

## Skills
없음

## 프로젝트 고유 규칙
- jabis 메인 앱의 /dev 경로로 라우팅되어 서빙되는 하위 프로젝트 (jabis.jinhakapply.com/dev)
- jabis-template을 복제하여 생성된 부서별 앱(jabis-{dept})의 첫 번째 사례
- UI 컴포넌트는 반드시 packages/ui/에 위치 → @jabis/ui import
- @jabis/ui 변경 후 개발 서버 재시작 필요
- **공통 메뉴는 `@jabis/menu`의 `buildMenu('developer', ..., { standalone: true })`로 관리** (하드코딩 금지)
- 개발 도구 메뉴만 App.jsx에서 직접 정의, 공통 그룹(업무/소통/사내생활/조직/기타)은 `common.*` 사용
