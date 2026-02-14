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

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|----------|------|------|--------|------|
| `page` | number | N | `1` | 페이지 번호 (1부터) |
| `pageSize` | number | N | `50` | 페이지 크기 (1~500) |
| `method` | string | N | — | HTTP 메서드 필터 (`GET` / `POST`) |
| `path` | string | N | — | 경로 부분 일치 검색 (ILIKE) |
| `startDate` | string (ISO 8601) | N | — | 조회 시작일시 |
| `endDate` | string (ISO 8601) | N | — | 조회 종료일시 |
| `sort` | string | N | `requested_at` | 정렬 기준 (`requested_at` / `response_time_ms` / `response_status`) |
| `order` | string | N | `desc` | 정렬 방향 (`asc` / `desc`) |

**Response: `200 OK`**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "requestId": "uuid-1234",
        "clientId": "user-001",
        "endpointId": "ep-001",
        "method": "GET",
        "path": "/api/users",
        "queryParams": { "status": "active" },
        "responseStatus": 200,
        "responseTimeMs": 45,
        "permissionStatus": "allowed",
        "clientIp": "192.168.1.100",
        "userAgent": "Mozilla/5.0 ...",
        "requestedAt": "2026-02-11T09:00:12.000Z"
      }
    ],
    "total": 150,
    "page": 1,
    "pageSize": 50
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

> API 응답은 camelCase, DB 컬럼은 snake_case. logRepository의 mapRow가 변환 담당.

| 응답 필드 (camelCase) | DB 컬럼 (snake_case) | 타입 | 설명 |
|----------------------|---------------------|------|------|
| `id` | `id` | number | 자동 증가 PK |
| `requestId` | `request_id` | string | 요청 고유 ID |
| `clientId` | `client_id` | string | 요청자 ID (JWT에서 추출) |
| `endpointId` | `endpoint_id` | string | 호출된 엔드포인트 ID |
| `method` | `method` | string | HTTP 메서드 (`GET` / `POST`) |
| `path` | `path` | string | 요청 경로 (쿼리스트링 제외) |
| `queryParams` | `query_params` | object | 쿼리 파라미터 (JSONB) |
| `responseStatus` | `response_status` | number | HTTP 응답 상태 코드 |
| `responseTimeMs` | `response_time_ms` | number | 응답 소요 시간 (ms) |
| `permissionStatus` | `permission_status` | string | 권한 상태 (`pending` / `allowed` / `blocked`) |
| `clientIp` | `client_ip` | string | 클라이언트 IP |
| `userAgent` | `user_agent` | string | User-Agent 헤더 |
| `requestedAt` | `requested_at` | string (ISO 8601) | 요청 시각 |

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
