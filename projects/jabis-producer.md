# jabis-producer

> 프로듀서(빌드/배포/인프라 관리) 전용 앱 — jabis.jinhakapply.com/producer 경로로 서빙

---

## 기술 스택
- React 18.2 (JSX)
- Vite 5
- pnpm monorepo
- Zustand
- React Router v6
- Tailwind CSS
- @jabis/ui, @jabis/layout, @jabis/auth, @jabis/core

## 프로젝트 구조
```
jabis-producer/
├── pnpm-workspace.yaml
├── packages/              # @jabis/common (git submodule)
│   ├── ui/                # @jabis/ui
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   └── core/              # @jabis/core
├── apps/
│   └── producer/
│       ├── src/
│       │   ├── App.jsx
│       │   ├── main.jsx
│       │   └── pages/
│       │       ├── DashboardPage.jsx
│       │       ├── MakerPage.jsx          # jabis-maker iframe
│       │       ├── ApiRequestLogPage.jsx   # API 요청 내역
│       │       └── ComingSoonPage.jsx
│       └── tailwind.config.js
└── dist -> apps/producer/dist
```

## 주요 명령어
```bash
pnpm install              # 의존성 설치
pnpm dev                  # 개발 서버 (--filter producer)
pnpm build                # 프로덕션 빌드
pnpm preview              # 빌드 결과 미리보기
```

## 포트
- 개발: 5173 (Vite)
- 프로덕션: 3000 (serve)

## 고유 기능

### JABIS Maker iframe
- jabis-maker (Claude Code 웹 UI)를 iframe으로 임베딩
- 전체 화면 토글, 새 탭 열기 지원
- `VITE_MAKER_URL` 환경변수로 URL 설정

### API 요청 내역
- jabis-api-gateway의 access log 조회
- 필터: 메서드, 경로, 상태코드
- 현재 mock 데이터 사용 (추후 gateway 엔드포인트 연동 예정)

### 간소화된 사이드바
- 대시보드
- JABIS Maker (iframe)
- API 요청 내역
- 공통 메뉴 (업무관리, 소통, 사내생활, 조직/관리, 기타)

## 배포
- Kubernetes (jabis-helm)
- 경로: jabis.jinhakapply.com/producer
- Docker 이미지: harbor-apply.jinhaksa.com/jabis/jabis-producer
