# Dev Tasks API — 외부 프로젝트 개발 작업 요청 가이드

> **Gateway 주소**: `http://localhost:3100` (로컬) / `https://jabis-gateway.jinhakapply.com` (운영)
>
> **인증**: `X-Dev-Tasks-Key` 헤더 (프로젝트별 API 키)
>
> **OpenAPI 명세**: `http://localhost:3100/docs` (Swagger UI)

---

## 인증

모든 Dev Tasks API 요청에는 `X-Dev-Tasks-Key` 헤더가 필요합니다.

```
X-Dev-Tasks-Key: dtk-yourproject-xxxxxxxx
```

- 키는 Gateway의 `DEV_TASKS_API_KEYS` 환경변수에 `프로젝트명:키` 형식으로 등록됩니다.
- 키에 바인딩된 프로젝트명과 요청의 `sourceProject`가 일치해야 합니다.
- 키가 없거나 잘못된 경우 `401` 또는 `403` 에러가 반환됩니다.

### 에러 응답
| HTTP 코드 | 상황 |
|-----------|------|
| 401 | `X-Dev-Tasks-Key` 헤더 누락 |
| 403 | 잘못된 키 또는 sourceProject 불일치 |

---

## 개요

Gateway에 API 구현 작업을 제출하면, 작업자가 구현 완료 후 엔드포인트가 실제로 동작하는지 자동 검증까지 수행하는 시스템입니다.

```
외부 프로젝트 → POST /api/dev-tasks (명세 제출)
  → 작업자가 GET으로 조회 → 구현 → POST status=completed
  → 외부 프로젝트가 POST /verify → 엔드포인트 자동 검증 → verified
```

### 상태 흐름

```
pending → in_progress → completed → verified
                ↑             ↓
              pending ← (검증실패/재작업)
                ↑
rejected ← (반려)
```

| 상태 | 의미 |
|------|------|
| `pending` | 제출 완료, 작업자 배정 대기 |
| `in_progress` | 작업자가 구현 중 |
| `completed` | 구현 완료, 검증 가능 |
| `verified` | 엔드포인트 검증 통과 (최종 완료) |
| `rejected` | 반려됨 |

---

## 사용법

### 1. 작업 제출 (외부 프로젝트 → Gateway)

```bash
POST /api/dev-tasks
Content-Type: application/json
X-Dev-Tasks-Key: dtk-nightbuilder-xxxxxxxx

{
  "title": "작업 제목 (간결하게)",
  "description": "작업에 대한 추가 설명 (선택)",
  "spec": "## 요구사항\n\n구현할 API의 상세 명세...\n\n### 엔드포인트\n...",
  "specFormat": "markdown",
  "sourceProject": "nightbuilder",
  "endpoints": [
    {"method": "GET", "path": "/api/some/endpoint", "description": "데이터 조회"},
    {"method": "POST", "path": "/api/some/action", "description": "액션 실행"}
  ],
  "priority": 10,
  "metadata": {}
}
```

**필수 필드**:
- `title` — 작업 제목
- `spec` — 구현 명세 본문 (마크다운 권장)
- `sourceProject` — 제출하는 프로젝트 이름

**선택 필드**:
- `description` — 추가 설명
- `specFormat` — 명세 형식 (`markdown` | `text` | `json`, 기본: `markdown`)
- `endpoints` — 구현 대상 엔드포인트 배열 (검증에 사용됨)
- `priority` — 우선순위 (높을수록 먼저 처리, 기본: `0`)
- `metadata` — 자유 형식 메타데이터

**응답** (201):
```json
{
  "id": "dtask-20260208-a1b2",
  "title": "작업 제목",
  "status": "pending",
  "sourceProject": "nightbuilder",
  "createdAt": "2026-02-08T10:00:00.000Z",
  ...
}
```

> `id`를 저장해두세요. 이후 상태 확인, 검증에 사용됩니다.

---

### 2. 상태 확인 (폴링)

```bash
GET /api/dev-tasks/{taskId}
```

`status`가 `completed`가 될 때까지 폴링합니다.

```json
{
  "id": "dtask-20260208-a1b2",
  "status": "completed",
  "assignee": "claude",
  "completedAt": "2026-02-08T14:30:00.000Z",
  ...
}
```

---

### 3. 엔드포인트 검증 (`completed` 상태에서만)

```bash
POST /api/dev-tasks/{taskId}/verify
```

등록된 `endpoints` 배열의 각 엔드포인트에 실제 HTTP 요청을 보내 존재 여부를 확인합니다.

**응답** (200):
```json
{
  "taskId": "dtask-20260208-a1b2",
  "allPassed": true,
  "results": [
    {
      "method": "GET",
      "path": "/api/some/endpoint",
      "status": 200,
      "success": true,
      "responseTime": 45
    },
    {
      "method": "POST",
      "path": "/api/some/action",
      "status": 400,
      "success": true,
      "responseTime": 12
    }
  ],
  "verifiedAt": "2026-02-08T15:30:00.000Z"
}
```

- `allPassed: true` → 자동으로 `verified` 상태 전환 (최종 완료)
- `allPassed: false` → `completed` 상태 유지, 검증 결과만 저장 (재검증 가능)
- `success` 판정: HTTP 404가 아니면 존재하는 것으로 판단 (400, 401 등도 존재로 인정)

---

### 4. 기타 API

```bash
# 작업 목록 조회 (필터링)
GET /api/dev-tasks?status=pending&sort=priority&order=desc&page=1&pageSize=20

# 전체 목록 (필터 없이)
GET /api/dev-tasks

# 상태 변경
POST /api/dev-tasks/{taskId}/status
{"status": "in_progress", "assignee": "claude"}

# 삭제 (verified 또는 rejected 상태만 가능)
POST /api/dev-tasks/{taskId}/delete
```

---

## AI 에이전트용 통합 플로우 예시

외부 프로젝트의 AI 에이전트가 Gateway에 구현 요청을 자동으로 제출하는 전체 플로우:

```
1. 구현이 필요한 API 명세를 마크다운으로 작성
2. POST /api/dev-tasks로 제출 → taskId 획득
3. 주기적으로 GET /api/dev-tasks/{taskId}로 상태 폴링
   - pending: 대기 중
   - in_progress: 구현 중
   - completed: 검증 단계로 진행
4. completed 확인 시 POST /api/dev-tasks/{taskId}/verify 호출
   - allPassed=true: 완료, 해당 엔드포인트 사용 가능
   - allPassed=false: 결과 확인 후 재요청 또는 대기
```

---

## spec 작성 가이드

`spec` 필드에 넣을 명세는 작업자(사람 또는 AI)가 읽고 구현할 수 있도록 충분한 정보를 포함해야 합니다.

### 권장 구조

```markdown
## 요구사항
구현할 기능의 목적과 배경

## 엔드포인트

### GET /api/example/items
- 설명: 아이템 목록 조회
- 쿼리 파라미터:
  - status (string, 선택): 상태 필터
  - page (integer, 기본 1): 페이지 번호
- 응답 (200):
  ```json
  {
    "items": [...],
    "total": 100,
    "page": 1,
    "pageSize": 20
  }
  ```

### POST /api/example/items
- 설명: 아이템 생성
- 요청 바디:
  ```json
  {
    "name": "string (필수)",
    "description": "string (선택)"
  }
  ```
- 응답 (201): 생성된 아이템 객체

## 데이터 모델
필요한 DB 테이블, 컬럼 정보

## 비즈니스 규칙
유효성 검증, 상태 전이 등 주의사항
```

---

## 에러 응답

모든 에러는 동일한 형식으로 반환됩니다:

```json
{
  "error": {
    "code": "BAD_REQUEST | NOT_FOUND | CONFLICT | INTERNAL_ERROR",
    "message": "에러 상세 메시지"
  }
}
```

| HTTP 코드 | code | 의미 |
|-----------|------|------|
| 400 | BAD_REQUEST | 필수 필드 누락 등 |
| 404 | NOT_FOUND | 작업 ID가 존재하지 않음 |
| 409 | CONFLICT | 잘못된 상태 전이 또는 삭제 불가 |
| 401 | UNAUTHORIZED | API 키 헤더 누락 |
| 403 | FORBIDDEN | 잘못된 키 또는 프로젝트 불일치 |
| 500 | INTERNAL_ERROR | 서버 내부 오류 |

---

## 외부 프로젝트 CLAUDE.md 설정 가이드

다른 프로젝트에서 Dev Tasks API를 사용하려면 해당 프로젝트의 `CLAUDE.md`에 아래 내용을 추가하세요.

### CLAUDE.md에 추가할 내용

```markdown
## Gateway Dev Tasks API 연동

이 프로젝트는 JABIS API Gateway에 개발 작업을 요청할 수 있습니다.

### 연동 정보
- **Gateway 주소**: http://localhost:3100
- **API 키**: dtk-{프로젝트명}-{발급받은키}
- **프로젝트 식별자**: {프로젝트명} (sourceProject 값)
- **인증 헤더**: `X-Dev-Tasks-Key: {API키}`

### Gateway에 API 구현 요청 시

Gateway에 새로운 API 엔드포인트 구현이 필요한 경우, 아래 절차를 따릅니다:

1. 구현할 API의 명세를 마크다운으로 작성합니다
2. Dev Tasks API를 통해 작업을 제출합니다:
   ```bash
   curl -X POST http://localhost:3100/api/dev-tasks \
     -H "Content-Type: application/json" \
     -H "X-Dev-Tasks-Key: {API키}" \
     -d '{
       "title": "작업 제목",
       "spec": "마크다운 명세 내용",
       "sourceProject": "{프로젝트명}",
       "endpoints": [{"method": "GET", "path": "/api/..."}],
       "priority": 10
     }'
   ```
3. 반환된 `id`로 상태를 폴링합니다:
   ```bash
   curl -H "X-Dev-Tasks-Key: {API키}" http://localhost:3100/api/dev-tasks/{taskId}
   ```
4. `status`가 `completed`가 되면 검증합니다:
   ```bash
   curl -X POST -H "X-Dev-Tasks-Key: {API키}" http://localhost:3100/api/dev-tasks/{taskId}/verify
   ```

### 명세(spec) 작성 규칙
- 마크다운 형식 권장
- 엔드포인트별 요청/응답 스키마 포함
- 비즈니스 규칙, 유효성 검증 조건 명시
- DB 테이블 변경이 필요한 경우 스키마 포함

상세 가이드: Gateway 프로젝트의 `docs/DEV_TASKS_API_GUIDE.md` 참조
```

### 환경변수 (.env)

프로젝트 `.env` 파일에 API 키를 추가하세요 (`.gitignore`에 포함되어야 함):

```env
GATEWAY_DEV_TASKS_KEY=dtk-{프로젝트명}-{발급받은키}
```
