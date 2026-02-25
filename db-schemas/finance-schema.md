# Finance Schema (법인카드 관리)

> **데이터베이스**: PostgreSQL (jabis)
> **스키마**: `finance`
> **배포 스크립트**: `sql/finance-schema.sql`
> **생성일**: 2026-02-23
> **관련 프론트엔드**: jabis-finance (`/card/*` 라우트)

## 개요

법인카드 불출/반납 전체 생명주기를 관리하는 스키마.
기안서 작성(사용자) → 승인(재무담당자) → 반납(사용자) → 반납 승인(재무담당자) 흐름을 지원한다.

> **설계 원칙**: 기존 `approval` 스키마(전자결재)와 독립적으로 운영.
> 법인카드 워크플로우는 결재선이 단순(기안자→재무담당)하고, 카드 상태와 기안서 상태가 강결합되므로
> 별도 스키마로 분리하여 트랜잭션 원자성을 보장한다.

## 스키마 다이어그램

```
finance
├── corporate_cards          법인카드 마스터
├── card_checkout_requests   불출 기안서
├── card_return_requests     반납 기안서
├── card_receipts            영수증 첨부 파일
├── card_users               기안서 작성 가능 사용자
├── card_departments         법인카드 관리용 부서
└── card_status_history      카드/기안서 상태 변경 이력
```

---

## ENUM 타입

### finance.card_status (카드 상태)

| 값 | 설명 |
|----|------|
| `available` | 불출 가능 |
| `in_use` | 불출 중 |

### finance.checkout_status (불출 기안서 상태)

| 값 | 설명 |
|----|------|
| `pending` | 결재 전 |
| `approved` | 불출 승인 (카드 사용 중) |
| `returned` | 반납 완료 |
| `rejected` | 반려 |

### finance.return_status (반납 기안서 상태)

| 값 | 설명 |
|----|------|
| `pending` | 반납 결재 중 |
| `returned` | 반납 승인 완료 |
| `rejected` | 반납 반려 |

### finance.history_action (이력 액션)

| 값 | 설명 |
|----|------|
| `checkout_request` | 불출 기안서 작성 |
| `checkout_approve` | 불출 승인 |
| `checkout_reject` | 불출 반려 |
| `checkout_delete` | 불출 기안서 삭제 |
| `return_request` | 반납 기안서 작성 |
| `return_approve` | 반납 승인 |
| `return_reject` | 반납 반려 |

---

## 테이블 정의

### 1. finance.corporate_cards (법인카드)

법인 보유 카드 마스터 테이블.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 카드 ID |
| `card_number` | VARCHAR(4) | NOT NULL, UNIQUE | — | 카드 뒷자리 4자리 |
| `company` | VARCHAR(50) | NOT NULL | — | 소속 법인 (진학사, 진학어플라이 등) |
| `status` | card_status | NOT NULL | 'available' | 카드 상태 |
| `current_holder_id` | UUID | FK → card_users.id | NULL | 현재 불출자 (in_use 시) |
| `description` | TEXT | | NULL | 카드 메모/설명 |
| `is_active` | BOOLEAN | NOT NULL | TRUE | 활성 여부 (비활성 시 불출 불가) |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | NOW() | 수정일 |

### 2. finance.card_departments (부서별 카드 설정)

부서별 카드 관리 설정 테이블. `organization.departments`와 `name` 컬럼으로 연동.
부서 마스터는 `organization.departments`가 원본이며, 이 테이블은 카드 관리용 정렬/설정만 저장한다.

> **organization 연동 방식**: `card_departments.name`과 `organization.departments.name`을 매칭하여 JOIN.
> API 응답에서 부서명은 `organization.departments.name`을 사용한다.
> `GET /api/finance/departments`는 `organization.departments` 기준으로 조회하고, `card_departments` 설정을 LEFT JOIN한다.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 부서 ID |
| `name` | VARCHAR(100) | NOT NULL, UNIQUE | — | 부서명 (organization.departments.name과 매칭) |
| `sort_order` | INTEGER | NOT NULL | 0 | 카드 관리용 정렬 순서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | NOW() | 수정일 |

### 3. finance.card_users (기안서 작성자)

법인카드 기안서 작성이 가능한 사용자 목록. 카드 관리자가 직접 등록/관리.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 사용자 ID |
| `name` | VARCHAR(100) | NOT NULL | — | 이름 |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE | — | 이메일 (로그인 계정과 매핑) |
| `company` | VARCHAR(50) | NOT NULL | — | 소속 법인 |
| `department_id` | UUID | FK → card_departments.id | NULL | 소속 부서 |
| `is_admin` | BOOLEAN | NOT NULL | FALSE | 관리자 권한 (승인 권한) |
| `is_active` | BOOLEAN | NOT NULL | TRUE | 활성 여부 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | NOW() | 수정일 |

### 4. finance.card_checkout_requests (불출 기안서)

법인카드 불출 요청 문서.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 기안서 ID |
| `request_number` | VARCHAR(20) | NOT NULL, UNIQUE | — | 기안번호 (FC-YYYYMMDD-NNN) |
| `card_id` | UUID | NOT NULL, FK → corporate_cards.id | — | 요청 카드 |
| `requester_id` | UUID | NOT NULL, FK → card_users.id | — | 기안자 |
| `content` | TEXT | NOT NULL | — | 사용 목적 및 내역 |
| `expected_amount` | INTEGER | NOT NULL | — | 사용 예정 금액 (원) |
| `status` | checkout_status | NOT NULL | 'pending' | 기안서 상태 |
| `approved_by` | UUID | FK → card_users.id | NULL | 승인자 |
| `approved_at` | TIMESTAMPTZ | | NULL | 승인 일시 |
| `reject_reason` | TEXT | | NULL | 반려 사유 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 작성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | NOW() | 수정일 |

### 5. finance.card_return_requests (반납 기안서)

불출 카드 반납 요청 문서. 불출 기안서 1건당 반납 기안서 최대 1건.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 반납 기안서 ID |
| `checkout_request_id` | UUID | NOT NULL, UNIQUE, FK → card_checkout_requests.id | — | 원본 불출 기안서 |
| `card_id` | UUID | NOT NULL, FK → corporate_cards.id | — | 반납 카드 |
| `requester_id` | UUID | NOT NULL, FK → card_users.id | — | 반납 기안자 |
| `document_number` | VARCHAR(50) | | NULL | 전자결재 문서번호 (외부 참조) |
| `content` | TEXT | NOT NULL | — | 사용 내역 적요 |
| `status` | return_status | NOT NULL | 'pending' | 반납 기안서 상태 |
| `approved_by` | UUID | FK → card_users.id | NULL | 반납 승인자 |
| `approved_at` | TIMESTAMPTZ | | NULL | 반납 승인 일시 |
| `reject_reason` | TEXT | | NULL | 반려 사유 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 작성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | NOW() | 수정일 |

### 6. finance.card_receipts (영수증 첨부)

반납 기안서에 첨부된 영수증 파일.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 영수증 ID |
| `return_request_id` | UUID | NOT NULL, FK → card_return_requests.id | — | 반납 기안서 |
| `file_name` | VARCHAR(255) | NOT NULL | — | 원본 파일명 |
| `file_path` | VARCHAR(500) | NOT NULL | — | 저장 경로 (MinIO/S3) |
| `file_size` | INTEGER | NOT NULL | — | 파일 크기 (bytes) |
| `mime_type` | VARCHAR(100) | NOT NULL | — | MIME 타입 |
| `sort_order` | INTEGER | NOT NULL | 0 | 정렬 순서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 업로드일 |

### 7. finance.card_status_history (상태 변경 이력)

카드 및 기안서의 모든 상태 변경을 감사 로그로 기록.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | UUID | PK | gen_random_uuid() | 이력 ID |
| `card_id` | UUID | NOT NULL, FK → corporate_cards.id | — | 대상 카드 |
| `checkout_request_id` | UUID | FK → card_checkout_requests.id | NULL | 관련 불출 기안서 |
| `return_request_id` | UUID | FK → card_return_requests.id | NULL | 관련 반납 기안서 |
| `action` | history_action | NOT NULL | — | 액션 타입 |
| `actor_id` | UUID | NOT NULL, FK → card_users.id | — | 수행자 |
| `previous_status` | VARCHAR(20) | | NULL | 이전 상태 |
| `new_status` | VARCHAR(20) | NOT NULL | — | 변경 후 상태 |
| `comment` | TEXT | | NULL | 비고 |
| `created_at` | TIMESTAMPTZ | NOT NULL | NOW() | 발생 일시 |

---

## 인덱스

```sql
-- corporate_cards
CREATE INDEX idx_finance_cards_status ON finance.corporate_cards(status);
CREATE INDEX idx_finance_cards_company ON finance.corporate_cards(company);

-- card_users
CREATE INDEX idx_finance_users_email ON finance.card_users(email);
CREATE INDEX idx_finance_users_department ON finance.card_users(department_id);
CREATE INDEX idx_finance_users_company ON finance.card_users(company);

-- card_checkout_requests
CREATE INDEX idx_finance_checkout_status ON finance.card_checkout_requests(status);
CREATE INDEX idx_finance_checkout_card ON finance.card_checkout_requests(card_id);
CREATE INDEX idx_finance_checkout_requester ON finance.card_checkout_requests(requester_id);
CREATE INDEX idx_finance_checkout_created ON finance.card_checkout_requests(created_at DESC);

-- card_return_requests
CREATE INDEX idx_finance_return_status ON finance.card_return_requests(status);
CREATE INDEX idx_finance_return_card ON finance.card_return_requests(card_id);
CREATE INDEX idx_finance_return_checkout ON finance.card_return_requests(checkout_request_id);
CREATE INDEX idx_finance_return_created ON finance.card_return_requests(created_at DESC);

-- card_receipts
CREATE INDEX idx_finance_receipts_return ON finance.card_receipts(return_request_id);

-- card_status_history
CREATE INDEX idx_finance_history_card ON finance.card_status_history(card_id);
CREATE INDEX idx_finance_history_created ON finance.card_status_history(created_at DESC);
```

---

## 제약 조건 요약

| 제약 | 테이블 | 설명 |
|------|--------|------|
| `uq_cards_number` | corporate_cards | 카드번호 유일성 |
| `uq_users_email` | card_users | 이메일 유일성 |
| `uq_checkout_number` | card_checkout_requests | 기안번호 유일성 |
| `uq_return_checkout` | card_return_requests | 불출 기안서당 반납 1건 |
| `fk_cards_holder` | corporate_cards | → card_users.id |
| `fk_users_department` | card_users | → card_departments.id |
| `fk_checkout_card` | card_checkout_requests | → corporate_cards.id |
| `fk_checkout_requester` | card_checkout_requests | → card_users.id |
| `fk_checkout_approver` | card_checkout_requests | → card_users.id |
| `fk_return_checkout` | card_return_requests | → card_checkout_requests.id |
| `fk_return_card` | card_return_requests | → corporate_cards.id |
| `fk_return_requester` | card_return_requests | → card_users.id |
| `fk_return_approver` | card_return_requests | → card_users.id |
| `fk_receipts_return` | card_receipts | → card_return_requests.id (CASCADE) |
| `fk_history_card` | card_status_history | → corporate_cards.id |

---

## ER 다이어그램

```
┌──────────────────────┐
│  card_departments    │
├──────────────────────┤
│ * id (PK)            │
│   name (UNIQUE)      │
│   sort_order         │
└──────────────────────┘
         │ 1:N
         ▼
┌──────────────────────┐       ┌──────────────────────┐
│  card_users          │       │  corporate_cards     │
├──────────────────────┤       ├──────────────────────┤
│ * id (PK)            │◄──┐   │ * id (PK)            │
│   name               │   │   │   card_number (UQ)   │
│   email (UNIQUE)     │   │   │   company             │
│   company            │   │   │   status              │
│   department_id (FK) │   └───│   current_holder_id   │
│   is_admin           │       │   is_active           │
└──────────────────────┘       └──────────────────────┘
     │ 1:N                          │ 1:N
     ▼                              ▼
┌────────────────────────────────────────────────┐
│  card_checkout_requests                        │
├────────────────────────────────────────────────┤
│ * id (PK)                                      │
│   request_number (UNIQUE)                      │
│   card_id (FK → corporate_cards)               │
│   requester_id (FK → card_users)               │
│   content                                      │
│   expected_amount                              │
│   status (pending → approved → returned)       │
│   approved_by (FK → card_users)                │
└────────────────────────────────────────────────┘
         │ 1:1
         ▼
┌────────────────────────────────────────────────┐
│  card_return_requests                          │
├────────────────────────────────────────────────┤
│ * id (PK)                                      │
│   checkout_request_id (FK, UNIQUE)             │
│   card_id (FK → corporate_cards)               │
│   requester_id (FK → card_users)               │
│   document_number                              │
│   content                                      │
│   status (pending → returned)                  │
│   approved_by (FK → card_users)                │
└────────────────────────────────────────────────┘
         │ 1:N
         ▼
┌──────────────────────┐    ┌──────────────────────────┐
│  card_receipts       │    │  card_status_history     │
├──────────────────────┤    ├──────────────────────────┤
│ * id (PK)            │    │ * id (PK)                │
│   return_request_id  │    │   card_id (FK)           │
│   file_name          │    │   checkout_request_id    │
│   file_path          │    │   return_request_id      │
│   file_size          │    │   action                 │
│   mime_type          │    │   actor_id (FK)          │
└──────────────────────┘    │   previous_status        │
                            │   new_status             │
                            └──────────────────────────┘
```

---

## 상태 전이 다이어그램

### 불출 기안서 (checkout_status)

```
              ┌──────────┐
              │ pending  │ ← 기안서 작성
              └────┬─────┘
                   │
          ┌────────┼────────┐
          ▼        │        ▼
   ┌──────────┐    │  ┌──────────┐
   │ approved │    │  │ rejected │ ← 반려
   └────┬─────┘    │  └──────────┘
        │          │
        ▼          │  (삭제: pending 상태에서만 가능)
   ┌──────────┐    │
   │ returned │    │
   └──────────┘    │
   ↑ 반납 승인 시  │
     자동 변경     │
```

### 반납 기안서 (return_status)

```
   ┌──────────┐
   │ pending  │ ← 반납 기안서 작성
   └────┬─────┘
        │
   ┌────┼────────┐
   ▼              ▼
┌──────────┐  ┌──────────┐
│ returned │  │ rejected │
└──────────┘  └──────────┘
```

### 카드 상태 (card_status) — 기안서 연동

```
   ┌───────────┐  불출 승인   ┌────────┐
   │ available │ ──────────► │ in_use │
   └───────────┘             └────────┘
        ▲                        │
        │      반납 승인          │
        └────────────────────────┘
```

---

## 비즈니스 규칙

### 1. 1인 1카드 정책
- 사용자가 `approved` 상태(반납 기안서 미작성)인 불출 기안서를 보유하면 추가 불출 불가
- 검증 쿼리:
```sql
SELECT EXISTS (
  SELECT 1 FROM finance.card_checkout_requests cr
  WHERE cr.requester_id = :userId
    AND cr.status = 'approved'
    AND NOT EXISTS (
      SELECT 1 FROM finance.card_return_requests rr
      WHERE rr.checkout_request_id = cr.id
    )
);
```

### 2. 불출 승인 트랜잭션
```sql
BEGIN;
  -- 1) 기안서 상태 변경
  UPDATE finance.card_checkout_requests
    SET status = 'approved', approved_by = :approverId, approved_at = NOW()
    WHERE id = :requestId AND status = 'pending';

  -- 2) 카드 상태 변경
  UPDATE finance.corporate_cards
    SET status = 'in_use', current_holder_id = :requesterId
    WHERE id = :cardId AND status = 'available';

  -- 3) 이력 기록
  INSERT INTO finance.card_status_history (card_id, checkout_request_id, action, actor_id, previous_status, new_status)
    VALUES (:cardId, :requestId, 'checkout_approve', :approverId, 'pending', 'approved');
COMMIT;
```

### 3. 반납 승인 트랜잭션
```sql
BEGIN;
  -- 1) 반납 기안서 상태 변경
  UPDATE finance.card_return_requests
    SET status = 'returned', approved_by = :approverId, approved_at = NOW()
    WHERE id = :returnId AND status = 'pending';

  -- 2) 원본 불출 기안서 상태 변경
  UPDATE finance.card_checkout_requests
    SET status = 'returned'
    WHERE id = :checkoutId AND status = 'approved';

  -- 3) 카드 상태 복원
  UPDATE finance.corporate_cards
    SET status = 'available', current_holder_id = NULL
    WHERE id = :cardId AND status = 'in_use';

  -- 4) 이력 기록
  INSERT INTO finance.card_status_history (card_id, checkout_request_id, return_request_id, action, actor_id, previous_status, new_status)
    VALUES (:cardId, :checkoutId, :returnId, 'return_approve', :approverId, 'pending', 'returned');
COMMIT;
```

### 4. 기안번호 채번 규칙
- 형식: `FC-YYYYMMDD-NNN` (예: FC-20260223-001)
- FC = Finance Card, 일별 순번 3자리
- 생성 쿼리:
```sql
SELECT 'FC-' || TO_CHAR(NOW(), 'YYYYMMDD') || '-' ||
  LPAD((COALESCE(MAX(
    CAST(SUBSTRING(request_number FROM 13) AS INTEGER)
  ), 0) + 1)::TEXT, 3, '0')
FROM finance.card_checkout_requests
WHERE request_number LIKE 'FC-' || TO_CHAR(NOW(), 'YYYYMMDD') || '%';
```

### 5. 삭제 정책
- 불출 기안서: `pending` 상태에서만 삭제 가능 (soft delete 아님, 실제 삭제)
- 반납 기안서: 삭제 불가 (반려 후 재작성)
- 카드: `available` 상태에서만 삭제 가능 (비활성화 권장)
- 부서: 소속 사용자 0명일 때만 삭제 가능
- 사용자: 진행 중인 기안서가 없을 때만 삭제 가능

---

## 외부 참조

| 참조 대상 | 참조 방법 | 용도 |
|----------|---------|------|
| `organization.users` | JWT email → `LOWER(u.email)` 매칭 | 로그인 사용자 식별 |
| `organization.employees` | `users.employee_id` → `employees.id` | 직원-부서 연결 |
| `organization.departments` | 재귀 CTE로 부서 트리 상향 탐색 (`type='company'`) | 사용자 소속 법인 조회 |
| `organization.departments` | `card_departments.name` = `organization.departments.name` JOIN | 부서명 원본 조회, 부서 목록 조회 |
| `approval.documents` | `card_return_requests.document_number` | 전자결재 문서 외부 참조 (선택) |
| jabis-storage (MinIO) | `card_receipts.file_path` | 영수증 파일 저장소 |

> **법인 조회 방식**: `organization.users.email` → `employees.department_id` → 부서 트리를 `parent_id`로 상향 탐색하여 `type='company'`인 노드의 `name`을 법인명으로 사용한다. 재귀 CTE 사용, 이메일 비교는 `LOWER()` 대소문자 무시.
> **참고**: `card_departments.name`으로 `organization.departments.name`과 매칭하여 부서명을 조회한다. API 응답의 부서명은 항상 `organization.departments`에서 가져온다.
> **인증**: card_users 기반 인증은 JWT roles 기반(`'finance'` 역할)으로 전환됨 (ADR-002).

---

## 시드 데이터

### 부서 초기 데이터
```sql
INSERT INTO finance.card_departments (name, sort_order) VALUES
  ('서비스본부', 1),
  ('기획본부', 2),
  ('AI본부', 3),
  ('경영지원본부', 4);
```

### 카드 초기 데이터
```sql
INSERT INTO finance.corporate_cards (card_number, company, status) VALUES
  ('1234', '진학사', 'available'),
  ('5678', '진학사', 'available'),
  ('9012', '진학사', 'available'),
  ('3456', '진학어플라이', 'available'),
  ('7890', '진학어플라이', 'available'),
  ('1313', '진학어플라이', 'available');
```

---

## 운영 정책

### 백업
- `card_status_history`는 감사 로그이므로 장기 보관 (최소 5년)
- `card_receipts`의 실제 파일은 MinIO에 별도 백업

### 정리
- `returned` 상태의 기안서 데이터는 보관 (삭제하지 않음)
- `card_status_history`는 아카이빙 정책에 따라 연 단위로 분리 가능

### 모니터링
- 미승인 기안서(pending) 3일 이상 방치 시 알림
- 카드 `in_use` 상태 30일 초과 시 알림 (장기 미반납)
