# JABIS 대시보드 API 규약서

> AI Orchestra 실행 내역 및 Night Builder 작업 내역을 조회하기 위한 REST API 규약.
> 웹 대시보드 프론트엔드가 이 문서를 기준으로 호출합니다.

---

## 목차

1. [공통 사항](#1-공통-사항)
2. [AI Orchestra API](#2-ai-orchestra-api)
3. [Night Builder API](#3-night-builder-api)
4. [에러 응답](#4-에러-응답)

---

## 1. 공통 사항

### Base URL

```
http://{host}:{port}/api/dashboard
```

### 인증

모든 요청에 Bearer 토큰을 포함합니다.

```
Authorization: Bearer {token}
```

jabis-cert를 통한 JWT 검증. 인증 실패 시 `401 Unauthorized`.

### HTTP 메서드

모든 API는 **GET** 또는 **POST**만 사용합니다.

- `GET` — 조회
- `POST` — 생성, 수정, 삭제 등 상태 변경 작업

> PUT, PATCH, DELETE 등 다른 메서드는 사용하지 않습니다.

### Content-Type

```
Content-Type: application/json
```

### 공통 페이징 파라미터

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|----------|------|------|--------|------|
| `page` | number | N | `1` | 페이지 번호 (1부터) |
| `pageSize` | number | N | `20` | 페이지 크기 (최대 100) |

### 공통 페이징 응답

```json
{
  "items": [...],
  "total": 150,
  "page": 1,
  "pageSize": 20
}
```

### 공통 날짜 필터 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `from` | string (ISO 8601) | N | 시작 일시 (inclusive) |
| `to` | string (ISO 8601) | N | 종료 일시 (exclusive) |

---

## 2. AI Orchestra API

### 2.1 세션 목록 조회

Orchestra 세션 실행 내역을 조회합니다.

```
GET /api/dashboard/orchestra/sessions
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | string | N | 상태 필터 (`running` / `completed` / `failed`) |
| `instanceId` | string | N | 인스턴스 ID 필터 |
| `developerId` | string | N | 개발자 ID 필터 |
| `projectName` | string | N | 프로젝트명 검색 (부분 일치) |
| `complexity` | string | N | 복잡도 필터 (`simple` / `medium` / `complex`) |
| `from` | string | N | 시작 일시 |
| `to` | string | N | 종료 일시 |
| `sort` | string | N | 정렬 기준 (기본값: `started_at`) |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "sessionId": "session-abc-123",
      "instanceId": "inst-001",
      "projectName": "jabis-api-gateway",
      "projectPath": "/home/user/projects/jabis-api-gateway",
      "developerId": "user@company.com",
      "developerName": "홍길동",
      "requirement": "로그인 페이지에 비밀번호 찾기 기능 추가",
      "complexity": "medium",
      "executionMode": "auto",
      "status": "completed",
      "resultSuccess": true,
      "resultErrorMsg": null,
      "resultFilesCreated": 3,
      "resultFilesModified": 5,
      "totalTokens": 125000,
      "totalCost": 1.2500,
      "startedAt": "2026-02-06T01:00:00Z",
      "endedAt": "2026-02-06T01:15:30Z",
      "durationMs": 930000,
      "phaseCount": 4
    }
  ],
  "total": 42,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.2 세션 상세 조회

특정 세션의 상세 정보를 Phase, 토큰 사용량, 에러 정보와 함께 조회합니다.

```
GET /api/dashboard/orchestra/sessions/{sessionId}
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `sessionId` | string | 세션 ID |

**Response: `200 OK`**

```json
{
  "session": {
    "id": "uuid",
    "sessionId": "session-abc-123",
    "instanceId": "inst-001",
    "projectName": "jabis-api-gateway",
    "projectPath": "/home/user/projects/jabis-api-gateway",
    "developerId": "user@company.com",
    "developerName": "홍길동",
    "machineName": "dev-server-01",
    "requirement": "로그인 페이지에 비밀번호 찾기 기능 추가",
    "complexity": "medium",
    "executionMode": "auto",
    "status": "completed",
    "resultSuccess": true,
    "resultErrorMsg": null,
    "resultFilesCreated": 3,
    "resultFilesModified": 5,
    "totalInputTokens": 80000,
    "totalOutputTokens": 45000,
    "totalTokens": 125000,
    "totalCost": 1.2500,
    "startedAt": "2026-02-06T01:00:00Z",
    "endedAt": "2026-02-06T01:15:30Z",
    "durationMs": 930000
  },
  "phases": [
    {
      "id": "uuid",
      "phaseId": "phase-001",
      "title": "요구사항 분석",
      "status": "completed",
      "agentRole": "analyst",
      "modelUsed": "claude-sonnet-4-5-20250929",
      "durationMs": 45000,
      "sortOrder": 1,
      "startedAt": "2026-02-06T01:00:00Z",
      "completedAt": "2026-02-06T01:00:45Z"
    }
  ],
  "tokenUsage": [
    {
      "phaseId": "phase-001",
      "agent": "analyst",
      "inputTokens": 20000,
      "outputTokens": 10000,
      "totalTokens": 30000,
      "estimatedCost": 0.3000,
      "recordedAt": "2026-02-06T01:00:45Z"
    }
  ],
  "errors": [
    {
      "id": "uuid",
      "originalMessage": "TypeError: Cannot read property 'name' of undefined",
      "stackPattern": "at UserService.getUser...",
      "agentRole": "coder",
      "phaseId": "phase-003",
      "relatedFiles": ["src/services/user.ts"],
      "occurredAt": "2026-02-06T01:10:00Z",
      "errorType": "TypeError",
      "autoFixable": true
    }
  ]
}
```

---

### 2.3 인스턴스 목록 조회

등록된 AI Orchestra 인스턴스 목록을 조회합니다.

```
GET /api/dashboard/orchestra/instances
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `isActive` | boolean | N | 활성 상태 필터 |
| `projectName` | string | N | 프로젝트명 검색 (부분 일치) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "instanceId": "inst-001",
      "projectName": "jabis-api-gateway",
      "projectPath": "/home/user/projects/jabis-api-gateway",
      "developerId": "user@company.com",
      "orchestraVersion": "1.2.0",
      "isActive": true,
      "firstSeenAt": "2026-01-15T09:00:00Z",
      "lastSeenAt": "2026-02-06T01:00:00Z",
      "sessionCount": 42
    }
  ],
  "total": 5,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.4 개발자 목록 조회

개발자별 통계를 조회합니다. (`orchestra.v_developer_stats` 뷰 기반)

```
GET /api/dashboard/orchestra/developers
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `sort` | string | N | 정렬 기준 (기본값: `total_sessions`). 가능 값: `total_sessions`, `total_tokens`, `total_cost`, `last_seen_at` |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "developerId": "user@company.com",
      "username": "user",
      "gitUserName": "홍길동",
      "gitUserEmail": "user@company.com",
      "totalSessions": 42,
      "completedSessions": 38,
      "failedSessions": 4,
      "totalTokens": 5200000,
      "totalCost": 52.50,
      "avgDurationMs": 720000,
      "firstSeenAt": "2026-01-15T09:00:00Z",
      "lastSeenAt": "2026-02-06T01:00:00Z"
    }
  ],
  "total": 8,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.5 개발자 상세 조회

특정 개발자의 상세 정보와 최근 세션 목록을 조회합니다.

```
GET /api/dashboard/orchestra/developers/{developerId}
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `developerId` | string | 개발자 UUID |

**Response: `200 OK`**

```json
{
  "developer": {
    "id": "uuid",
    "developerId": "user@company.com",
    "username": "user",
    "userDomain": "company.com",
    "gitUserName": "홍길동",
    "gitUserEmail": "user@company.com",
    "totalSessions": 42,
    "completedSessions": 38,
    "failedSessions": 4,
    "totalTokens": 5200000,
    "totalCost": 52.50,
    "avgDurationMs": 720000,
    "firstSeenAt": "2026-01-15T09:00:00Z",
    "lastSeenAt": "2026-02-06T01:00:00Z"
  },
  "machines": [
    {
      "machineId": "machine-001",
      "hostname": "dev-server-01",
      "platform": "linux",
      "sessionCount": 30,
      "lastSeenAt": "2026-02-06T01:00:00Z"
    }
  ],
  "recentSessions": [
    {
      "sessionId": "session-abc-123",
      "projectName": "jabis-api-gateway",
      "requirement": "로그인 페이지에 비밀번호 찾기 기능 추가",
      "status": "completed",
      "resultSuccess": true,
      "totalTokens": 125000,
      "durationMs": 930000,
      "startedAt": "2026-02-06T01:00:00Z"
    }
  ]
}
```

---

### 2.6 머신 목록 조회

머신별 통계를 조회합니다. (`orchestra.v_machine_stats` 뷰 기반)

```
GET /api/dashboard/orchestra/machines
```

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "machineId": "machine-001",
      "hostname": "dev-server-01",
      "platform": "linux",
      "osType": "Linux",
      "cpuModel": "Intel i9-13900K",
      "cpuCores": 24,
      "totalMemoryGb": 64.0,
      "totalSessions": 120,
      "completedSessions": 110,
      "failedSessions": 10,
      "totalTokens": 15000000,
      "totalCost": 150.00,
      "firstSeenAt": "2026-01-10T08:00:00Z",
      "lastSeenAt": "2026-02-06T01:00:00Z"
    }
  ],
  "total": 3,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.7 에러 패턴 목록 조회

수집된 에러 패턴 목록을 조회합니다.

```
GET /api/dashboard/orchestra/errors
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `errorType` | string | N | 에러 타입 필터 (예: `TypeError`, `SyntaxError`) |
| `autoFixable` | boolean | N | 자동 수정 가능 여부 필터 |
| `sort` | string | N | 정렬 기준 (기본값: `occurrence_count`). 가능 값: `occurrence_count`, `last_occurrence`, `best_success_rate` |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "normalizedMessage": "Cannot read property 'x' of undefined",
      "errorType": "TypeError",
      "autoFixable": true,
      "occurrenceCount": 15,
      "bestSuccessRate": 0.8667,
      "firstOccurrence": "2026-01-20T10:00:00Z",
      "lastOccurrence": "2026-02-05T14:30:00Z",
      "solutions": [
        {
          "id": "uuid",
          "description": "null 체크 추가",
          "appliedCount": 12,
          "successCount": 10,
          "successRate": 0.8333
        }
      ]
    }
  ],
  "total": 25,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.8 작업 패턴 목록 조회

수집된 작업 패턴 목록을 조회합니다.

```
GET /api/dashboard/orchestra/tasks
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `category` | string | N | 카테고리 필터 (예: `feature`, `bugfix`, `refactor`) |
| `sort` | string | N | 정렬 기준 (기본값: `occurrence_count`). 가능 값: `occurrence_count`, `success_rate`, `avg_duration_min` |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "uuid",
      "category": "feature",
      "keywords": ["로그인", "인증", "비밀번호"],
      "normalizedRequest": "인증 관련 기능 구현",
      "occurrenceCount": 8,
      "avgComplexity": "medium",
      "avgDurationMin": 12.50,
      "successRate": 0.8750,
      "commonFiles": ["src/services/auth.ts", "src/routes/auth.ts"],
      "firstSeen": "2026-01-20T10:00:00Z",
      "lastSeen": "2026-02-05T14:30:00Z"
    }
  ],
  "total": 18,
  "page": 1,
  "pageSize": 20
}
```

---

### 2.9 세션 삭제

특정 세션과 하위 연관 데이터를 모두 삭제합니다.

```
POST /api/dashboard/orchestra/sessions/{sessionId}/delete
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `sessionId` | string | 세션 ID |

**Request Body:** 없음

**삭제 대상 테이블 (트랜잭션):**

1. `session_error_occurrences` (session_id)
2. `session_task_occurrences` (session_id)
3. `token_usage_history` (session_id)
4. `session_phases` (session_id)
5. `events` (session_id)
6. `sessions` (본체)

**Response: `200 OK`**

```json
{
  "message": "Session and related data deleted",
  "sessionId": "session-abc-123"
}
```

**Response: `404 Not Found`**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Session not found: session-abc-999"
  }
}
```

---

### 2.10 통합 통계 조회

기간별 집계 통계를 조회합니다.

```
GET /api/dashboard/orchestra/stats
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `period` | string | N | 집계 단위 (기본값: `daily`). 가능 값: `daily`, `weekly`, `monthly` |
| `from` | string | N | 시작 날짜 (기본값: 30일 전) |
| `to` | string | N | 종료 날짜 (기본값: 오늘) |

**Response: `200 OK`**

```json
{
  "summary": {
    "totalSessions": 420,
    "completedSessions": 380,
    "failedSessions": 40,
    "successRate": 0.9048,
    "totalTokens": 52000000,
    "totalCost": 520.00,
    "avgTokensPerSession": 123810,
    "avgSessionDurationMin": 12.5,
    "activeDevelopers": 8,
    "activeInstances": 5
  },
  "timeline": [
    {
      "date": "2026-02-06",
      "totalSessions": 15,
      "completedSessions": 14,
      "failedSessions": 1,
      "successRate": 0.9333,
      "totalTokens": 1875000,
      "totalCost": 18.75,
      "complexityDistribution": {
        "simple": 5,
        "medium": 8,
        "complex": 2
      }
    }
  ],
  "categoryDistribution": {
    "feature": 180,
    "bugfix": 120,
    "refactor": 80,
    "test": 40
  }
}
```

---

## 3. Night Builder API

> 기존 운영 API (`NIGHT_BUILDER_API_SPECIFICATION.md`)와 별도로, 대시보드 조회용 API입니다.

### 3.1 기능 요청 목록 조회 (대시보드용)

모든 상태의 기능 요청을 통합 조회합니다.

```
GET /api/dashboard/nbuilder/requests
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | string | N | 상태 필터 (복수 가능, 쉼표 구분). 예: `queued,planning,in_progress` |
| `repositoryUrl` | string | N | 저장소 URL 필터 (부분 일치) |
| `search` | string | N | 제목/설명 검색 (부분 일치) |
| `from` | string | N | 생성일 시작 |
| `to` | string | N | 생성일 종료 |
| `sort` | string | N | 정렬 기준 (기본값: `created_at`). 가능 값: `created_at`, `updated_at`, `priority`, `status` |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

**Response: `200 OK`**

```json
{
  "items": [
    {
      "id": "req-20260203-001",
      "title": "로그인 페이지에 비밀번호 찾기 추가",
      "description": "이메일 인증 기반의 비밀번호 재설정 기능을 추가합니다.",
      "repositoryUrl": "https://github.com/company/my-project.git",
      "branch": "develop",
      "targetBranch": "develop",
      "status": "completed",
      "priority": 1,
      "hasPlan": true,
      "planComplexity": "medium",
      "planApprovedAt": "2026-02-03T12:00:00Z",
      "createdAt": "2026-02-03T10:00:00Z",
      "updatedAt": "2026-02-03T15:30:00Z",
      "lastStatusMessage": "Feature implementation completed",
      "lastResult": {
        "branchName": "feature/req-20260203-001",
        "prUrl": "https://github.com/company/my-project/pull/42",
        "phasesExecuted": 3
      }
    }
  ],
  "total": 15,
  "page": 1,
  "pageSize": 20
}
```

---

### 3.2 기능 요청 상세 조회

특정 기능 요청의 전체 정보 (계획 + 상태 이력 포함)를 조회합니다.

```
GET /api/dashboard/nbuilder/requests/{requestId}
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `requestId` | string | 기능 요청 ID |

**Response: `200 OK`**

```json
{
  "request": {
    "id": "req-20260203-001",
    "title": "로그인 페이지에 비밀번호 찾기 추가",
    "description": "이메일 인증 기반의 비밀번호 재설정 기능을 추가합니다.",
    "prompt": "로그인 페이지에 비밀번호 찾기 링크를 추가하고...",
    "repositoryUrl": "https://github.com/company/my-project.git",
    "branch": "develop",
    "targetBranch": "develop",
    "status": "completed",
    "priority": 1,
    "planApprovedAt": "2026-02-03T12:00:00Z",
    "metadata": {},
    "createdAt": "2026-02-03T10:00:00Z",
    "updatedAt": "2026-02-03T15:30:00Z"
  },
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
  "statusHistory": [
    {
      "id": 1,
      "fromStatus": null,
      "toStatus": "queued",
      "message": null,
      "result": null,
      "changedBy": "user",
      "createdAt": "2026-02-03T10:00:00Z"
    },
    {
      "id": 2,
      "fromStatus": "queued",
      "toStatus": "planning",
      "message": "JABIS Planning Service: 계획 수립 시작",
      "result": null,
      "changedBy": "planning_service",
      "createdAt": "2026-02-03T10:05:00Z"
    },
    {
      "id": 3,
      "fromStatus": "planning",
      "toStatus": "plan_review",
      "message": "Plan submitted for review",
      "result": null,
      "changedBy": "planning_service",
      "createdAt": "2026-02-03T10:10:00Z"
    },
    {
      "id": 4,
      "fromStatus": "plan_review",
      "toStatus": "approved",
      "message": "사용자 승인",
      "result": null,
      "changedBy": "user",
      "createdAt": "2026-02-03T12:00:00Z"
    },
    {
      "id": 5,
      "fromStatus": "approved",
      "toStatus": "in_progress",
      "message": "JABIS Night Builder started processing",
      "result": null,
      "changedBy": "night_builder",
      "createdAt": "2026-02-03T14:00:00Z"
    },
    {
      "id": 6,
      "fromStatus": "in_progress",
      "toStatus": "completed",
      "message": "Feature implementation completed",
      "result": {
        "branchName": "feature/req-20260203-001",
        "prUrl": "https://github.com/company/my-project/pull/42",
        "testResults": [
          { "command": "npm test", "passed": true, "output": "Tests passed" }
        ],
        "phasesExecuted": 3
      },
      "changedBy": "night_builder",
      "createdAt": "2026-02-03T15:30:00Z"
    }
  ]
}
```

---

### 3.3 디버그 태스크 목록 조회 (대시보드용)

모든 상태의 디버그 태스크를 통합 조회합니다.

```
GET /api/dashboard/nbuilder/debug-tasks
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | string | N | 상태 필터 (복수 가능, 쉼표 구분) |
| `repositoryUrl` | string | N | 저장소 URL 필터 (부분 일치) |
| `search` | string | N | 에러 메시지 검색 (부분 일치) |
| `from` | string | N | 생성일 시작 |
| `to` | string | N | 생성일 종료 |
| `sort` | string | N | 정렬 기준 (기본값: `created_at`). 가능 값: `created_at`, `updated_at`, `priority`, `status` |
| `order` | `asc` / `desc` | N | 정렬 방향 (기본값: `desc`) |

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
      "status": "completed",
      "priority": 2,
      "createdAt": "2026-02-03T14:30:00Z",
      "updatedAt": "2026-02-03T15:00:00Z",
      "lastStatusMessage": "Bug fix applied successfully",
      "lastResult": {
        "branchName": "fix/debug-20260203-001"
      }
    }
  ],
  "total": 8,
  "page": 1,
  "pageSize": 20
}
```

---

### 3.4 디버그 태스크 상세 조회

특정 디버그 태스크의 전체 정보 (상태 이력 포함)를 조회합니다.

```
GET /api/dashboard/nbuilder/debug-tasks/{taskId}
```

**Path Parameters:**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| `taskId` | string | 디버그 태스크 ID |

**Response: `200 OK`**

```json
{
  "task": {
    "id": "debug-20260203-001",
    "errorMessage": "TypeError: Cannot read property 'name' of undefined",
    "stackTrace": "at UserService.getUser (src/services/user.ts:42:18)\nat async handler (src/routes/users.ts:15:20)",
    "repositoryUrl": "https://github.com/company/my-project.git",
    "branch": "develop",
    "filePath": "src/services/user.ts",
    "context": "사용자 조회 시 탈퇴한 사용자에 대해 null 체크 누락",
    "status": "completed",
    "priority": 2,
    "metadata": {},
    "createdAt": "2026-02-03T14:30:00Z",
    "updatedAt": "2026-02-03T15:00:00Z"
  },
  "statusHistory": [
    {
      "id": 1,
      "fromStatus": null,
      "toStatus": "queued",
      "message": null,
      "result": null,
      "changedBy": "system",
      "createdAt": "2026-02-03T14:30:00Z"
    },
    {
      "id": 2,
      "fromStatus": "queued",
      "toStatus": "in_progress",
      "message": "JABIS Auto Debugger started processing",
      "result": null,
      "changedBy": "auto_debugger",
      "createdAt": "2026-02-03T14:35:00Z"
    },
    {
      "id": 3,
      "fromStatus": "in_progress",
      "toStatus": "completed",
      "message": "Bug fix applied successfully",
      "result": {
        "branchName": "fix/debug-20260203-001"
      },
      "changedBy": "auto_debugger",
      "createdAt": "2026-02-03T15:00:00Z"
    }
  ]
}
```

---

### 3.5 Night Builder 통합 통계 조회

기능 요청 + 디버그 태스크의 통합 통계를 조회합니다.

```
GET /api/dashboard/nbuilder/stats
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `from` | string | N | 시작 날짜 (기본값: 30일 전) |
| `to` | string | N | 종료 날짜 (기본값: 오늘) |

**Response: `200 OK`**

```json
{
  "requests": {
    "total": 50,
    "byStatus": {
      "queued": 3,
      "planning": 1,
      "plan_review": 2,
      "approved": 1,
      "in_progress": 1,
      "completed": 35,
      "failed": 5,
      "cancelled": 2
    },
    "avgCompletionTimeMs": 5400000,
    "successRate": 0.8750,
    "complexityDistribution": {
      "low": 15,
      "medium": 25,
      "high": 10
    }
  },
  "debugTasks": {
    "total": 30,
    "byStatus": {
      "queued": 2,
      "in_progress": 1,
      "completed": 24,
      "failed": 3
    },
    "avgCompletionTimeMs": 1800000,
    "successRate": 0.8889
  },
  "timeline": [
    {
      "date": "2026-02-06",
      "requestsCreated": 2,
      "requestsCompleted": 3,
      "requestsFailed": 0,
      "debugTasksCreated": 1,
      "debugTasksCompleted": 2,
      "debugTasksFailed": 0
    }
  ]
}
```

---

## 4. 에러 응답

### 에러 응답 형식

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Session not found: session-abc-999"
  }
}
```

### HTTP 상태 코드

| 코드 | 의미 | 설명 |
|------|------|------|
| `200` | 성공 | 정상 조회 |
| `400` | 잘못된 요청 | 잘못된 파라미터 |
| `401` | 인증 실패 | 토큰 미제공 또는 만료 |
| `403` | 권한 없음 | 대시보드 접근 권한 없음 |
| `404` | 리소스 없음 | 요청한 세션/요청/태스크가 존재하지 않음 |
| `500` | 서버 오류 | 내부 오류 |

---

## API 요약 테이블

| # | Method | 경로 | 설명 |
|---|--------|------|------|
| 1 | `GET` | `/api/dashboard/orchestra/sessions` | Orchestra 세션 목록 조회 |
| 2 | `GET` | `/api/dashboard/orchestra/sessions/{sessionId}` | Orchestra 세션 상세 (phases, tokens, errors 포함) |
| 3 | `POST` | `/api/dashboard/orchestra/sessions/{sessionId}/delete` | Orchestra 세션 + 하위 데이터 삭제 |
| 4 | `GET` | `/api/dashboard/orchestra/instances` | Orchestra 인스턴스 목록 |
| 5 | `GET` | `/api/dashboard/orchestra/developers` | 개발자 목록 + 통계 |
| 6 | `GET` | `/api/dashboard/orchestra/developers/{developerId}` | 개발자 상세 + 머신 + 최근 세션 |
| 7 | `GET` | `/api/dashboard/orchestra/machines` | 머신 목록 + 통계 |
| 8 | `GET` | `/api/dashboard/orchestra/errors` | 에러 패턴 목록 + 솔루션 |
| 9 | `GET` | `/api/dashboard/orchestra/tasks` | 작업 패턴 목록 |
| 10 | `GET` | `/api/dashboard/orchestra/stats` | Orchestra 기간별 통합 통계 |
| 11 | `GET` | `/api/dashboard/nbuilder/requests` | 기능 요청 목록 (전체 상태) |
| 12 | `GET` | `/api/dashboard/nbuilder/requests/{requestId}` | 기능 요청 상세 (계획 + 이력) |
| 13 | `GET` | `/api/dashboard/nbuilder/debug-tasks` | 디버그 태스크 목록 (전체 상태) |
| 14 | `GET` | `/api/dashboard/nbuilder/debug-tasks/{taskId}` | 디버그 태스크 상세 (이력 포함) |
| 15 | `GET` | `/api/dashboard/nbuilder/stats` | Night Builder 통합 통계 |

---

## DB 테이블 매핑

### AI Orchestra (스키마: `orchestra`)

| API | 주요 테이블 |
|-----|------------|
| sessions | `sessions`, `developers`, `machines` |
| sessions/{id} | `sessions`, `session_phases`, `token_usage_history`, `session_error_occurrences`, `error_patterns` |
| sessions/{id}/delete | `sessions`, `session_phases`, `token_usage_history`, `session_error_occurrences`, `session_task_occurrences`, `events` |
| instances | `instances` |
| developers | `v_developer_stats` (뷰) |
| developers/{id} | `developers`, `developer_machines`, `machines`, `sessions` |
| machines | `v_machine_stats` (뷰) |
| errors | `error_patterns`, `error_solutions` |
| tasks | `task_patterns` |
| stats | `company_stats`, `sessions` (실시간 집계) |

### Night Builder (스키마: `nbuilder`)

| API | 주요 테이블 |
|-----|------------|
| requests | `feature_requests`, `implementation_plans`, `status_histories` |
| requests/{id} | `feature_requests`, `implementation_plans`, `plan_steps`, `status_histories` |
| debug-tasks | `debug_tasks`, `status_histories` |
| debug-tasks/{id} | `debug_tasks`, `status_histories` |
| stats | `feature_requests`, `debug_tasks`, `implementation_plans` |
