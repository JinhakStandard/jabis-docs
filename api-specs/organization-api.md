# Organization API 명세

> **서버**: jabis-api-gateway
> **Base URL**: `/api/organization`
> **인증**: JWT (jabis-cert 연동)
> **HTTP 메서드**: GET (조회), POST (변경 — action 필드로 구분)

---

## 1. 부서 관리

### GET /api/organization/departments

전체 부서 목록 조회 (sort_order, name 정렬)

**응답**
```json
{
  "success": true,
  "data": [
    {
      "id": "company",
      "code": "JH",
      "name": "㈜진학사",
      "parentId": null,
      "type": "company",
      "status": "active",
      "description": null,
      "sortOrder": 0,
      "createdAt": "2026-02-11T...",
      "updatedAt": "2026-02-11T..."
    }
  ]
}
```

### POST /api/organization/departments

부서 생성/수정/삭제

#### action: create
```json
{
  "action": "create",
  "data": {
    "id": "new-dept",
    "code": "ND",
    "name": "신규부서",
    "parentId": "division-support",
    "type": "team",
    "status": "active",
    "description": "설명",
    "sortOrder": 0
  }
}
```

#### action: update
```json
{
  "action": "update",
  "id": "new-dept",
  "data": {
    "name": "수정된 부서명",
    "status": "inactive"
  }
}
```

#### action: delete
```json
{
  "action": "delete",
  "id": "new-dept"
}
```

---

## 2. 직원 관리

### GET /api/organization/employees

전체 직원 목록 조회 (sort_order, name 정렬)

**응답**
```json
{
  "success": true,
  "data": [
    {
      "id": "emp-001",
      "employeeNumber": null,
      "name": "홍길동",
      "title": "팀장",
      "email": null,
      "phone": null,
      "departmentId": "team-dev-1",
      "status": "active",
      "sortOrder": 0,
      "createdAt": "2026-02-11T...",
      "updatedAt": "2026-02-11T..."
    }
  ]
}
```

### GET /api/organization/employees/search

직원 이름 검색 (완전 일치 우선, LIKE 검색 후순위, 최대 10건)

**파라미터**
| 이름 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `name` | query string | ✅ | 검색할 이름 |

**응답**: `{ success: true, data: Employee[] }`

### POST /api/organization/employees

직원 생성/수정/삭제 (부서 API와 동일한 action 패턴)

#### action: create
```json
{
  "action": "create",
  "data": {
    "id": "emp-new",
    "name": "김신입",
    "title": "사원",
    "departmentId": "team-dev-1",
    "employeeNumber": "2026001",
    "email": "new@jinhak.com",
    "phone": "010-1234-5678",
    "status": "active",
    "sortOrder": 0
  }
}
```

#### action: update
```json
{
  "action": "update",
  "id": "emp-new",
  "data": { "title": "선임", "departmentId": "team-dev-2" }
}
```

#### action: delete
```json
{ "action": "delete", "id": "emp-new" }
```

---

## 3. 사용자 관리

### GET /api/organization/me

현재 로그인 사용자 정보 + 권한 조회 (JWT 필수)

- 사용자 ID: JWT `sub` 또는 `email` 에서 `@` 앞부분만 추출 (절대 규칙)
- 사용자가 없으면 자동 생성 (UPSERT)
- `isSuperAdmin = true`이면 모든 권한 자동 부여

**응답**
```json
{
  "success": true,
  "data": {
    "id": "ka3794",
    "email": "ka3794@jinhak.com",
    "displayName": "임형섭",
    "employeeId": "emp-001",
    "isSuperAdmin": true,
    "mappedAt": "2026-02-11T...",
    "createdAt": "2026-02-11T...",
    "updatedAt": "2026-02-11T...",
    "employee": {
      "id": "emp-001",
      "name": "임형섭",
      "title": "팀장",
      "departmentId": "team-dev-1"
    },
    "department": {
      "id": "team-dev-1",
      "name": "개발1팀",
      "type": "team"
    },
    "permissions": ["page:portal", "page:developer", "feature:org:edit", "..."]
  }
}
```

### POST /api/organization/me/map-employee

현재 사용자에 직원 매칭

```json
{ "employeeId": "emp-001" }
```

### GET /api/organization/users

전체 사용자 목록 조회

**응답**: `{ success: true, data: OrgUser[] }`

---

## 4. 권한 관리

### GET /api/organization/permissions

전체 권한 목록 조회

**응답**
```json
{
  "success": true,
  "data": [
    {
      "id": "page:portal",
      "category": "page",
      "name": "포털",
      "description": "메인 포털 페이지 접근",
      "parentId": null
    },
    {
      "id": "feature:org:edit",
      "category": "feature",
      "name": "조직 관리 편집",
      "description": "부서/직원 생성·수정·삭제",
      "parentId": null
    }
  ]
}
```

### GET /api/organization/users/:userId/permissions

특정 사용자의 권한 ID 배열 조회

**응답**: `{ success: true, data: ["page:portal", "feature:org:view"] }`

### POST /api/organization/users/:userId/permissions

권한 부여/해제/일괄설정/슈퍼관리자 설정

#### action: grant
```json
{ "action": "grant", "permissionId": "page:developer" }
```

#### action: revoke
```json
{ "action": "revoke", "permissionId": "page:developer" }
```

#### action: set (일괄 설정 — 트랜잭션)
```json
{
  "action": "set",
  "permissionIds": ["page:portal", "page:developer", "feature:org:view"]
}
```

#### action: set-super-admin
```json
{ "action": "set-super-admin", "isSuperAdmin": true }
```

---

## 5. 권한 체계

### 5.1 페이지 권한 (15개)

| ID | 이름 |
|----|------|
| `page:portal` | 포털 |
| `page:developer` | 개발 |
| `page:operation` | 운영 |
| `page:sales` | 영업 |
| `page:design` | 디자인 |
| `page:planner` | 기획 |
| `page:marketer` | 마케팅 |
| `page:finance` | 재무 |
| `page:hr` | 인사 |
| `page:teamlead` | 팀장 |
| `page:depthead` | 부장 |
| `page:executive` | 본부장 |
| `page:aiadmin` | AI관리 |
| `page:ceo` | CEO |
| `page:superadmin` | 슈퍼관리자 |

### 5.2 기능 권한 (9개)

| ID | 이름 |
|----|------|
| `feature:nightbuilder:create` | NB 기능요청 생성 |
| `feature:nightbuilder:approve` | NB 기능요청 승인 |
| `feature:orchestra:view` | Orchestra 조회 |
| `feature:org:view` | 조직 조회 |
| `feature:org:edit` | 조직 편집 |
| `feature:permission:manage` | 권한 관리 |
| `feature:dbconsole:access` | DB 콘솔 접근 |
| `feature:approval:create` | 결재 생성 |
| `feature:approval:approve` | 결재 승인 |

### 5.3 슈퍼관리자

- `is_super_admin = TRUE` → 모든 권한 자동 부여
- 기본 슈퍼관리자: `ka3794`

---

## 6. 에러 응답

```json
{ "error": { "code": "ERROR_CODE", "message": "설명" } }
```

| 코드 | HTTP | 설명 |
|------|------|------|
| `INVALID_REQUEST` | 400 | 필수 파라미터 누락 |
| `INVALID_ACTION` | 400 | 알 수 없는 action 값 |
| `UNAUTHORIZED` | 401 | JWT 토큰 없거나 무효 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `INTERNAL_ERROR` | 500 | 서버 오류 |

---

## 7. 구현 일자

| 날짜 | 커밋 | 내용 |
|------|------|------|
| 2026-02-11 | `1b7abc3` | 사용자 조직 선택 저장 API 추가 |
| 2026-02-11 | `3c92003` | 조직도 + 권한 관리 API 핵심 구현 (+1,130줄) |
| 2026-02-11 | `70bf6d7` | 사용자 ID `@` 제거 규칙 + 실제 조직도 데이터 |
| 2026-02-13 | `a7a3cf9` | DDL 자동실행 제거 (보안 — DBA 수동 관리) |
