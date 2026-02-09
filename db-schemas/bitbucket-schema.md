# JABIS Bitbucket Sync - Database Schema

> 이 문서는 jabis-bitbucket-sync 프로젝트의 DB 스키마 및 API Gateway 엔드포인트 정보를 기록합니다.
> 작업 시 이 문서를 참고하세요.

## 1. PostgreSQL Schema: `bitbucket`

### 1.1 Enum Types

```sql
-- 프로젝트 타입
CREATE TYPE bitbucket.project_type AS ENUM ('backend', 'frontend', 'fullstack', 'library', 'infrastructure', 'documentation', 'other');

-- 저장소 상태
CREATE TYPE bitbucket.repo_status AS ENUM ('active', 'archived', 'deprecated');

-- 패키지 매니저
CREATE TYPE bitbucket.package_manager AS ENUM ('npm', 'yarn', 'pnpm', 'pip', 'poetry', 'maven', 'gradle', 'composer', 'cargo', 'go', 'nuget', 'other');

-- 업데이트 타입
CREATE TYPE bitbucket.update_type AS ENUM ('major', 'minor', 'patch', 'prerelease', 'unknown');

-- 동기화 타입
CREATE TYPE bitbucket.sync_type AS ENUM ('full', 'incremental', 'repository', 'branch', 'dependency');

-- 동기화 상태
CREATE TYPE bitbucket.sync_status AS ENUM ('running', 'completed', 'failed', 'cancelled');
```

### 1.2 Tables

#### bb_workspaces (워크스페이스)
```sql
CREATE TABLE bitbucket.bb_workspaces (
    id SERIAL PRIMARY KEY,
    uuid VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### bb_repositories (저장소)
```sql
CREATE TABLE bitbucket.bb_repositories (
    id SERIAL PRIMARY KEY,
    uuid VARCHAR(100) UNIQUE NOT NULL,
    workspace_uuid VARCHAR(100) NOT NULL REFERENCES bitbucket.bb_workspaces(uuid),
    slug VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    project_type bitbucket.project_type DEFAULT 'other',
    description TEXT,
    language VARCHAR(50),
    main_branch VARCHAR(100) DEFAULT 'main',
    is_private BOOLEAN DEFAULT true,
    status bitbucket.repo_status DEFAULT 'active',
    has_issues BOOLEAN DEFAULT false,
    has_wiki BOOLEAN DEFAULT false,
    last_commit_at TIMESTAMP,
    last_commit_hash VARCHAR(40),
    clone_url_https TEXT,
    clone_url_ssh TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_bb_repositories_workspace ON bitbucket.bb_repositories(workspace_uuid);
CREATE INDEX idx_bb_repositories_status ON bitbucket.bb_repositories(status);
CREATE INDEX idx_bb_repositories_language ON bitbucket.bb_repositories(language);
```

#### bb_branches (브랜치)
```sql
CREATE TABLE bitbucket.bb_branches (
    id SERIAL PRIMARY KEY,
    repository_uuid VARCHAR(100) NOT NULL REFERENCES bitbucket.bb_repositories(uuid),
    name VARCHAR(255) NOT NULL,
    commit_hash VARCHAR(40),
    commit_message TEXT,
    commit_author VARCHAR(255),
    commit_date TIMESTAMP,
    is_default BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(repository_uuid, name)
);

CREATE INDEX idx_bb_branches_repository ON bitbucket.bb_branches(repository_uuid);
```

#### bb_dependencies (의존성)
```sql
CREATE TABLE bitbucket.bb_dependencies (
    id SERIAL PRIMARY KEY,
    repository_uuid VARCHAR(100) NOT NULL REFERENCES bitbucket.bb_repositories(uuid),
    package_manager bitbucket.package_manager NOT NULL,
    package_name VARCHAR(255) NOT NULL,
    current_version VARCHAR(50),
    latest_version VARCHAR(50),
    update_type bitbucket.update_type,
    is_dev_dependency BOOLEAN DEFAULT false,
    file_path VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(repository_uuid, package_manager, package_name)
);

CREATE INDEX idx_bb_dependencies_repository ON bitbucket.bb_dependencies(repository_uuid);
CREATE INDEX idx_bb_dependencies_outdated ON bitbucket.bb_dependencies(update_type) WHERE update_type IN ('major', 'minor');
```

#### bb_sync_logs (동기화 로그)
```sql
CREATE TABLE bitbucket.bb_sync_logs (
    id SERIAL PRIMARY KEY,
    sync_type bitbucket.sync_type NOT NULL,
    status bitbucket.sync_status NOT NULL,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    items_synced INTEGER DEFAULT 0,
    error_message TEXT,
    details JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_bb_sync_logs_type_status ON bitbucket.bb_sync_logs(sync_type, status);
CREATE INDEX idx_bb_sync_logs_started ON bitbucket.bb_sync_logs(started_at DESC);
```

---

## 2. API Gateway Endpoints

### 2.1 gateway.api_endpoints 테이블 스키마

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary Key (auto-generated) |
| name | VARCHAR | 엔드포인트 이름 |
| path_pattern | VARCHAR | URL 패턴 |
| method | VARCHAR | HTTP 메소드 (GET, POST) |
| endpoint_type | VARCHAR | 'query' 또는 'proxy' |
| upstream_service | VARCHAR | 프록시 대상 서비스 (nullable) |
| upstream_path | VARCHAR | 프록시 대상 경로 (nullable) |
| query | TEXT | 실행할 SQL 쿼리 (query 타입일 때) |
| description | VARCHAR | 설명 |
| auth_required | BOOLEAN | 인증 필요 여부 |
| required_scopes | JSONB | 필요한 스코프 배열 |
| allow_pending_access | BOOLEAN | 대기 중 접근 허용 |
| status | VARCHAR | 'active', 'deprecated', 'disabled' |
| version | VARCHAR | API 버전 (예: 'v1') |
| owner_id | VARCHAR | 소유자 ID (nullable) |
| default_rate_limit | INTEGER | 기본 요청 제한 (nullable) |
| default_daily_quota | INTEGER | 일일 할당량 (nullable) |
| created_at | TIMESTAMP | 생성일 |
| updated_at | TIMESTAMP | 수정일 |

### 2.2 등록된 Bitbucket 엔드포인트 (12개)

#### GET 엔드포인트 (7개) - 조회용

| Name | Path | Description |
|------|------|-------------|
| bitbucket-repositories | /api/bitbucket/repositories | 저장소 목록 조회 (최근 커밋순, 100개) |
| bitbucket-repository-detail | /api/bitbucket/repositories/:uuid | 특정 저장소 상세 조회 |
| bitbucket-repository-dependencies | /api/bitbucket/repositories/:uuid/dependencies | 특정 저장소 의존성 목록 |
| bitbucket-outdated-packages | /api/bitbucket/outdated-packages | 업데이트 필요한 패키지 목록 (major/minor) |
| bitbucket-repository-branches | /api/bitbucket/repositories/:uuid/branches | 특정 저장소 브랜치 목록 |
| bitbucket-sync-logs | /api/bitbucket/sync-logs | 동기화 로그 (최근 50개) |
| bitbucket-stats-summary | /api/bitbucket/stats/summary | 통계 요약 |

#### POST 엔드포인트 (5개) - 저장용

| Name | Path | Description | Scope |
|------|------|-------------|-------|
| bitbucket-sync-workspace | /api/bitbucket/workspaces | 워크스페이스 UPSERT | bitbucket:write |
| bitbucket-sync-repository | /api/bitbucket/repositories | 저장소 UPSERT | bitbucket:write |
| bitbucket-sync-branch | /api/bitbucket/branches | 브랜치 UPSERT | bitbucket:write |
| bitbucket-sync-dependency | /api/bitbucket/dependencies | 의존성 UPSERT | bitbucket:write |
| bitbucket-sync-log | /api/bitbucket/sync-logs | 동기화 로그 INSERT | bitbucket:write |

### 2.3 POST 요청 파라미터 (Request Body)

#### POST /api/bitbucket/workspaces
```json
{
  "uuid": "string",
  "slug": "string",
  "name": "string",
  "is_active": "boolean"
}
```

#### POST /api/bitbucket/repositories
```json
{
  "uuid": "string",
  "workspace_uuid": "string",
  "slug": "string",
  "name": "string",
  "full_name": "string",
  "project_type": "backend|frontend|fullstack|library|infrastructure|documentation|other",
  "description": "string|null",
  "language": "string|null",
  "main_branch": "string",
  "is_private": "boolean",
  "status": "active|archived|deprecated",
  "has_issues": "boolean",
  "has_wiki": "boolean",
  "last_commit_at": "timestamp|null",
  "last_commit_hash": "string|null",
  "clone_url_https": "string|null",
  "clone_url_ssh": "string|null"
}
```

#### POST /api/bitbucket/branches
```json
{
  "repository_uuid": "string",
  "name": "string",
  "commit_hash": "string|null",
  "commit_message": "string|null",
  "commit_author": "string|null",
  "commit_date": "timestamp|null",
  "is_default": "boolean"
}
```

#### POST /api/bitbucket/dependencies
```json
{
  "repository_uuid": "string",
  "package_manager": "npm|yarn|pnpm|pip|poetry|maven|gradle|composer|cargo|go|nuget|other",
  "package_name": "string",
  "current_version": "string|null",
  "latest_version": "string|null",
  "update_type": "major|minor|patch|prerelease|unknown|null",
  "is_dev_dependency": "boolean",
  "file_path": "string|null"
}
```

#### POST /api/bitbucket/sync-logs
```json
{
  "sync_type": "full|incremental|repository|branch|dependency",
  "status": "running|completed|failed|cancelled",
  "started_at": "timestamp",
  "completed_at": "timestamp|null",
  "items_synced": "integer",
  "error_message": "string|null",
  "details": "object|null"
}
```

---

## 3. 아키텍처

```
┌─────────────────────────┐
│   Bitbucket Cloud API   │
└───────────┬─────────────┘
            │
┌───────────▼─────────────┐
│  jabis-bitbucket-sync   │
│   (Node.js/TypeScript)  │
│                         │
│  - Bitbucket OAuth 인증  │
│  - 데이터 수집/가공       │
│  - node-cron 스케줄러    │
└───────────┬─────────────┘
            │ HTTP (fetch)
┌───────────▼─────────────┐
│   jabis-api-gateway     │
│                         │
│  GET  /api/bitbucket/*  │  ← 조회 (SELECT)
│  POST /api/bitbucket/*  │  ← 저장 (INSERT/UPSERT)
└───────────┬─────────────┘
            │
┌───────────▼─────────────┐
│      PostgreSQL         │
│    (bitbucket schema)   │
└─────────────────────────┘
```

**핵심 원칙**: jabis-bitbucket-sync는 DB에 직접 연결하지 않고, jabis-api-gateway를 통해서만 데이터를 읽고 쓴다.

---

## 4. 환경변수 (jabis-bitbucket-sync)

```env
# Bitbucket OAuth
BITBUCKET_CLIENT_ID=xxx
BITBUCKET_CLIENT_SECRET=xxx
BITBUCKET_WORKSPACE=jinhaksa

# API Gateway
API_GATEWAY_URL=https://jabis-api-gateway.jinhakapply.com
API_GATEWAY_TOKEN=xxx  # bitbucket:write 스코프 필요

# Cron Schedule (optional)
SYNC_CRON_SCHEDULE=0 2 * * *  # 매일 새벽 2시
```

---

## 5. 변경 이력

| 날짜 | 작업 내용 |
|------|----------|
| 2026-01-27 | bitbucket 스키마 및 테이블 생성 |
| 2026-01-27 | GET 엔드포인트 7개 등록 |
| 2026-01-27 | POST 엔드포인트 5개 등록 |

---

## 6. TODO (다음 작업)

### Phase 1: jabis-bitbucket-sync 코드 수정 ✅ 완료 (2026-01-28)
- [x] Prisma 제거 (DB 직접 연결 X)
- [x] API Gateway HTTP 클라이언트 구현
  - `src/lib/apiGateway.ts` - fetch 기반 클라이언트
  - GET/POST 메소드, 인증 토큰 처리
- [x] 기존 DB 레이어를 API Gateway 호출로 교체
- [x] vault.ts 제거 (DATABASE_URL 불필요)
- [x] 환경변수 정리 (API_GATEWAY_URL, API_GATEWAY_TOKEN만 필요)

### Phase 2: jabis-helm 설정 추가 ✅ 완료 (2026-01-28)
- [x] `jabis-bitbucket-sync` 앱 설정 추가 (`prod-applications-values.yaml`)
- [x] Vault 연동 설정 (vaultconfig)
- [x] 환경변수 설정 (BITBUCKET_WORKSPACE, API_GATEWAY_URL, 스케줄러 크론)

### Phase 3: Bitbucket Pipeline 설정 ✅ 완료 (2026-01-28)
- [x] `Dockerfile` 작성 (Node.js 20-alpine, multi-stage build)
- [x] `bitbucket-pipelines.yml` 작성
- [x] Teams 배포 알림 설정
- [x] Vault 시크릿 로더 추가 (`src/utils/vault.ts`)

### Phase 4: 테스트 & 배포
- [ ] Vault 시크릿 등록 (아래 참조)
- [ ] Bitbucket 리포지토리 생성 & push
- [ ] 첫 번째 파이프라인 실행
- [ ] ArgoCD 동기화 확인
- [ ] 동기화 스케줄러 동작 확인

---

## 7. Vault 시크릿 등록 (필수)

jabis-bitbucket-sync가 정상 동작하려면 아래 시크릿을 Vault에 등록해야 합니다.

**경로**: `secret/jabis/jabis-bitbucket-sync`

| Key | Description | 예시 값 |
|-----|-------------|---------|
| BITBUCKET_CLIENT_SECRET | Bitbucket OAuth Consumer Secret | `xxx` |
| API_GATEWAY_TOKEN | API Gateway 서비스 토큰 (bitbucket:write 스코프) | `jabis-gateway-xxx` |

**Vault CLI 등록 예시**:
```bash
vault kv put secret/jabis/jabis-bitbucket-sync \
  BITBUCKET_CLIENT_SECRET="your-bitbucket-secret" \
  API_GATEWAY_TOKEN="your-api-gateway-token"
```

**참고**: BITBUCKET_CLIENT_ID는 민감하지 않으므로 환경변수로 설정합니다.
