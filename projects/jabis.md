# jabis

> 메인 운영 앱 — jabis-{dept} 부서별 앱의 홈 역할 (역할별 대시보드, OAuth 클라이언트)

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
- Recharts
- React Flow

## 프로젝트 구조
```
jabis/
├── pnpm-workspace.yaml
├── packages/
│   ├── ui/                # @jabis/ui (jabis-common에서 clone)
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   └── core/              # @jabis/core
├── apps/
│   └── prototype/
│       ├── src/
│       │   ├── components/
│       │   │   ├── common/
│       │   │   ├── dashboard/
│       │   │   ├── layout/
│       │   │   └── widgets/
│       │   ├── pages/
│       │   ├── stores/
│       │   ├── data/
│       │   ├── App.jsx
│       │   └── main.jsx
│       └── tailwind.config.js
└── public/
    └── serve.json         # SPA 라우팅 설정
```

## 주요 명령어
```bash
pnpm install              # 의존성 설치
pnpm dev                  # 개발 서버 (--filter prototype)
pnpm build                # 프로덕션 빌드
pnpm preview              # 빌드 결과 미리보기
```

## 포트
- 3000 (serve)
- 5173 (Vite dev)

## 환경변수 (Dockerfile)
- NODE_OPTIONS (--max-old-space-size=8192)
- VITE_BUILD_MINIFY (esbuild)

## Dockerfile
- node:20-alpine multi-stage pnpm 빌드
- jabis-common git clone → packages 빌드 → prototype 빌드
- serve -s dist -l 3000

## Bitbucket Pipeline
- Pattern A
- master → 수동 deploy
- alpha → 수동 deploy
- jabis-common을 git clone으로 받아옴

## Skills
없음 (.claude/commands, .claude/hooks만 있음)

## 역할 (Roles)
`@jabis/menu`의 `roles.js`에 정의된 역할 목록:
- developer(개발자), operation(운영자), sales(영업), design(디자이너), admin(관리자)
- superadmin(시스템 관리자), finance(재무), teamlead(팀장), depthead(부장), executive(본부장)
- hr(인사), ceo(CEO), aiadmin(AI 관리자), portal(AI 포탈), planner(기획자), marketer(마케터)
- **producer(프로듀서)** — JABIS 에코시스템 자체를 빌드/배포/관리하는 역할. developer(jabis 사용자 중 개발 직군)와 구분됨.

## 프로젝트 고유 규칙
- jabis-{dept} 부서별 앱(jabis-dev 등)의 홈 앱 — 공통 진입점 및 메인 대시보드 제공
- UI 컴포넌트는 반드시 packages/ui/에 위치 → @jabis/ui import
- POST + action 패턴으로 CRUD 처리
- ID 형식: prefix + 고유값 (dept-001, user-ceo)
- Zustand 스토어: 상태/액션/getter/다이얼로그 구조화, localStorage 연동
- 새 UI 컴포넌트: packages/ui/src/components/ → src/index.js export → @jabis/ui import
- @jabis/ui 변경 후 개발 서버 재시작 필요
- 빌드 오류 시 dist 삭제 후 재빌드
