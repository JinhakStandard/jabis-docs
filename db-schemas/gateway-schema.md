# Gateway 스키마 (gateway)

> jabis-api-gateway의 API 관리 관련 테이블.

---

## gateway.api_endpoints

동적 SQL 기반 API 엔드포인트 정의.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | VARCHAR(30) PK | 엔드포인트 ID |
| `name` | VARCHAR(200) | 이름 |
| `description` | TEXT | 설명 |
| `method` | VARCHAR(10) | HTTP 메서드 (GET/POST) |
| `path_pattern` | VARCHAR(300) | URL 경로 패턴 |
| `query` | TEXT | SQL 쿼리 |
| `parameters` | JSONB | 파라미터 정의 |
| `auth_required` | BOOLEAN | 인증 필요 여부 |
| `required_scopes` | JSONB | 필요 스코프 |
| `allow_pending_access` | BOOLEAN | 미승인 접근 허용 |
| `default_rate_limit` | INTEGER | 기본 Rate Limit |
| `default_daily_quota` | INTEGER | 기본 일일 쿼터 |
| `status` | VARCHAR(20) | `active`/`deprecated`/`disabled` |
| `owner_id` | VARCHAR(100) | 소유자 ID |
| `created_at` | TIMESTAMPTZ | 생성일 |
| `updated_at` | TIMESTAMPTZ | 수정일 |

---

## gateway.api_access_logs

API 접근 로그. 동적 API 엔드포인트 호출 시 `logService.saveAccessLog()`에 의해 비동기 자동 기록.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | SERIAL PK | 자동 증가 ID |
| `request_id` | VARCHAR(255) NOT NULL | 요청 고유 ID (Fastify requestId) |
| `client_id` | VARCHAR(255) | 클라이언트 ID (JWT에서 추출) |
| `endpoint_id` | VARCHAR(50) | 호출된 엔드포인트 ID |
| `method` | VARCHAR(10) NOT NULL | HTTP 메서드 (GET/POST) |
| `path` | TEXT NOT NULL | 요청 경로 (쿼리스트링 제외) |
| `query_params` | JSONB | 쿼리 파라미터 |
| `response_status` | INTEGER NOT NULL | HTTP 응답 상태 코드 |
| `response_time_ms` | INTEGER NOT NULL | 응답 소요 시간 (ms) |
| `permission_status` | VARCHAR(20) | 권한 상태 (`pending`/`allowed`/`blocked`) |
| `client_ip` | VARCHAR(45) | 클라이언트 IP |
| `user_agent` | TEXT | User-Agent 헤더 |
| `requested_at` | TIMESTAMPTZ NOT NULL | 요청 시각 (기본값: NOW()) |

**인덱스:**
- `idx_access_logs_request_id` — `request_id`
- `idx_access_logs_client_id` — `client_id`
- `idx_access_logs_endpoint_id` — `endpoint_id`
- `idx_access_logs_requested_at` — `requested_at DESC`
- `idx_access_logs_path` — `path`
- `idx_access_logs_method` — `method`

**DDL 파일:** `jabis-api-gateway/sql/gateway-schema.sql`

---

## gateway.client_api_permissions

클라이언트별 API 접근 권한.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | VARCHAR(30) PK | 권한 ID |
| `client_id` | VARCHAR | 클라이언트 ID |
| `endpoint_id` | VARCHAR FK | 엔드포인트 ID |
| `permission_status` | VARCHAR | `pending`/`allowed`/`blocked` |
| `rate_limit` | INTEGER | Rate Limit |
| `daily_quota` | INTEGER | 일일 쿼터 |
| `approved_by` | VARCHAR | 승인자 |
| `approved_at` | TIMESTAMPTZ | 승인일 |
| `blocked_reason` | VARCHAR | 차단 사유 |
| `created_at` | TIMESTAMPTZ | 생성일 |
| `updated_at` | TIMESTAMPTZ | 수정일 |

---

## gateway.api_tasks

프로젝트 단위 API 요청 관리.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | VARCHAR(30) PK | `atask-{YYYYMMDD}-{rand}` |
| `title` | VARCHAR(300) NOT NULL | 요청 제목 |
| `description` | TEXT | 설명 |
| `spec` | TEXT NOT NULL | 요청 명세 |
| `spec_format` | VARCHAR(20) | `markdown`/`text`/`json` |
| `requested_by` | VARCHAR(200) | 요청자 email (JWT 자동 추출) |
| `project` | VARCHAR(50) NOT NULL | 대상 프로젝트명 (예: `jabis-producer`) |
| `status` | VARCHAR(20) NOT NULL | `pending`/`in_progress`/`completed`/`verified`/`rejected` |
| `priority` | INTEGER NOT NULL | 우선순위 (0: 보통, 5: 높음, 10: 긴급) |
| `assignee` | VARCHAR(100) | 구현 담당자 |
| `endpoints` | JSONB | 엔드포인트 참조 |
| `result_note` | TEXT | 처리 결과 메모 |
| `metadata` | JSONB | 자유 메타데이터 |
| `created_at` | TIMESTAMPTZ NOT NULL | 생성일 |
| `updated_at` | TIMESTAMPTZ NOT NULL | 수정일 |
| `completed_at` | TIMESTAMPTZ | 완료일 |

**인덱스:**
- `idx_api_tasks_project` — `project`
- `idx_api_tasks_status` — `status`
- `idx_api_tasks_requested_by` — `requested_by`
