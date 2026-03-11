# Alpha(스테이징) 환경 구축 계획

> 작성일: 2026-03-10
> 최종 수정: 2026-03-11
> 상태: **인프라 설정 진행 중** (TLS 인증서 대기)

## 1. 배경

JABIS 프로젝트에 여러 개발자가 참여하게 되면서, main push = 운영 배포 방식은 위험도가 높아짐.
운영 배포 전 테스트할 수 있는 Alpha(스테이징) 환경이 필요.

## 2. 핵심 정책 결정

| # | 항목 | 결정 |
|---|------|------|
| 1 | 전체 구성 | **옵션 B: FE + GW + DB 완전 격리** |
| 2 | 브랜칭 | `alpha` 브랜치 push → alpha 배포, `alpha` → `main` merge → 운영 배포 |
| 3 | 도메인 | `*-alpha.jinhakapply.com` |
| 4 | DB | 같은 PostgreSQL 서버에 `jabis_alpha` DB 추가 |
| 5 | jabis-cert | **운영 공유** (alpha 전용 client_id 별도 발급, cert 프론트에서 등록) |
| 6 | MinIO | **운영 공유** (같은 버킷, `alpha/` prefix로 경로 분리) |
| 7 | Redis | **운영 공유** (같은 인스턴스, DB 번호 분리 — 운영 DB 0, alpha DB 1) |
| 8 | jabis-storage | **alpha 별도 배포** (`STORAGE_PREFIX=alpha/` 환경변수로 경로 분리) |
| 9 | 접근 제어 | 별도 제한 없음 (운영 DB 권한 기반 그대로) |
| 10 | 이메일/알림 | 메일 모듈 미구현 상태, 해당 없음 |

## 3. Alpha 환경 아키텍처

```
┌─ Alpha 환경 (jabis-alpha 네임스페이스) ─────────────────────┐
│                                                              │
│  jabis-alpha.jinhakapply.com                                 │
│  ├─ /         → jabis (alpha)                                │
│  ├─ /hr       → jabis-hr (alpha)                             │
│  ├─ /dev      → jabis-dev (alpha)                            │
│  ├─ /producer → jabis-producer (alpha)                       │
│  ├─ /finance  → jabis-finance (alpha)                        │
│  ├─ /superadmin → jabis-sysadmin (alpha)                     │
│  └─ /teamlead → jabis-teamlead (alpha)                      │
│                                                              │
│  jabis-gateway-alpha.jinhakapply.com                         │
│  └─ /         → jabis-api-gateway (alpha) → Alpha DB        │
│                                                              │
│  jabis-storage (alpha)                                       │
│  └─ STORAGE_PREFIX=alpha/ → 운영 MinIO (같은 버킷)           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  공유 인프라 (운영과 동일)                                      │
│  - jabis-cert (prod) ← OAuth 인증 (alpha client_id 별도)     │
│  - Redis (prod) ← DB 0: 운영, DB 1: alpha                   │
│  - MinIO (prod) ← 같은 버킷, alpha/ prefix로 분리            │
│  - LDAP (prod) ← 사내 인증                                    │
│  - Vault (prod) ← 시크릿                                     │
└──────────────────────────────────────────────────────────────┘
```

### 요청 흐름

```
alpha FE → 운영 jabis-cert (로그인, alpha client_id 사용)
         → alpha gateway (REDIS_URL=redis://host:6379/1)
         → alpha DB (jabis_alpha)
         → alpha storage (STORAGE_PREFIX=alpha/) → 운영 MinIO
```

## 4. 서비스별 Alpha 배포 여부

| 서비스 | Alpha 별도 배포 | 비고 |
|--------|:-:|------|
| jabis (메인 FE) | O | alpha 빌드 |
| jabis-hr, dev, producer, finance, sysadmin, teamlead | O | alpha 빌드 |
| jabis-api-gateway | O | alpha DB 연결, Redis DB 1 |
| jabis-storage | O | STORAGE_PREFIX=alpha/ |
| jabis-cert | **X (공유)** | alpha client_id만 DB에 등록 |
| MinIO | **X (공유)** | alpha/ prefix로 경로 분리 |
| Redis | **X (공유)** | DB 번호 분리 (DB 0=운영, DB 1=alpha) |
| PostgreSQL | **X (공유)** | 같은 서버, jabis_alpha DB 추가 |

## 5. jabis-common submodule 동기화 플로우

jabis-common은 alpha 브랜치 없이 **master만 사용**. Pipeline이 없는 순수 소스 저장소이므로 브랜치 분리 불필요.

```
[1] shared-pages 수정 → jabis-common master에 push

[2] 소비 프로젝트 alpha 브랜치에서:
    git submodule update --remote packages
    → alpha 파이프라인 → alpha 환경 배포 → 테스트

[3] alpha 검증 통과 후:
    소비 프로젝트 alpha → master 머지
    → 운영 파이프라인 → 운영 배포
```

- submodule 참조 커밋이 잠금장치 역할: master 브랜치는 머지 전까지 이전 커밋 참조 유지
- alpha → master 머지 시 submodule 참조 커밋도 함께 머지됨

## 6. DB 관리 전략

### 초기 데이터 동기화
- 운영 DB → alpha DB로 전체 복제 (스키마 + 데이터)
- 필요 시 수동으로 재복제 가능

### 관리 서비스 (향후 구현)
| 기능 | 담당 서비스 | 설명 |
|------|------------|------|
| DB 복제 | jabis-emergency-console | 운영 → alpha 전체 복사 |
| DB 마이그레이션 | jabis-emergency-console | alpha DB에 스키마/테이블 변경 배포 |
| 소스 배포 | jabis-maker | alpha 환경에 코드 빌드/배포 |

## 7. 구현 단계 (배포 순서)

### Phase 0: 인프라 준비 (수동 작업)

| # | 작업 | 상태 | 상세 |
|---|------|:----:|------|
| 0-1 | K3S 네임스페이스 생성 | ✅ | `jabis-alpha` 네임스페이스 생성 완료 |
| 0-2 | Alpha PostgreSQL DB 생성 | ✅ | pgAdmin에서 `jabis_alpha` DB 생성, 운영 스키마/데이터 복제 완료 |
| 0-3 | DNS 등록 | ⏳ | `jabis-alpha.jinhakapply.com`, `jabis-gateway-alpha.jinhakapply.com` 요청 완료 |
| 0-4 | TLS 인증서 | ⏳ | alpha 도메인용 인증서 — 시스템팀 작업 대기 |
| 0-5 | Harbor 네임스페이스 | ✅ | `jabis-alpha` 프로젝트 생성 완료 |
| 0-6 | Vault ConfigMap 복사 | ✅ | `jabis-alpha` 네임스페이스에 vault-agent-config 복사 |
| 0-7 | NetworkPolicy 허용 | ✅ | postgresql-helm, redis-helm의 allowedNamespaces에 `jabis-alpha` 추가 |

### Phase 1: 기반 서비스

| # | 작업 | 상태 | 상세 |
|---|------|:----:|------|
| 1-1 | jabis-helm alpha values 생성 | ✅ | alpha-ingress-values.yaml, alpha-applications-values.yaml |
| 1-2 | jabis-cert alpha 클라이언트 등록 | ✅ | FE용 `jabis_49c11699f14214d3`, GW용 `jabis_61b2e91c1e4408a7` |
| 1-3 | jabis-storage Dockerfile.alpha + Pipeline | ✅ | STORAGE_PREFIX=alpha/, Dockerfile.alpha 패턴 적용 |

### Phase 2: 백엔드

| # | 작업 | 상태 | 상세 |
|---|------|:----:|------|
| 2-1 | jabis-api-gateway Dockerfile.alpha + Pipeline | ✅ | alpha DB, Redis DB 1, alpha cert client |

### Phase 3: 메인 프론트엔드

| # | 작업 | 상태 | 상세 |
|---|------|:----:|------|
| 3-1 | jabis alpha 브랜치 + Pipeline + .env.alpha | ✅ | alpha OAuth, gateway URL 설정 |

### Phase 4: dept 프로젝트

| # | 프로젝트 | 상태 | 상세 |
|---|---------|:----:|------|
| 4-1 | jabis-hr | ✅ | alpha 브랜치 push, .env.alpha OAuth 설정 |
| 4-2 | jabis-dev | ✅ | 동일 |
| 4-3 | jabis-producer | ✅ | 동일 |
| 4-4 | jabis-finance | ✅ | 동일 |
| 4-5 | jabis-sysadmin | ✅ | 동일 |
| 4-6 | jabis-teamlead | ✅ | 동일 |

### Phase 5: 문서화

- [x] staging.md 완성 (이 문서)
- [x] deployment.md 업데이트 (alpha 배포 절차, K8S 서비스 URL, 인프라 정보, 환경변수 정책 추가)
- [x] git-workflow.md 업데이트 (alpha 브랜치 전략, submodule 동기화 플로우 추가)
- [x] environments.md 업데이트 (Alpha 환경 섹션, 환경 비교 표 추가)

### 남은 작업

| # | 작업 | 담당 | 상태 |
|---|------|------|:----:|
| 1 | TLS 인증서 설정 | 시스템팀 | ⏳ |
| 2 | DNS 등록 완료 확인 | 시스템팀 | ⏳ |
| 3 | alpha 환경 E2E 테스트 | AI/사용자 | 대기 (TLS 후) |

## 8. Git 워크플로우

```
feature/* ──(PR)──→ alpha ──(push)──→ Alpha 배포 & 테스트
                       │
                       └──(merge)──→ main ──(push)──→ 운영 배포
```

- alpha 브랜치 push → Bitbucket Pipeline → alpha 환경 자동 배포
- alpha → main 머지 → Bitbucket Pipeline → 운영 환경 자동 배포
- jabis-common은 master만 사용 (submodule 참조로 버전 관리)

## 9. Dockerfile.alpha 패턴 (환경변수 관리)

Helm values의 `env` 키는 common-helm에서 무시되므로, **Dockerfile에서 환경변수를 관리**한다.
Alpha 환경은 운영과 다른 값이 필요하므로, `Dockerfile.alpha`를 별도 생성하여 Pipeline에서 분기한다.

### 구조

```
프로젝트/
├── Dockerfile         # 운영용 (master 브랜치)
├── Dockerfile.alpha   # Alpha용 (alpha 브랜치)
└── bitbucket-pipelines.yml
    └── alpha 브랜치: DOCKERFILE=Dockerfile.alpha
    └── master 브랜치: DOCKERFILE=Dockerfile (기본값)
```

### Dockerfile.alpha에서 운영 대비 변경되는 값

**jabis-api-gateway:**

| 환경변수 | 운영 | Alpha |
|---------|------|-------|
| DATABASE_NAME | jabis | jabis_alpha |
| REDIS_DB | 0 (기본) | 1 |
| CERT_CLIENT_ID | jabis_5ad89312e42883fa | jabis_61b2e91c1e4408a7 |
| CERT_CLIENT_SECRET | (운영 시크릿) | (alpha 시크릿) |
| STORAGE_SERVICE_URL | jabis-storage-prod-service.jabis-prod:3400 | jabis-storage-alpha-service.jabis-alpha:3400 |

> DATABASE_HOST(`postgresql-helm.jabis`), REDIS_HOST(`redis-helm.jabis`)는 **운영과 동일** — PostgreSQL/Redis가 `jabis` 네임스페이스에 배포되어 있음.

**jabis-storage:**

| 환경변수 | 운영 | Alpha |
|---------|------|-------|
| STORAGE_PREFIX | (빈 문자열) | alpha/ |

## 10. NetworkPolicy 설정

PostgreSQL과 Redis는 `jabis` 네임스페이스에 배포되며, NetworkPolicy로 접근을 제어한다.
Alpha 환경(`jabis-alpha` 네임스페이스)에서 접근하려면 allowedNamespaces에 추가가 필요하다.

**리포지토리:**
- `postgresql-helm` (bitbucket.org/jinhaksa/postgresql-helm)
- `redis-helm` (bitbucket.org/jinhaksa/redis-helm)

**values-prod.yaml 설정:**

```yaml
networkPolicy:
  enabled: true
  allowedNamespaces:
    - jabis
    - jabis-prod
    - jabis-alpha    # ← alpha 환경 접근 허용
    - monitoring
```

> **새 환경 추가 시** 반드시 postgresql-helm, redis-helm 양쪽 모두 allowedNamespaces에 추가해야 한다.

## 11. OAuth 클라이언트 등록

jabis-cert는 운영과 공유하며, alpha 전용 client_id를 별도 발급한다.

| 용도 | client_id | 등록 위치 |
|------|-----------|----------|
| Alpha FE (jabis, dept 전체) | `jabis_49c11699f14214d3` | 각 프로젝트 `.env.alpha` |
| Alpha Gateway | `jabis_61b2e91c1e4408a7` | jabis-api-gateway `Dockerfile.alpha` |

> **운영 client_id와 혼동 금지** — alpha와 운영은 별도 클라이언트를 사용한다.

## 12. Pipeline alpha 지원 현황

| 프로젝트 | alpha Pipeline | alpha 브랜치 | 비고 |
|---------|:---:|:---:|------|
| jabis-api-gateway | ✅ | ✅ | Dockerfile.alpha + set-alpha-variables |
| jabis-storage | ✅ | ✅ | Dockerfile.alpha + set-alpha-variables |
| jabis | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-hr | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-dev | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-producer | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-finance | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-sysadmin | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-teamlead | ✅ | ✅ | .env.alpha, alpha Pipeline |
| jabis-helm | ✅ | - | deploy-alpha custom pipeline |

## 13. 알려진 이슈

| 이슈 | 원인 | 해결 |
|------|------|------|
| `crypto.randomUUID is not a function` | alpha 도메인에 TLS 미설정 → Insecure Context | TLS 인증서 설정 후 해결 |
