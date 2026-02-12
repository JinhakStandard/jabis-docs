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
│       │   ├── pages/
│       │   │   ├── DashboardPage.jsx
│       │   │   ├── MakerPage.jsx          # jabis-maker iframe
│       │   │   ├── ApiRequestPage.jsx     # 일반 API 요청 (Dev Tasks)
│       │   │   ├── ApiBuilderPage.jsx     # 개발자 간이 API 빌더
│       │   │   ├── ApiRequestLogPage.jsx  # API 로그 모니터링
│       │   │   └── ComingSoonPage.jsx
│       │   ├── stores/
│       │   │   ├── apiRequestStore.js     # API 요청 상태 관리
│       │   │   └── apiBuilderStore.js     # API 빌더 상태 관리
│       │   └── services/
│       │       ├── gatewayApi.js           # Gateway 통신 (axios)
│       │       └── devTasksApi.js          # Dev Tasks API 서비스
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

## 환경변수
| 변수 | 설명 | 기본값 |
|------|------|--------|
| `VITE_OAUTH_ENABLED` | OAuth 활성화 | `false` |
| `VITE_MAKER_URL` | JABIS Maker URL | - |
| `VITE_API_GATEWAY_URL` | API Gateway URL | `https://jabis-gateway.jinhakapply.com` |
| `VITE_OAUTH_AUTH_SERVER` | 인증 서버 URL | `http://localhost:3000` |

## 고유 기능

### JABIS Maker iframe
- jabis-maker (Claude Code 웹 UI)를 iframe으로 임베딩
- 전체 화면 토글, 새 탭 열기 지원
- `VITE_MAKER_URL` 환경변수로 URL 설정

### API 관리 (3개 페이지)

#### 1. API 요청하기 (`/api-request`)
- **대상**: 일반 작업자 (비개발 직군)
- 텍스트 기반 API 요청 제출 폼 (제목, 설명, 요청 내용 마크다운)
- 우선순위 설정 (보통/높음/긴급)
- Dev Tasks API 연동 (`POST /api/dev-tasks`)
- 요청 목록 + 상태 추적 (pending → in_progress → completed → verified)

#### 2. API 빌더 (`/api-builder`)
- **대상**: 개발자 (producer/developer 역할)
- SQL 쿼리 기반 API 엔드포인트 직접 생성
- 엔드포인트 CRUD: name, method(GET/POST), pathPattern, SQL query
- 파라미터 정의 UI (name, type, required, default, description)
- 인증/권한 설정 (authRequired, requiredScopes)
- 쿼리 테스트 실행
- Gateway Endpoint Management API 연동 (`/api/endpoints`)

#### 3. API 로그 (`/api-logs`)
- API Gateway 접근 로그 모니터링
- 필터: 메서드, 경로
- Gateway `/api/access-logs` 연동 (미연동 시 데모 데이터 fallback)

### 사이드바 구조
- 대시보드
- JABIS Maker (iframe)
- API 관리 (그룹)
  - API 요청하기
  - API 빌더
  - API 로그
- 공통 메뉴 (업무관리, 소통, 사내생활, 조직/관리, 기타)

## Gateway 연동
- Vite 프록시: `/gateway-api` → `VITE_API_GATEWAY_URL`
- Bearer 토큰 자동 주입 (axios 인터셉터, @jabis/auth 토큰)
- API: 엔드포인트 CRUD, Dev Tasks, Access Logs

## 배포
- Kubernetes (jabis-helm)
- 경로: jabis.jinhakapply.com/producer
- Docker 이미지: harbor-apply.jinhaksa.com/jabis/jabis-producer
