# jabis-teamlead

> 팀장 전용 앱 — teamlead 역할 기반 대시보드 (팀원 관리, 결재 처리, 팀원 근태, 시스템 모니터링)

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
jabis-teamlead/
├── pnpm-workspace.yaml
├── packages/              # git submodule (jabis-common)
│   ├── ui/                # @jabis/ui
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   ├── core/              # @jabis/core
│   ├── menu/              # @jabis/menu
│   └── shared-pages/      # @jabis/shared-pages
├── apps/
│   └── teamlead/
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
├── .gitmodules
├── CLAUDE.md
├── Dockerfile
├── bitbucket-pipelines.yml
└── dist → apps/teamlead/dist   # symlink (jabis-maker 미리보기용)
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
- `/teamlead/*` — 팀장 전용 경로

## 환경변수
- `VITE_OAUTH_ENABLED` — OAuth 활성화 여부
- `VITE_OAUTH_CLIENT_ID` — OAuth 클라이언트 ID (jabis 메인과 동일)
- `VITE_OAUTH_CLIENT_SECRET` — OAuth 클라이언트 Secret (jabis 메인과 동일)
- `VITE_OAUTH_REDIRECT_URI` — https://jabis.jinhakapply.com/teamlead/auth/callback
- `VITE_OAUTH_AUTH_SERVER` — https://jabis-cert.jinhakapply.com
- `VITE_GATEWAY_URL` — https://jabis-gateway.jinhakapply.com
- `VITE_KAKAO_MAP_KEY` — 카카오맵 API 키

## Dockerfile
- node:20-alpine multi-stage pnpm 빌드
- jabis-common git clone → packages 빌드 → teamlead 앱 빌드
- serve -s dist -l 3000
- SPA rewrite: `/teamlead/**` → `/teamlead/index.html`

## Bitbucket Pipeline
- master → 자동 deploy (push = 배포)
- jabis-common을 git clone으로 받아옴
- Harbor push → Helm update → Teams 알림

## 역할
- teamlead (팀장) — 메인 역할
- @jabis/menu의 ROLE_URL_MAP에서 `teamlead` → `/teamlead/` 매핑
- 역할 전환 드롭다운으로 다른 역할로 이동 가능

## 팀장 전용 메뉴

### 팀 관리 그룹
- **AI 팀 업무 취합** (Sparkles, `/ai-team-report`) — AI 기반 팀 업무 리포트
- **시스템 모니터링** (Shield, `/monitoring`) — 서버 상태, CPU/메모리/디스크
- **팀원 관리** (Users, `/team`) — 팀원 출근상태, 업무 진행률, 연락처
- **팀원 근태 현황** (Clock, `/team/attendance`) — 팀원 출퇴근 기록, 지각/조퇴/결근 통계
- **대학 문의** (Building2, `/requests`) — 대학 문의 관리

### 공통 메뉴 (buildMenu)
- **업무 관리** (work) — 할일, 문서작성, 목표, 일정, 전자결재, 근태, 카드신청
- **소통** (communication) — 메신저, 메일, 회의실
- **사내 생활** (life) — 보물블로그, 아침식사, 뉴스, 맛집, 추억, 바우처
- **조직/관리** (organization) — 조직도, 인사정보, 사이트
- **기타** (etc) — 기타 유틸리티

## 대시보드 위젯
jabis 메인 앱의 teamlead 위젯 지원 목록:
- projectChart (프로젝트 현황 차트)
- projectList (프로젝트 목록)
- taskKanban (할일 칸반보드)
- githubCommit (개발 활동)
- systemStatus (시스템 상태)
- siteStatus (사이트 운영 상태)

## 핵심 기능: 전자결재 처리

팀장의 주요 업무 중 하나가 전자결재(승인/반려) 처리. `@jabis/shared-pages`의 ApprovalPage를 통해:
- **미결함**: 본인에게 결재 요청이 온 문서 목록 → 승인/반려 처리
- **진행함**: 본인이 기안한 문서 진행 상태 확인
- **결재선**: step_order 기반 순차 결재 (결재/합의/참조)
- **후처리**: 부서별 후처리 완료/반려

## 핵심 기능: 팀원 관리 & 근태

팀장이 소속 팀원을 관리하고 근태 현황을 조회:
- 팀원 상태 (근무중/회의중/외근/휴가) 실시간 확인
- 팀원 출퇴근 기록 조회 (일별/주별/월별)
- 지각/조퇴/결근 통계
- 향후: 근태 관련 결재 처리 (휴가 승인 등)

## jabis 메인 앱과의 관계

현재 jabis 메인 앱(`/teamlead/*`)에 teamlead 페이지가 있음:
- `TeamLeadDashboard.jsx` — 대시보드
- `TeamMembersPage.jsx` — 팀원 관리
- `SystemMonitoringPage.jsx` — 시스템 모니터링

jabis-teamlead 프로젝트 생성 후, 이 기능들을 독립 앱으로 이전하여
jabis 메인 앱의 teamlead 라우트는 `/teamlead/`로 리다이렉트 처리.

---

## 프로젝트 고유 규칙
- jabis-{dept} 부서별 앱 패턴 준수
- UI 컴포넌트는 @jabis/ui import, packages/ 수정 금지
- POST + action 패턴으로 CRUD 처리
- **공통 메뉴는 `@jabis/menu`의 `buildMenu('teamlead', ..., { standalone: true })`로 관리** (하드코딩 금지)
- 팀장 전용 메뉴(팀 관리, AI 리포트, 모니터링)만 App.jsx에서 직접 정의
- basename: `/teamlead`
- 미리보기 base path: `/preview/jabis-teamlead/`
- 운영 base path: `/teamlead/`
