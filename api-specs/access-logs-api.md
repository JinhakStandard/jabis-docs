# Access Logs API 명세

> Gateway에 기록된 API 접근 로그를 조회하는 REST API.
> jabis-producer의 API 로그 페이지에서 사용합니다.

---

## 공통 사항

### Base URL
```
http://{host}:{port}/api/access-logs
```

### 인증
모든 요청에 Bearer 토큰 필수.
```
Authorization: Bearer {token}
```
jabis-cert를 통한 JWT 검증. 허용 역할: `producer`, `developer`, `admin`, `superadmin`

### HTTP 메서드
- `GET` — 조회

---

## 1. 접근 로그 목록 조회

```
GET /api/access-logs
```

**Query Parameters:**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `method` | string | N | HTTP 메서드 필터 (`GET` / `POST`) |
| `path` | string | N | 경로 부분 일치 검색 (ILIKE) |
| `startDate` | string (ISO 8601) | N | 조회 시작일시 |
| `endDate` | string (ISO 8601) | N | 조회 종료일시 |
| `sort` | string | N | 정렬 기준 (`requested_at` / `response_time_ms` / `response_status`, 기본값: `requested_at`) |
| `order` | string | N | 정렬 방향 (`asc` / `desc`, 기본값: `desc`) |
| `limit` | number | N | 조회 개수 (기본값: 100, 최대: 500) |
| `offset` | number | N | 건너뛸 개수 (기본값: 0) |

**Response: `200 OK`**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "request_id": "uuid-1234",
        "client_id": "user-001",
        "endpoint_id": "ep-001",
        "method": "GET",
        "path": "/api/users",
        "query_params": { "status": "active" },
        "response_status": 200,
        "response_time_ms": 45,
        "permission_status": "allowed",
        "client_ip": "192.168.1.100",
        "user_agent": "Mozilla/5.0 ...",
        "requested_at": "2026-02-11T09:00:12Z"
      }
    ]
  }
}
```

---

## 2. 접근 로그 단건 조회

```
GET /api/access-logs/{requestId}
```

**Response: `200 OK`**
```json
{
  "success": true,
  "data": { /* ApiAccessLog 객체 */ }
}
```

**에러:**
- `404` — 해당 requestId의 로그가 존재하지 않음

---

## 데이터 모델: ApiAccessLog

| 필드 | 타입 | 설명 |
|------|------|------|
| `request_id` | string (UUID) | 요청 고유 ID |
| `client_id` | string | 요청자 ID (JWT에서 추출) |
| `endpoint_id` | string | 호출된 엔드포인트 ID |
| `method` | string | HTTP 메서드 (`GET` / `POST`) |
| `path` | string | 요청 경로 (쿼리스트링 제외) |
| `query_params` | object | 쿼리 파라미터 (JSONB) |
| `response_status` | number | HTTP 응답 상태 코드 |
| `response_time_ms` | number | 응답 소요 시간 (ms) |
| `permission_status` | string | 권한 상태 (`pending` / `allowed` / `blocked`) |
| `client_ip` | string | 클라이언트 IP |
| `user_agent` | string | User-Agent 헤더 |
| `requested_at` | string (ISO 8601) | 요청 시각 |

---

## 에러 응답

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND | INTERNAL_ERROR",
    "message": "에러 상세 메시지"
  }
}
```

---

## 관련 테이블

- `gateway.api_access_logs` — 접근 로그 저장 테이블
- 로그는 동적 API 엔드포인트(`/api/*` 와일드카드) 호출 시 자동 기록됨
