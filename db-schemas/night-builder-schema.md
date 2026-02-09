# JABIS API Gateway - 데이터베이스 스키마 명세서

> JABIS Night Builder & Auto Debugger API Gateway 서버에 필요한 DB 테이블 정의.
> 스키마명: `nbuilder`

---

## 목차

1. [개요](#1-개요)
2. [스키마 생성](#2-스키마-생성)
3. [공통 타입](#3-공통-타입)
4. [테이블 정의](#4-테이블-정의)
5. [인덱스](#5-인덱스)
6. [제약 조건 요약](#6-제약-조건-요약)
7. [ERD](#7-erd)
8. [초기 데이터](#8-초기-데이터)
9. [운영 참고 사항](#9-운영-참고-사항)

---

## 1. 개요

### 대상 DBMS

PostgreSQL 15+

### 스키마

`nbuilder`

### 테이블 목록

| # | 테이블명 | 설명 |
|---|----------|------|
| 1 | `feature_requests` | 기능 개발 요청 |
| 2 | `implementation_plans` | 구현 계획 (Claude AI 수립) |
| 3 | `plan_steps` | 구현 계획 단계 |
| 4 | `debug_tasks` | 디버그(에러 수정) 태스크 |
| 5 | `status_histories` | 상태 변경 이력 |

---

## 2. 스키마 생성

```sql
CREATE SCHEMA IF NOT EXISTS nbuilder;

-- ENUM 타입 정의
CREATE TYPE nbuilder.request_status AS ENUM (
    'queued',
    'planning',
    'plan_review',
    'approved',
    'in_progress',
    'completed',
    'failed',
    'cancelled'
);

CREATE TYPE nbuilder.complexity_level AS ENUM (
    'low',
    'medium',
    'high'
);
```

---

## 3. 공통 타입

### 3.1 request_status

| 값 | 설명 | 적용 대상 |
|----|------|-----------|
| `queued` | 등록 직후 초기 상태 | feature_requests, debug_tasks |
| `planning` | Planning Service가 계획 수립 중 | feature_requests |
| `plan_review` | 계획 수립 완료, 사용자 검토 대기 | feature_requests |
| `approved` | 사용자 승인 완료, Night Builder 실행 대상 | feature_requests |
| `in_progress` | Night Builder / Auto Debugger 실행 중 | feature_requests, debug_tasks |
| `completed` | 처리 완료 | feature_requests, debug_tasks |
| `failed` | 처리 실패 | feature_requests, debug_tasks |
| `cancelled` | 사용자 취소 | feature_requests |

### 3.2 complexity_level

| 값 | 설명 |
|----|------|
| `low` | 낮은 복잡도 |
| `medium` | 중간 복잡도 |
| `high` | 높은 복잡도 |

---

## 4. 테이블 정의

### 4.1 feature_requests (기능 요청)

사용자가 등록하는 AI 기능 개발 요청.

```sql
CREATE TABLE nbuilder.feature_requests (
    id              VARCHAR(50)                 NOT NULL,
    title           VARCHAR(500)                NOT NULL,
    description     TEXT                        NOT NULL,
    prompt          TEXT                        NOT NULL,
    repository_url  VARCHAR(1000)               NOT NULL,
    branch          VARCHAR(255)                NULL,
    target_branch   VARCHAR(255)                NULL,
    status          nbuilder.request_status     NOT NULL DEFAULT 'queued',
    priority        INTEGER                     NOT NULL DEFAULT 0,
    plan_approved_at TIMESTAMPTZ               NULL,
    metadata        JSONB                       NULL     DEFAULT '{}',
    created_at      TIMESTAMPTZ                 NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ                 NOT NULL DEFAULT now(),

    CONSTRAINT pk_feature_requests PRIMARY KEY (id)
);

COMMENT ON TABLE  nbuilder.feature_requests IS '기능 개발 요청';
COMMENT ON COLUMN nbuilder.feature_requests.id IS '고유 식별자 (예: req-20260203-001)';
COMMENT ON COLUMN nbuilder.feature_requests.title IS '요청 제목';
COMMENT ON COLUMN nbuilder.feature_requests.description IS '요청 상세 설명';
COMMENT ON COLUMN nbuilder.feature_requests.prompt IS 'ai-orchestra에 전달할 프롬프트';
COMMENT ON COLUMN nbuilder.feature_requests.repository_url IS 'Git 저장소 URL';
COMMENT ON COLUMN nbuilder.feature_requests.branch IS '작업 기준 브랜치 (NULL이면 기본 브랜치)';
COMMENT ON COLUMN nbuilder.feature_requests.target_branch IS 'PR 대상 브랜치 (NULL이면 기본 브랜치)';
COMMENT ON COLUMN nbuilder.feature_requests.status IS '현재 상태';
COMMENT ON COLUMN nbuilder.feature_requests.priority IS '우선순위 (높을수록 먼저 처리)';
COMMENT ON COLUMN nbuilder.feature_requests.plan_approved_at IS '계획 승인 시각 (사용자가 approved 시 설정)';
COMMENT ON COLUMN nbuilder.feature_requests.metadata IS '추가 메타데이터 (JSON)';
COMMENT ON COLUMN nbuilder.feature_requests.created_at IS '생성 시각';
COMMENT ON COLUMN nbuilder.feature_requests.updated_at IS '최종 수정 시각';
```

**컬럼 상세:**

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | VARCHAR(50) | N | - | PK. 형식: `req-YYYYMMDD-NNN` |
| `title` | VARCHAR(500) | N | - | 요청 제목 |
| `description` | TEXT | N | - | 요청 상세 설명 |
| `prompt` | TEXT | N | - | ai-orchestra 전달 프롬프트 |
| `repository_url` | VARCHAR(1000) | N | - | Git 저장소 URL |
| `branch` | VARCHAR(255) | Y | NULL | 작업 기준 브랜치 |
| `target_branch` | VARCHAR(255) | Y | NULL | PR 대상 브랜치 |
| `status` | request_status | N | `'queued'` | 현재 상태 |
| `priority` | INTEGER | N | `0` | 우선순위 (DESC 정렬) |
| `plan_approved_at` | TIMESTAMPTZ | Y | NULL | 계획 승인 시각 |
| `metadata` | JSONB | Y | `'{}'` | 추가 메타데이터 |
| `created_at` | TIMESTAMPTZ | N | `now()` | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | N | `now()` | 최종 수정 시각 |

---

### 4.2 implementation_plans (구현 계획)

Planning Service(Claude AI)가 수립한 구현 계획. `feature_requests`와 1:1 관계.

```sql
CREATE TABLE nbuilder.implementation_plans (
    id                   BIGINT GENERATED ALWAYS AS IDENTITY,
    request_id           VARCHAR(50)                 NOT NULL,
    summary              TEXT                        NOT NULL,
    estimated_complexity nbuilder.complexity_level    NOT NULL DEFAULT 'medium',
    affected_areas       JSONB                       NOT NULL DEFAULT '[]',
    created_at           TIMESTAMPTZ                 NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ                 NOT NULL DEFAULT now(),

    CONSTRAINT pk_implementation_plans PRIMARY KEY (id),
    CONSTRAINT uq_implementation_plans_request_id UNIQUE (request_id),
    CONSTRAINT fk_implementation_plans_request
        FOREIGN KEY (request_id) REFERENCES nbuilder.feature_requests(id)
        ON DELETE CASCADE
);

COMMENT ON TABLE  nbuilder.implementation_plans IS '구현 계획 (Claude AI 수립)';
COMMENT ON COLUMN nbuilder.implementation_plans.id IS '자동 생성 PK';
COMMENT ON COLUMN nbuilder.implementation_plans.request_id IS '기능 요청 ID (FK, UNIQUE)';
COMMENT ON COLUMN nbuilder.implementation_plans.summary IS '계획 요약 (한국어, 비개발자 대상)';
COMMENT ON COLUMN nbuilder.implementation_plans.estimated_complexity IS '예상 복잡도 (low/medium/high)';
COMMENT ON COLUMN nbuilder.implementation_plans.affected_areas IS '영향 받는 영역 목록 (JSON 배열)';
COMMENT ON COLUMN nbuilder.implementation_plans.created_at IS '생성 시각';
COMMENT ON COLUMN nbuilder.implementation_plans.updated_at IS '최종 수정 시각';
```

**컬럼 상세:**

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | BIGINT (IDENTITY) | N | 자동 | PK |
| `request_id` | VARCHAR(50) | N | - | FK → feature_requests.id (UNIQUE) |
| `summary` | TEXT | N | - | 전체 계획 요약 (한국어) |
| `estimated_complexity` | complexity_level | N | `'medium'` | 예상 복잡도 |
| `affected_areas` | JSONB | N | `'[]'` | 영향 영역 목록 (예: `["로그인 페이지", "이메일 발송"]`) |
| `created_at` | TIMESTAMPTZ | N | `now()` | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | N | `now()` | 최종 수정 시각 |

**affected_areas 예시:**

```json
["로그인 페이지", "이메일 발송", "사용자 인증"]
```

---

### 4.3 plan_steps (계획 단계)

구현 계획의 세부 단계. `implementation_plans`와 1:N 관계.

```sql
CREATE TABLE nbuilder.plan_steps (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    plan_id     BIGINT                      NOT NULL,
    step_order  INTEGER                     NOT NULL,
    title       VARCHAR(500)                NOT NULL,
    description TEXT                        NOT NULL,

    CONSTRAINT pk_plan_steps PRIMARY KEY (id),
    CONSTRAINT uq_plan_steps_order UNIQUE (plan_id, step_order),
    CONSTRAINT fk_plan_steps_plan
        FOREIGN KEY (plan_id) REFERENCES nbuilder.implementation_plans(id)
        ON DELETE CASCADE
);

COMMENT ON TABLE  nbuilder.plan_steps IS '구현 계획 단계';
COMMENT ON COLUMN nbuilder.plan_steps.id IS '자동 생성 PK';
COMMENT ON COLUMN nbuilder.plan_steps.plan_id IS '구현 계획 ID (FK)';
COMMENT ON COLUMN nbuilder.plan_steps.step_order IS '단계 순서 (1부터 시작)';
COMMENT ON COLUMN nbuilder.plan_steps.title IS '단계 제목';
COMMENT ON COLUMN nbuilder.plan_steps.description IS '단계 상세 설명';
```

**컬럼 상세:**

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | BIGINT (IDENTITY) | N | 자동 | PK |
| `plan_id` | BIGINT | N | - | FK → implementation_plans.id |
| `step_order` | INTEGER | N | - | 단계 순서 (plan_id + step_order UNIQUE) |
| `title` | VARCHAR(500) | N | - | 단계 제목 |
| `description` | TEXT | N | - | 단계 상세 설명 |

---

### 4.4 debug_tasks (디버그 태스크)

에러 자동 수정 태스크.

```sql
CREATE TABLE nbuilder.debug_tasks (
    id              VARCHAR(50)                 NOT NULL,
    error_message   TEXT                        NOT NULL,
    stack_trace     TEXT                        NOT NULL,
    repository_url  VARCHAR(1000)               NOT NULL,
    branch          VARCHAR(255)                NOT NULL,
    file_path       VARCHAR(1000)               NULL,
    context         TEXT                        NULL,
    status          nbuilder.request_status     NOT NULL DEFAULT 'queued',
    priority        INTEGER                     NOT NULL DEFAULT 0,
    metadata        JSONB                       NULL     DEFAULT '{}',
    created_at      TIMESTAMPTZ                 NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ                 NOT NULL DEFAULT now(),

    CONSTRAINT pk_debug_tasks PRIMARY KEY (id)
);

COMMENT ON TABLE  nbuilder.debug_tasks IS '디버그(에러 수정) 태스크';
COMMENT ON COLUMN nbuilder.debug_tasks.id IS '고유 식별자 (예: debug-20260203-001)';
COMMENT ON COLUMN nbuilder.debug_tasks.error_message IS '에러 메시지';
COMMENT ON COLUMN nbuilder.debug_tasks.stack_trace IS '스택 트레이스';
COMMENT ON COLUMN nbuilder.debug_tasks.repository_url IS 'Git 저장소 URL';
COMMENT ON COLUMN nbuilder.debug_tasks.branch IS '작업 대상 브랜치';
COMMENT ON COLUMN nbuilder.debug_tasks.file_path IS '에러 발생 추정 파일 경로';
COMMENT ON COLUMN nbuilder.debug_tasks.context IS '추가 컨텍스트 정보';
COMMENT ON COLUMN nbuilder.debug_tasks.status IS '현재 상태';
COMMENT ON COLUMN nbuilder.debug_tasks.priority IS '우선순위 (높을수록 먼저 처리)';
COMMENT ON COLUMN nbuilder.debug_tasks.metadata IS '추가 메타데이터 (JSON)';
COMMENT ON COLUMN nbuilder.debug_tasks.created_at IS '생성 시각';
COMMENT ON COLUMN nbuilder.debug_tasks.updated_at IS '최종 수정 시각';
```

**컬럼 상세:**

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | VARCHAR(50) | N | - | PK. 형식: `debug-YYYYMMDD-NNN` |
| `error_message` | TEXT | N | - | 에러 메시지 |
| `stack_trace` | TEXT | N | - | 스택 트레이스 |
| `repository_url` | VARCHAR(1000) | N | - | Git 저장소 URL |
| `branch` | VARCHAR(255) | N | - | 작업 대상 브랜치 |
| `file_path` | VARCHAR(1000) | Y | NULL | 에러 발생 추정 파일 경로 |
| `context` | TEXT | Y | NULL | 추가 컨텍스트 |
| `status` | request_status | N | `'queued'` | 현재 상태 |
| `priority` | INTEGER | N | `0` | 우선순위 (DESC 정렬) |
| `metadata` | JSONB | Y | `'{}'` | 추가 메타데이터 |
| `created_at` | TIMESTAMPTZ | N | `now()` | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | N | `now()` | 최종 수정 시각 |

---

### 4.5 status_histories (상태 변경 이력)

모든 상태 변경을 기록하는 감사 테이블.

```sql
CREATE TABLE nbuilder.status_histories (
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    entity_type     VARCHAR(20)                 NOT NULL,
    entity_id       VARCHAR(50)                 NOT NULL,
    from_status     nbuilder.request_status     NULL,
    to_status       nbuilder.request_status     NOT NULL,
    message         TEXT                        NULL,
    result          JSONB                       NULL,
    changed_by      VARCHAR(100)                NOT NULL,
    created_at      TIMESTAMPTZ                 NOT NULL DEFAULT now(),

    CONSTRAINT pk_status_histories PRIMARY KEY (id),
    CONSTRAINT ck_status_histories_entity_type
        CHECK (entity_type IN ('feature_request', 'debug_task'))
);

COMMENT ON TABLE  nbuilder.status_histories IS '상태 변경 이력 (감사 로그)';
COMMENT ON COLUMN nbuilder.status_histories.id IS '자동 생성 PK';
COMMENT ON COLUMN nbuilder.status_histories.entity_type IS '대상 엔티티 타입 (feature_request / debug_task)';
COMMENT ON COLUMN nbuilder.status_histories.entity_id IS '대상 엔티티 ID';
COMMENT ON COLUMN nbuilder.status_histories.from_status IS '변경 전 상태 (최초 등록 시 NULL)';
COMMENT ON COLUMN nbuilder.status_histories.to_status IS '변경 후 상태';
COMMENT ON COLUMN nbuilder.status_histories.message IS '상태 변경 사유/설명';
COMMENT ON COLUMN nbuilder.status_histories.result IS '처리 결과 데이터 (completed/failed 시)';
COMMENT ON COLUMN nbuilder.status_histories.changed_by IS '변경 주체 (planning_service / night_builder / auto_debugger / user / system)';
COMMENT ON COLUMN nbuilder.status_histories.created_at IS '변경 시각';
```

**컬럼 상세:**

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| `id` | BIGINT (IDENTITY) | N | 자동 | PK |
| `entity_type` | VARCHAR(20) | N | - | `'feature_request'` 또는 `'debug_task'` |
| `entity_id` | VARCHAR(50) | N | - | 대상 엔티티 ID |
| `from_status` | request_status | Y | NULL | 변경 전 상태 |
| `to_status` | request_status | N | - | 변경 후 상태 |
| `message` | TEXT | Y | NULL | 변경 사유 |
| `result` | JSONB | Y | NULL | 처리 결과 (completed/failed 시) |
| `changed_by` | VARCHAR(100) | N | - | 변경 주체 |
| `created_at` | TIMESTAMPTZ | N | `now()` | 변경 시각 |

**changed_by 값:**

| 값 | 설명 |
|----|------|
| `planning_service` | Planning Service에 의한 변경 |
| `night_builder` | Night Builder에 의한 변경 |
| `auto_debugger` | Auto Debugger에 의한 변경 |
| `user` | 사용자(웹 UI)에 의한 변경 |
| `system` | 시스템 자동 복구 (타임아웃 등) |

**result 예시 (completed):**

```json
{
  "branchName": "feature/req-20260203-001",
  "prUrl": "https://github.com/company/my-project/pull/42",
  "testResults": [
    { "command": "npm test", "passed": true, "output": "..." },
    { "command": "npm run lint", "passed": true, "output": "..." }
  ],
  "phasesExecuted": 3
}
```

**result 예시 (failed):**

```json
{
  "testResults": [
    { "command": "npm test", "passed": false, "output": "FAIL src/..." }
  ]
}
```

---

## 5. 인덱스

### 5.1 feature_requests 인덱스

```sql
-- 상태별 조회 (Planning Service: queued, Night Builder: approved)
-- priority DESC 정렬 포함
CREATE INDEX idx_feature_requests_status_priority
    ON nbuilder.feature_requests (status, priority DESC);

-- 저장소별 조회 (동시성 관리용)
CREATE INDEX idx_feature_requests_repository_url
    ON nbuilder.feature_requests (repository_url)
    WHERE status IN ('approved', 'in_progress');

-- 생성일시 범위 조회
CREATE INDEX idx_feature_requests_created_at
    ON nbuilder.feature_requests (created_at DESC);
```

### 5.2 implementation_plans 인덱스

```sql
-- request_id UNIQUE 인덱스는 제약 조건에 의해 자동 생성됨
```

### 5.3 plan_steps 인덱스

```sql
-- plan_id + step_order UNIQUE 인덱스는 제약 조건에 의해 자동 생성됨

-- plan_id 단독 조회용
CREATE INDEX idx_plan_steps_plan_id
    ON nbuilder.plan_steps (plan_id);
```

### 5.4 debug_tasks 인덱스

```sql
-- 상태별 조회 (Auto Debugger: queued)
CREATE INDEX idx_debug_tasks_status_priority
    ON nbuilder.debug_tasks (status, priority DESC);

-- 저장소별 조회
CREATE INDEX idx_debug_tasks_repository_url
    ON nbuilder.debug_tasks (repository_url)
    WHERE status IN ('queued', 'in_progress');

-- 생성일시 범위 조회
CREATE INDEX idx_debug_tasks_created_at
    ON nbuilder.debug_tasks (created_at DESC);
```

### 5.5 status_histories 인덱스

```sql
-- 엔티티별 이력 조회
CREATE INDEX idx_status_histories_entity
    ON nbuilder.status_histories (entity_type, entity_id, created_at DESC);

-- 시간순 조회
CREATE INDEX idx_status_histories_created_at
    ON nbuilder.status_histories (created_at DESC);
```

---

## 6. 제약 조건 요약

### 6.1 Primary Key

| 테이블 | 제약 조건명 | 컬럼 |
|--------|-------------|------|
| feature_requests | pk_feature_requests | id |
| implementation_plans | pk_implementation_plans | id |
| plan_steps | pk_plan_steps | id |
| debug_tasks | pk_debug_tasks | id |
| status_histories | pk_status_histories | id |

### 6.2 Foreign Key

| 제약 조건명 | 테이블 | 컬럼 | 참조 | ON DELETE |
|-------------|--------|------|------|-----------|
| fk_implementation_plans_request | implementation_plans | request_id | feature_requests(id) | CASCADE |
| fk_plan_steps_plan | plan_steps | plan_id | implementation_plans(id) | CASCADE |

### 6.3 Unique

| 제약 조건명 | 테이블 | 컬럼 |
|-------------|--------|------|
| uq_implementation_plans_request_id | implementation_plans | request_id |
| uq_plan_steps_order | plan_steps | (plan_id, step_order) |

### 6.4 Check

| 제약 조건명 | 테이블 | 조건 |
|-------------|--------|------|
| ck_status_histories_entity_type | status_histories | entity_type IN ('feature_request', 'debug_task') |

---

## 7. ERD

```
┌───────────────────────────┐
│    feature_requests       │
├───────────────────────────┤
│ * id          VARCHAR(50) │──────┐
│   title       VARCHAR(500)│      │
│   description TEXT        │      │
│   prompt      TEXT        │      │
│   repository_url VARCHAR  │      │
│   branch      VARCHAR(255)│      │
│   target_branch VARCHAR   │      │
│   status      request_status     │
│   priority    INTEGER     │      │
│   plan_approved_at TIMESTAMPTZ   │
│   metadata    JSONB       │      │
│   created_at  TIMESTAMPTZ │      │
│   updated_at  TIMESTAMPTZ │      │
└───────────────────────────┘      │
           │                       │
           │ 1:1                   │
           ▼                       │
┌───────────────────────────┐      │
│  implementation_plans     │      │
├───────────────────────────┤      │
│ * id          BIGINT (PK) │      │
│   request_id  VARCHAR(50) │──────┘ FK (UNIQUE)
│   summary     TEXT        │
│   estimated_complexity    │
│               complexity_level   │
│   affected_areas JSONB    │──────┐
│   created_at  TIMESTAMPTZ │      │
│   updated_at  TIMESTAMPTZ │      │
└───────────────────────────┘      │
           │                       │
           │ 1:N                   │
           ▼                       │
┌───────────────────────────┐      │
│      plan_steps           │      │
├───────────────────────────┤      │
│ * id          BIGINT (PK) │      │
│   plan_id     BIGINT      │──────┘ FK
│   step_order  INTEGER     │
│   title       VARCHAR(500)│
│   description TEXT        │
└───────────────────────────┘


┌───────────────────────────┐
│      debug_tasks          │
├───────────────────────────┤
│ * id          VARCHAR(50) │
│   error_message TEXT      │
│   stack_trace TEXT        │
│   repository_url VARCHAR  │
│   branch      VARCHAR(255)│
│   file_path   VARCHAR(1000)
│   context     TEXT        │
│   status      request_status
│   priority    INTEGER     │
│   metadata    JSONB       │
│   created_at  TIMESTAMPTZ │
│   updated_at  TIMESTAMPTZ │
└───────────────────────────┘


┌───────────────────────────┐
│    status_histories       │
├───────────────────────────┤
│ * id          BIGINT (PK) │
│   entity_type VARCHAR(20) │ ─── 'feature_request' → feature_requests.id
│   entity_id   VARCHAR(50) │ ─── 'debug_task'      → debug_tasks.id
│   from_status request_status
│   to_status   request_status
│   message     TEXT        │
│   result      JSONB       │
│   changed_by  VARCHAR(100)│
│   created_at  TIMESTAMPTZ │
└───────────────────────────┘
```

### 관계 요약

```
feature_requests  1 ──── 1  implementation_plans  (request_id FK, UNIQUE)
implementation_plans  1 ──── N  plan_steps          (plan_id FK)
feature_requests  1 ──── N  status_histories        (entity_type='feature_request')
debug_tasks       1 ──── N  status_histories        (entity_type='debug_task')
```

---

## 8. 초기 데이터

별도의 초기 데이터는 필요하지 않습니다. ENUM 타입(`request_status`, `complexity_level`)이 스키마 생성 시 함께 정의됩니다.

---

## 9. 운영 참고 사항

### 9.1 updated_at 자동 갱신 트리거

```sql
CREATE OR REPLACE FUNCTION nbuilder.update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_feature_requests_updated_at
    BEFORE UPDATE ON nbuilder.feature_requests
    FOR EACH ROW EXECUTE FUNCTION nbuilder.update_updated_at();

CREATE TRIGGER trg_implementation_plans_updated_at
    BEFORE UPDATE ON nbuilder.implementation_plans
    FOR EACH ROW EXECUTE FUNCTION nbuilder.update_updated_at();

CREATE TRIGGER trg_debug_tasks_updated_at
    BEFORE UPDATE ON nbuilder.debug_tasks
    FOR EACH ROW EXECUTE FUNCTION nbuilder.update_updated_at();
```

### 9.2 상태 전이 검증

API 서버에서 상태 변경 시 유효한 전이인지 검증하는 것을 권장합니다.

**feature_requests 허용 전이:**

```
queued       → planning        (Planning Service)
planning     → plan_review     (Planning Service - 계획 등록과 동시)
planning     → queued          (Planning Service - 실패 시 복원)
plan_review  → approved        (사용자 - 웹 UI)
plan_review  → cancelled       (사용자 - 웹 UI)
approved     → in_progress     (Night Builder)
in_progress  → completed       (Night Builder)
in_progress  → failed          (Night Builder)
```

**debug_tasks 허용 전이:**

```
queued       → in_progress     (Auto Debugger)
in_progress  → completed       (Auto Debugger)
in_progress  → failed          (Auto Debugger)
```

### 9.3 낙관적 잠금 (동시성 보호)

여러 JABIS 인스턴스가 동시에 같은 요청을 가져갈 수 있으므로, 상태 변경 시 현재 상태를 WHERE 조건에 포함하는 것을 권장합니다.

```sql
-- 예: queued → planning 전이 시
UPDATE nbuilder.feature_requests
SET status = 'planning', updated_at = now()
WHERE id = :request_id AND status = 'queued';

-- affected rows = 0 이면 이미 다른 인스턴스가 처리 중 → 409 Conflict 반환
```

### 9.4 타임아웃 복구

`planning` 또는 `in_progress` 상태에서 장시간(예: 2시간) 경과된 요청을 자동 복원하는 스케줄 작업을 권장합니다.

```sql
-- planning 상태 2시간 초과 → queued로 복원
UPDATE nbuilder.feature_requests
SET status = 'queued', updated_at = now()
WHERE status = 'planning'
  AND updated_at < now() - INTERVAL '2 hours';

-- in_progress 상태 2시간 초과 → approved로 복원 (재처리 대상)
UPDATE nbuilder.feature_requests
SET status = 'approved', updated_at = now()
WHERE status = 'in_progress'
  AND updated_at < now() - INTERVAL '2 hours';

-- debug_tasks: in_progress 2시간 초과 → queued로 복원
UPDATE nbuilder.debug_tasks
SET status = 'queued', updated_at = now()
WHERE status = 'in_progress'
  AND updated_at < now() - INTERVAL '2 hours';
```

### 9.5 데이터 보관 정책

완료/실패/취소된 오래된 데이터는 별도 아카이브 테이블로 이동하거나 삭제하는 정책을 권장합니다.

```sql
-- 예: 90일 경과된 completed/failed/cancelled 데이터 삭제
DELETE FROM nbuilder.feature_requests
WHERE status IN ('completed', 'failed', 'cancelled')
  AND updated_at < now() - INTERVAL '90 days';

DELETE FROM nbuilder.debug_tasks
WHERE status IN ('completed', 'failed')
  AND updated_at < now() - INTERVAL '90 days';

-- status_histories는 별도 보관 주기 적용
DELETE FROM nbuilder.status_histories
WHERE created_at < now() - INTERVAL '180 days';
```

### 9.6 API 응답 시 계획 데이터 조합

`GET /api/requests?status=approved` 응답 시 `plan` 필드를 포함해야 합니다. 서버에서 JOIN으로 조합합니다.

```sql
-- feature_request + implementation_plan + plan_steps 조합 쿼리
SELECT
    fr.*,
    json_build_object(
        'summary', ip.summary,
        'estimatedComplexity', ip.estimated_complexity,
        'affectedAreas', ip.affected_areas,
        'steps', (
            SELECT json_agg(
                json_build_object(
                    'order', ps.step_order,
                    'title', ps.title,
                    'description', ps.description
                ) ORDER BY ps.step_order
            )
            FROM nbuilder.plan_steps ps
            WHERE ps.plan_id = ip.id
        )
    ) AS plan
FROM nbuilder.feature_requests fr
LEFT JOIN nbuilder.implementation_plans ip ON ip.request_id = fr.id
WHERE fr.status = 'approved'
ORDER BY fr.priority DESC;
```

### 9.7 전체 DDL 실행 순서

```
1. CREATE SCHEMA nbuilder
2. CREATE TYPE nbuilder.request_status
3. CREATE TYPE nbuilder.complexity_level
4. CREATE TABLE nbuilder.feature_requests
5. CREATE TABLE nbuilder.implementation_plans
6. CREATE TABLE nbuilder.plan_steps
7. CREATE TABLE nbuilder.debug_tasks
8. CREATE TABLE nbuilder.status_histories
9. CREATE INDEX (전체)
10. CREATE FUNCTION nbuilder.update_updated_at()
11. CREATE TRIGGER (전체)
```
