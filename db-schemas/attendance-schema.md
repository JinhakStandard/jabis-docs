# 근태 관리 DB 스키마

> **데이터베이스**: PostgreSQL (jabis)
> **스키마**: `organization`
> **관련 프로젝트**: jabis-api-gateway
> **생성일**: 2026-03-03

---

## 스키마 개요

```
organization.attendance_types          근태 유형 마스터 (출근, 연차, 외근 등)
    │
    │ 1:N
    ▼
organization.attendance_records        일별 근태 기록 (출퇴근 시간, GPS, 상태)
    │
    │ N:1
    ▼
organization.employees                 직원 (부서 소속)
```

---

## status 값 정의

| 값 | 한글 | 설명 | 설정 방식 |
|----|------|------|----------|
| `pending` | 대기 | 아직 결정되지 않은 상태 | 시스템 |
| `normal` | 정상 | 09:00 이전 출근 | 시스템 (clock-in 시 자동 판별) |
| `late` | 지각 | 09:00 이후 출근 | 시스템 (clock-in 시 자동 판별) |
| `early_leave` | 조퇴 | 정규 퇴근 시간 전 퇴근 | 시스템 |
| `absent` | 결근 | 출근 기록 없음 | 배치/관리자 |
| `registered` | 등록 | 연차/외근/출장 등 수동 등록 | 사용자 (register action) |

---

## 테이블

### 1. organization.attendance_types (근태 유형)

근태 종류를 정의하는 마스터 테이블.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | VARCHAR(30) | PK | - | 근태 유형 ID (예: `normal`, `annual-leave`) |
| `name` | TEXT | NOT NULL | - | 근태 유형명 (예: `정상출근`, `연차`) |
| `category` | TEXT | - | NULL | 분류 (예: `출근`, `연차`, `외근`, `기타`) |
| `description` | TEXT | - | NULL | 설명 |
| `requires_approval` | BOOLEAN | NOT NULL | `FALSE` | 결재 필요 여부 |
| `is_active` | BOOLEAN | NOT NULL | `TRUE` | 활성화 여부 |
| `sort_order` | INTEGER | NOT NULL | `0` | 정렬 순서 |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |

#### DDL

```sql
CREATE TABLE IF NOT EXISTS organization.attendance_types (
  id            VARCHAR(30) PRIMARY KEY,
  name          TEXT NOT NULL,
  category      TEXT,
  description   TEXT,
  requires_approval BOOLEAN NOT NULL DEFAULT FALSE,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  sort_order    INTEGER NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### 기본 데이터 (10개)

| id | name | category | requires_approval | sort_order |
|----|------|----------|-------------------|------------|
| `normal` | 정상출근 | 출근 | false | 0 |
| `late` | 지각 | 출근 | false | 1 |
| `early-leave` | 조퇴 | 출근 | false | 2 |
| `annual-leave` | 연차 | 연차 | true | 10 |
| `half-day-am` | 오전 반차 | 연차 | true | 11 |
| `half-day-pm` | 오후 반차 | 연차 | true | 12 |
| `business-trip` | 출장 | 외근 | true | 20 |
| `outside-work` | 외근 | 외근 | false | 21 |
| `remote-work` | 재택근무 | 기타 | false | 30 |
| `sick-leave` | 병가 | 기타 | true | 31 |

---

### 2. organization.attendance_records (근태 기록)

직원별 일별 근태 기록. 출퇴근 시간과 GPS 좌표를 포함.

| 컬럼 | 타입 | 제약 | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | VARCHAR(50) | PK | - | 레코드 ID (UUID) |
| `employee_id` | TEXT | FK → employees(id), NOT NULL | - | 직원 ID |
| `date` | DATE | NOT NULL | - | 근태 날짜 |
| `attendance_type_id` | VARCHAR(30) | FK → attendance_types(id) | NULL | 근태 유형 ID |
| `clock_in_at` | TIMESTAMPTZ | - | NULL | 출근 시각 |
| `clock_in_latitude` | NUMERIC(10,7) | - | NULL | 출근 GPS 위도 |
| `clock_in_longitude` | NUMERIC(10,7) | - | NULL | 출근 GPS 경도 |
| `clock_in_accuracy` | NUMERIC(10,2) | - | NULL | 출근 GPS 정확도 (m) |
| `clock_out_at` | TIMESTAMPTZ | - | NULL | 퇴근 시각 |
| `clock_out_latitude` | NUMERIC(10,7) | - | NULL | 퇴근 GPS 위도 |
| `clock_out_longitude` | NUMERIC(10,7) | - | NULL | 퇴근 GPS 경도 |
| `clock_out_accuracy` | NUMERIC(10,2) | - | NULL | 퇴근 GPS 정확도 (m) |
| `status` | TEXT | NOT NULL | `'pending'` | 근태 상태 (status 값 정의 참조) |
| `note` | TEXT | - | NULL | 비고 (최대 200자) |
| `registered_by` | TEXT | - | NULL | 등록자 ID (수동 등록 시) |
| `created_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 생성일 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | `NOW()` | 수정일 |

**UNIQUE 제약**: `(employee_id, date)` — 직원당 하루에 1건만 허용

#### DDL

```sql
CREATE TABLE IF NOT EXISTS organization.attendance_records (
  id                  VARCHAR(50) PRIMARY KEY,
  employee_id         TEXT NOT NULL REFERENCES organization.employees(id),
  date                DATE NOT NULL,
  attendance_type_id  VARCHAR(30) REFERENCES organization.attendance_types(id),
  clock_in_at         TIMESTAMPTZ,
  clock_in_latitude   NUMERIC(10,7),
  clock_in_longitude  NUMERIC(10,7),
  clock_in_accuracy   NUMERIC(10,2),
  clock_out_at        TIMESTAMPTZ,
  clock_out_latitude  NUMERIC(10,7),
  clock_out_longitude NUMERIC(10,7),
  clock_out_accuracy  NUMERIC(10,2),
  status              TEXT NOT NULL DEFAULT 'pending',
  note                TEXT,
  registered_by       TEXT,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (employee_id, date)
);
```

---

## 인덱스

| 인덱스 | 테이블 | 컬럼 | 용도 |
|--------|--------|------|------|
| `idx_att_records_employee` | attendance_records | `employee_id` | 직원별 근태 조회 |
| `idx_att_records_date` | attendance_records | `date` | 날짜별 근태 조회 |
| `idx_att_records_status` | attendance_records | `status` | 상태별 필터링 |
| `idx_att_records_employee_date` | attendance_records | `employee_id, date` | 직원+날짜 복합 조회 (UNIQUE 인덱스) |
| `idx_att_types_active` | attendance_types | `is_active, sort_order` | 활성 유형 정렬 조회 |

---

## ER 다이어그램

```
attendance_types (근태 유형)
  ├── id (PK)
  ├── name
  ├── category
  └── ...
       │
       │ 1:N
       ▼
attendance_records (근태 기록)
  ├── id (PK)
  ├── employee_id (FK → employees.id)   ──── N:1 ──→  employees (직원)
  ├── date                                              ├── id (PK)
  ├── attendance_type_id (FK → attendance_types.id)     ├── name
  ├── clock_in_at / clock_out_at                        └── department_id (FK → departments.id)
  ├── clock_in_latitude / longitude / accuracy
  ├── clock_out_latitude / longitude / accuracy
  ├── status
  └── UNIQUE (employee_id, date)
```

---

## 시간 정책

- 모든 시간 컬럼은 **TIMESTAMPTZ** (UTC 저장)
- 출퇴근 판별 시 **KST (Asia/Seoul)** 변환: `(NOW() AT TIME ZONE 'Asia/Seoul')::TIME`
- 지각 기준: KST 09:00 이후 출근 → `status = 'late'`
- 프론트엔드 표시 시 KST로 변환

---

## 운영 정책

- **DDL 관리**: DBA가 수동 SQL 스크립트로 실행 (앱에서 자동 DDL 실행 금지)
- **파라미터화 쿼리**: SQL Injection 방지 필수 ($1, $2)
- **UPSERT**: 근태 등록(register) 시 `ON CONFLICT (employee_id, date) DO UPDATE` 사용
- **비동기 처리 없음**: 모든 쿼리는 동기 응답 (access-logs와 달리 즉시 결과 반환)
