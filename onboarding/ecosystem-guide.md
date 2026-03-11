# JABIS 에코시스템 전체 가이드

> JABIS(Jinhak AI Based Integration System) 에코시스템을 빠르게 파악하기 위한 종합 가이드입니다.
> 새로운 AI 세션이나 신규 참여자가 이 문서를 먼저 읽으면 전체 그림을 잡을 수 있습니다.

---

## 1. 시스템 개요

JABIS는 진학어플라이(교육기업)의 사내 워크플로우 관리 시스템이다.
22개 프로젝트가 마이크로서비스 아키텍처로 구성되어 있고, K3S(경량 Kubernetes) 위에서 운영된다.

### 핵심 아키텍처

```
[사용자 브라우저]
    ↓ HTTPS (*.jinhakapply.com)
[K3S Ingress - Nginx]
    ├── /            → jabis (메인 앱, 17개 역할 기반 대시보드)
    ├── /hr/         → jabis-hr (인사담당자)
    ├── /dev/        → jabis-dev (개발자)
    ├── /producer/   → jabis-producer (프로듀서)
    ├── /finance/    → jabis-finance (재무)
    ├── /sysadmin/   → jabis-sysadmin (시스템관리자)
    ├── /teamlead/   → jabis-teamlead (팀리더)
    ├── /cert/       → jabis-cert (인증서버, OAuth 2.0 + LDAP)
    ├── /gateway/    → jabis-api-gateway (API 게이트웨이)
    └── /storage/    → jabis-storage (파일 스토리지)

[내부 서비스]
    ├── jabis-night-builder  — AI 자동화 (기능요청 → 코드생성 → PR)
    ├── jabis-bitbucket-sync — Git 저장소 동기화
    ├── jabis-emergency-console — DB 긴급 관리
    └── jabis-maker — Claude Code 원격 실행 플랫폼 (로컬 PM2)
```

---

## 2. 프로젝트 분류

### 2-1. 프론트엔드 (React + Vite + Tailwind)

| 프로젝트 | 역할 | 구조 |
|---------|------|------|
| **jabis** | 메인 앱 (17개 역할 대시보드) | 모노레포 `apps/prototype/` |
| **jabis-hr** | 인사담당자 전용 | 모노레포 `apps/hr/` |
| **jabis-dev** | 개발자 전용 | 모노레포 `apps/dev/` |
| **jabis-producer** | 프로듀서 전용 (Maker iframe, API 로그) | 모노레포 `apps/producer/` |
| **jabis-finance** | 재무담당자 전용 (법인카드 관리) | 모노레포 `apps/finance/` |
| **jabis-sysadmin** | 시스템 관리자 전용 (역할/권한) | 모노레포 `apps/sysadmin/` |
| **jabis-teamlead** | 팀리더 전용 | 모노레포 `apps/teamlead/` |
| **jabis-design-system** | 디자인 시스템 쇼케이스 | 단독 |
| **jabis-maker-admin** | jabis-maker 관리 UI | 단독 |

> **dept 프로젝트**(jabis-hr, jabis-dev 등)는 모두 동일한 모노레포 구조:
> `apps/{role}/` + `packages/` (jabis-common을 git submodule로 참조)

### 2-2. 백엔드

| 프로젝트 | 역할 | 스택 |
|---------|------|------|
| **jabis-api-gateway** | REST API 집약, SQL 실행 | Fastify, PostgreSQL, Redis |
| **jabis-cert** | OAuth 2.0 + PKCE, LDAP, JWT | Next.js 14, MSAL, jose |
| **jabis-storage** | MinIO 파일 스토리지 | Fastify, MinIO |
| **jabis-night-builder** | AI 자동화 서비스 | Node.js, Winston, PM2 |
| **jabis-bitbucket-sync** | Git 저장소 동기화 | Prisma, node-cron |
| **jabis-emergency-console** | DB 긴급 관리 콘솔 | Express, Monaco Editor |
| **jabis-maker** | Claude Code 원격 실행 + NightExecutor | Fastify, WebSocket, SQLite |

### 2-3. 공유/인프라

| 프로젝트 | 역할 |
|---------|------|
| **jabis-common** | 공유 라이브러리 원본 (@jabis/ui, layout, auth, core, menu, shared-pages) |
| **jabis-design-package** | @jabis/ui NPM 배포용 |
| **jabis-helm** | K3S Helm Chart 중앙 관리 |
| **jabis-docs** | 전체 문서 (정책, API 스펙, DB 스키마, 아키텍처) |

> 프로젝트별 상세: [프로젝트 맵](project-map.md), 각 프로젝트 문서는 `../projects/` 참조

---

## 3. 핵심 흐름

### 3-1. 인증 흐름

```
사용자 → jabis-cert OAuth 2.0 + PKCE (Microsoft Azure AD)
    → JWT 발급 (roles: organization.user_roles에서 조회)
    → 프론트엔드 JWT 쿠키 저장 → API 요청 시 자동 첨부
    → jabis-api-gateway → jabis-cert JWT introspection (circuit breaker)
```

- 모든 dept 프로젝트는 jabis 메인과 **동일한 CLIENT_ID + CLIENT_SECRET** 사용
- `VITE_OAUTH_REDIRECT_URI`만 프로젝트별 경로 (예: `/hr/auth/callback`)
- 상세: [jabis-cert 문서](../projects/jabis-cert.md), [보안 정책](../policies/security.md)

### 3-2. API 패턴

- **GET과 POST만 사용** (PUT/DELETE 없음)
- POST body에 `action` 필드로 mutation 구분: `create` / `update` / `delete`
- 응답 형식: `{ success: true/false, data/error }`
- API Gateway가 SQL 쿼리를 직접 실행 (동적 엔드포인트)
- 상세: [바이브 코딩 가이드 3절](vibe-coding-guide.md), [API 스펙](../api-specs/)

### 3-3. 공유 코드 흐름 (jabis-common → 소비 프로젝트)

```
1. jabis-common에서 수정 → 커밋 → push
2. 소비 프로젝트에서 git submodule update --remote packages
3. git add packages → 커밋 → push
```

**이 플로우를 안 거치면 소비 프로젝트는 이전 코드를 참조함** (submodule이 이전 커밋 가리킴)

소비 프로젝트 목록 (jabis-lab, jabis-template 제외):
- jabis, jabis-hr, jabis-dev, jabis-producer, jabis-design-system, jabis-finance

상세: [jabis-common 문서](../projects/jabis-common.md)

### 3-4. 배포 흐름

```
코드 push → Bitbucket Pipeline → Docker 빌드 → Harbor push
    → Helm 업데이트 → ArgoCD → K3S 배포
```

- **push = 배포** (자동, 수동 트리거 불필요)
- **DB 마이그레이션**: 쿼리 실행 먼저 → 코드 push (순서 중요)
- 여러 프로젝트 동시 push 가능 (retry 메커니즘이 Helm 충돌 자동 해결)
- 상세: [배포 정책](../policies/deployment.md), [Git 워크플로우](../policies/git-workflow.md)

---

## 4. DB 구조

PostgreSQL, **스키마로 프로젝트 분리**:

| 스키마 | 용도 | 핵심 테이블 |
|--------|------|------------|
| **organization** | 조직/인사 | departments(47+), employees(63+), users, user_roles, permissions |
| **gateway** | API 관리 | api_endpoints, api_access_logs, api_tasks |
| **cert** | 인증 | 세션, 토큰 |
| **attendance** | 출결 | GPS 기반 출퇴근 |
| **approval** | 결재 | 문서, 결재선 |
| **finance** | 법인카드 | 카드요청, 사용내역 |
| **bitbucket** | Git 동기화 | 저장소, 커밋 |

> **중요**: 전체 직원 목록은 `organization.employees` 기준 (`users`는 로그인한 사람만)

상세: [DB 스키마 문서](../db-schemas/)

---

## 5. 역할 체계 (17개)

```
portal, developer, hr, producer, finance, sysadmin, teamlead,
secretary, officer, auditor, planner, researcher, instructor,
manager, designer, marketer, support
```

- 각 역할마다 대시보드가 다름 (jabis 메인에서 전환)
- dept 프로젝트(jabis-hr 등)는 특정 역할 전용 앱
- 역할 목록/URL은 `@jabis/menu`에서 중앙 관리 (`availableRoles`, `getRoleUrl`)

---

## 6. 핵심 공유 패키지 (@jabis/*)

jabis-common이 원본이며, 소비 프로젝트에서 git submodule로 참조:

| 패키지 | 역할 |
|--------|------|
| **@jabis/ui** | UI 컴포넌트 (Radix UI + Tailwind), `useToast` 등 |
| **@jabis/layout** | AppLayout, 사이드바, 헤더 |
| **@jabis/auth** | OAuth 인증 로직 |
| **@jabis/core** | 공통 유틸리티 |
| **@jabis/menu** | 공통 메뉴 구성 (buildMenu, availableRoles, getRoleUrl) |
| **shared-pages** | 공통 페이지 컴포넌트 (법인카드 등) |

> **주의**: `useToast`는 `showSuccess/showError/showInfo/showWarning` 사용 (`toast` 프로퍼티 없음)

---

## 7. 개발 워크플로우 요약

```
1. jabis-docs에서 관련 정책/스펙 읽기
2. git pull (최신 코드 동기화)
3. 코드 작성 (정책 준수)
4. pnpm build:preview (미리보기 빌드 — pnpm build 아님!)
5. ai-e2e-tester로 E2E 테스트
6. 커밋 & push (Conventional Commits, Co-Authored-By 추가)
7. 운영 URL에서 E2E 테스트로 배포 확인
```

상세: [개발 워크플로우](../policies/work-flow.md), [E2E 테스트 정책](../policies/e2e-testing.md)

---

## 8. 주요 정책 요약

| 정책 | 핵심 내용 | 문서 |
|------|----------|------|
| 프로젝트 구조 | jabis-common은 루트에만 패키지, `packages/` 금지 | [project-structure.md](../policies/project-structure.md) |
| 공유 코드 | shared-pages 수정은 jabis-common에서만 | [jabis-common.md](../projects/jabis-common.md) |
| 메뉴 | dept 프로젝트는 @jabis/menu buildMenu 사용 (하드코딩 금지) | [jabis-common.md](../projects/jabis-common.md) |
| DB 마이그레이션 | 쿼리는 채팅에 출력, 실행 → push 순서 | [deployment.md](../policies/deployment.md) |
| K3S URL | `http://{서비스명}-prod-service.jabis-prod:{포트}` | [deployment.md](../policies/deployment.md) |
| 코딩 스타일 | GET/POST만, Zod 검증, Winston 로깅, @jabis/ui 필수 | [coding-style.md](../policies/coding-style.md) |
| 보안 | 파라미터화 쿼리, Vault 시크릿, JWT 검증 | [security.md](../policies/security.md) |
| 미리보기 | `build:preview` 사용 (운영 빌드하면 흰 화면) | [deployment.md](../policies/deployment.md) |

---

## 9. 자주 하는 실수 & 주의사항

| 실수 | 결과 | 올바른 방법 |
|------|------|------------|
| `pnpm build`로 미리보기 | 흰 화면 | `pnpm build:preview` |
| `organization.users`로 직원 목록 조회 | 미로그인 직원 누락 | `employees` 테이블 사용 |
| jabis-common 수정 후 submodule 미업데이트 | 소비 프로젝트가 구버전 참조 | 전체 소비 프로젝트 submodule update |
| 모노레포에서 dist 심볼릭 링크 누락 | "No build output found" | `ln -s apps/{앱}/dist dist` |
| ROLE_URL_MAP에 새 역할 미추가 | 역할 전환 시 jabis 메인으로 돌아감 | `jabis-common/menu/src/roles.js` 수정 |
| DB 마이그레이션 후 코드 push (순서 바꿈) | 새 스키마 참조 쿼리 실패 | 쿼리 실행 먼저 → push |
| `const { toast } = useToast()` | undefined 런타임 에러 | `showSuccess/showError` 사용 |
| K3S URL 네임스페이스 `jabis` | 서비스 못 찾음 | `jabis-prod` (namespace-stage 규칙) |

---

## 10. 문서 파악 추천 순서

처음 접하는 경우 아래 순서로 읽으면 전체 그림이 빠르게 잡힌다:

1. **이 문서** — 전체 에코시스템 개요 (현재)
2. [architecture/system-overview.md](../architecture/system-overview.md) — 상세 아키텍처
3. [onboarding/project-map.md](project-map.md) — 프로젝트 목록 & 관계
4. [policies/project-structure.md](../policies/project-structure.md) — 프로젝트 구조 정책
5. [policies/coding-style.md](../policies/coding-style.md) — 코딩 컨벤션
6. [onboarding/vibe-coding-guide.md](vibe-coding-guide.md) — 바이브 코딩 대전제
7. [policies/deployment.md](../policies/deployment.md) — 배포 정책
8. [projects/jabis-common.md](../projects/jabis-common.md) — 공유 라이브러리 (핵심)
9. [projects/jabis-api-gateway.md](../projects/jabis-api-gateway.md) — API Gateway
10. [projects/jabis-cert.md](../projects/jabis-cert.md) — 인증

---

## 참조

- [프로젝트 맵](project-map.md) — 프로젝트 전체 목록 & Bitbucket URL
- [바이브 코딩 가이드](vibe-coding-guide.md) — UI/API/상태관리 코딩 규칙
- [AI 표준 세팅 가이드](ai-standard-setup.md) — 프로젝트별 AI 표준 적용
- [OAuth 가이드](oauth-guide.md) — OAuth 설정 상세

---

> 문서 버전: 1.0.0
> 최종 수정: 2026-03-06
