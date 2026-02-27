# JABIS 코딩 스타일 정책

> 이 문서는 JABIS 전체 프로젝트에 적용되는 코딩 스타일 정책입니다.
> 프로젝트별 고유 규칙은 각 프로젝트의 CLAUDE.md를 참조하세요.

---

## 1. 언어 규칙

- 코드 주석, 커밋 메시지, 문서, AI 응답: **한국어 우선**
- 변수/함수명: 영어

---

## 2. 네이밍 컨벤션

| 대상 | 규칙 | 예시 |
|------|------|------|
| 컴포넌트 파일/이름 | PascalCase | `DashboardLayout.jsx`, `PageHeader` |
| 함수/변수 | camelCase | `setViewMode`, `handleSubmit` |
| 상수 | UPPER_SNAKE_CASE | `STORAGE_KEY`, `WIDGET_DEFINITIONS` |
| 인터페이스 (TS) | PascalCase + `I` prefix | `IUserInfo`, `ISeoData` |
| Enum/Type (TS) | PascalCase | `UserRole`, `ButtonType` |
| 스토어 파일 | camelCase + Store | `dashboardStore.js` |
| CSS 클래스 | kebab-case | Tailwind 사용 |
| 서비스 파일 (TS) | camelCase + .service | `query.service.ts` |

---

## 3. 프로젝트별 기술 스택

> [프로젝트 맵](../onboarding/project-map.md) 참조

---

## 4. HTTP Method 규칙

> **GET, POST만 사용. PUT, PATCH, DELETE 사용 금지.**

| Method | 용도 |
|--------|------|
| `GET` | 데이터 조회 (쿼리 파라미터로 필터링) |
| `POST` | 생성, 수정, 삭제 (`action` 필드로 동작 구분) |

```javascript
// POST + action 패턴
POST /api/users
body: { action: 'create', data: {...} }
body: { action: 'update', id: '123', data: {...} }
body: { action: 'delete', id: '123' }
```

---

## 5. 필수 규칙

- **ID 컬럼**: 기본 varchar(30), 필요 시 varchar(50)까지 허용 — PostgreSQL native UUID 타입 사용 금지
- **파일 크기**: 최대 400줄 권장, 800줄 절대 상한
- **불변성**: 불변성(immutability) 우선
- **하드코딩 금지**: 환경변수 또는 Vault 시크릿 사용

---

## 6. 금지 사항

| 금지 항목 | 대안 |
|----------|------|
| `console.log` (프로덕션) | 백엔드: `logger` 사용 / 프론트: 삭제 |
| `any` 타입 (TypeScript) | 구체적 타입 정의 |
| 하드코딩된 URL/포트/비밀키 | 환경변수 또는 Vault |
| `git push --force` | 일반 push 사용 |
| 주석 처리된 코드 방치 | 삭제 |
| 파일 전체 재작성 | 부분 수정으로 대체 |

---

## 7. Toast / 알림 사용 규칙

> **@jabis/ui의 `useToast()`는 shadcn/ui와 API가 다르다.** `const { toast } = useToast()` 패턴은 사용 금지.

### 올바른 사용법

```jsx
import { useToast } from '@jabis/ui'

const { showSuccess, showError, showInfo, showWarning } = useToast()

// 성공 알림 (우측 하단 토스트)
showSuccess('저장 완료', '문서가 저장되었습니다.')

// 에러 알림 (화면 중앙 모달, 10초 후 자동 닫힘)
showError('저장 실패', '문서 저장 중 오류가 발생했습니다.')

// 정보 알림 (우측 하단 토스트)
showInfo('안내', '이미 최신 버전입니다.')

// 경고 알림 (우측 하단 토스트)
showWarning('주의', '저장하지 않은 변경사항이 있습니다.')
```

### 함수 시그니처

```typescript
showSuccess(title: string, message?: string, duration?: number): number
showError(title: string, message?: string, duration?: number): number
showInfo(title: string, message?: string, duration?: number): number
showWarning(title: string, message?: string, duration?: number): number
addToast({ type, title, message, duration }): number  // 저수준 API
```

### 금지 패턴

```jsx
// ❌ 금지 — useToast()에 toast 프로퍼티 없음 → undefined → "X is not a function" 런타임 에러
const { toast } = useToast()
toast({ title: '완료', description: '...', variant: 'destructive' })

// ❌ 금지 — shadcn/ui 스타일 객체 인자
showSuccess({ title: '완료', description: '...' })
```

### 타입별 동작 차이

| 메서드 | 표시 위치 | 자동 닫힘 | 용도 |
|--------|----------|----------|------|
| `showInfo` | 우측 하단 토스트 | 5초 | 일반 안내 |
| `showWarning` | 우측 하단 토스트 | 5초 | 주의/경고 |
| `showSuccess` | 우측 하단 토스트 | 5초 | 성공 피드백 |
| `showError` | 화면 중앙 모달 | 10초 | 오류 알림 |

---

## 8. 프론트엔드 API 클라이언트 규칙

> **shared-pages 내 API 호출은 반드시 글로벌 API 클라이언트(`apiClient.js`)를 통해야 한다.**
> 개별 store/모듈마다 자체 API 클라이언트를 만드는 것은 금지.

### 배경

`@jabis/shared-pages`는 라이브러리이므로 게이트웨이 URL이나 인증 토큰을 직접 알 수 없다.
소비 앱(jabis, jabis-finance 등)이 부팅 시 `setSharedApiClient()`를 **1회** 호출하여 API 함수를 주입하면,
shared-pages 내 모든 store와 페이지가 해당 함수를 사용한다.

### 구조

```
shared-pages/src/stores/apiClient.js    ← 유일한 API 클라이언트 (setSharedApiClient, apiGet, apiPost, apiUpload, getApiBaseUrl)
shared-pages/src/stores/approvalStore.js  ← import { apiGet, apiPost } from './apiClient'
shared-pages/src/stores/cardApi.js        ← import { apiGet, apiPost, apiUpload } from './apiClient'
shared-pages/src/stores/organizationStore.js ← import { apiGet, apiPost } from './apiClient'
```

### 소비 프로젝트 main.jsx — 올바른 패턴

```jsx
import { setSharedApiClient } from '@jabis/shared-pages'

// 앱 부팅 시 1회 호출 — 이것만으로 모든 shared-pages API가 게이트웨이를 향함
setSharedApiClient({ apiGet: makeApiGet, apiPost: makeApiPost, apiUpload: makeApiUpload, apiBaseUrl: GATEWAY_BASE })

// <a href>나 window.open 등 비-fetch 용도에서 게이트웨이 URL 사용:
// import { getApiBaseUrl } from '@jabis/shared-pages'
// <a href={`${getApiBaseUrl()}/api/storage/download?...`}>다운로드</a>
```

### 소비 프로젝트 로컬 API 모듈 — 올바른 패턴

소비 프로젝트에 자체 API 모듈(예: `financeApi.js`)이 있는 경우에도
`@jabis/shared-pages`에서 `apiGet`/`apiPost`/`apiUpload`를 import하여 사용한다.
자체 `let client` + `setXxxApiClient` 패턴을 만들지 않는다.

```jsx
// ✅ 올바른 패턴
import { apiGet, apiPost, apiUpload } from '@jabis/shared-pages'

export const financeApi = {
  getMe: () => apiGet('/api/finance/me'),
  getCards: () => apiGet('/api/finance/cards'),
  uploadFile: (formData) => apiUpload('/api/storage/upload', formData),
}

// ❌ 금지 패턴 — 개별 setter 생성
let client = { apiGet: null, apiPost: null }
export function setFinanceApiClient(c) { client = c }

// ❌ 금지 패턴 — fetch 직접 호출 (multipart 포함)
const res = await fetch('/api/storage/upload', { method: 'POST', body: formData })
```

### 금지 패턴 요약

| 금지 패턴 | 대안 |
|----------|------|
| store마다 `let _apiGet`, `setXxxApiClient` 생성 | `import { apiGet } from './apiClient'` |
| 소비 프로젝트에서 `setXxxApiClient` 여러 개 호출 | `setSharedApiClient()` 1회 호출 |
| 로컬 API 모듈에서 자체 client 패턴 | `import { apiGet, apiPost, apiUpload } from '@jabis/shared-pages'` |
| `fetch(path)` 직접 호출 (상대 경로) | `apiGet(path)` / `apiUpload(path, formData)` 사용 — 게이트웨이 URL이 자동 적용 |
| `fetch` + `FormData`로 파일 업로드 | `apiUpload(path, formData)` 사용 — multipart도 게이트웨이 경유 필수 |

### 왜 이 규칙이 필요한가

- **상대 경로 fetch 문제**: `fetch('/api/finance/me')`는 현재 호스트(`jabis.jinhakapply.com`)로 요청하여 SPA fallback HTML을 반환받는다. 게이트웨이(`jabis-gateway.jinhakapply.com`)로 보내야 JSON을 받을 수 있다.
- **N개 setter 문제**: API 모듈이 추가될 때마다 소비 앱 5개의 main.jsx에 setter를 추가해야 하며, 하나라도 빠뜨리면 운영에서 API 실패가 발생한다.
- **단일 주입점**: `setSharedApiClient()` 1회 호출로 모든 store/페이지에 일괄 적용되므로 누락이 불가능하다.

---

## 9. 로깅 규칙

| 환경 | 허용 | 금지 |
|------|------|------|
| 백엔드 (프로덕션) | `logger.info/warn/error` | `console.log/error` |
| 프론트 (프로덕션) | 없음 | `console.log` |
| 프론트 (예외 처리) | `console.warn` (localStorage 실패 등) | - |
| 개발 환경 | 모두 허용 | - |

---

## 10. AI 안티패턴 금지

Claude는 다음 안티패턴을 감지하면 **즉시 경고하고 대안을 제시**합니다.

| 분류 | 안티패턴 | Claude 대응 |
|------|---------|------------|
| 위험한 요청 | `push --force`, `reset --hard`, `--no-verify` 등 위험 명령 | 실행 거부 + 안전한 대안 제시 |
| 위험한 요청 | 프로덕션 DB 직접 조작 요청 | 거부 + 스테이징 환경 사용 안내 |
| 민감 정보 노출 | 프롬프트에 비밀번호, API Key, 개인정보 포함 | 경고 + 마스킹/가명화 요청 |
| 민감 정보 노출 | `.env` 파일 내용 공유 요청 | 거부 + Vault 사용 안내 |
| 품질 저하 | "전체를 처음부터 다시 작성해줘" | 부분 수정 제안 |
| 품질 저하 | 순차 의존성 있는 5개 이상 기능 동시 요청 | 단계별 분할 제안 (독립 작업은 Agent Teams 활용 가능) |

**3중 방어 구조:**
1. **CLAUDE.md 규칙** — 자연어 감지로 안티패턴 경고 + 대안 제시
2. **settings.json deny 규칙** — `--no-verify`, `push --force` 등 물리적 차단 (우회 불가)
3. **hooks 보조 경고** — 파일 수정 전 보안 경고 주입

> 상세 내용은 [AI 협업 정책](ai-collaboration.md) 참조

---

## 11. 작업 기록 (.ai/ 폴더)

> **Agent Memory 활용**: Claude Code는 프로젝트별 메모리를 자동으로 기록/회상합니다. `.ai/` 파일은 Agent Memory의 보조 수단으로, 팀원 간 공유와 Git 추적이 필요한 핵심 사항만 기록합니다.

모든 프로젝트는 `.ai/` 폴더에 작업 이력을 관리합니다:

```
.ai/
├── SESSION_LOG.md      # 세션별 작업 기록 (요약 위주)
├── CURRENT_SPRINT.md   # 현재 진행/대기 작업 현황
├── DECISIONS.md        # 기술 의사결정 기록 (ADR)
├── ARCHITECTURE.md     # 시스템 구조 문서
└── CONVENTIONS.md      # 코딩 컨벤션
```

### 세션 시작 시
1. `.ai/CURRENT_SPRINT.md` 읽어 진행 중인 작업 파악
2. `.ai/SESSION_LOG.md`에서 최근 작업 확인 (필요 시)
3. Agent Memory가 이전 세션 컨텍스트를 자동으로 로드

### 작업 완료 후 (필수)
1. `.ai/CURRENT_SPRINT.md` 진행 상태 업데이트
2. `.ai/SESSION_LOG.md`에 요약 기록 (핵심 변경사항 위주)
3. 중요 기술 결정 시 `.ai/DECISIONS.md` 업데이트

### SESSION_LOG.md 기록 형식
```markdown
## YYYY-MM-DD

### 세션 요약
- 작업 1 설명
- 작업 2 설명

### 주요 변경
- `파일경로` - 변경 내용

### 커밋
- `해시` 커밋 메시지
```

> 상세 세션 관리 규칙은 [AI 협업 정책](ai-collaboration.md) 참조

---

## 8. 사용자 목록 기준 테이블 정책

**전체 직원 목록이 필요한 곳에서는 반드시 `organization.employees`를 기준 테이블로 사용한다.**

| 기준 | 설명 |
|------|------|
| `organization.employees` | **전체 직원** — 인사 시스템에서 동기화된 마스터 테이블 |
| `organization.users` | **로그인 이력이 있는 사용자만** — OAuth 최초 로그인 시 생성 |

### 올바른 패턴 (employees 기준)

```sql
FROM organization.employees e
LEFT JOIN organization.users u ON u.employee_id = e.id
LEFT JOIN organization.departments d ON d.id = e.department_id
WHERE e.status = 'active'
```

### 잘못된 패턴 (users 기준) — 사용 금지

```sql
-- ❌ 로그인하지 않은 직원이 누락됨
FROM organization.users u
LEFT JOIN organization.employees e ON e.id = u.employee_id
```

### 적용 범위

- 법인 매핑 관리 (CardSettingsPage — 전체 직원 대상 법인 매핑)
- 사용자 선택/검색 UI (직원 목록, 부서 트리 등)
- 관리자 설정 페이지 (역할 부여, 권한 설정 등)

### 예외

- 현재 로그인 사용자 정보 조회 (`/me` 등) — `users` 테이블에서 JWT 기반 조회
- 로그인 세션 관리 — `users` 테이블 직접 사용
