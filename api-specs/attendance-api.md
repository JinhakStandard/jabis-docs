# 근태 관리 API 명세

> **서버**: jabis-api-gateway
> **Base URL**: `/api/attendance`
> **인증**: JWT (jabis-cert 연동)
> **HTTP 메서드**: GET (조회), POST (변경 — action 필드로 구분)

---

## 공통 사항

### 인증 흐름

```
JWT Bearer → email 추출 → organization.users 조회/생성 → employeeId 매핑
```

- 모든 엔드포인트에 JWT 인증 필수
- JWT에서 `email` 또는 `preferred_username` 추출
- `organization.users` 테이블에서 `employeeId` 매핑 필요
- `employeeId`가 없으면 `403 NO_EMPLOYEE` 반환

### HR 전용 엔드포인트

`/admin`, `/stats` 엔드포인트는 HR 관리자 권한 필수:
- `organization.user_roles`에서 `hr` 또는 `superadmin` 역할 확인
- 권한 없으면 `403 FORBIDDEN` 반환

### 시간 정책

- 모든 시간 필드는 **TIMESTAMPTZ** (UTC 저장)
- 출퇴근 시각 판별은 **KST (Asia/Seoul)** 기준
- 지각 기준: KST 09:00 이후 출근
- 프론트엔드에서 KST로 변환하여 표시

### 공통 응답 형식

**성공**
```json
{
  "success": true,
  "data": { /* 실제 데이터 */ }
}
```

**에러**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "에러 설명"
  }
}
```

---

## 1. GET /api/attendance/types

근태 유형 목록을 조회합니다. 활성화된 유형만 반환하며 `sort_order` 순으로 정렬됩니다.

**파라미터**: 없음

**응답: `200 OK`**
```json
{
  "success": true,
  "data": [
    {
      "id": "normal",
      "name": "정상출근",
      "category": "출근",
      "description": null,
      "requiresApproval": false,
      "isActive": true,
      "sortOrder": 0,
      "createdAt": "2026-03-01T00:00:00.000Z"
    },
    {
      "id": "annual-leave",
      "name": "연차",
      "category": "연차",
      "description": null,
      "requiresApproval": true,
      "isActive": true,
      "sortOrder": 10,
      "createdAt": "2026-03-01T00:00:00.000Z"
    }
  ]
}
```

---

## 2. GET /api/attendance/today

오늘 내 출근 기록을 조회합니다. KST 기준 오늘 날짜의 기록을 반환합니다.

**파라미터**: 없음 (JWT에서 직원 자동 식별)

**응답: `200 OK`** (기록 있음)
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "employeeId": "emp-001",
    "date": "2026-03-03",
    "attendanceTypeId": "normal",
    "clockInAt": "2026-03-03T00:05:00.000Z",
    "clockInLatitude": 37.5665,
    "clockInLongitude": 126.978,
    "clockInAccuracy": 15.5,
    "clockOutAt": null,
    "clockOutLatitude": null,
    "clockOutLongitude": null,
    "clockOutAccuracy": null,
    "status": "normal",
    "note": null,
    "registeredBy": null,
    "createdAt": "2026-03-03T00:05:00.000Z",
    "updatedAt": "2026-03-03T00:05:00.000Z"
  }
}
```

**응답: `200 OK`** (기록 없음)
```json
{
  "success": true,
  "data": null
}
```

---

## 3. POST /api/attendance

출퇴근 및 근태 등록을 처리합니다. `action` 필드로 동작을 구분합니다.

### action: clock-in (출근)

GPS 좌표와 함께 출근을 기록합니다. KST 09:00 기준으로 `normal`/`late` 상태가 자동 결정됩니다.

**요청**
```json
{
  "action": "clock-in",
  "latitude": 37.5665,
  "longitude": 126.978,
  "accuracy": 15.5
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `action` | string | O | `"clock-in"` |
| `latitude` | number | N | GPS 위도 |
| `longitude` | number | N | GPS 경도 |
| `accuracy` | number | N | GPS 정확도 (미터) |

**응답: `200 OK`**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "employeeId": "emp-001",
    "date": "2026-03-03",
    "attendanceTypeId": "normal",
    "clockInAt": "2026-03-03T00:05:00.000Z",
    "clockInLatitude": 37.5665,
    "clockInLongitude": 126.978,
    "clockInAccuracy": 15.5,
    "clockOutAt": null,
    "clockOutLatitude": null,
    "clockOutLongitude": null,
    "clockOutAccuracy": null,
    "status": "normal",
    "note": null,
    "registeredBy": null,
    "createdAt": "2026-03-03T00:05:00.000Z",
    "updatedAt": "2026-03-03T00:05:00.000Z"
  }
}
```

**에러**: 이미 출근한 경우 `409 ALREADY_CLOCKED_IN`

### action: clock-out (퇴근)

오늘 출근 기록에 퇴근 시간을 기록합니다.

**요청**
```json
{
  "action": "clock-out",
  "latitude": 37.5665,
  "longitude": 126.978,
  "accuracy": 12.0
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `action` | string | O | `"clock-out"` |
| `latitude` | number | N | GPS 위도 |
| `longitude` | number | N | GPS 경도 |
| `accuracy` | number | N | GPS 정확도 (미터) |

**응답: `200 OK`** — clock-in 응답과 동일 구조 (`clockOutAt` 채워짐)

**에러**:
- 출근 기록 없음: `400 NO_CLOCK_IN`
- 이미 퇴근: `409 ALREADY_CLOCKED_OUT`

### action: register (근태 등록)

연차/외근/출장 등을 수동으로 등록합니다. 동일 날짜 기록이 있으면 UPSERT 처리됩니다.

**요청**
```json
{
  "action": "register",
  "date": "2026-03-01",
  "typeId": "annual-leave",
  "note": "개인 사유"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `action` | string | O | `"register"` |
| `date` | string | O | 근태 날짜 (YYYY-MM-DD, 과거날짜만) |
| `typeId` | string | O | 근태 유형 ID (attendance_types.id) |
| `note` | string | N | 비고 (최대 200자) |

**응답: `200 OK`** — AttendanceRecord 구조 (`status = 'registered'`)

**에러**:
- 미래 날짜: `400 FUTURE_DATE`
- date 누락: `400 INVALID_REQUEST`
- typeId 누락: `400 INVALID_REQUEST`

---

## 4. GET /api/attendance

내 근태 기록을 기간별로 조회합니다. 날짜 내림차순 정렬.

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `startDate` | string | O | 시작일 (YYYY-MM-DD) |
| `endDate` | string | O | 종료일 (YYYY-MM-DD) |

**응답: `200 OK`**
```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-...",
      "employeeId": "emp-001",
      "date": "2026-03-03",
      "attendanceTypeId": "normal",
      "clockInAt": "2026-03-03T00:05:00.000Z",
      "clockInLatitude": 37.5665,
      "clockInLongitude": 126.978,
      "clockInAccuracy": 15.5,
      "clockOutAt": "2026-03-03T09:00:00.000Z",
      "clockOutLatitude": 37.5665,
      "clockOutLongitude": 126.978,
      "clockOutAccuracy": 12.0,
      "status": "normal",
      "note": null,
      "registeredBy": null,
      "createdAt": "2026-03-03T00:05:00.000Z",
      "updatedAt": "2026-03-03T09:00:00.000Z"
    }
  ]
}
```

---

## 5. GET /api/attendance/admin

> **HR 전용**: `hr` 또는 `superadmin` 역할 필수

전 직원 근태 기록을 필터링하여 조회합니다. 페이지네이션 지원.

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|----------|------|------|--------|------|
| `startDate` | string | O | - | 시작일 (YYYY-MM-DD) |
| `endDate` | string | O | - | 종료일 (YYYY-MM-DD) |
| `departmentId` | string | N | - | 부서 ID 필터 |
| `employeeId` | string | N | - | 직원 ID 필터 |
| `status` | string | N | - | 상태 필터 (`normal`, `late`, `early_leave`, `absent`, `registered`) |
| `limit` | number | N | `20` | 페이지 크기 (최대 100) |
| `offset` | number | N | `0` | 시작 위치 |

**응답: `200 OK`**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "550e8400-...",
        "employeeId": "emp-001",
        "date": "2026-03-03",
        "attendanceTypeId": "normal",
        "clockInAt": "2026-03-03T00:05:00.000Z",
        "clockInLatitude": 37.5665,
        "clockInLongitude": 126.978,
        "clockInAccuracy": 15.5,
        "clockOutAt": "2026-03-03T09:00:00.000Z",
        "clockOutLatitude": null,
        "clockOutLongitude": null,
        "clockOutAccuracy": null,
        "status": "normal",
        "note": null,
        "registeredBy": null,
        "createdAt": "2026-03-03T00:05:00.000Z",
        "updatedAt": "2026-03-03T09:00:00.000Z",
        "employeeName": "홍길동",
        "departmentName": "개발1팀"
      }
    ],
    "total": 150
  }
}
```

**추가 필드** (AdminAttendanceRecord):

| 필드 | 타입 | 설명 |
|------|------|------|
| `employeeName` | string | 직원명 (employees.name JOIN) |
| `departmentName` | string \| null | 부서명 (departments.name LEFT JOIN) |

---

## 6. GET /api/attendance/stats

> **HR 전용**: `hr` 또는 `superadmin` 역할 필수

기간별 근태 통계를 조회합니다. 부서 필터 가능.

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| `startDate` | string | O | 시작일 (YYYY-MM-DD) |
| `endDate` | string | O | 종료일 (YYYY-MM-DD) |
| `departmentId` | string | N | 부서 ID 필터 |

**응답: `200 OK`**
```json
{
  "success": true,
  "data": {
    "totalCount": 150,
    "normalCount": 120,
    "lateCount": 15,
    "earlyLeaveCount": 5,
    "registeredCount": 10
  }
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `totalCount` | number | 전체 근태 기록 수 |
| `normalCount` | number | 정상 출근 수 |
| `lateCount` | number | 지각 수 |
| `earlyLeaveCount` | number | 조퇴 수 |
| `registeredCount` | number | 등록 (연차/외근 등) 수 |

---

## 7. GET /api/attendance/zones/active

활성 상태인 근태 영역 목록을 조회합니다. **HR 권한 불필요** — 모든 인증된 사용자가 접근 가능.
프론트엔드에서 출근 시 Kakao Maps 다이얼로그에 영역을 표시할 때 사용합니다.

**파라미터**: 없음

**응답: `200 OK`**
```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-...",
      "name": "본사",
      "latitude": 37.5726545,
      "longitude": 126.9701371,
      "radiusMeters": 200,
      "isActive": true,
      "description": "본사 사무실 (서울)",
      "createdBy": "emp-001",
      "createdAt": "2026-03-03T00:00:00.000Z",
      "updatedAt": "2026-03-03T00:00:00.000Z"
    }
  ]
}
```

> **참고**: 비활성 영역은 반환하지 않음. HR 관리자가 전체 목록(비활성 포함)을 조회하려면 `GET /api/attendance/zones`를 사용.

---

## 8. GET /api/attendance/zones

> **HR 전용**: `hr` 또는 `superadmin` 역할 필수

근태 영역(출근 허용 구역) 목록을 조회합니다.

**Query Parameters**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|----------|------|------|--------|------|
| `includeInactive` | string | N | `"false"` | `"true"`이면 비활성 영역도 포함 |

**응답: `200 OK`**
```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-...",
      "name": "본사",
      "latitude": 37.5726545,
      "longitude": 126.9701371,
      "radiusMeters": 200,
      "isActive": true,
      "description": "본사 사무실 (서울)",
      "createdBy": "emp-001",
      "createdAt": "2026-03-03T00:00:00.000Z",
      "updatedAt": "2026-03-03T00:00:00.000Z"
    }
  ]
}
```

---

## 9. POST /api/attendance/zones

> **HR 전용**: `hr` 또는 `superadmin` 역할 필수

근태 영역 CRUD. `action` 필드로 동작을 구분합니다.

### action: create

```json
{ "action": "create", "name": "본사", "latitude": 37.5726545, "longitude": 126.9701371, "radiusMeters": 200, "description": "본사 사무실" }
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | string | O | 영역 이름 |
| `latitude` | number | O | 위도 (-90~90) |
| `longitude` | number | O | 경도 (-180~180) |
| `radiusMeters` | number | N | 반경 (기본 200m, 50~5000m) |
| `description` | string | N | 설명 |

### action: update

```json
{ "action": "update", "id": "uuid-xxx", "name": "본사 (수정)", "latitude": 37.5726545, "longitude": 126.9701371, "radiusMeters": 300 }
```

### action: toggle

```json
{ "action": "toggle", "id": "uuid-xxx", "isActive": false }
```

### action: delete

```json
{ "action": "delete", "id": "uuid-xxx" }
```

---

## 에러 응답

| 코드 | HTTP | 설명 |
|------|------|------|
| `UNAUTHORIZED` | 401 | JWT 토큰 없거나 무효 |
| `NO_EMPLOYEE` | 403 | 직원 정보가 연결되지 않음 (users.employeeId 없음) |
| `FORBIDDEN` | 403 | HR 관리자 권한 필요 (admin, stats, zones 엔드포인트) |
| `INVALID_REQUEST` | 400 | 필수 파라미터 누락 (startDate, endDate, action 등) |
| `INVALID_ACTION` | 400 | 알 수 없는 action 값 |
| `FUTURE_DATE` | 400 | 미래 날짜에 근태 등록 시도 |
| `ALREADY_CLOCKED_IN` | 409 | 이미 오늘 출근 기록 존재 |
| `ALREADY_CLOCKED_OUT` | 409 | 이미 퇴근 처리됨 |
| `NO_CLOCK_IN` | 400 | 오늘 출근 기록 없이 퇴근 시도 |
| `GPS_REQUIRED` | 400 | 근태 영역 활성화 상태에서 GPS 좌표 없음 |
| `OUT_OF_ZONE` | 403 | 등록된 출근 영역 밖에서 출근 시도 |
| `ZONE_NOT_FOUND` | 404 | 해당 영역 ID를 찾을 수 없음 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

---

## 엔드포인트 요약

| 메서드 | 경로 | 설명 | 권한 |
|--------|------|------|------|
| GET | `/api/attendance/types` | 근태 유형 목록 | 인증 필수 |
| GET | `/api/attendance/today` | 오늘 내 출근 기록 | 인증 필수 |
| POST | `/api/attendance` | 출퇴근/근태 등록 (action 기반) | 인증 필수 |
| GET | `/api/attendance` | 내 근태 기간 조회 | 인증 필수 |
| GET | `/api/attendance/admin` | HR 관리자 근태 조회 | hr / superadmin |
| GET | `/api/attendance/stats` | 근태 통계 | hr / superadmin |
| GET | `/api/attendance/zones/active` | 활성 근태 영역 목록 | 인증 필수 |
| GET | `/api/attendance/zones` | 근태 영역 목록 (전체) | hr / superadmin |
| POST | `/api/attendance/zones` | 근태 영역 CRUD (action 기반) | hr / superadmin |
