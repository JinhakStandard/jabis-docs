# JABIS API Gateway - API 규약서

> JABIS Night Builder & Auto Debugger가 API Gateway와 통신하기 위한 REST API 규약.
> API 서버 구축 시 이 문서를 기준으로 구현합니다.

---

## 목차

1. [공통 사항](#1-공통-사항)
2. [데이터 모델](#2-데이터-모델)
3. [기능 요청 (Feature Request) API](#3-기능-요청-feature-request-api)
4. [디버그 태스크 (Debug Task) API](#4-디버그-태스크-debug-task-api)
5. [상태 흐름](#5-상태-흐름)
6. [에러 응답](#6-에러-응답)
7. [호출 시나리오](#7-호출-시나리오)

---

## 1. 공통 사항

### Base URL

```
http://{host}:{port}
```

환경별 설정:
- 개발: `http://localhost:8080`
- 운영: 별도 도메인

### 인증

모든 요청에 Bearer 토큰을 포함합니다.

```
Authorization: Bearer {token}
```

인증 실패 시 `401 Unauthorized` 또는 `403 Forbidden`을 반환합니다.
이 두 상태 코드는 재시도하지 않습니다.

### Content-Type

모든 요청/응답은 JSON 형식입니다.

```
Content-Type: application/json
```

### 타임아웃

클라이언트 기본 타임아웃: **30초** (설정 가능)

### 재시도 정책

클라이언트는 5xx 에러 및 네트워크 오류 시 **최대 3회** exponential backoff로 재시도합니다.
- 1차: 1초 후
- 2차: 2초 후
- 3차: 4초 후

---

## 2. 데이터 모델

### 2.1 RequestStatus (공통 상태 enum)

```
queued | planning | plan_review | approved | in_progress | completed | failed | cancelled
```

| 값 | 설명 |
|----|------|
| `queued` | 사용자가 등록한 초기 상태 |
| `planning` | Planning Service가 계획 수립 중 |
| `plan_review` | 계획 수립 완료, 사용자 검토 대기 |
| `approved` | 사용자가 계획 승인, Night Builder 실행 대상 |
| `in_progress` | Night Builder / Auto Debugger 실행 중 |
| `completed` | 처리 완료 |
| `failed` | 처리 실패 |
| `cancelled` | 사용자가 취소 |

### 2.2 FeatureRequest (기능 요청)

```json
{
  "id": "req-20260203-001",
  "title": "로그인 페이지에 비밀번호 찾기 추가",
  "description": "이메일 인증 기반의 비밀번호 재설정 기능을 추가합니다.",
  "prompt": "로그인 페이지에 비밀번호 찾기 링크를 추가하고, 이메일 입력 → 인증 코드 발송 → 비밀번호 재설정 플로우를 구현해주세요.",
  "repositoryUrl": "https://github.com/company/my-project.git",
  "branch": "develop",
  "targetBranch": "develop",
  "status": "queued",
  "priority": 1,
  "createdAt": "2026-02-03T10:00:00Z",
  "updatedAt": "2026-02-03T10:00:00Z",
  "plan": null,
  "planApprovedAt": null,
  "metadata": {}
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | string | Y | 고유 식별자 |
| `title` | string | Y | 요청 제목 |
| `description` | string | Y | 요청 상세 설명 |
| `prompt` | string | Y | ai-orchestra에 전달할 프롬프트 |
| `repositoryUrl` | string | Y | Git 저장소 URL |
| `branch` | string \| null | N | 작업 기준 브랜치 (미지정 시 기본 브랜치 사용) |
| `targetBranch` | string \| null | N | PR 대상 브랜치 (미지정 시 기본 브랜치 사용) |
| `status` | RequestStatus | Y | 현재 상태 |
| `priority` | number | Y | 우선순위 (높을수록 먼저 처리) |
| `createdAt` | string (ISO 8601) | Y | 생성 시각 |
| `updatedAt` | string (ISO 8601) | Y | 최종 수정 시각 |
| `plan` | ImplementationPlan \| null | N | 수립된 구현 계획 |
| `planApprovedAt` | string (ISO 8601) \| null | N | 계획 승인 시각 |
| `metadata` | object \| null | N | 추가 메타데이터 |

### 2.3 ImplementationPlan (구현 계획)

Planning Service가 수립하여 `submitPlan` API로 등록하는 구조체입니다.

```json
{
  "summary": "로그인 페이지에 비밀번호 찾기 기능을 추가합니다. 이메일 입력 후 인증 코드를 발송하고, 코드 확인 후 새 비밀번호를 설정할 수 있는 화면을 만듭니다.",
  "steps": [
    {
      "order": 1,
      "title": "비밀번호 찾기 화면 추가",
      "description": "로그인 페이지에 '비밀번호를 잊으셨나요?' 링크를 추가하고, 이메일 입력 화면을 만듭니다."
    },
    {
      "order": 2,
      "title": "인증 코드 발송 기능",
      "description": "입력된 이메일로 인증 코드를 발송하는 서버 기능을 구현합니다."
    }
  ],
  "estimatedComplexity": "medium",
  "affectedAreas": ["로그인 페이지", "이메일 발송", "사용자 인증"]
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `summary` | string | 전체 계획 요약 (한국어, 비개발자 대상) |
| `steps` | PlanStep[] | 구현 단계 목록 |
| `steps[].order` | number | 단계 순서 |
| `steps[].title` | string | 단계 제목 |
| `steps[].description` | string | 단계 상세 설명 |
| `estimatedComplexity` | `"low"` \| `"medium"` \| `"high"` | 예상 복잡도 |
| `affectedAreas` | string[] | 영향 받는 영역 목록 |

### 2.4 DebugTask (디버그 태스크)

```json
{
  "id": "debug-20260203-001",
  "errorMessage": "TypeError: Cannot read property 'name' of undefined",
  "stackTrace": "at UserService.getUser (src/services/user.ts:42:18)\nat async handler (src/routes/users.ts:15:20)",
  "repositoryUrl": "https://github.com/company/my-project.git",
  "branch": "develop",
  "filePath": "src/services/user.ts",
  "context": "사용자 조회 시 탈퇴한 사용자에 대해 null 체크 누락",
  "status": "queued",
  "priority": 2,
  "createdAt": "2026-02-03T14:30:00Z",
  "updatedAt": "2026-02-03T14:30:00Z",
  "metadata": {}
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `id` | string | Y | 고유 식별자 |
| `errorMessage` | string | Y | 에러 메시지 |
| `stackTrace` | string | Y | 스택 트레이스 |
| `repositoryUrl` | string | Y | Git 저장소 URL |
| `branch` | string | Y | 작업 대상 브랜치 |
| `filePath` | string \| null | N | 에러 발생 추정 파일 경로 |
| `context` | string \| null | N | 추가 컨텍스트 정보 |
| `status` | RequestStatus | Y | 현재 상태 |
| `priority` | number | Y | 우선순위 |
| `createdAt` | string (ISO 8601) | Y | 생성 시각 |
| `updatedAt` | string (ISO 8601) | Y | 최종 수정 시각 |
| `metadata` | object \| null | N | 추가 메타데이터 |

### 2.5 StatusUpdate (상태 변경 요청)

POST 요청 시 전송되는 body 구조입니다.

```json
{
  "status": "in_progress",
  "message": "JABIS Night Builder started processing",
  "result": null
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `status` | RequestStatus | Y | 변경할 상태 |
| `message` | string \| null | N | 상태 변경 사유 / 설명 |
| `result` | object \| null | N | 처리 결과 데이터 (completed/failed 시) |

`result` 필드에 들어갈 수 있는 값은 상태에 따라 다릅니다:

**completed 시:**
```json
{
  "branchName": "feature/req-20260203-001",
  "prUrl": "https://github.com/company/my-project/pull/42",
  "testResults": [
    { "command": "npm test", "passed": true, "output": "..." },
    { "command": "npm run lint", "passed": true, "output": "..." }
  ],
  "phasesExecuted": 3
}
```

**failed 시:**
```json
{
  "testResults": [
    { "command": "npm test", "passed": false, "output": "FAIL src/..." }
  ]
}
```

---

## 3. 기능 요청 (Feature Request) API

### 3.1 기능 요청 목록 조회

상태별로 기능 요청 목록을 조회합니다.

```
GET /api/requests?status={status}&sort=priority&order=desc
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | RequestStatus | Y | 조회할 상태 |
| `sort` | string | N | 정렬 기준 필드 (기본값: `priority`) |
| `order` | `asc` \| `desc` | N | 정렬 방향 (기본값: `desc`) |

**호출 주체별 사용 상태:**

| 호출 주체 | 조회 상태 | 용도 |
|-----------|-----------|------|
| Planning Service | `queued` | 계획 수립 대상 요청 감지 |
| Night Builder | `approved` | 실행 대상 요청 감지 |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "req-20260203-001",
      "title": "로그인 페이지에 비밀번호 찾기 추가",
      "description": "...",
      "prompt": "...",
      "repositoryUrl": "https://github.com/company/my-project.git",
      "branch": "develop",
      "targetBranch": "develop",
      "status": "queued",
      "priority": 1,
      "createdAt": "2026-02-03T10:00:00Z",
      "updatedAt": "2026-02-03T10:00:00Z",
      "plan": null,
      "planApprovedAt": null,
      "metadata": {}
    }
  ],
  "total": 1,
  "page": 1,
  "pageSize": 20
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `items` | FeatureRequest[] | 요청 목록 |
| `total` | number | 전체 건수 |
| `page` | number | 현재 페이지 |
| `pageSize` | number | 페이지 크기 |

> **중요**: `status=approved`로 조회 시 `plan` 필드에 승인된 계획서가 포함되어야 합니다. Night Builder가 이 계획서를 기반으로 상세 Phase를 수립합니다.

---

### 3.2 기능 요청 상태 변경

기능 요청의 상태를 변경합니다.

```
POST /api/requests/{requestId}/status
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `requestId` | string | 기능 요청 ID |

**Request Body:**

```json
{
  "status": "in_progress",
  "message": "JABIS Night Builder started processing",
  "result": null
}
```

**JABIS가 요청하는 상태 전이:**

| 현재 상태 | 변경 상태 | 호출 주체 | message 예시 |
|-----------|-----------|-----------|-------------|
| `queued` | `planning` | Planning Service | `"JABIS Planning Service: 계획 수립 시작"` |
| `planning` (실패 시) | `queued` | Planning Service | `"계획 수립 실패, 재시도 대기: {에러}"` |
| `approved` | `in_progress` | Night Builder | `"JABIS Night Builder started processing"` |
| `in_progress` | `completed` | Night Builder | `"Feature implementation completed"` |
| `in_progress` | `failed` | Night Builder | `"Tests failed after 3 attempts"` 또는 에러 메시지 |

**Response: `200 OK` 또는 `204 No Content`**

성공 시 body 없이 응답합니다.

---

### 3.3 구현 계획 등록

Planning Service가 수립한 구현 계획을 등록하고, 상태를 `plan_review`로 변경합니다.

```
POST /api/requests/{requestId}/plan
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `requestId` | string | 기능 요청 ID |

**Request Body:**

```json
{
  "plan": {
    "summary": "사용자 로그인 페이지에 비밀번호 찾기 기능을 추가합니다.",
    "steps": [
      {
        "order": 1,
        "title": "비밀번호 찾기 화면 추가",
        "description": "로그인 페이지에 '비밀번호를 잊으셨나요?' 링크를 추가하고, 이메일 입력 화면을 만듭니다."
      },
      {
        "order": 2,
        "title": "인증 코드 발송 기능",
        "description": "입력된 이메일로 인증 코드를 발송하는 서버 기능을 구현합니다."
      }
    ],
    "estimatedComplexity": "medium",
    "affectedAreas": ["로그인 페이지", "이메일 발송"]
  },
  "status": "plan_review"
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `plan` | ImplementationPlan | 구현 계획 객체 |
| `status` | `"plan_review"` | 변경할 상태 (항상 `plan_review`) |

**서버 처리 사항:**

1. 해당 요청의 `plan` 필드에 계획 객체 저장
2. `status`를 `plan_review`로 변경
3. `updatedAt`을 현재 시각으로 갱신

**Response: `200 OK` 또는 `204 No Content`**

---

## 4. 디버그 태스크 (Debug Task) API

### 4.1 디버그 태스크 목록 조회

대기 중인 디버그 태스크를 조회합니다.

```
GET /api/debug-tasks?status=queued&sort=priority&order=desc
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | RequestStatus | Y | 조회할 상태 (항상 `queued`) |
| `sort` | string | N | 정렬 기준 필드 (기본값: `priority`) |
| `order` | `asc` \| `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "debug-20260203-001",
      "errorMessage": "TypeError: Cannot read property 'name' of undefined",
      "stackTrace": "at UserService.getUser (src/services/user.ts:42:18)...",
      "repositoryUrl": "https://github.com/company/my-project.git",
      "branch": "develop",
      "filePath": "src/services/user.ts",
      "context": "사용자 조회 시 탈퇴한 사용자에 대해 null 체크 누락",
      "status": "queued",
      "priority": 2,
      "createdAt": "2026-02-03T14:30:00Z",
      "updatedAt": "2026-02-03T14:30:00Z",
      "metadata": {}
    }
  ],
  "total": 1,
  "page": 1,
  "pageSize": 20
}
```

---

### 4.2 디버그 태스크 상태 변경

디버그 태스크의 상태를 변경합니다.

```
POST /api/debug-tasks/{taskId}/status
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `taskId` | string | 디버그 태스크 ID |

**Request Body:**

```json
{
  "status": "in_progress",
  "message": "JABIS Auto Debugger started processing",
  "result": null
}
```

**JABIS가 요청하는 상태 전이:**

| 현재 상태 | 변경 상태 | message 예시 | result |
|-----------|-----------|-------------|--------|
| `queued` | `in_progress` | `"JABIS Auto Debugger started processing"` | null |
| `in_progress` | `completed` | `"Bug fix applied successfully"` | `{ "branchName": "fix/debug-001" }` |
| `in_progress` | `failed` | `"Could not fix the issue after all retry attempts"` 또는 에러 메시지 | null |

**Response: `200 OK` 또는 `204 No Content`**

---

## 5. 상태 흐름

### 5.1 기능 요청 상태 전이 다이어그램

```
                        [사용자 등록]
                             │
                             ▼
                          queued ◄──────────────── (계획 수립 실패 시 복원)
                             │
                    Planning Service 감지
                             │
                             ▼
                         planning
                             │
                    Claude API 계획 수립
                             │
                             ▼
                       plan_review ──────────────► cancelled
                             │                     [사용자 취소]
                       [사용자 승인]
                             │
                             ▼
                         approved
                             │
                    Night Builder 감지
                             │
                             ▼
                       in_progress
                           ┌─┴─┐
                           │   │
                           ▼   ▼
                     completed  failed
```

### 5.2 디버그 태스크 상태 전이 다이어그램

```
                        [시스템 등록]
                             │
                             ▼
                          queued
                             │
                    Auto Debugger 감지
                             │
                             ▼
                       in_progress
                           ┌─┴─┐
                           │   │
                           ▼   ▼
                     completed  failed
```

### 5.3 상태 변경 주체 정리

| API 엔드포인트 | 변경 주체 | 비고 |
|---------------|-----------|------|
| `POST /api/requests/{id}/status` (→ planning) | Planning Service | |
| `POST /api/requests/{id}/status` (→ queued) | Planning Service | 계획 수립 실패 시 복원 |
| `POST /api/requests/{id}/plan` (→ plan_review) | Planning Service | 계획 등록과 동시에 |
| `POST /api/requests/{id}/status` (→ approved) | **웹 UI (사용자)** | 서버에서 직접 구현 |
| `POST /api/requests/{id}/status` (→ cancelled) | **웹 UI (사용자)** | 서버에서 직접 구현 |
| `POST /api/requests/{id}/status` (→ in_progress) | Night Builder | |
| `POST /api/requests/{id}/status` (→ completed) | Night Builder | |
| `POST /api/requests/{id}/status` (→ failed) | Night Builder | |
| `POST /api/debug-tasks/{id}/status` (→ in_progress) | Auto Debugger | |
| `POST /api/debug-tasks/{id}/status` (→ completed) | Auto Debugger | |
| `POST /api/debug-tasks/{id}/status` (→ failed) | Auto Debugger | |

---

## 6. 에러 응답

### 에러 응답 형식

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Request not found: req-20260203-999"
  }
}
```

### HTTP 상태 코드

| 코드 | 의미 | JABIS 동작 |
|------|------|-----------|
| `200` | 성공 | 정상 처리 |
| `204` | 성공 (body 없음) | 정상 처리 |
| `400` | 잘못된 요청 | 로그 후 해당 요청 건너뜀 |
| `401` | 인증 실패 | 재시도 안함, 로그 후 중단 |
| `403` | 권한 없음 | 재시도 안함, 로그 후 중단 |
| `404` | 리소스 없음 | 로그 후 해당 요청 건너뜀 |
| `409` | 상태 충돌 (이미 처리 중 등) | 로그 후 해당 요청 건너뜀 |
| `429` | 요청 제한 초과 | Retry-After 헤더 존중 후 재시도 |
| `500` | 서버 오류 | exponential backoff 재시도 (최대 3회) |
| `502`/`503`/`504` | 서버 일시 장애 | exponential backoff 재시도 (최대 3회) |

---

## 7. 호출 시나리오

### 7.1 Planning Service 시나리오

Planning Service는 10~30초 주기로 폴링합니다.

```
┌─────────────────────┐                        ┌──────────────┐
│  Planning Service   │                        │  API Gateway │
└─────────┬───────────┘                        └──────┬───────┘
          │                                           │
          │  GET /api/requests?status=queued           │
          │  &sort=priority&order=desc                 │
          ├──────────────────────────────────────────►│
          │                                           │
          │  200 OK { items: [...], total: 1, ... }   │
          │◄──────────────────────────────────────────┤
          │                                           │
          │  POST /api/requests/req-001/status        │
          │  { status: "planning", message: "..." }   │
          ├──────────────────────────────────────────►│
          │                                           │
          │  204 No Content                           │
          │◄──────────────────────────────────────────┤
          │                                           │
          │  ... Claude API로 계획 수립 ...             │
          │                                           │
          │  POST /api/requests/req-001/plan            │
          │  { plan: {...}, status: "plan_review" }    │
          ├──────────────────────────────────────────►│
          │                                           │
          │  204 No Content                           │
          │◄──────────────────────────────────────────┤
          │                                           │
```

**계획 수립 실패 시:**

```
          │  POST /api/requests/req-001/status        │
          │  { status: "queued",                       │
          │    message: "계획 수립 실패, 재시도 대기" }   │
          ├──────────────────────────────────────────►│
```

---

### 7.2 Night Builder 시나리오

Night Builder는 30초 주기로 폴링합니다.

```
┌─────────────────────┐                        ┌──────────────┐
│   Night Builder     │                        │  API Gateway │
└─────────┬───────────┘                        └──────┬───────┘
          │                                           │
          │  GET /api/requests?status=approved         │
          │  &sort=priority&order=desc                 │
          ├──────────────────────────────────────────►│
          │                                           │
          │  200 OK { items: [...], total: 2, ... }   │
          │◄──────────────────────────────────────────┤
          │                                           │
          │  POST /api/requests/req-001/status        │
          │  { status: "in_progress", message: "..." } │
          ├──────────────────────────────────────────►│
          │                                           │
          │  204 No Content                           │
          │◄──────────────────────────────────────────┤
          │                                           │
          │  ... Git clone/pull, Phase 수립,           │
          │      ai-orchestra 실행, 테스트 ...          │
          │                                           │
          │  [성공 시]                                  │
          │  POST /api/requests/req-001/status        │
          │  { status: "completed",                    │
          │    message: "Feature implementation ...",   │
          │    result: {                               │
          │      branchName: "feature/req-001",        │
          │      prUrl: "https://github.com/.../42",   │
          │      testResults: [...],                   │
          │      phasesExecuted: 3                     │
          │    }                                       │
          │  }                                         │
          ├──────────────────────────────────────────►│
          │                                           │
          │  [실패 시]                                  │
          │  POST /api/requests/req-001/status        │
          │  { status: "failed",                       │
          │    message: "Tests failed after 3 ...",    │
          │    result: { testResults: [...] }          │
          │  }                                         │
          ├──────────────────────────────────────────►│
          │                                           │
```

---

### 7.3 Auto Debugger 시나리오

Auto Debugger는 30초 주기로 폴링합니다.

```
┌─────────────────────┐                        ┌──────────────┐
│   Auto Debugger     │                        │  API Gateway │
└─────────┬───────────┘                        └──────┬───────┘
          │                                           │
          │  GET /api/debug-tasks?status=queued        │
          │  &sort=priority&order=desc                 │
          ├──────────────────────────────────────────►│
          │                                           │
          │  200 OK { items: [...] }                   │
          │◄──────────────────────────────────────────┤
          │                                           │
          │  POST /api/debug-tasks/debug-001/status   │
          │  { status: "in_progress", message: "..." } │
          ├──────────────────────────────────────────►│
          │                                           │
          │  ... Git clone, ai-orchestra 실행 ...       │
          │                                           │
          │  [성공 시]                                  │
          │  POST /api/debug-tasks/debug-001/status   │
          │  { status: "completed",                    │
          │    message: "Bug fix applied ...",         │
          │    result: { branchName: "fix/debug-001" } │
          │  }                                         │
          ├──────────────────────────────────────────►│
          │                                           │
```

---

## API 요약 테이블

| # | Method | 경로 | 설명 | 호출 주체 |
|---|--------|------|------|-----------|
| 1 | `GET` | `/api/requests?status={status}&sort=priority&order=desc` | 기능 요청 목록 조회 | Planning Service, Night Builder |
| 2 | `POST` | `/api/requests/{requestId}/status` | 기능 요청 상태 변경 | Planning Service, Night Builder |
| 3 | `POST` | `/api/requests/{requestId}/plan` | 구현 계획 등록 + 상태 변경 | Planning Service |
| 4 | `GET` | `/api/debug-tasks?status=queued&sort=priority&order=desc` | 디버그 태스크 목록 조회 | Auto Debugger |
| 5 | `POST` | `/api/debug-tasks/{taskId}/status` | 디버그 태스크 상태 변경 | Auto Debugger |

---

## 서버 구현 시 참고 사항

### 동시성 보호

- 여러 JABIS 인스턴스가 동시에 같은 요청을 폴링할 수 있습니다.
- `POST /status`에서 `queued → planning`, `approved → in_progress` 전이 시 **낙관적 잠금** 또는 **상태 전이 조건 검증**을 권장합니다.
- 예: 이미 `planning` 상태인 요청에 다시 `planning` 전이 요청이 오면 `409 Conflict` 반환.

### 타임아웃 복구

- `planning` 또는 `in_progress` 상태에서 일정 시간(예: 2시간) 경과 시 `queued` 또는 `approved`로 자동 복원하는 타임아웃 정책을 서버에서 구현하는 것을 권장합니다.
- JABIS 프로세스가 비정상 종료되면 진행 중이던 요청이 해당 상태로 남기 때문입니다.

### 정렬 및 페이징

- 현재 JABIS는 `sort=priority&order=desc`만 사용합니다.
- 페이징은 현재 사용하지 않지만, 대량 요청 대비 `page`, `pageSize` 파라미터를 지원하는 것을 권장합니다.

### plan 필드 관리

- `POST /api/requests/{id}/plan` 호출 시 `plan` 필드를 저장하고 `status`를 `plan_review`로 변경합니다.
- 사용자가 웹 UI에서 계획을 수정할 수 있도록, `plan` 필드의 개별 수정 API도 별도로 구현을 권장합니다 (JABIS는 호출하지 않음).
- 사용자가 승인(`approved`)하면 `planApprovedAt`을 현재 시각으로 설정합니다.
