# JABIS Cert - Database Schema

> 이 문서는 jabis-cert 프로젝트의 DB 스키마를 기록합니다.
> 통합인증 서버(OAuth 2.0, LDAP, TOTP)의 데이터 구조입니다.

## 1. PostgreSQL Schema: `cert`

- Database: `jabis`
- Schema: `cert`
- User: `jabis_admin`

### 1.1 Tables

#### users (사용자)

```sql
CREATE TABLE cert.users (
    id VARCHAR(50) PRIMARY KEY,            -- 이메일 local part (예: hong@jinhak.com -> hong)
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255) NOT NULL,    -- 최초 가입 시 1회만 저장
    role_level INTEGER NOT NULL DEFAULT 1, -- 0:USER, 1:DEVELOPER, 2:APPROVER, 3:ADMIN
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- active/dormant/locked
    totp_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    totp_secret VARCHAR(255) NULL,
    backup_codes TEXT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ NULL,
    dormant_at TIMESTAMPTZ NULL,           -- 90일 미로그인 시 자동 휴면
    reactivated_by VARCHAR(50) NULL,
    reactivated_at TIMESTAMPTZ NULL
);
```

- **인덱스**: `idx_users_email`, `idx_users_role_level`, `idx_users_status`
- **휴면 정책**: 90일 미로그인 시 `status = 'dormant'`, 관리자 복원 필요

#### registered_clients (등록된 클라이언트 서비스)

```sql
CREATE TABLE cert.registered_clients (
    id VARCHAR(50) PRIMARY KEY,
    client_id VARCHAR(100) NOT NULL UNIQUE,      -- OAuth Client ID (jabis_xxxxxxxxxxxxxxxx)
    client_secret_hash VARCHAR(255) NOT NULL,    -- bcrypt 해시
    service_name VARCHAR(255) NOT NULL,
    description TEXT NULL,
    allowed_domain VARCHAR(255) NOT NULL,         -- 허용 도메인 (https:// 자동 적용)
    callback_path VARCHAR(255) NOT NULL,          -- 콜백 경로 (예: /auth/callback)
    allowed_scopes TEXT NOT NULL,                 -- JSON 배열 (["openid","profile","email"])
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending/approved/rejected/revoked
    owner_id VARCHAR(50) NOT NULL REFERENCES cert.users(id),
    approved_by_id VARCHAR(50) NULL REFERENCES cert.users(id),
    approved_at TIMESTAMPTZ NULL,
    rejection_reason TEXT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- **인덱스**: `idx_registered_clients_owner_id`, `idx_registered_clients_status`, `idx_registered_clients_client_id`
- **상태 흐름**: `pending` -> `approved` (관리자 승인) 또는 `rejected` / `revoked`
- **localhost**: 모든 포트에서 자동 허용 (개발 환경용)

#### refresh_tokens (리프레시 토큰)

```sql
CREATE TABLE cert.refresh_tokens (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL REFERENCES cert.users(id),
    token_hash VARCHAR(255) NOT NULL,
    client_id VARCHAR(100) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    revoked_at TIMESTAMPTZ NULL
);
```

- **인덱스**: `idx_refresh_tokens_user_id`, `idx_refresh_tokens_token_hash`

#### authorization_codes (OAuth Authorization Code)

```sql
CREATE TABLE cert.authorization_codes (
    code VARCHAR(100) PRIMARY KEY,
    client_id VARCHAR(100) NOT NULL,
    user_id VARCHAR(50) NOT NULL REFERENCES cert.users(id),
    redirect_uri VARCHAR(500) NOT NULL,
    scope TEXT NOT NULL,
    state VARCHAR(255) NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- **인덱스**: `idx_authorization_codes_client_id`, `idx_authorization_codes_expires_at`
- **수명**: 단기 (10분)

#### audit_logs (감사 로그)

```sql
CREATE TABLE cert.audit_logs (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    user_id VARCHAR(50) NULL,
    client_ip VARCHAR(50) NOT NULL,
    user_agent VARCHAR(500) NULL,
    details JSONB NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- **인덱스**: `idx_audit_logs_user_id`, `idx_audit_logs_event_type`, `idx_audit_logs_created_at`

### 1.2 Functions

```sql
-- 만료된 토큰 및 코드 정리
CREATE OR REPLACE FUNCTION cert.cleanup_expired_tokens()
RETURNS void AS $$
BEGIN
    DELETE FROM cert.refresh_tokens WHERE expires_at < NOW() OR revoked_at IS NOT NULL;
    DELETE FROM cert.authorization_codes WHERE expires_at < NOW() OR used_at IS NOT NULL;
END;
$$ LANGUAGE plpgsql;
```

## 2. 등록된 OAuth 클라이언트 목록

| ID | Client ID | 서비스명 | 도메인 | 콜백 경로 | 스코프 | 등록 SQL |
|----|-----------|---------|--------|----------|--------|---------|
| api-gateway-client-001 | jabis_5ad89312e42883fa | JABIS API Gateway | (내부) | (없음) | introspect | `register-api-gateway-client.sql` |
| jabis-client-001 | jabis_d93fb708be5b5eb7 | JABIS 메인 프론트엔드 | jabis.jinhakapply.com | /auth/callback | openid, profile, email | `register-jabis-client.sql` |
| jabis-producer-client-001 | jabis_producer_a7c3e91f4d2b8e06 | JABIS 프로듀서 | jabis.jinhakapply.com | /producer/auth/callback | openid, profile, email | `register-jabis-producer-client.sql` |

### Client ID 공유 정책

**같은 도메인(`jabis.jinhakapply.com`)의 dept 프로젝트는 jabis 메인 client_id를 공유한다.**

jabis-cert는 redirect_uri 검증 시 도메인만 확인하므로, 같은 도메인 내 서브 경로(`/hr/auth/callback`, `/finance/auth/callback` 등)는 별도 클라이언트 등록 없이 동일한 client_id로 인증이 가능하다.

| 프로젝트 | 사용하는 Client ID | 비고 |
|---------|-------------------|------|
| jabis (메인) | `jabis_d93fb708be5b5eb7` | 원본 등록 |
| jabis-hr | `jabis_d93fb708be5b5eb7` | 메인과 공유 |
| jabis-dev | `jabis_d93fb708be5b5eb7` | 메인과 공유 |
| jabis-finance | `jabis_d93fb708be5b5eb7` | 메인과 공유 |
| jabis-producer | `jabis_producer_a7c3e91f4d2b8e06` | 별도 등록 (레거시) |

> **새 dept 프로젝트 생성 시** `.env.production`의 `VITE_OAUTH_CLIENT_ID`에 `jabis_d93fb708be5b5eb7`를 사용한다. 별도 클라이언트 등록은 불필요.

## 3. SQL 파일 위치

| 파일 | 용도 |
|------|------|
| `sql/init.sql` | 스키마 초기화 (테이블, 인덱스, 함수 생성) |
| `sql/drop.sql` | 스키마 삭제 |
| `sql/register-api-gateway-client.sql` | API Gateway 클라이언트 등록 |
| `sql/register-jabis-client.sql` | jabis 메인 프론트엔드 클라이언트 등록 |
| `sql/register-jabis-producer-client.sql` | jabis-producer 프로듀서 앱 클라이언트 등록 |
