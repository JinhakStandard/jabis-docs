# 엔드포인트 관리 API 명세

> Gateway에 등록된 동적 API 엔드포인트를 CRUD 관리하는 REST API.
> jabis-producer의 API 빌더 페이지에서 사용합니다.

---

## 공통 사항

### Base URL
```
http://{host}:{port}/api/endpoints
```

### 인증
모든 요청에 Bearer 토큰 필수.
```
Authorization: Bearer {token}
```
jabis-cert를 통한 JWT 검증. 허용 역할: `producer`, `developer`, `admin`, `superadmin`

### HTTP 메서드
- `GET` — 조회
- `POST` — 생성, 수정, 삭제 (action 패턴)

---

## 1. 엔드포인트 목록 조회

```
GET /api/endpoints
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `status` | string | N | 상태 필터 (`active` / `deprecated` / `disabled`) |
| `method` | string | N | 메서드 필터 (`GET` / `POST`) |

**Response: `200 OK`**
```json
{
  "success": true,
  "data": [
    {
      "id": "ep-001",
      "name": "user-list",
      "description": "사용자 목록 조회",
      "method": "GET",
      "pathPattern": "/api/users",
      "query": "SELECT id, name, email FROM users WHERE status = :status",
      "parameters": [
        { "name": "status", "type": "string", "required": false, "default": "active" }
      ],
      "authRequired": true,
      "requiredScopes": ["user:read"],
      "allowPendingAccess": false,
      "status": "active",
      "ownerId": "user-001",
      "createdAt": "2026-02-10T10:00:00Z",
      "updatedAt": "2026-02-10T10:00:00Z"
    }
  ],
  "total": 15
}
```

---

## 2. 엔드포인트 단건 조회

```
GET /api/endpoints/{id}
```

**Response: `200 OK`**
```json
{
  "success": true,
  "data": { /* ApiEndpoint 객체 */ }
}
```

---

## 3. 엔드포인트 생성

```
POST /api/endpoints
```

**Request Body:**
```json
{
  "name": "user-list",
  "description": "사용자 목록 조회",
  "method": "GET",
  "pathPattern": "/api/users",
  "query": "SELECT id, name, email FROM users WHERE status = :status",
  "parameters": [
    { "name": "status", "type": "string", "required": false, "default": "active", "description": "사용자 상태" }
  ],
  "authRequired": true,
  "requiredScopes": ["user:read"],
  "allowPendingAccess": false,
  "status": "active"
}
```

**필수 필드**: `name`, `method`, `pathPattern`, `query`

**SQL 쿼리 제약사항:**
- `SELECT` 또는 `WITH` 절로만 시작 허용
- INSERT, UPDATE, DELETE, DROP, ALTER, CREATE 등 DML/DDL 차단
- 다중 쿼리 (세미콜론 분리) 차단

**Response: `201 Created`**
```json
{
  "success": true,
  "data": { /* 생성된 ApiEndpoint 객체 */ }
}
```

**에러:**
- `400` — 필수 필드 누락, SQL 검증 실패
- `409` — 동일 method/pathPattern 중복

---

## 4. 엔드포인트 수정

```
POST /api/endpoints/{id}
```

**Request Body:**
```json
{
  "action": "update",
  "data": {
    "name": "updated-name",
    "description": "변경된 설명",
    "query": "SELECT ...",
    "parameters": [...],
    "status": "deprecated"
  }
}
```

**Response: `200 OK`**

---

## 5. 엔드포인트 삭제

```
POST /api/endpoints/{id}
```

**Request Body:**
```json
{
  "action": "delete"
}
```

**Response: `200 OK`**
```json
{
  "success": true,
  "message": "엔드포인트가 삭제되었습니다."
}
```

---

## 6. 쿼리 테스트 실행

```
POST /api/endpoints/{id}/test
```

**Request Body:** (테스트 파라미터)
```json
{
  "status": "active",
  "department": "개발팀"
}
```

**Response: `200 OK`**
```json
{
  "success": true,
  "data": {
    "rows": [...],
    "rowCount": 5,
    "performance": {
      "queryDuration": 45,
      "parameterCount": 2,
      "rowsReturned": 5
    }
  }
}
```

---

## 에러 응답

```json
{
  "success": false,
  "error": {
    "code": "BAD_REQUEST | NOT_FOUND | CONFLICT | QUERY_VALIDATION_FAILED | INTERNAL_ERROR",
    "message": "에러 상세 메시지"
  }
}
```
