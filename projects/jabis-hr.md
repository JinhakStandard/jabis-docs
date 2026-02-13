# jabis-hr

> 인사담당자 전용 앱 — HR 역할 기반 대시보드 (채용, 인사, 급여, 교육/평가)

---

## 기술 스택
- React 18.2 (JSX)
- Vite 5
- pnpm monorepo
- Zustand
- React Router v6
- Tailwind CSS 3.3 + CVA
- @jabis/ui
- Lucide React

## 프로젝트 구조
```
jabis-hr/
├── pnpm-workspace.yaml
├── packages/              # git submodule (jabis-common)
│   ├── ui/                # @jabis/ui
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   └── core/              # @jabis/core
├── apps/
│   └── hr/
│       ├── src/
│       │   ├── components/
│       │   ├── pages/
│       │   │   ├── DashboardPage.jsx
│       │   │   └── ComingSoonPage.jsx
│       │   ├── stores/
│       │   ├── App.jsx
│       │   └── main.jsx
│       ├── .env.local
│       ├── .env.preview
│       ├── .env.production
│       ├── vite.config.js
│       ├── tailwind.config.js
│       └── package.json
├── .ai/
├── .claude/
├── CLAUDE.md
├── Dockerfile
└── bitbucket-pipelines.yml
```

## 주요 명령어
```bash
pnpm install              # 의존성 설치
pnpm dev                  # 개발 서버 (localhost:5173)
pnpm build                # 프로덕션 빌드
pnpm build:preview        # 미리보기 빌드
```

## 포트
- 3000 (serve)
- 5173 (Vite dev)

## 라우팅 경로
- `/hr/*` — 인사관리 전용 경로

## 환경변수
- `VITE_OAUTH_ENABLED` — OAuth 활성화 여부
- `VITE_OAUTH_CLIENT_ID` — OAuth 클라이언트 ID
- `VITE_OAUTH_REDIRECT_URI` — https://jabis.jinhakapply.com/hr/auth/callback
- `VITE_OAUTH_AUTH_SERVER` — https://jabis-cert.jinhakapply.com

## Dockerfile
- node:20-alpine multi-stage pnpm 빌드
- jabis-common git clone → packages 빌드 → hr 앱 빌드
- serve -s dist -l 3000
- SPA rewrite: `/hr/**` → `/hr/index.html`

## Bitbucket Pipeline
- master → 수동 deploy
- jabis-common을 git clone으로 받아옴
- Harbor push → Helm update → Teams 알림

## 역할
- hr (인사담당자) — 메인 역할
- 역할 전환 드롭다운으로 다른 역할로 이동 가능

## HR 전용 메뉴
- 채용 관리 (공고, 지원자, 면접, 현황)
- 인사 관리 (직원정보, 인사이동, 직급/직책, 근태, 휴가)
- 급여/복리후생 (급여, 복리후생, 4대보험)
- 교육/평가 (교육 프로그램, 교육 일정, 인사 평가, 목표)
- 업무 관리 (할일, 문서작성, 전자결재, 일정) — 공통
- 소통 (메신저, 메일, 회의실) — 공통
- 사내 생활 — 공통
- 조직/관리, 기타 — 공통

## 프로젝트 고유 규칙
- jabis-{dept} 부서별 앱 패턴 준수
- UI 컴포넌트는 @jabis/ui import, packages/ 수정 금지
- POST + action 패턴으로 CRUD 처리
- basename: `/hr`
- 미리보기 base path: `/preview/jabis-hr/`
- 운영 base path: `/hr/`
