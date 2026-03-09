# JABIS 온톨로지 (Ontology)

> JABIS 시스템의 핵심 개념, 분류 체계, 관계를 정의합니다.
> 새로운 개발자 온보딩 및 시스템 리뷰 시 참고 자료입니다.

---

## 1. JABIS란?

**JABIS** (Jinhak Business Integration System) — 진학어플라이의 엔터프라이즈급 통합 업무 시스템.

- 20+ 개의 상호 연결된 프로젝트로 구성
- 마이크로서비스 아키텍처 (K3S 기반)
- 17개 역할 기반 접근 제어
- AI 자동화 내장 (Night Builder, Auto Debugger)

---

## 2. 최상위 개념 분류

```
JABIS
├── 시스템 (System)           — 어떤 프로젝트들이 있는가
├── 사용자 (Actor)            — 누가 쓰는가
├── 데이터 (Data)             — 무엇을 저장하는가
├── 프로세스 (Process)        — 어떻게 동작하는가
└── 인프라 (Infrastructure)   — 어디서 돌아가는가
```

---

## 3. 시스템 분류 체계 (System Taxonomy)

### 3.1 프로젝트 유형

JABIS 프로젝트는 5가지 유형으로 분류됩니다.

| 유형 | 정의 | 프로젝트 |
|------|------|----------|
| **포탈 (Portal)** | 전 직원이 사용하는 메인 앱. 17개 역할별 대시보드 제공 | jabis |
| **부서앱 (Dept App)** | 특정 부서 전용 앱. 모노레포 구조, jabis-common을 submodule로 소비 | jabis-hr, jabis-finance, jabis-dev, jabis-producer, jabis-sysadmin |
| **코어 서비스 (Core Service)** | 인증, API, 스토리지 등 핵심 백엔드 서비스 | jabis-api-gateway, jabis-cert, jabis-storage |
| **자동화 서비스 (Automation)** | AI 기반 또는 주기적 자동화 서비스 | jabis-night-builder, jabis-bitbucket-sync, jabis-maker |
| **공유 라이브러리 (Shared Library)** | 프론트엔드 공용 컴포넌트/로직 | jabis-common |

### 3.2 프로젝트 분류도

```
JABIS 프로젝트
│
├── 프론트엔드 (Frontend) ─── React + Vite + pnpm
│   ├── 포탈
│   │   └── jabis ─── 메인 앱 (17개 역할 대시보드)
│   ├── 부서앱
│   │   ├── jabis-hr ─── 인사 (채용, 인사, 급여, 교육)
│   │   ├── jabis-finance ─── 재무 (법인카드, 정산, 예산)
│   │   ├── jabis-dev ─── 개발 (작업 관리, AI 태스크)
│   │   ├── jabis-producer ─── 프로듀서 (API 관리, Maker)
│   │   └── jabis-sysadmin ─── 시스템 관리 (역할/권한)
│   └── 도구앱
│       ├── jabis-design-system ─── 디자인 시스템 쇼케이스
│       ├── jabis-maker-admin ─── Maker 관리 UI
│       ├── jabis-lab ─── 실험 환경
│       └── jabis-template ─── 신규 프로젝트 템플릿
│
├── 백엔드 (Backend) ─── Node.js + TypeScript + npm
│   ├── 코어 서비스
│   │   ├── jabis-api-gateway ─── SQL 쿼리 실행 게이트웨이
│   │   ├── jabis-cert ─── 통합 인증 서버 (OAuth 2.0 + LDAP)
│   │   └── jabis-storage ─── 파일 스토리지 (MinIO 프록시)
│   ├── 자동화 서비스
│   │   ├── jabis-night-builder ─── AI 자동 구현 서비스
│   │   ├── jabis-bitbucket-sync ─── 저장소/의존성 동기화
│   │   ├── jabis-maker ─── 빌드/미리보기 서버 + NightExecutor
│   │   └── jabis-emergency-console ─── DB 긴급 관리
│   └── 공유 라이브러리
│       ├── jabis-common ─── @jabis/* 패키지 원본
│       └── jabis-design-package ─── @jabis/ui NPM 배포용
│
└── 배포 (Deployment)
    └── jabis-helm ─── K3S Helm Chart 관리
```

### 3.3 핵심 관계 (Relationships)

| 주체 | 관계 | 대상 | 설명 |
|------|------|------|------|
| 부서앱 | **소비한다 (consumes)** | jabis-common | git submodule로 @jabis/* 패키지를 가져옴 |
| 모든 앱 | **인증받는다 (authenticates via)** | jabis-cert | OAuth 2.0 + PKCE로 JWT 발급 |
| 모든 앱 | **쿼리한다 (queries via)** | jabis-api-gateway | GET/POST로 SQL 실행 |
| jabis-api-gateway | **접근한다 (accesses)** | PostgreSQL | 파라미터화된 SQL 쿼리 실행 |
| jabis-night-builder | **폴링한다 (polls)** | jabis-api-gateway | 기능 요청을 주기적으로 가져옴 |
| jabis-maker | **서빙한다 (serves)** | dist/ 빌드 결과 | 미리보기 환경 제공 |
| jabis-helm | **배포한다 (deploys)** | K3S 클러스터 | 모든 서비스를 Helm으로 배포 |
| jabis-bitbucket-sync | **동기화한다 (syncs)** | Bitbucket 저장소 | 리포/브랜치/의존성 정보 수집 |

---

## 4. 사용자/역할 체계 (Actor Taxonomy)

### 4.1 역할 분류 (17개)

JABIS는 **역할 기반 접근 제어 (RBAC)**를 사용합니다. 사용자는 하나 이상의 역할을 가질 수 있습니다.

```
역할 (Role)
├── 관리 계층 (Administrative)
│   ├── superadmin ─── 시스템 전체 관리자 (모든 권한)
│   ├── admin ─── 글로벌 관리자
│   ├── aiadmin ─── AI 시스템 관리자
│   └── sysadmin ─── 시스템 설정 관리자
│
├── 경영 계층 (Executive)
│   ├── ceo ─── 대표
│   ├── executive ─── 본부장/임원
│   ├── depthead ─── 부장
│   └── teamlead ─── 팀장
│
├── 직무 계층 (Functional)
│   ├── developer ─── 개발자
│   ├── operation ─── 운영자
│   ├── sales ─── 영업
│   ├── design ─── 디자인
│   ├── planner ─── 기획/PM
│   ├── marketer ─── 마케팅
│   ├── finance ─── 재무
│   ├── hr ─── 인사
│   └── producer ─── 프로듀서 (API/서비스 관리)
│
└── 기본 (Default)
    └── portal ─── 포탈 접근 (전 직원 기본 역할)
```

### 4.2 역할 → 앱 매핑

역할에 따라 전용 앱으로 라우팅됩니다. 전용 앱이 없는 역할은 jabis 포탈 내에서 대시보드를 제공합니다.

| 역할 | URL 경로 | 앱 |
|------|----------|------|
| developer | `/dev/` | jabis-dev |
| hr | `/hr/` | jabis-hr |
| finance | `/finance/` | jabis-finance |
| producer | `/producer/` | jabis-producer |
| sysadmin | `/sysadmin/` | jabis-sysadmin |
| 나머지 12개 | `/{roleId}` | jabis (포탈 내 대시보드) |

> 역할 목록과 URL 매핑은 `@jabis/menu`의 `availableRoles`, `getRoleUrl`에서 중앙 관리됩니다.

---

## 5. 데이터 계층 (Data Taxonomy)

### 5.1 데이터베이스 스키마 구조

JABIS는 단일 PostgreSQL에 **스키마 분리** 전략을 사용합니다.

```
PostgreSQL
├── organization ─── 조직 구조 (★ 핵심)
│   ├── departments ─── 부서
│   ├── employees ─── 직원 (사용자 목록의 기준 테이블)
│   ├── users ─── 로그인한 사용자 (employees의 부분집합)
│   ├── user_roles ─── 역할 할당
│   └── permissions ─── 권한
│
├── gateway ─── API 관리
│   ├── api_endpoints ─── 등록된 API 엔드포인트 정의
│   ├── api_access_logs ─── API 접근 로그
│   └── api_tasks ─── AI 작업 요청/결과
│
├── approval ─── 전자결재
│   ├── documents ─── 결재 문서
│   ├── templates ─── 결재 양식
│   ├── approvers ─── 결재선 (결재자 목록)
│   └── timeline ─── 결재 이력
│
├── finance ─── 재무
│   ├── corporate_cards ─── 법인카드
│   ├── card_checkout_requests ─── 카드 불출 요청
│   ├── card_return_requests ─── 카드 반납 요청
│   ├── settlements ─── 정산
│   └── budgets ─── 예산
│
├── attendance ─── 근태
│   ├── attendance_records ─── 출퇴근 기록
│   └── schedules ─── 근무 일정
│
├── bitbucket_sync ─── 저장소 동기화
│   ├── repositories ─── 저장소 목록
│   └── dependencies ─── 의존성 관계
│
└── night_builder ─── AI 자동화
    ├── feature_requests ─── 기능 요청
    ├── test_results ─── 테스트 결과
    └── pull_requests ─── PR 기록
```

### 5.2 핵심 데이터 원칙

| 원칙 | 설명 |
|------|------|
| **employees = 기준 테이블** | 전체 직원 목록은 반드시 `organization.employees` 기준 (로그인 여부 무관) |
| **users ≠ 전체 직원** | `organization.users`는 로그인한 사람만. 단독 사용 시 미로그인 직원 누락 |
| **스키마 분리** | 도메인별로 스키마를 분리하여 관심사 격리 |
| **파라미터화된 쿼리** | SQL Injection 방지를 위해 반드시 파라미터 바인딩 사용 |

---

## 6. 프로세스/흐름 체계 (Process Taxonomy)

### 6.1 인증 흐름 (Authentication)

```
사용자 → 프론트엔드 앱 → jabis-cert (PKCE 시작)
→ Microsoft Azure AD (OAuth 2.0)
→ jabis-cert (콜백, 세션 쿠키 설정)
→ 프론트엔드 (JWT 수령, API 호출에 사용)
```

### 6.2 API 호출 패턴

JABIS API는 **GET과 POST만** 사용합니다. PUT/PATCH/DELETE는 사용하지 않습니다.

```
조회: GET  /api/{resource}?param=value
변경: POST /api/{resource}  { "action": "create|update|delete", ...data }
응답: { "success": true|false, "data": {...} | "error": "..." }
```

### 6.3 배포 파이프라인

```
코드 push (main 브랜치)
→ Bitbucket Pipeline 트리거
→ Docker 이미지 빌드 → Harbor 레지스트리에 push
→ jabis-helm values 업데이트
→ ArgoCD 자동 감지 → K3S 클러스터 배포
→ Teams 알림
```

> push = 배포. main에 push하면 자동으로 운영 배포가 실행됩니다.

### 6.4 공유 라이브러리 수정 흐름

jabis-common은 모든 프론트엔드의 **Single Source of Truth**입니다. 수정 시 아래 순서를 따릅니다:

```
1. jabis-common에서 수정 → 커밋 → push
2. 각 소비 프로젝트에서:
   git submodule update --remote packages
   git add packages
   커밋 → push
3. 미리보기 빌드로 검증
```

> 이 흐름을 거치지 않으면 소비 프로젝트는 수정 전 코드를 계속 사용합니다 (submodule이 이전 커밋을 참조).

### 6.5 AI 자동화 흐름

```
기능 요청 등록 (DB)
→ Night Builder 폴링
→ AI 코드 생성 (ai-orchestra)
→ 자동 테스트
→ PR 생성 → Teams 알림
```

### 6.6 미리보기 흐름

```
코드 수정
→ pnpm build:preview (base path: /preview/{프로젝트명}/)
→ jabis-maker가 dist/ 서빙
→ https://jabis-maker.ngrok-free.dev/preview/{프로젝트명}/
```

---

## 7. 인프라 계층 (Infrastructure Taxonomy)

### 7.1 실행 환경

| 환경 | NODE_ENV | 도메인 패턴 | 용도 |
|------|----------|------------|------|
| Local | local | localhost | 개발 |
| Alpha | alpha | *-alpha.jinhakapply.com | 스테이징 (구축 중) |
| Production | production | *.jinhakapply.com | 운영 |

### 7.2 인프라 구성 요소

```
인프라
├── 컨테이너/오케스트레이션
│   ├── K3S ─── 경량 Kubernetes 클러스터
│   ├── Helm ─── 패키지/배포 관리
│   └── ArgoCD ─── GitOps 기반 자동 배포
│
├── 레지스트리
│   └── Harbor ─── Docker 이미지 저장소 (harbor-apply.jinhaksa.com)
│
├── 데이터 저장소
│   ├── PostgreSQL ─── 주 데이터베이스 (스키마 분리)
│   ├── Redis ─── 캐시, 세션, Rate Limiting
│   └── MinIO ─── 오브젝트 스토리지 (파일 업로드)
│
├── 외부 서비스
│   ├── Microsoft Azure AD ─── OAuth 인증 제공자
│   ├── Bitbucket ─── 소스코드 저장소 + CI/CD Pipeline
│   ├── Microsoft Teams ─── 알림
│   └── ngrok ─── 로컬 터널링 (미리보기, E2E 테스트)
│
└── 내부 통신
    └── K3S 서비스 URL: http://{서비스명}-prod-service.jabis-prod:{포트}
```

---

## 8. 공유 라이브러리 체계 (@jabis/*)

jabis-common은 6개의 패키지를 제공합니다. 모든 패키지의 원본은 jabis-common에만 존재합니다.

| 패키지 | 역할 | 주요 export |
|--------|------|------------|
| **@jabis/ui** | UI 컴포넌트 키트 | Button, Dialog, Table, Toast 등 30+ 컴포넌트 |
| **@jabis/layout** | 레이아웃 프리미티브 | DashboardLayout, Header, Sidebar |
| **@jabis/auth** | 인증 로직 | LoginPage, CallbackPage, useAuthStore |
| **@jabis/core** | 코어 프로바이더 | ThemeProvider, ThemeToggle |
| **@jabis/menu** | 메뉴 빌더 | buildMenu(), availableRoles, getRoleUrl() |
| **@jabis/shared-pages** | 공유 페이지 템플릿 | 26개 페이지, 30+ 컴포넌트 |

### 소비 방식

```
jabis-common (원본)
    ↓ git submodule
소비 프로젝트의 packages/ 디렉토리
    ↓ pnpm workspace
import { Button } from '@jabis/ui'
import { buildMenu } from '@jabis/menu'
```

---

## 9. 핵심 용어 사전 (Glossary)

| 용어 | 정의 |
|------|------|
| **JABIS** | Jinhak Business Integration System. 진학어플라이 통합 업무 시스템 |
| **포탈 (Portal)** | jabis 메인 앱. 17개 역할 기반 대시보드를 하나의 앱에서 제공 |
| **부서앱 (Dept App)** | 특정 부서 전용 앱. 모노레포 구조(`apps/{role}/`), jabis-common을 submodule로 소비 |
| **역할 (Role)** | 사용자의 직무 유형. 17개 정의 (developer, finance, hr 등) |
| **@jabis/*** | jabis-common에서 제공하는 공유 패키지군 (ui, layout, auth, core, menu, shared-pages) |
| **Submodule** | git submodule로 jabis-common을 dept 프로젝트의 packages/에 연결하는 방식 |
| **API Gateway** | jabis-api-gateway. SQL 쿼리를 직접 실행하는 백엔드 (ORM 미사용) |
| **action 필드** | POST 요청에서 mutation 종류를 구분하는 필드 (create / update / delete) |
| **Night Builder** | AI 기반 자동 구현 서비스. 기능 요청 → 코드 생성 → 테스트 → PR 생성 |
| **NightExecutor** | jabis-maker의 대량 작업 배치 시스템. "night mode로 작업" = NightExecutor 실행 |
| **Auto Debugger** | AI 기반 자동 디버깅 서비스. 버그 감지 → 수정 → 테스트 → PR |
| **jabis-maker** | 로컬 빌드/미리보기 서버 (포트 3200). dist/ 파일을 서빙하고 NightExecutor 실행 |
| **미리보기 (Preview)** | `build:preview`로 빌드 → jabis-maker가 서빙하는 테스트 환경 |
| **Vibe Coding** | 디자인 시스템 기반 빠른 AI 협업 개발 방식 |
| **Single Source of Truth** | 유일한 진실의 원천. jabis-docs = 정책/스펙, jabis-common = 공유 코드 |
| **ai-e2e-tester** | AI가 브라우저를 직접 조작하여 UI를 자동 테스트하는 서비스 |
| **AI Orchestra** | AI 세션/이벤트 관리 시스템 (세션 시작/업데이트/종료 이벤트) |
| **Babel** | jabis-bitbucket-sync의 별칭. Bitbucket 저장소/의존성 동기화 서비스 |
| **공통 그룹** | @jabis/menu에서 제공하는 5개 메뉴 그룹 (work, communication, life, organization, etc) |
| **PKCE** | OAuth 2.0의 보안 확장. 프론트엔드에서 Client Secret 없이 인증 |

---

## 10. 전체 관계도

```
                         ┌──────────────┐
                         │  Azure AD    │
                         └──────┬───────┘
                                │ OAuth 2.0 (PKCE)
                         ┌──────▼───────┐
                         │  jabis-cert  │ ← JWT 발급
                         └──────┬───────┘
                                │
    ┌───────────────────────────┼───────────────────────────┐
    │                    프론트엔드 계층                       │
    │                                                        │
    │   jabis (포탈)  jabis-hr  jabis-finance  jabis-dev ...  │
    │        │            │          │            │           │
    │        └────────────┴──────────┴────────────┘           │
    │                         │                               │
    │                  ┌──────▼──────┐                        │
    │                  │jabis-common │ @jabis/* (submodule)    │
    │                  └─────────────┘                        │
    └────────────────────────┬────────────────────────────────┘
                             │ API 호출 (GET/POST + JWT)
                      ┌──────▼──────┐
                      │ API Gateway │ ← SQL 직접 실행
                      └──────┬──────┘
                             │
                 ┌───────────┼───────────┐
                 │           │           │
           ┌─────▼──┐ ┌─────▼──┐ ┌──────▼────┐
           │PostgreSQL│ │ Redis  │ │   MinIO   │
           │(스키마별)│ │ (캐시) │ │(스토리지) │
           └────┬────┘ └────────┘ └───────────┘
                │
       ┌────────▼────────┐
       │  Night Builder   │ ← AI 자동화 (폴링 → 구현 → 테스트 → PR)
       └────────┬────────┘
                │
       ┌────────▼────────┐
       │   jabis-helm     │ ← K3S 배포 (ArgoCD)
       └─────────────────┘
```

---

## 참고 문서

| 문서 | 경로 | 내용 |
|------|------|------|
| 시스템 아키텍처 | [system-overview.md](./system-overview.md) | 서비스 구조, 포트, 통신 |
| 프로젝트 맵 | [project-map.md](../onboarding/project-map.md) | 전체 프로젝트 목록 |
| 프로젝트 구조 정책 | [project-structure.md](../policies/project-structure.md) | 디렉토리/패키지 규칙 |
| 코딩 스타일 | [coding-style.md](../policies/coding-style.md) | 네이밍, API 패턴, Toast |
| 배포 정책 | [deployment.md](../policies/deployment.md) | Pipeline, Helm, DB 마이그레이션 |
| 디자인 시스템 | [DESIGN-SYSTEM.md](../design-system/DESIGN-SYSTEM.md) | 컬러, 타이포, 컴포넌트 |
