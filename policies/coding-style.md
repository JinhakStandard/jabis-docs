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

## 7. 로깅 규칙

| 환경 | 허용 | 금지 |
|------|------|------|
| 백엔드 (프로덕션) | `logger.info/warn/error` | `console.log/error` |
| 프론트 (프로덕션) | 없음 | `console.log` |
| 프론트 (예외 처리) | `console.warn` (localStorage 실패 등) | - |
| 개발 환경 | 모두 허용 | - |

---

## 8. AI 안티패턴 금지

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

## 9. 작업 기록 (.ai/ 폴더)

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
