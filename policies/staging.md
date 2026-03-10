# Alpha(스테이징) 환경 구축 계획

> 작성일: 2026-03-10
> 최종 수정: 2026-03-10
> 상태: **구현 진행 중** (Phase 5 문서화 진행)

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

| # | 작업 | 상세 |
|---|------|------|
| 0-1 | K3S 네임스페이스 생성 | `kubectl create namespace jabis-alpha` |
| 0-2 | Alpha PostgreSQL DB 생성 | 같은 서버에 `jabis_alpha` DB 생성, 운영 스키마/데이터 복제 |
| 0-3 | DNS 확인 | `*-alpha.jinhakapply.com` 와일드카드 |
| 0-4 | TLS 인증서 | alpha 도메인용 인증서 |
| 0-5 | Harbor 네임스페이스 확인 | `jabis-alpha` 프로젝트 |

### Phase 1: 기반 서비스

| # | 작업 | 담당 | 상세 |
|---|------|------|------|
| 1-1 | jabis-helm alpha values 생성 | AI | alpha-ingress-values.yaml, alpha-applications-values.yaml 등 |
| 1-2 | jabis-cert alpha 클라이언트 등록 | 사용자 | cert 프론트에서 alpha 전용 client_id 발급 |
| 1-3 | jabis-storage STORAGE_PREFIX 구현 | AI | 환경변수 기반 prefix 로직 추가 + alpha 배포 설정 |

### Phase 2: 백엔드

| # | 작업 | 담당 | 상세 |
|---|------|------|------|
| 2-1 | jabis-api-gateway alpha 설정 | AI | .env.alpha, alpha 브랜치, 파이프라인 (이미 일부 존재) |

### Phase 3: 메인 프론트엔드

| # | 작업 | 담당 | 상세 |
|---|------|------|------|
| 3-1 | jabis alpha 설정 | AI | .env.alpha, build:alpha, 파이프라인, alpha 브랜치 |

### Phase 4: dept 프로젝트 (병렬 가능)

| # | 프로젝트 | 담당 | 상세 |
|---|---------|------|------|
| 4-1 | jabis-hr | AI | Phase 3과 동일 패턴 |
| 4-2 | jabis-dev | AI | Phase 3과 동일 패턴 |
| 4-3 | jabis-producer | AI | Phase 3과 동일 패턴 |
| 4-4 | jabis-finance | AI | Phase 3과 동일 패턴 |
| 4-5 | jabis-sysadmin | AI | Phase 3과 동일 패턴 |
| 4-6 | jabis-teamlead | AI | Phase 3과 동일 패턴 |

### Phase 5: 문서화 (진행 중)

- [x] staging.md 완성 (이 문서)
- [x] deployment.md 업데이트 (alpha 배포 절차, K8S 서비스 URL, 인프라 정보 추가)
- [x] git-workflow.md 업데이트 (alpha 브랜치 전략, submodule 동기화 플로우 추가)
- [x] environments.md 업데이트 (Alpha 환경 섹션, 환경 비교 표 추가)

## 8. Git 워크플로우

```
feature/* ──(PR)──→ alpha ──(push)──→ Alpha 배포 & 테스트
                       │
                       └──(merge)──→ main ──(push)──→ 운영 배포
```

- alpha 브랜치 push → Bitbucket Pipeline → alpha 환경 자동 배포
- alpha → main 머지 → Bitbucket Pipeline → 운영 환경 자동 배포
- jabis-common은 master만 사용 (submodule 참조로 버전 관리)

## 9. 현재 Pipeline alpha 지원 현황

| 프로젝트 | alpha Pipeline | alpha 브랜치 | 비고 |
|---------|---------------|-------------|------|
| jabis-api-gateway | O (완성) | 미생성 | set-alpha-variables 구현됨 |
| jabis | X | 미생성 | prod만 구현 |
| jabis-hr | X | 미생성 | prod만 구현 |
| jabis-dev | X | 미생성 | prod만 구현 |
| jabis-producer | X | 미생성 | prod만 구현 |
| jabis-finance | X | 미생성 | prod만 구현 |
| jabis-sysadmin | X | 미생성 | prod만 구현 |
| jabis-teamlead | X | 미생성 | prod만 구현 |
| jabis-helm | O (수동만) | - | deploy-alpha custom pipeline |
| jabis-storage | X | 미생성 | STORAGE_PREFIX 미구현 |
