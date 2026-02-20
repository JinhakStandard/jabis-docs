# Approval (전자결재) DB 스키마

> **데이터베이스**: PostgreSQL (jabis)
> **스키마**: `approval`
> **배포 스크립트**: `jabis-api-gateway/sql/approval-schema.sql`
> **생성일**: 2026-02-16

---

## 스키마 개요

```
approval.form_categories              양식 카테고리 (8개 시드)
    └── approval.document_forms       결재 양식 템플릿 (13개 시드)

approval.documents                    결재 문서
    ├── approval.lines                결재선 (순서별 결재자)
    ├── approval.references           참조자/회람
    ├── approval.related_docs         관련문서 연결
    ├── approval.processing_history   후처리 이력
    └── approval.attachments          첨부파일
```

---

## ENUM 타입

### approval.document_status (문서 상태)
| 값 | 설명 |
|----|------|
| `draft` | 임시저장 |
| `in_progress` | 결재 진행중 |
| `approved` | 승인완료 |
| `rejected` | 반려 |
| `processing` | 후처리중 |
| `returned` | 회수/반환 |
| `cancelled` | 취소 |

### approval.line_status (결재선 상태)
| 값 | 설명 |
|----|------|
| `waiting` | 대기 (아직 차례 아님) |
| `current` | 현재 결재 차례 |
| `approved` | 승인 완료 |
| `rejected` | 반려 |

### approval.line_type (결재선 유형)
| 값 | 설명 |
|----|------|
| `approve` | 결재 (승인/반려 가능) |
| `agree` | 합의 (의견만, 자동 통과) |
| `notify` | 통보 (알림만 수신) |

### approval.reference_type (참조 유형)
| 값 | 설명 |
|----|------|
| `pre` | 사전참조 (결재 전, 의견 제시 가능) |
| `post` | 사후참조 (결재 완료 후, 열람만) |

### approval.related_doc_relationship (관련문서 관계)
| 값 | 설명 |
|----|------|
| `reference` | 참고 |
| `supplement` | 보충 |
| `revision` | 수정 |
| `follow_up` | 후속 |

### approval.processing_status (후처리 상태)
| 값 | 설명 |
|----|------|
| `pending` | 대기 |
| `in_progress` | 처리중 |
| `completed` | 완료 |
| `rejected` | 반려 |

### approval.priority_level (우선순위)
| 값 | 설명 |
|----|------|
| `low` | 낮음 |
| `normal` | 보통 |
| `high` | 높음 |

### approval.form_category (양식 카테고리)
| 값 | 설명 |
|----|------|
| `general` | 기안서 |
| `contract` | 인감/계약 |
| `hr` | 인사 |
| `welfare` | 복리후생 |
| `education` | 교육 |
| `purchase` | 구매 |
| `security` | 보안 |
| `etc` | 기타 |

---

## 테이블

### 1. approval.form_categories (양식 카테고리)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 카테고리 ID (예: `general`, `hr`) |
| `name` | TEXT | NOT NULL | - | 카테고리명 |
| `description` | TEXT | - | NULL | 설명 |
| `sort_order` | INTEGER | - | `0` | 정렬 순서 |
| `is_active` | BOOLEAN | NOT NULL | `TRUE` | 활성 여부 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

**시드 데이터**: general(기안서), contract(인감/계약), hr(인사), welfare(복리후생), education(교육), purchase(구매), security(보안), etc(기타)

### 2. approval.document_forms (결재 양식 템플릿)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 양식 ID (예: `form-leave`) |
| `name` | TEXT | NOT NULL | - | 양식명 |
| `category_id` | TEXT | FK → form_categories(id) | - | 카테고리 |
| `description` | TEXT | - | NULL | 설명 |
| `fields` | JSONB | NOT NULL | `'[]'` | 양식 필드 정의 배열 |
| `default_approval_line` | TEXT[] | - | `'{}'` | 기본 결재선 역할 배열 |
| `default_processing_dept_id` | TEXT | - | NULL | 1차 처리부서 |
| `second_processing_dept_id` | TEXT | - | NULL | 2차 처리부서 |
| `sort_order` | INTEGER | - | `0` | 정렬 순서 |
| `is_active` | BOOLEAN | NOT NULL | `TRUE` | 활성 여부 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

**fields 예시**: `[{"name":"reason","label":"사유","type":"textarea","required":true}]`

### 3. approval.documents (결재 문서)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 문서 ID |
| `document_number` | TEXT | - | NULL | 문서번호 (예: `APV-2026-0001`) |
| `template_id` | TEXT | FK → document_forms(id) | NULL | 양식 ID |
| `title` | TEXT | NOT NULL | - | 제목 |
| `content` | JSONB | - | `'{}'` | 양식 필드별 입력 데이터 |
| `author_id` | TEXT | NOT NULL | - | 기안자 (organization.users.id) |
| `author_employee_id` | TEXT | - | NULL | 기안자 직원 ID |
| `author_dept_id` | TEXT | - | NULL | 기안자 부서 ID |
| `status` | document_status | NOT NULL | `'draft'` | 문서 상태 |
| `priority` | priority_level | NOT NULL | `'normal'` | 우선순위 |
| `is_urgent` | BOOLEAN | NOT NULL | `FALSE` | 긴급 여부 |
| `due_date` | DATE | - | NULL | 마감일 |
| `current_approver_order` | INTEGER | - | `0` | 현재 결재 순서 |
| `processing_dept_id` | TEXT | - | NULL | 1차 후처리 부서 |
| `second_processing_dept_id` | TEXT | - | NULL | 2차 후처리 부서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |
| `completed_at` | TIMESTAMPTZ | - | NULL | 최종 완료 시점 |

**인덱스**: status, author_id, (author_id, status), template_id, processing_dept_id, created_at DESC

### 4. approval.lines (결재선)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 결재선 ID |
| `document_id` | UUID | FK → documents(id) CASCADE | - | 문서 ID |
| `user_id` | TEXT | NOT NULL | - | 결재자 ID |
| `employee_id` | TEXT | - | NULL | 직원 ID (이름/직급 조회용) |
| `step_order` | INTEGER | NOT NULL | - | 결재 순서 (1부터) |
| `type` | line_type | NOT NULL | `'approve'` | 결재 유형 |
| `status` | line_status | NOT NULL | `'waiting'` | 결재 상태 |
| `comment` | TEXT | - | NULL | 승인/반려 사유 |
| `received_at` | TIMESTAMPTZ | - | NULL | 결재 수신 시간 |
| `action_at` | TIMESTAMPTZ | - | NULL | 결재 처리 시간 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

**제약**: UNIQUE(document_id, step_order)
**인덱스**: document_id, user_id, (user_id, status), (document_id, step_order), user_id WHERE status='current' (partial)

### 5. approval.references (참조/회람)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 참조 ID |
| `document_id` | UUID | FK → documents(id) CASCADE | - | 문서 ID |
| `user_id` | TEXT | NOT NULL | - | 참조자 ID |
| `employee_id` | TEXT | - | NULL | 직원 ID |
| `type` | reference_type | NOT NULL | - | 참조 유형 |
| `opinion` | TEXT | - | NULL | 사전참조 의견 (pre만) |
| `read_at` | TIMESTAMPTZ | - | NULL | 열람 시간 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

**제약**: UNIQUE(document_id, user_id)
**인덱스**: document_id, user_id, (user_id, type)

### 6. approval.related_docs (관련문서)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 관련문서 연결 ID |
| `document_id` | UUID | FK → documents(id) CASCADE | - | 원본 문서 ID |
| `related_document_id` | UUID | FK → documents(id) CASCADE | - | 관련 문서 ID |
| `relationship` | related_doc_relationship | NOT NULL | `'reference'` | 관계 유형 |
| `created_by` | TEXT | NOT NULL | - | 연결 생성자 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

**제약**: UNIQUE(document_id, related_document_id)
**인덱스**: document_id, related_document_id

### 7. approval.processing_history (후처리 이력)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 후처리 이력 ID |
| `document_id` | UUID | FK → documents(id) CASCADE | - | 문서 ID |
| `dept_id` | TEXT | NOT NULL | - | 처리부서 ID |
| `status` | processing_status | NOT NULL | `'pending'` | 처리 상태 |
| `stage` | INTEGER | NOT NULL | `1` | 1차 또는 2차 |
| `processed_by_user_id` | TEXT | - | NULL | 처리자 ID |
| `comment` | TEXT | - | NULL | 처리 코멘트 |
| `processed_at` | TIMESTAMPTZ | - | NULL | 처리 시간 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

**인덱스**: document_id, dept_id, (dept_id, status)

### 8. approval.attachments (첨부파일)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | `gen_random_uuid()` | 첨부파일 ID |
| `document_id` | UUID | FK → documents(id) CASCADE | - | 문서 ID |
| `file_name` | TEXT | NOT NULL | - | 파일명 |
| `file_size` | BIGINT | - | NULL | 파일 크기 (bytes) |
| `file_type` | TEXT | - | NULL | MIME 타입 |
| `file_url` | TEXT | NOT NULL | - | 저장 경로/URL |
| `uploaded_by` | TEXT | NOT NULL | - | 업로드한 사용자 ID |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

**인덱스**: document_id

---

## 상태 전이 다이어그램

```
문서 상태:
  draft → in_progress (상신)
  in_progress → approved (마지막 결재자 승인, 후처리부서 없음)
  in_progress → processing (마지막 결재자 승인, 후처리부서 있음)
  in_progress → rejected (결재 반려)
  in_progress → returned (기안자 회수)
  in_progress → cancelled (관리자 취소)
  processing → approved (모든 후처리 완료)
  processing → returned (후처리 반려)
  processing → cancelled (관리자 취소)
  returned → in_progress (재상신)
  draft → (삭제 가능)
  returned → (삭제 가능)

결재선 상태:
  waiting → current (차례 도래)
  current → approved (승인)
  current → rejected (반려)
  * → waiting (회수 시 초기화)
```

---

## 외부 참조

| 참조 테이블 | 참조 컬럼 | 사용 위치 |
|-----------|---------|---------|
| organization.users.id | author_id, user_id, created_by 등 | documents, lines, references, related_docs |
| organization.employees.id | employee_id, author_employee_id | documents, lines, references |
| organization.departments.id | dept_id, author_dept_id, processing_dept_id | documents, processing_history |
