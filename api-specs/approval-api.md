# Approval (전자결재) API 명세

> **서버**: jabis-api-gateway
> **Base URL**: `/api/approval`
> **인증**: JWT (jabis-cert 연동) 또는 `x-user-email` / `x-user-dept-id` 헤더
> **HTTP 메서드**: GET (조회), POST (변경 — action 필드로 구분)
> **DB 스키마**: `approval.*` (approval-schema.sql 참조)

---

## 1. 문서함 조회

### GET /api/approval/documents

문서함별 결재 문서 목록 조회 (페이징 지원)

**쿼리 파라미터**

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|-------|------|
| boxType | string | `all` | 문서함 유형: `pending`, `progress`, `completed`, `rejected`, `draft`, `reference`, `scheduled`, `department`, `all` |
| deptId | string | — | 부서 ID (없으면 x-user-dept-id 헤더 사용) |
| page | number | 1 | 페이지 번호 |
| pageSize | number | 20 | 페이지 크기 (최대 100) |
| searchQuery | string | — | 제목 검색어 |

**응답**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "uuid",
        "documentNumber": "APPR-2026-0001",
        "templateId": "form-leave",
        "title": "연차 신청",
        "content": {},
        "authorId": "user1",
        "authorEmployeeId": "EMP001",
        "authorDeptId": "dept-dev",
        "status": "in_progress",
        "priority": "normal",
        "isUrgent": false,
        "dueDate": null,
        "currentApproverOrder": 2,
        "processingDeptId": null,
        "secondProcessingDeptId": null,
        "createdAt": "2026-02-16T...",
        "updatedAt": "2026-02-16T...",
        "completedAt": null
      }
    ],
    "total": 42,
    "page": 1,
    "pageSize": 20
  }
}
```

### GET /api/approval/documents/:id

문서 상세 조회 (결재선, 참조자, 관련문서, 후처리이력, 첨부 포함)

**응답**
```json
{
  "success": true,
  "data": {
    "document": { "...ApprovalDocument" },
    "lines": [
      {
        "id": "uuid", "documentId": "uuid", "userId": "user1",
        "employeeId": "EMP001", "stepOrder": 1, "type": "approve",
        "status": "approved", "comment": "확인", "receivedAt": "...",
        "actionAt": "...", "createdAt": "...",
        "userName": "홍길동", "userTitle": "팀장", "userDepartment": "개발팀"
      }
    ],
    "references": [
      {
        "id": "uuid", "documentId": "uuid", "userId": "user2",
        "employeeId": "EMP002", "type": "pre", "opinion": null,
        "readAt": null, "createdAt": "...",
        "userName": "김철수", "userTitle": "과장"
      }
    ],
    "relatedDocs": [
      {
        "id": "uuid", "documentId": "uuid", "relatedDocumentId": "uuid",
        "relationship": "reference", "createdBy": "user1", "createdAt": "...",
        "relatedTitle": "관련 문서 제목", "relatedDocumentNumber": "APPR-2026-0002",
        "relatedStatus": "approved"
      }
    ],
    "processingHistory": [
      {
        "id": "uuid", "documentId": "uuid", "deptId": "dept-hr",
        "status": "completed", "stage": 1, "processedByUserId": "user3",
        "comment": "처리 완료", "processedAt": "...", "createdAt": "..."
      }
    ],
    "attachments": [
      {
        "id": "uuid", "documentId": "uuid", "fileName": "첨부.pdf",
        "fileSize": 102400, "fileType": "application/pdf",
        "fileUrl": "/files/...", "uploadedBy": "user1", "createdAt": "..."
      }
    ]
  }
}
```

---

## 2. 통계

### GET /api/approval/stats

8종 문서함 통계 조회

**쿼리 파라미터**

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| deptId | string | 부서 ID (없으면 x-user-dept-id 헤더 사용) |

**응답**
```json
{
  "success": true,
  "data": {
    "pending": 3,
    "progress": 5,
    "completed": 12,
    "rejected": 1,
    "draft": 2,
    "reference": 7,
    "scheduled": 0,
    "department": 4
  }
}
```

---

## 3. 문서 CRUD

### POST /api/approval/documents

문서 생성/수정/상신/삭제

#### action: create
```json
{
  "action": "create",
  "data": {
    "title": "연차 신청",
    "templateId": "form-leave",
    "content": { "startDate": "2026-02-17", "endDate": "2026-02-17" },
    "authorEmployeeId": "EMP001",
    "authorDeptId": "dept-dev",
    "priority": "normal",
    "isUrgent": false,
    "dueDate": "2026-02-20",
    "processingDeptId": "dept-hr",
    "secondProcessingDeptId": null,
    "lines": [
      { "userId": "manager1", "employeeId": "EMP010", "stepOrder": 1, "type": "approve" },
      { "userId": "director1", "employeeId": "EMP020", "stepOrder": 2, "type": "approve" }
    ],
    "references": [
      { "userId": "ref1", "employeeId": "EMP030", "type": "pre" }
    ]
  }
}
```

#### action: update
기안자 본인만 가능, status = `draft`일 때만

```json
{
  "action": "update",
  "id": "document-uuid",
  "data": {
    "title": "수정된 제목",
    "content": { "...수정된 내용" }
  }
}
```

#### action: submit
기안자 본인만 가능, status = `draft` 또는 `returned`

```json
{
  "action": "submit",
  "id": "document-uuid"
}
```

#### action: delete
기안자 본인만 가능, status = `draft` 또는 `returned`

```json
{
  "action": "delete",
  "id": "document-uuid"
}
```

---

## 4. 결재 액션

### POST /api/approval/documents/:id/approve

결재/반려/회수/취소

#### action: approve
현재 결재 순서의 결재자만 가능

```json
{
  "action": "approve",
  "comment": "확인했습니다"
}
```

**비즈니스 로직**:
- 현재 결재자의 status → `approved`, 다음 결재자의 status → `current`
- 마지막 결재자가 승인 시:
  - `processing_dept_id`가 있으면 → 문서 status = `processing` (후처리 대기)
  - 없으면 → 문서 status = `approved`

#### action: reject
```json
{
  "action": "reject",
  "reason": "내용 수정 필요"
}
```

**비즈니스 로직**: 문서 status → `rejected`, 해당 결재선 status → `rejected`

#### action: withdraw
기안자가 상신 취소 (status = `in_progress`일 때)

```json
{
  "action": "withdraw"
}
```

**비즈니스 로직**: 모든 결재선 status → `waiting`, 문서 status → `draft`

#### action: cancel
```json
{
  "action": "cancel"
}
```

---

## 5. 후처리

### POST /api/approval/documents/:id/processing

후처리 부서의 완료/반려

#### action: complete
```json
{
  "action": "complete",
  "deptId": "dept-hr",
  "comment": "처리 완료"
}
```

**비즈니스 로직**:
- stage 1 (processing_dept_id) 완료 후 second_processing_dept_id가 있으면 → stage 2 pending 생성
- 모든 후처리 완료 시 → 문서 status = `approved`

#### action: reject
```json
{
  "action": "reject",
  "deptId": "dept-hr",
  "comment": "서류 미비"
}
```

**비즈니스 로직**: 문서 status → `returned`

---

## 6. 참조자 관리

### POST /api/approval/documents/:id/references

참조자 추가/제거/의견/읽음 처리

#### action: add
```json
{
  "action": "add",
  "userId": "ref-user1",
  "employeeId": "EMP040",
  "type": "pre"
}
```

#### action: remove
```json
{
  "action": "remove",
  "refId": "reference-uuid"
}
```

#### action: opinion
사전 참조자(`pre`)만 의견 작성 가능

```json
{
  "action": "opinion",
  "refId": "reference-uuid",
  "opinion": "특이사항 없음"
}
```

#### action: read
```json
{
  "action": "read",
  "refId": "reference-uuid"
}
```

---

## 7. 관련문서

### POST /api/approval/documents/:id/related

관련문서 연결/해제

#### action: add
```json
{
  "action": "add",
  "relatedDocumentId": "other-document-uuid",
  "relationship": "reference"
}
```

relationship 종류: `reference`, `supplement`, `revision`, `follow_up`

#### action: remove
```json
{
  "action": "remove",
  "id": "related-doc-uuid"
}
```

---

## 8. 양식 관리

### GET /api/approval/forms

양식 목록 조회

**쿼리 파라미터**

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| categoryId | string | 카테고리별 필터 |

**응답**
```json
{
  "success": true,
  "data": [
    {
      "id": "form-leave",
      "name": "연차/휴가 신청서",
      "categoryId": "cat-hr",
      "description": "연차/반차/병가 등 휴가 신청",
      "fields": [],
      "defaultApprovalLine": [],
      "defaultProcessingDeptId": "dept-hr",
      "secondProcessingDeptId": null,
      "sortOrder": 1,
      "isActive": true,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

### GET /api/approval/forms/:id

양식 상세 조회

### POST /api/approval/forms

양식 생성/수정/삭제 (관리자)

#### action: create
```json
{
  "action": "create",
  "data": {
    "id": "form-new",
    "name": "신규 양식",
    "categoryId": "cat-general",
    "description": "설명",
    "fields": [],
    "defaultApprovalLine": [],
    "defaultProcessingDeptId": null,
    "sortOrder": 0
  }
}
```

#### action: update
```json
{
  "action": "update",
  "id": "form-leave",
  "data": { "name": "수정된 양식명" }
}
```

#### action: delete
```json
{
  "action": "delete",
  "id": "form-unused"
}
```

---

## 9. 카테고리 관리

### GET /api/approval/categories

양식 카테고리 목록 조회

**응답**
```json
{
  "success": true,
  "data": [
    {
      "id": "cat-hr",
      "name": "인사/근태",
      "description": "인사 및 근태 관련 양식",
      "sortOrder": 1,
      "isActive": true,
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

### POST /api/approval/categories

카테고리 생성/수정/삭제 (관리자)

#### action: create
```json
{
  "action": "create",
  "data": {
    "id": "cat-new",
    "name": "신규 카테고리",
    "description": "설명",
    "sortOrder": 0
  }
}
```

#### action: update
```json
{
  "action": "update",
  "id": "cat-hr",
  "data": { "name": "수정된 카테고리명" }
}
```

#### action: delete
```json
{
  "action": "delete",
  "id": "cat-unused"
}
```

---

## 엔드포인트 요약

| 메서드 | 경로 | 설명 |
|-------|------|------|
| GET | `/api/approval/documents` | 문서함별 목록 조회 |
| GET | `/api/approval/documents/:id` | 문서 상세 조회 |
| GET | `/api/approval/stats` | 8종 문서함 통계 |
| GET | `/api/approval/forms` | 양식 목록 |
| GET | `/api/approval/forms/:id` | 양식 상세 |
| GET | `/api/approval/categories` | 카테고리 목록 |
| POST | `/api/approval/documents` | 문서 create/update/submit/delete |
| POST | `/api/approval/documents/:id/approve` | 결재 approve/reject/withdraw/cancel |
| POST | `/api/approval/documents/:id/processing` | 후처리 complete/reject |
| POST | `/api/approval/documents/:id/references` | 참조자 add/remove/opinion/read |
| POST | `/api/approval/documents/:id/related` | 관련문서 add/remove |
| POST | `/api/approval/forms` | 양식 create/update/delete |
| POST | `/api/approval/categories` | 카테고리 create/update/delete |

## 상태 ENUM 참조

| ENUM | 값 |
|------|---|
| DocumentStatus | `draft`, `in_progress`, `approved`, `rejected`, `processing`, `returned`, `cancelled` |
| LineStatus | `waiting`, `current`, `approved`, `rejected` |
| LineType | `approve`, `agree`, `notify` |
| ReferenceType | `pre`, `post` |
| RelatedDocRelationship | `reference`, `supplement`, `revision`, `follow_up` |
| ProcessingStatus | `pending`, `in_progress`, `completed`, `rejected` |
| PriorityLevel | `low`, `normal`, `high` |

## 소스 파일

| 파일 | 설명 |
|-----|------|
| `src/routes/approval.ts` | 라우트 정의 (13개 엔드포인트) |
| `src/services/approvalService.ts` | 서비스 레이어 (비즈니스 검증) |
| `src/repositories/approvalRepository.ts` | 레포지토리 (DB 쿼리, 트랜잭션) |
| `src/repositories/approvalFormRepository.ts` | 양식/카테고리 레포지토리 |
| `src/types/approval.ts` | 타입 정의 |
| `sql/approval-schema.sql` | DB 스키마 (10 테이블, 6 ENUM, 시드 데이터) |
