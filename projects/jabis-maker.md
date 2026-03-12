# jabis-maker

> Claude Code 원격 실행 플랫폼 — 세션 관리, Ultra 실행, NightExecutor(배치 자동화), 프로젝트 미리보기

---

## 기술 스택
- Fastify 4.26 + TypeScript (strict mode)
- Node.js 20+
- better-sqlite3 (사용자/세션/대화 로그)
- jose (JWT HS256)
- Winston (로깅)
- Zod + js-yaml (설정 검증/로딩)
- nodemailer (이메일 알림)
- React 18 + Vite 5 (관리자 프론트엔드, client/)
- PM2 (프로세스 관리)

## 프로젝트 구조
```
jabis-maker/
├── src/
│   ├── server.ts              # 진입점 (포트 3200)
│   ├── app.ts                 # Fastify 앱 빌더
│   ├── config/                # YAML 설정 로더 + Zod 스키마
│   ├── core/                  # logger, lifecycle, errors
│   ├── middleware/             # JWT 인증, 역할 체크, 에러 핸들러
│   ├── routes/
│   │   ├── auth.ts            # POST /api/auth (login/setup/refresh)
│   │   ├── sessions.ts        # POST /api/sessions (create/kill)
│   │   ├── ws.ts              # WebSocket 핸들러 (GET /ws)
│   │   ├── preview.ts         # GET /preview/:projectName/* (SPA 서빙)
│   │   ├── projects.ts        # GET /api/projects
│   │   ├── admin.ts           # GET /api/admin/* (관리자 전용)
│   │   ├── deploy.ts           # 배포 대시보드 API (status/stream, promote, sync)
│   │   └── users.ts           # POST /api/users (사용자 관리)
│   ├── services/
│   │   ├── sessionManager.ts  # 세션 생명주기 관리
│   │   ├── claudeProcess.ts   # Claude CLI 프로세스 실행/스트리밍
│   │   ├── ultraExecutor.ts   # Ultra 실행 모드 (다단계 프롬프트)
│   │   ├── nightExecutor.ts   # NightExecutor (배치 자동화)
│   │   ├── nightPersistence.ts # 나이트모드 SQLite 저장
│   │   ├── projectScanner.ts  # 프로젝트 스캐닝
│   │   ├── deployManager.ts   # 배포 상태 조회/promote/sync + 인메모리 캐시
│   │   ├── contextResolver.ts # jabis-docs 컨텍스트 로딩
│   │   ├── emailNotifier.ts   # 이메일 알림
│   │   └── teamsNotifier.ts   # Teams 웹훅 알림
│   ├── auth/                  # 사용자 DB, JWT, 비밀번호, 토큰 관리
│   └── types/                 # TypeScript 타입 정의
├── client/                    # React 관리자 프론트엔드
│   └── src/
│       ├── components/
│       ├── hooks/             # useWebSocket, useProjects
│       └── stores/            # Zustand
├── config/
│   ├── default.yaml           # 기본 설정
│   └── development.yaml       # 개발 오버라이드
├── data/
│   └── jabis-maker.db         # SQLite DB
├── logs/                      # 로그 파일
├── pm2.config.cjs             # PM2 설정
├── package.json
└── tsconfig.json
```

## 주요 명령어
```bash
npm install               # 의존성 설치
npm run build             # TypeScript 컴파일
npm run dev               # 개발 실행 (tsx watch)
npm start                 # 프로덕션 실행 (dist/server.js)
npm test                  # Vitest 테스트
npm run pm2:start         # PM2 시작
npm run pm2:restart       # 빌드 + PM2 재시작
npm run pm2:logs          # 최근 로그 50줄
npm run pm2:status        # 상태 확인
```

## 포트
- 3200 (로컬)
- ngrok: `https://jabis-maker.ngrok-free.dev`

## 환경변수
| 변수 | 설명 | 설정 위치 |
|------|------|----------|
| `JABIS_AUTH__JWT__SECRET` | JWT 서명 시크릿 | pm2.config.cjs |
| `JABIS_EMAIL__PASSWORD` | SMTP 비밀번호 | .env |
| `JABIS_TEAMS__WEBHOOK_URL` | Teams 웹훅 URL | pm2.config.cjs |
| `NODE_ENV` | 실행 환경 | pm2.config.cjs (`production`) |

## 데이터베이스 (SQLite)

경로: `./data/jabis-maker.db`

| 테이블 | 설명 |
|--------|------|
| `users` | 사용자 (id, username, display_name, password_hash, role) |
| `refresh_tokens` | JWT 리프레시 토큰 (자동 회전) |
| `conversation_logs` | 대화 기록 (프롬프트, 응답 요약, 토큰 수) |
| `user_projects` | 사용자별 프로젝트 접근 권한 |
| `deploy_projects` | 배포 대시보드 대상 프로젝트 (name TEXT PK) |

## 사용자 역할

| 역할 | 권한 |
|------|------|
| `admin` | 전체 접근 — 모든 세션 관리, 사용자 관리, 나이트모드 |
| `user` | 자신의 세션 생성/실행, 할당된 프로젝트만 접근 |
| `viewer` | 읽기 전용 — 세션 관찰만 가능, 프롬프트/abort 불가 |

## 핵심 기능

### 1. 원격 Claude Code 세션
- WebSocket으로 `claude` CLI 프로세스를 원격 실행
- stream-json 출력을 실시간 스트리밍
- `--resume`으로 이전 대화 이어서 진행
- 다중 클라이언트가 같은 세션에 attach 가능 (관전 모드)

### 2. Ultra 실행 모드
- 다단계 프롬프트 + 체크리스트 검증
- 자동 재시도 (최대 3회)
- jabis-docs 컨텍스트 자동 주입

### 3. NightExecutor (나이트모드)
- 복수 태스크를 의존성 순서에 따라 순차 실행
- 태스크별 quality gate (npm test, build 등)
- 완료 후 자동 git commit + push
- 이메일/Teams 알림
- SQLite 영속화 (크래시 복구)
- 상세 정책: `policies/night-mode.md`

### 4. 프로젝트 미리보기
- `/preview/:projectName/`으로 빌드된 프로젝트 서빙
- SPA 폴백 (index.html 리라이트)
- `<base>` 태그 자동 주입
- `build:preview` 빌드 후 자동으로 접근 가능

## REST API 주요 엔드포인트

| Method | 경로 | 인증 | 설명 |
|--------|------|------|------|
| POST | `/api/auth` | - | 로그인/회원가입/토큰 갱신 (`action` 기반) |
| GET | `/api/projects` | JWT | 접근 가능한 프로젝트 목록 |
| POST | `/api/sessions` | JWT | 세션 생성/종료 (`action: create/kill`) |
| GET | `/api/conversations` | JWT | 대화 기록 조회 |
| POST | `/api/users` | Admin | 사용자 CRUD |
| GET | `/api/admin/online` | Admin | 온라인 사용자 목록 |
| GET | `/api/admin/logs` | Admin | 대화 로그 필터링 조회 |
| GET | `/api/deploy/status` | Admin | 전체 배포 상태 조회 (일괄) |
| GET | `/api/deploy/status/stream` | Admin | SSE 스트리밍 배포 상태 (`?refresh=true`로 캐시 무효화) |
| GET | `/api/deploy/status/:name` | Admin | 특정 프로젝트 배포 상태 |
| POST | `/api/deploy/promote` | Admin | alpha → main 운영 배포 (`force` 옵션) |
| POST | `/api/deploy/sync` | Admin | main → alpha 역동기화 (`force` 옵션) |
| GET | `/api/deploy/projects` | Admin | 배포 대상 프로젝트 목록 조회 |
| POST | `/api/deploy/projects` | Admin | 배포 대상 프로젝트 추가/삭제 (action: add/remove) |
| GET | `/preview/:project/*` | - | 프로젝트 미리보기 서빙 |

## 배포 대시보드 (Deploy Dashboard)

Alpha → 운영 배포를 관리하는 대시보드. 프론트엔드는 `package/src/components/DeployDashboard.tsx`.

### 아키텍처

- **DB 기반 프로젝트 목록**: `deploy_projects` 테이블에 저장 (초기 시드: 9개 프로젝트, 관리 UI로 추가/삭제)
- **SSE 스트리밍**: `/api/deploy/status/stream` — 프로젝트별 상태를 완료되는 즉시 전송
- **인메모리 캐시**: 상태를 서버 메모리에 캐시 — 재방문 시 캐시 즉시 반환 + 백그라운드 갱신
- **git fetch --prune**: stale remote ref 자동 정리 (삭제된 브랜치 참조 방지)
- **getMainBranch**: `git rev-parse --verify`로 정확한 main/master 판별

### 대상 프로젝트 관리

- 프로젝트 목록은 SQLite `deploy_projects` 테이블에 저장
- 초기 시드: jabis, jabis-api-gateway, jabis-dev, jabis-finance, jabis-hr, jabis-producer, jabis-storage, jabis-sysadmin, jabis-teamlead
- 관리 UI: 배포 대시보드 상단 "관리" 버튼 → 프로젝트 추가/삭제
- API: `GET /api/deploy/projects`, `POST /api/deploy/projects` (action: add/remove)

### SSE 이벤트 플로우

| 이벤트 | 데이터 | 설명 |
|--------|--------|------|
| `projects` | `string[]` | 프로젝트 이름 목록 (스켈레톤 렌더링용) |
| `status` | `DeployStatus` | 개별 프로젝트 상태 (도착 즉시) |
| `skip` | `{ projectName }` | alpha 없는 프로젝트 스킵 알림 |
| `cached` | `{}` | 캐시 데이터 전송 완료 (이후 fresh 갱신 시작) |
| `done` | `{}` | 전체 완료 |

### 머지 충돌 처리

- `promote`/`sync` 시 머지 충돌 감지 → `conflictFiles` 배열 반환
- 프론트엔드에서 충돌 파일 목록 표시 + "강제 머지" 버튼 제공
- 강제 머지: `git merge -X theirs` (소스 브랜치 우선 자동 해결)

### DeployStatus 타입

```typescript
interface DeployStatus {
  projectName: string;
  hasAlpha: boolean;
  hasMaster: boolean;
  alphaAhead: number;    // alpha에만 있는 커밋 수
  mainAhead: number;     // main에만 있는 커밋 수 (핫픽스 등)
  alphaCommits: CommitInfo[];
  mainCommits: CommitInfo[];
  lastAlphaCommit?: CommitInfo;
  lastMainCommit?: CommitInfo;
}
```

## WebSocket 프로토콜

연결: `ws://localhost:3200/ws`

### 클라이언트 → 서버
| 타입 | 설명 |
|------|------|
| `auth` | JWT 토큰 인증 |
| `attach` / `detach` | 세션 연결/해제 |
| `prompt` | Claude에 프롬프트 전송 |
| `abort` | 세션 종료 |
| `abort_prompt` | 현재 프롬프트만 중단 |
| `interrupt` | 현재 작업 중단 + 새 프롬프트 |
| `ultra_execute` / `ultra_abort` | Ultra 모드 실행/중단 |
| `night_start` / `night_abort` / `night_status` | 나이트모드 시작/중단/상태 |
| `answer_question` | Claude의 질문에 응답 |

### 서버 → 클라이언트
| 타입 | 설명 |
|------|------|
| `auth_ok` / `auth_fail` | 인증 결과 |
| `stream` | Claude 출력 스트림 |
| `session_ended` | 세션 종료 |
| `night_progress` | 나이트모드 진행 상태 |
| `night_task_start` / `night_task_complete` | 태스크 시작/완료 |
| `night_complete` | 나이트모드 전체 완료 |
| `night_error` | 태스크 에러 |
| `error` | 일반 에러 |

## 설정 파일 (config/default.yaml)

주요 설정 항목:

| 항목 | 기본값 | 설명 |
|------|--------|------|
| `server.port` | 3200 | 서버 포트 |
| `claude.model` | opus | Claude 모델 |
| `claude.effort` | high | 실행 노력 수준 |
| `claude.maxTurns` | 80 | 최대 턴 수 |
| `sessions.maxConcurrent` | 5 | 동시 세션 수 |
| `sessions.timeoutMs` | 86400000 | 세션 타임아웃 (24시간) |
| `nightMode.maxTaskRetries` | 2 | 태스크 실패 시 재시도 횟수 |
| `nightMode.maxTotalDurationMs` | 36000000 | 나이트모드 최대 시간 (10시간) |
| `nightMode.autoPush` | true | 완료 후 자동 git push |
| `nightMode.interTaskCooldownMs` | 5000 | 태스크 간 대기 시간 |

## 배포
- PM2 + 로컬 실행 (K3S 배포 대상 아님 — 로컬 개발 도구)
- ngrok으로 외부 접근: `jabis-maker.ngrok-free.dev`
- 프론트엔드 미리보기도 ngrok 경유로 접근 가능

## 관련 문서

| 문서 | 설명 |
|------|------|
| `policies/night-mode.md` | 나이트모드 실행 정책 (프로그래밍 방식 제출 포함) |
| `policies/concurrent-sessions.md` | 동시 작업 세션 정책 (세션 디렉토리 격리, 머지 충돌 처리) |
| `policies/deployment.md` | 배포 정책 (push=배포, 30초 간격) |
| `policies/staging.md` | Alpha 환경 정책 (배포 대시보드의 배포 대상 환경) |
| `projects/jabis-night-builder.md` | jabis-night-builder (별도 시스템 — 폴링 기반 자동 빌더) |
