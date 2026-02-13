# API Tasks API 명세

> 프로젝트 단위 API 요청 관리 시스템.
> jabis-producer의 API 요청 페이지에서 사용합니다.
> `nbuilder.dev_tasks`(nightbuilder용)와 별개로 `gateway.api_tasks` 테이블을 사용합니다.

---

## 공통 사항

### Base URL
```
http://{host}:{port}/api/api-tasks
```

### 인증
모든 요청에 Bearer 토큰 필수.
```
Authorization: Bearer {token}
```
jabis-cert를 통한 JWT 검증. 허용 역할: `producer`, `developer`, `admin`, `superadmin`

### 접근 제어
- `admin`, `superadmin`: 전체 요청 조회 가능
- `producer`, `developer`: 본인이 제출한 요청만 조회

### HTTP 메서드
- `GET` — 조회
- `POST` — 생성, 상태 변경, 삭제

---

## 1. 작업 생성

```
POST /api/api-tasks
```

**Request Body:**
```json
{
  "title": "부서별 사용자 조회 API",
  "description": "부서 코드로 해당 부서 사용자 목록 조회",
  "spec": "## 요구사항\n- 부서 코드(dept_code) 파라미터\n- 이름 검색\n- 페이지네이션",
  "specFormat": "markdown",
  "project": "jabis-producer",
  "priority": 5,
  "endpoints": [
    { "method": "GET", "path": "/api/users/by-dept", "description": "부서별 사용자 조회" }
  ],
  "metadata": {}
}
```

**필수 필드**: `title`, `spec`

`requested_by`는 JWT에서 자동 추출됩니다. `project`는 프론트엔드에서 선택하며, 생략 시 기본값 `jabis-producer`.

**Response: `201 Created`**
```json
{
  "success": true,
  "data": {
    "id": "atask-20260213-a1b2",
    "title": "부서별 사용자 조회 API",
    "requestedBy": "user@jinhakapply.com",
    "project": "jabis-producer",
    "status": "pending",
    ...
  }
}
```

---

## 2. 작업 목록 조회

```
GET /api/api-tasks
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | string | N | 상태 필터 (`pending`/`in_progress`/`completed`/`verified`/`rejected`) |
| `project` | string | N | 프로젝트 필터 (예: `jabis-producer`, `jabis`, `jabis-api-gateway`) |
| `requestedBy` | string | N | 요청자 필터 (admin/superadmin만 사용 가능) |
| `sort` | string | N | 정렬 기준 (`priority`/`createdAt`/`updatedAt`, 기본값: `priority`) |
| `order` | string | N | 정렬 방향 (`asc`/`desc`, 기본값: `desc`) |
| `page` | number | N | 페이지 번호 (기본값: 1) |
| `pageSize` | number | N | 페이지 크기 (기본값: 20, 최대: 100) |

**Response: `200 OK`**
```json
{
  "success": true,
  "data": {
    "items": [ ... ],
    "total": 15,
    "page": 1,
    "pageSize": 20
  }
}
```

---

## 3. 작업 단건 조회

```
GET /api/api-tasks/{taskId}
```

**Response: `200 OK`**
```json
{
  "success": true,
  "data": { /* ApiTask 객체 */ }
}
```

---

## 4. 상태 변경

```
POST /api/api-tasks/{taskId}/status
```

**Request Body:**
```json
{
  "status": "in_progress",
  "assignee": "developer@jinhakapply.com",
  "resultNote": "구현 시작"
}
```

**상태 전이 규칙:**
```
pending      → in_progress, rejected
in_progress  → completed, pending
completed    → verified, rejected, in_progress
verified     → (최종 상태)
rejected     → pending
```

**Response: `204 No Content`**

---

## 5. 작업 삭제

```
POST /api/api-tasks/{taskId}/delete
```

`verified` 또는 `rejected` 상태에서만 삭제 가능.

**Response: `204 No Content`**

---

## 데이터 모델: ApiTask

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | string | PK, `atask-{YYYYMMDD}-{rand}` |
| `title` | string | 요청 제목 |
| `description` | string | 설명 (선택) |
| `spec` | string | 요청 명세 |
| `specFormat` | string | `markdown`/`text`/`json` |
| `requestedBy` | string | 요청자 email (JWT 자동 추출) |
| `project` | string | 대상 프로젝트명 (예: `jabis-producer`, `jabis`, `jabis-api-gateway`) |
| `status` | string | `pending`/`in_progress`/`completed`/`verified`/`rejected` |
| `priority` | number | 우선순위 (0: 보통, 5: 높음, 10: 긴급) |
| `assignee` | string | 구현 담당자 |
| `endpoints` | array | 생성된 엔드포인트 참조 `[{method, path, description}]` |
| `resultNote` | string | 처리 결과 메모 |
| `metadata` | object | 자유 메타데이터 (JSONB) |
| `createdAt` | string (ISO 8601) | 생성일 |
| `updatedAt` | string (ISO 8601) | 수정일 |
| `completedAt` | string (ISO 8601) | 완료일 |

---

## 에러 응답

```json
{
  "success": false,
  "error": {
    "code": "BAD_REQUEST | NOT_FOUND | CONFLICT | INTERNAL_ERROR",
    "message": "에러 상세 메시지"
  }
}
```

---

## dev_tasks와의 차이

| 항목 | `nbuilder.dev_tasks` | `gateway.api_tasks` |
|------|---------------------|---------------------|
| 스키마 | nbuilder (nightbuilder용) | gateway (API 관리용) |
| 인증 | X-Dev-Tasks-Key (API 키) | JWT Bearer (사용자 인증) |
| 추적 단위 | 프로젝트 (`source_project`) | 프로젝트 (`project`) + 요청자 (`requested_by`) |
| 용도 | 기계 간 요청 (nightbuilder, 자동화) | 사용자 UI 요청 (jabis-producer) |
