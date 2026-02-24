# jabis-sysadmin

> 시스템 관리자(superadmin) 전용 앱 — 사용자 역할/권한 관리, 시스템 설정

---

## 기본 정보

| 항목 | 값 |
|------|-----|
| 역할 | superadmin (시스템 관리자) |
| 기술 스택 | React 18, Vite 5, Zustand, Tailwind CSS |
| 패키지 매니저 | pnpm |
| base path | `/superadmin/` (운영), `/preview/jabis-sysadmin/` (미리보기) |
| 공유 패키지 | @jabis/ui, @jabis/layout, @jabis/auth, @jabis/core, @jabis/menu, @jabis/shared-pages |
| submodule | packages/ → jabis-common |
| 백엔드 API | jabis-api-gateway `/api/sysadmin/*` |

## 빌드 명령

```bash
pnpm install          # 의존성 설치
pnpm dev              # 개발 서버
pnpm build            # 운영 빌드
pnpm build:preview    # 미리보기 빌드
```

## 프로젝트 구조

```
jabis-sysadmin/
├── pnpm-workspace.yaml
├── packages/              # git submodule (jabis-common)
│   ├── ui/                # @jabis/ui
│   ├── layout/            # @jabis/layout
│   ├── auth/              # @jabis/auth
│   ├── core/              # @jabis/core
│   └── menu/              # @jabis/menu
├── apps/
│   └── sysadmin/
│       ├── src/
│       │   ├── App.jsx
│       │   ├── main.jsx
│       │   ├── pages/
│       │   │   ├── DashboardPage.jsx
│       │   │   ├── RoleManagementPage.jsx     # 역할 관리 (P0, 완료)
│       │   │   ├── UserManagementPage.jsx     # 사용자 관리 (P1)
│       │   │   ├── SystemSettingsPage.jsx     # 시스템 설정 (P2)
│       │   │   └── ComingSoonPage.jsx
│       │   ├── lib/
│       │   │   └── sysadminApi.js             # API 클라이언트
│       │   └── stores/
│       ├── .env.local
│       ├── .env.preview
│       ├── .env.production
│       ├── vite.config.js
│       ├── tailwind.config.js
│       └── package.json
├── Dockerfile
├── bitbucket-pipelines.yml
├── CLAUDE.md
└── dist -> apps/sysadmin/dist     # 심볼릭 링크 (jabis-maker 미리보기용)
```

## 기능 우선순위

### P0: 권한 관리 (MVP — 최초 배포 대상)

역할(role) 부여/회수 관리. `organization.user_roles` 테이블 CRUD.

| 기능 | 설명 |
|------|------|
| 역할 부여 | 사용자에게 역할 부여 (INSERT into organization.user_roles) |
| 역할 회수 | 사용자의 역할 제거 (DELETE from organization.user_roles) |
| 역할 현황 조회 | 역할별 사용자 목록, 사용자별 역할 목록 |
| 역할 목록 | 17개 역할 표시 (@jabis/menu availableRoles 기반) |
| 부여 이력 | granted_by, granted_at 기록 조회 |

**DB 테이블**: `organization.user_roles`
```sql
CREATE TABLE organization.user_roles (
  employee_id UUID NOT NULL REFERENCES organization.employees(id),
  role TEXT NOT NULL,           -- 역할 ID (portal, developer, finance, hr 등)
  granted_by TEXT,              -- 부여자 (user_id 또는 'system')
  granted_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (employee_id, role)
);
```

> **employee_id 기반**: OAuth 로그인 없이도 직원에게 역할 부여 가능 (2024-02 마이그레이션 완료)

**17개 역할 (availableRoles)**:
portal, developer, operation, sales, design, planner, marketer, finance, hr, teamlead, depthead, executive, producer, aiadmin, ceo, admin, superadmin

**주의**: 역할 부여/회수는 `organization.user_roles` 레코드만 변경. 실제 권한 적용은 jabis-cert JWT에 roles 배열이 포함되므로, 다음 로그인(또는 세션 갱신) 시 반영됨.

### P1: 사용자 관리

`organization.users`와 `organization.employees` 조회 + 역할 현황 표시.

| 기능 | 설명 |
|------|------|
| 사용자 목록 | organization.users 전체 목록 (이름, 이메일, 부서) |
| 역할 현황 | 사용자별 보유 역할 뱃지 표시 |
| 사용자 검색 | 이름, 이메일, 부서별 필터 |
| 사용자 상세 | 선택 시 보유 역할 + 부여 이력 표시 |

**참조 테이블**:
- `organization.users` — id, email, display_name, employee_id
- `organization.employees` — id, name, email, department_id
- `organization.departments` — id, name

### P2: 시스템 설정

시스템 수준 설정 관리 (향후 확장).

| 기능 | 설명 |
|------|------|
| 세션 정책 | 세션 만료 시간, 최대 연장 시간 조회 (읽기 전용) |
| 역할 설명 관리 | 각 역할의 설명 문구 관리 |
| 감사 로그 | cert.audit_logs 최근 이벤트 조회 |

### P3: API 키 관리 (향후)

등록된 OAuth 클라이언트 및 API 키 관리 (cert.registered_clients).

## 메뉴 구조

### 시스템 관리 전용 메뉴

| 메뉴 | 경로 | 우선순위 | 상태 |
|------|------|----------|------|
| 대시보드 | /dashboard | - | 완료 |
| 역할 관리 | /roles | P0 | 완료 |
| 사용자 관리 | /users | P1 | 개발 예정 |
| 시스템 설정 | /settings | P2 | 개발 예정 |

### 공통 메뉴 (@jabis/menu)
- 업무 관리: 할일, 문서작성, 전자결재, 일정, 법인카드 신청
- 소통: 메신저, 메일, 회의실 예약
- 사내 생활: 트래져 블로그, 조식, 뉴스, 맛집, 추억 공유
- 조직/관리: 조직도
- 기타: 회계 전표, 사내 사이트

**공통 메뉴는 `buildMenu('superadmin', ..., { standalone: true })`로 관리** (하드코딩 금지)

## DashboardLayout 설정

```javascript
headerConfig: { title: 'JABIS', subtitle: '시스템 관리' }
menuStateKey: 'sysadmin'
userRole: 'superadmin'
roleBadgeLabel: '시스템 관리자'
```

## 백엔드 API (jabis-api-gateway)

### 라우트: `/api/sysadmin/*`

**인증**: JWT 필수, `roles` 배열에 `superadmin` 포함 필수

#### P0 — 역할 관리 API

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/sysadmin/roles` | 전체 역할 목록 + 역할별 사용자 수 |
| GET | `/api/sysadmin/roles/:role/users` | 특정 역할의 사용자 목록 |
| GET | `/api/sysadmin/users/:userId/roles` | 특정 사용자의 역할 목록 |
| POST | `/api/sysadmin/roles` | action: grant — 역할 부여 |
| POST | `/api/sysadmin/roles` | action: revoke — 역할 회수 |

**POST body 예시**:
```json
// 역할 부여
{ "action": "grant", "data": { "employeeId": "uuid-...", "role": "finance" } }

// 역할 회수
{ "action": "revoke", "data": { "employeeId": "uuid-...", "role": "finance" } }
```

#### P1 — 사용자 조회 API

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/sysadmin/users` | 사용자 목록 (organization.users + employees + roles) |
| GET | `/api/sysadmin/users/:userId` | 사용자 상세 (역할 + 부여 이력) |

## 환경 설정

| 환경 | OAuth | Gateway URL |
|------|-------|-------------|
| .env.local | 비활성 | localhost:3000 |
| .env.preview | 비활성 (DEV_TOKEN) | jabis-gateway.jinhakapply.com |
| .env.production | 활성 (client_id 미발급) | jabis-gateway.jinhakapply.com |

## ROLE_URL_MAP 변경 필요

`@jabis/menu`의 `roles.js`에서 superadmin을 외부 앱으로 등록:

```javascript
// 현재: superadmin → /superadmin (jabis 메인 내부 라우팅)
// 변경: superadmin → /superadmin/ (독립 앱으로 외부 라우팅)
ROLE_URL_MAP: {
  ...기존,
  superadmin: '/superadmin/',
}
```

## Dockerfile

- node:20-alpine multi-stage pnpm 빌드
- jabis-common git clone → packages 빌드 → sysadmin 빌드
- `dist/superadmin/` 하위에 빌드 결과 복사 (Vite base path `/superadmin/` 대응)
- serve.json: `/superadmin/**` → `/superadmin/index.html` (SPA rewrite)
- serve -s dist -l 3000

### 구현 완료 기능 (역할 관리)

- 17개 역할 카드 뷰 (사용자 수 표시)
- 부서별 트리 구조 직원 선택 (검색 + 자동 펼침)
- 부서 단위 일괄 역할 부여 ("전체 부여" 버튼)
- 역할 사용자 목록 부서별 그룹핑
- 역할 부여/회수 + 토스트 알림

## 배포 상태

- [x] Bitbucket 리포 생성
- [x] jabis-api-gateway에 `/api/sysadmin/*` 라우트 추가
- [x] ROLE_URL_MAP에 superadmin 외부 앱 등록
- [ ] jabis-cert OAuth client_id 발급
- [ ] jabis-helm ingress/values 설정
- [ ] .env.production에 client_id 반영

## 프로젝트 고유 규칙

- jabis-{dept} 부서별 앱 패턴 준수
- UI 컴포넌트는 @jabis/ui import, packages/ 수정 금지
- POST + action 패턴으로 CRUD 처리
- 공통 메뉴는 `buildMenu('superadmin', ..., { standalone: true })` 사용 (하드코딩 금지)
- superadmin 전용 메뉴(권한 관리, 사용자 관리, 시스템 설정)만 App.jsx에서 직접 정의
- basename: `/superadmin`
- 미리보기 base path: `/preview/jabis-sysadmin/`
- 운영 base path: `/superadmin/`
- **superadmin 역할이 없는 사용자는 접근 불가** — JWT roles 체크 필수
