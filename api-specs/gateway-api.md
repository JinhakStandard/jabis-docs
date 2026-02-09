# JABIS API 엔드포인트 명세

> AI Orchestra 클라이언트(`JabisClient`)가 호출하는 API 명세.
> JABIS 서버(별도 프로젝트)에서 이 엔드포인트를 구현해야 합니다.

---

## 공통 사항

### Base URL
```
http(s)://<jabis-server-host>:<port>
```

### 공통 요청 헤더
| 헤더 | 필수 | 설명 |
|------|------|------|
| `Content-Type` | O | `application/json` |
| `Authorization` | △ | `Bearer <api_key>` (API 키 인증 시) |
| `X-Instance-ID` | O | 클라이언트 인스턴스 고유 ID (예: `jabis-1706000000000-abc123def`) |
| `X-Orchestra-Version` | O | ai-orchestra 버전 (예: `1.0.0`) |

### 공통 응답 형식
성공 시:
```json
{
  "success": true,
  "message": "처리 결과 메시지"
}
```
에러 시:
```json
{
  "success": false,
  "message": "에러 설명",
  "error": "상세 에러"
}
```

### 공통 HTTP 상태 코드
| 코드 | 설명 |
|------|------|
| 200 | 성공 |
| 400 | 잘못된 요청 (필수 필드 누락, JSON 파싱 실패) |
| 401 | 인증 실패 (API 키 없거나 유효하지 않음) |
| 500 | 서버 내부 오류 |

### 클라이언트 타임아웃 참고
| 엔드포인트 | 클라이언트 타임아웃 |
|-----------|-------------------|
| `GET /health` | 5초 |
| `POST /api/v1/orchestra/events` | 10초 |
| `POST /api/v1/orchestra/sync` | 15초 |

---

## 1. GET /health

헬스체크. 서버 연결 확인용.

> **인증 불필요**: 이 엔드포인트는 `Authorization` 헤더 없이 접근 가능해야 합니다.
> 클라이언트가 API 키 설정 전에도 연결 확인용으로 호출합니다.

### 요청
```
GET /health
```

### 응답 (200 OK)
```json
{
  "status": "healthy",
  "timestamp": "2026-02-03T10:00:00.000Z"
}
```

### 사용처
- `JabisClient.checkConnection()` — 서버 연결 여부 확인
- `JabisClient.connect()` — 초기 연결 시도

---

## 2. POST /api/v1/orchestra/events

모든 세션 이벤트를 수신하는 단일 엔드포인트.
`type` 필드로 이벤트 종류를 구분합니다.

### 요청
```
POST /api/v1/orchestra/events
Content-Type: application/json
```

### 이벤트 타입별 페이로드

> **참고**: 클라이언트 타입 정의에는 `analytics_sync` 타입도 존재하지만,
> 분석 동기화는 이 events 엔드포인트가 아닌 별도의 `POST /api/v1/orchestra/sync` 엔드포인트를 사용합니다.
> 따라서 이 엔드포인트에서 처리할 `type`은 `session_start`, `session_update`, `session_end` 3종류입니다.

---

### 2-1. session_start

세션 시작 시 전송.

```json
{
  "type": "session_start",
  "timestamp": "2026-02-03T10:00:00.000Z",
  "source": {
    "instanceId": "jabis-1706000000000-abc123def",
    "projectName": "my-project",
    "version": "1.0.0"
  },
  "data": {
    "sessionId": "session-1706000000000",
    "instanceId": "jabis-1706000000000-abc123def",
    "projectName": "my-project",
    "projectPath": "/home/user/projects/my-project",
    "developerId": "dev-anonymous-hash",
    "startTime": "2026-02-03T10:00:00.000Z",
    "status": "running",
    "requirement": "로그인 API에 JWT 인증 추가",
    "complexity": "medium",
    "executionMode": "auto",
    "phases": [
      {
        "id": "phase-1",
        "title": "인증 미들웨어 구현",
        "status": "pending",
        "duration": null
      },
      {
        "id": "phase-2",
        "title": "JWT 토큰 발급/검증",
        "status": "pending",
        "duration": null
      }
    ]
  }
}
```

**`data` 필드 설명:**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `sessionId` | string | O | 클라이언트 세션 고유 ID |
| `instanceId` | string | O | 인스턴스 ID |
| `projectName` | string | O | 프로젝트 이름 |
| `projectPath` | string | O | 프로젝트 절대 경로 |
| `developerId` | string | △ | 익명 개발자 식별자 |
| `startTime` | ISO8601 | O | 세션 시작 시간 |
| `status` | enum | O | `"running"` (시작 시 항상 `running`) |
| `requirement` | string | O | 사용자 요구사항 원문 |
| `complexity` | enum | O | `"simple"` \| `"medium"` \| `"complex"` |
| `executionMode` | enum | O | `"auto"` \| `"simple"` \| `"medium"` \| `"full"` \| `"run"` \| `"fix"` \| `"task"` \| `"build"` \| `"analyze"` |
| `phases` | array | O | Phase 목록 (id, title, status, duration). status 가능 값: `"pending"` \| `"running"` \| `"success"` \| `"failed"` \| `"blocked"` \| `"skipped"` |

### 응답 (200)
```json
{
  "success": true,
  "message": "Session started"
}
```

---

### 2-2. session_update

세션 진행 중 주기적으로 전송 (토큰 사용량, Phase 상태 변경 등).

```json
{
  "type": "session_update",
  "timestamp": "2026-02-03T10:05:00.000Z",
  "source": {
    "instanceId": "jabis-1706000000000-abc123def",
    "projectName": "my-project",
    "version": "1.0.0"
  },
  "data": {
    "sessionId": "session-1706000000000",
    "timestamp": "2026-02-03T10:05:00.000Z",
    "tokenUsage": {
      "totalInputTokens": 15000,
      "totalOutputTokens": 8000,
      "totalTokens": 23000,
      "totalCost": 0.045,
      "usageHistory": [
        {
          "inputTokens": 5000,
          "outputTokens": 3000,
          "totalTokens": 8000,
          "estimatedCost": 0.015,
          "timestamp": "2026-02-03T10:02:00.000Z",
          "agent": "coder",
          "phaseId": "phase-1"
        }
      ]
    },
    "errorPatterns": [],
    "taskPatterns": [],
    "session": {
      "sessionId": "session-1706000000000",
      "instanceId": "jabis-1706000000000-abc123def",
      "projectName": "my-project",
      "projectPath": "/home/user/projects/my-project",
      "startTime": "2026-02-03T10:00:00.000Z",
      "status": "running",
      "requirement": "로그인 API에 JWT 인증 추가",
      "complexity": "medium",
      "executionMode": "auto",
      "phases": [
        {
          "id": "phase-1",
          "title": "인증 미들웨어 구현",
          "status": "running",
          "duration": 120000
        },
        {
          "id": "phase-2",
          "title": "JWT 토큰 발급/검증",
          "status": "pending",
          "duration": null
        }
      ]
    }
  }
}
```

**`data` 필드 설명:**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `sessionId` | string | O | 세션 ID |
| `timestamp` | ISO8601 | O | 업데이트 시점 |
| `tokenUsage` | object | O | 토큰 사용량 누적 (아래 참조) |
| `errorPatterns` | array | O | 에러 패턴 요약 (보통 빈 배열) |
| `taskPatterns` | array | O | 작업 패턴 요약 (보통 빈 배열) |
| `session` | object | O | 최신 세션 상태 전체 (session_start의 data와 동일 구조) |

**`tokenUsage` 구조:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `totalInputTokens` | number | 누적 입력 토큰 |
| `totalOutputTokens` | number | 누적 출력 토큰 |
| `totalTokens` | number | 누적 총 토큰 |
| `totalCost` | number | 누적 비용 (USD) |
| `usageHistory` | array | 개별 사용 기록 배열 |

**`usageHistory[]` 항목:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `inputTokens` | number | 이 호출의 입력 토큰 |
| `outputTokens` | number | 이 호출의 출력 토큰 |
| `totalTokens` | number | 이 호출의 총 토큰 |
| `estimatedCost` | number | 이 호출의 추정 비용 (USD) |
| `timestamp` | ISO8601 | 기록 시점 |
| `agent` | string? | 에이전트 이름 (`planner`, `coder`, `debugger`, `reviewer`) |
| `phaseId` | string? | Phase ID |

### 응답 (200)
```json
{
  "success": true,
  "message": "Session updated"
}
```

---

### 2-3. session_end

세션 종료 시 전송. 최종 결과, 에러 패턴, 작업 패턴 포함.

```json
{
  "type": "session_end",
  "timestamp": "2026-02-03T10:15:00.000Z",
  "source": {
    "instanceId": "jabis-1706000000000-abc123def",
    "projectName": "my-project",
    "version": "1.0.0"
  },
  "data": {
    "sessionId": "session-1706000000000",
    "timestamp": "2026-02-03T10:15:00.000Z",
    "tokenUsage": {
      "totalInputTokens": 45000,
      "totalOutputTokens": 22000,
      "totalTokens": 67000,
      "totalCost": 0.125,
      "usageHistory": [
        {
          "inputTokens": 5000,
          "outputTokens": 3000,
          "totalTokens": 8000,
          "estimatedCost": 0.015,
          "timestamp": "2026-02-03T10:02:00.000Z",
          "agent": "coder",
          "phaseId": "phase-1"
        },
        {
          "inputTokens": 8000,
          "outputTokens": 4000,
          "totalTokens": 12000,
          "estimatedCost": 0.022,
          "timestamp": "2026-02-03T10:08:00.000Z",
          "agent": "debugger",
          "phaseId": "phase-1"
        }
      ]
    },
    "errorPatterns": [
      {
        "normalizedMessage": "TypeError: Cannot read properties of undefined (reading '<PROP>')",
        "errorType": "TypeError",
        "occurrenceCount": 3,
        "autoFixable": true,
        "bestSolutionSuccessRate": 0.85
      }
    ],
    "taskPatterns": [
      {
        "category": "feature_add",
        "keywords": ["JWT", "인증", "미들웨어", "토큰"],
        "occurrenceCount": 1,
        "avgComplexity": "medium",
        "successRate": 1.0
      }
    ],
    "session": {
      "sessionId": "session-1706000000000",
      "instanceId": "jabis-1706000000000-abc123def",
      "projectName": "my-project",
      "projectPath": "/home/user/projects/my-project",
      "developerId": "dev-anonymous-hash",
      "startTime": "2026-02-03T10:00:00.000Z",
      "endTime": "2026-02-03T10:15:00.000Z",
      "status": "completed",
      "requirement": "로그인 API에 JWT 인증 추가",
      "complexity": "medium",
      "executionMode": "auto",
      "phases": [
        {
          "id": "phase-1",
          "title": "인증 미들웨어 구현",
          "status": "success",
          "duration": 480000
        },
        {
          "id": "phase-2",
          "title": "JWT 토큰 발급/검증",
          "status": "success",
          "duration": 320000
        }
      ],
      "result": {
        "success": true,
        "errorMessage": null,
        "filesCreated": 3,
        "filesModified": 5
      }
    }
  }
}
```

**추가 필드 (`session_update` 대비):**

| 필드 | 타입 | 설명 |
|------|------|------|
| `session.endTime` | ISO8601 | 세션 종료 시간 |
| `session.status` | enum | `"completed"` \| `"failed"` \| `"cancelled"` |
| `session.result` | object | 최종 결과 (아래 참조) |
| `errorPatterns` | array | 에러 패턴 요약 배열 |
| `taskPatterns` | array | 작업 패턴 요약 배열 |

**`session.result` 구조:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `success` | boolean | 성공 여부 |
| `errorMessage` | string? | 실패 시 에러 메시지 |
| `filesCreated` | number | 생성된 파일 수 |
| `filesModified` | number | 수정된 파일 수 |

**`errorPatterns[]` 항목:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `normalizedMessage` | string | 정규화된 에러 메시지 (변수/경로 제거됨) |
| `errorType` | string | 에러 유형 (`TypeError`, `SyntaxError` 등) |
| `occurrenceCount` | number | 이 세션에서 발생 횟수 |
| `autoFixable` | boolean | 자동 수정 가능 여부 |
| `bestSolutionSuccessRate` | number | 최고 해결 성공률 (0-1) |

**`taskPatterns[]` 항목:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `category` | enum | 작업 카테고리 (아래 참조) |
| `keywords` | string[] | 주요 키워드 |
| `occurrenceCount` | number | 발생 횟수 |
| `avgComplexity` | enum | `"simple"` \| `"medium"` \| `"complex"` |
| `successRate` | number | 성공률 (0-1) |

**작업 카테고리 목록:**
`bug_fix`, `feature_add`, `refactoring`, `performance`, `testing`, `documentation`, `security`, `ui_ux`, `api`, `database`, `config`, `other`

### 응답 (200)
```json
{
  "success": true,
  "message": "Session ended"
}
```

---

## 3. POST /api/v1/orchestra/sync

분석 데이터 동기화. 로컬 패턴을 서버에 push하고, 회사 전체 집계 데이터를 pull.

### 요청
```
POST /api/v1/orchestra/sync
Content-Type: application/json
```

```json
{
  "instanceId": "jabis-1706000000000-abc123def",
  "errorPatterns": [
    {
      "normalizedMessage": "TypeError: Cannot read properties of undefined (reading '<PROP>')",
      "errorType": "TypeError",
      "occurrenceCount": 15,
      "autoFixable": true,
      "bestSolutionSuccessRate": 0.85
    }
  ],
  "taskPatterns": [
    {
      "category": "bug_fix",
      "keywords": ["타입", "에러", "undefined"],
      "occurrenceCount": 8,
      "avgComplexity": "simple",
      "successRate": 0.9
    }
  ]
}
```

### 응답 (200)

서버는 회사 전체 집계 데이터를 응답합니다.

```json
{
  "companyPatterns": {
    "topCategories": [
      { "category": "bug_fix", "percentage": 35.5 },
      { "category": "feature_add", "percentage": 28.2 },
      { "category": "refactoring", "percentage": 15.0 }
    ],
    "commonErrors": [
      {
        "normalizedMessage": "TypeError: Cannot read properties of undefined (reading '<PROP>')",
        "errorType": "TypeError",
        "occurrenceCount": 142,
        "autoFixable": true,
        "bestSolutionSuccessRate": 0.88
      }
    ],
    "recommendations": [
      "TypeScript strict 모드 활성화 권장 (타입 에러 45% 감소 효과)",
      "API 작업 시 OpenAPI 스펙 컨텍스트 제공 시 성공률 20% 향상"
    ]
  },
  "benchmarks": {
    "avgTokensPerSession": 45000,
    "avgSessionDuration": 12.5,
    "successRate": 0.82
  },
  "lastSyncTime": "2026-02-03T10:15:00.000Z"
}
```

**응답 필드 설명:**

| 필드 | 타입 | 설명 |
|------|------|------|
| `companyPatterns.topCategories` | array | 상위 작업 카테고리 (percentage 합 = 100) |
| `companyPatterns.commonErrors` | array | 빈출 에러 패턴 (occurrenceCount >= 3) |
| `companyPatterns.recommendations` | string[] | 활성 권장사항 목록 |
| `benchmarks.avgTokensPerSession` | number | 최근 30일 세션당 평균 토큰 |
| `benchmarks.avgSessionDuration` | number | 최근 30일 평균 세션 시간 (분) |
| `benchmarks.successRate` | number | 최근 30일 성공률 (0-1) |
| `lastSyncTime` | ISO8601 | 동기화 시점 |

---

## 참고: jabis-schema.sql 테이블과 API 매핑

| 테이블 | 관련 API | 설명 |
|--------|---------|------|
| `instances` | events (모든 타입) | `X-Instance-ID` 헤더로 자동 upsert |
| `sessions` | events (start/update/end) | `data.session` 또는 `data` 직접 |
| `session_phases` | events (start/update/end) | `data.session.phases` 또는 `data.phases` |
| `token_usage_history` | events (update/end) | `data.tokenUsage.usageHistory` |
| `error_patterns` | events (end), sync | `data.errorPatterns` |
| `error_solutions` | — | 서버 내부 관리 (학습 기반) |
| `session_error_occurrences` | events (end) | `errorPatterns` → 세션 연결 |
| `task_patterns` | events (end), sync | `data.taskPatterns` |
| `session_task_occurrences` | events (end) | `taskPatterns` → 세션 연결 |
| `events` | events (모든 타입) | 원시 페이로드 JSONB 저장 |
| `company_stats` | — | 서버 내부 주기적 집계 |
| `sync_history` | sync | 동기화 기록 |
| `company_recommendations` | sync (응답) | 서버 내부 관리 |

---

## 참고: 클라이언트 동작 흐름

```
1. 서버 연결 확인
   JabisClient.connect()
   → GET /health (200이면 연결 성공)

2. 세션 시작
   Orchestrator 시작 시
   → POST /api/v1/orchestra/events { type: "session_start", ... }

3. 세션 진행 중 업데이트 (토큰 사용량 변경 시)
   → POST /api/v1/orchestra/events { type: "session_update", ... }

4. 세션 종료
   Orchestrator 완료/실패 시
   → POST /api/v1/orchestra/events { type: "session_end", ... }
   (에러 패턴, 작업 패턴, 최종 결과 포함)

5. 분석 데이터 동기화 (주기적)
   → POST /api/v1/orchestra/sync
   ← 회사 전체 패턴 + 벤치마크 + 권장사항

※ 오프라인 시: 로컬 큐(.orchestra/jabis-queue.json)에 저장 → 연결 복구 시 재전송
※ 재시도: 지수 백오프 (1s, 2s, 4s) 최대 3회
```
