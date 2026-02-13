# Organization DB 스키마

> **데이터베이스**: PostgreSQL (jabis)
> **스키마**: `organization`
> **배포 스크립트**: `jabis-api-gateway/sql/organization-deploy.sql`
> **생성일**: 2026-02-11

---

## 스키마 개요

```
organization.departments          부서 (계층구조)
    ├── organization.employees    직원 (부서 소속)
    └── ...
organization.users                사용자 (OAuth 계정 — 직원 매핑)
    └── organization.user_permissions  사용자별 권한
organization.permissions          권한 정의
```

---

## 테이블

### 1. organization.departments (부서)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 부서 ID (예: `division-support`) |
| `code` | TEXT | UNIQUE NOT NULL | - | 부서 코드 (예: `SUP`) |
| `name` | TEXT | NOT NULL | - | 부서명 |
| `parent_id` | TEXT | FK → departments(id) | NULL | 상위 부서 ID (계층구조) |
| `type` | TEXT | NOT NULL | - | 부서 유형 |
| `status` | TEXT | NOT NULL | `'active'` | 상태 |
| `description` | TEXT | - | NULL | 설명 |
| `sort_order` | INTEGER | - | `0` | 정렬 순서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

**부서 유형 (type)**
| 값 | 설명 | 계층 |
|----|------|------|
| `company` | 회사 | 1단계 |
| `division` | 본부 | 2단계 |
| `department` | 부 | 3단계 |
| `team` | 팀 | 4단계 |
| `part` | 파트 | 5단계 |

### 2. organization.employees (직원)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 직원 ID (예: `emp-001`) |
| `employee_number` | TEXT | - | NULL | 사번 |
| `name` | TEXT | NOT NULL | - | 이름 |
| `title` | TEXT | NOT NULL | - | 직급 (대표이사, 본부장, 팀장 등) |
| `email` | TEXT | - | NULL | 이메일 |
| `phone` | TEXT | - | NULL | 전화번호 |
| `department_id` | TEXT | FK → departments(id) | NULL | 소속 부서 ID |
| `status` | TEXT | NOT NULL | `'active'` | 상태 |
| `sort_order` | INTEGER | - | `0` | 정렬 순서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

### 3. organization.users (사용자)

OAuth 계정과 직원을 매핑하는 테이블

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 사용자 ID (JWT sub에서 `@` 앞부분) |
| `email` | TEXT | - | NULL | 이메일 |
| `display_name` | TEXT | - | NULL | 표시 이름 |
| `employee_id` | TEXT | FK → employees(id) | NULL | 매칭된 직원 ID |
| `is_super_admin` | BOOLEAN | NOT NULL | `FALSE` | 슈퍼관리자 여부 |
| `mapped_at` | TIMESTAMPTZ | - | NULL | 직원 매칭 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

**사용자 ID 규칙**: JWT의 `sub` 또는 `email`에서 `@` 앞부분만 추출 (절대 규칙)
- 예: `navskh@jinhak.com` → `navskh`

### 4. organization.permissions (권한 정의)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | TEXT | PK | - | 권한 ID (예: `page:portal`, `feature:org:edit`) |
| `category` | TEXT | NOT NULL | - | `page` 또는 `feature` |
| `name` | TEXT | NOT NULL | - | 권한 이름 |
| `description` | TEXT | - | NULL | 설명 |
| `parent_id` | TEXT | - | NULL | 상위 권한 (계층 구조용) |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

### 5. organization.user_permissions (사용자별 권한)

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `user_id` | TEXT | PK, FK → users(id) ON DELETE CASCADE | - | 사용자 ID |
| `permission_id` | TEXT | PK, FK → permissions(id) ON DELETE CASCADE | - | 권한 ID |
| `granted_by` | TEXT | - | NULL | 권한 부여자 |
| `granted_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 권한 부여 시각 |

---

## 인덱스

| 인덱스 | 테이블 | 컬럼 | 용도 |
|--------|--------|------|------|
| `idx_org_employees_department` | employees | `department_id` | 부서별 직원 조회 |
| `idx_org_employees_name` | employees | `name` | 이름 검색 |
| `idx_org_users_employee` | users | `employee_id` | 직원 매핑 조회 |
| `idx_org_user_permissions_user` | user_permissions | `user_id` | 사용자 권한 조회 |

---

## ER 다이어그램

```
departments (부서)
  ├── id (PK)
  ├── parent_id (FK → departments.id, self-referencing)
  └── ...
       │
       │ 1:N
       ▼
employees (직원)
  ├── id (PK)
  ├── department_id (FK → departments.id)
  └── ...
       │
       │ 1:1
       ▼
users (사용자)
  ├── id (PK)
  ├── employee_id (FK → employees.id)
  └── ...
       │
       │ N:M
       ▼
user_permissions
  ├── user_id (FK → users.id)
  └── permission_id (FK → permissions.id)
                          │
                          ▼
                    permissions (권한)
                      ├── id (PK)
                      └── ...
```

---

## 시드 데이터

### 배포 스크립트
- **파일**: `jabis-api-gateway/sql/organization-deploy.sql`
- 스키마 생성 + 시드 데이터 통합
- `BEGIN...COMMIT` 트랜잭션으로 원자성 보장
- `ON CONFLICT DO NOTHING`으로 중복 방지

### 데이터 규모
| 테이블 | 건수 | 비고 |
|--------|------|------|
| departments | 47+ | 진학사/진학어플라이 전체 조직도 |
| employees | 63+ | 본부장~사원 |
| users | 1 | ka3794 (슈퍼관리자) |
| permissions | 24 | 15 페이지 + 9 기능 |

### 배포 방법
```bash
psql -h <host> -U <user> -d jabis -f sql/organization-deploy.sql
```

---

## 운영 정책

- **DDL 관리**: DBA가 수동 SQL 스크립트로 실행 (앱에서 자동 DDL 실행 금지)
- **파라미터화 쿼리**: SQL Injection 방지 필수
- **트랜잭션**: 권한 일괄 설정 시 DELETE + INSERT 트랜잭션 사용
