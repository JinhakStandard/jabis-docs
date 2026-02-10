# JABIS 시스템 아키텍처 개요

> JABIS(진학 통합 시스템) 전체 마이크로서비스 구조와 서비스 간 관계를 정의합니다.

---

## 1. 서비스 구조

```
┌──────────────────────────────────────────────────────────────┐
│                        프론트엔드 앱                          │
│  jabis          - 메인 앱 (운영)                              │
│  jabis-template - 템플릿/데모                                 │
│  jabis-lab      - 실험적 개발 플랫폼                          │
│  jabis-dev      - 개발부서 전용                               │
│  jabis-design-system - 디자인 시스템 문서                     │
└──────────┬──────────────────────────────────┬────────────────┘
           │ OAuth (PKCE)                     │ REST API
           ▼                                  ▼
┌──────────────────┐            ┌─────────────────────────┐
│   jabis-cert     │            │  jabis-api-gateway      │
│  통합인증 서버    │            │  SQL 쿼리 전용 Gateway   │
│  (Next.js 14)    │◄───────────│  (Fastify + TypeScript)  │
│  Microsoft OAuth │  토큰 검증  │  GET/POST만 사용         │
│  + PKCE          │            │                         │
└──────────────────┘            └─────────┬───────────────┘
                                          │
                                          │ 폴링 (REST)
                                          ▼
                                ┌─────────────────────────┐
                                │  jabis-night-builder    │
                                │  24시간 자동 구현 콘솔   │
                                │  Night Builder          │
                                │  Auto Debugger          │
                                │  ai-orchestra 연동      │
                                └─────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                        지원 서비스                            │
│  jabis-bitbucket-sync  - Bitbucket 리포/브랜치/의존성 동기화  │
│  jabis-emergency-console - 긴급 DB 콘솔                      │
│  jabis-common          - @jabis/ui, layout, auth, core       │
│  jabis-helm            - 전체 Helm 배포 관리                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 서비스별 상세

> 기술 스택은 [프로젝트 맵](../onboarding/project-map.md) 참조

| 서비스 | 포트 | 도메인 | 역할 |
|--------|------|--------|------|
| jabis-cert | 3000 | jabis-cert.jinhakapply.com | 통합인증 (OAuth + PKCE) |
| jabis-api-gateway | 3100 | jabis-gateway.jinhakapply.com | SQL 쿼리 실행, Dashboard API |
| jabis | 3000 | jabis.jinhakapply.com | 메인 운영 앱 |
| jabis-template | 3000 | jabis-template.jinhakapply.com | 템플릿/데모 |
| jabis-lab | 3000 | jabis-lab.jinhakapply.com | 실험적 개발 플랫폼 |
| jabis-dev | 3000 | jabis.jinhakapply.com/dev | 개발부서 전용 |
| jabis-night-builder | - | - | 야간 자동 구현 콘솔 |
| jabis-bitbucket-sync | 3010 | jabis-bitbucket-sync.jinhakapply.com | Bitbucket 동기화 |
| jabis-emergency-console | 3005 | jabis-emergency.jinhakapply.com | 긴급 DB 접근 |
| jabis-design-system | 3000 | jabis-design.jinhakapply.com | 디자인 시스템 문서 |

---

## 3. 인증 플로우

```
1. 클라이언트 앱 → jabis-cert/api/oauth/authorize (로그인 시작)
2. jabis-cert → Microsoft OAuth (PKCE 사용)
3. Microsoft → jabis-cert/api/auth/microsoft/callback (인증 완료)
4. jabis-cert → 클라이언트/auth/callback (authorization code 전달)
5. 클라이언트 → jabis-cert/api/oauth/userinfo (쿠키 인증으로 사용자 정보)
```

- Client Secret은 프론트엔드에서 사용하지 않음 (쿠키 기반)
- jabis-cert가 세션 쿠키 설정, 클라이언트는 `withCredentials: true`

---

## 4. API Gateway 구조

> HTTP Method 규칙은 [코딩 스타일](../policies/coding-style.md#4-http-method-규칙) 참조

- 모든 엔드포인트는 `gateway.api_endpoints` 테이블에 등록
- parameters는 JSONB로 정의: `[{name, type, required, default, description}]`
- 파라미터 검증(`validateParams`) 후 SQL 쿼리 실행

### API 그룹
| 그룹 | 경로 | 용도 |
|------|------|------|
| AI Orchestra | `/api/v1/orchestra/*` | 세션 이벤트, 동기화 |
| Night Builder | `/api/requests/*`, `/api/debug-tasks/*` | 기능 요청, 디버그 태스크 |
| Dashboard | `/api/dashboard/*` | 대시보드 조회 |
| Dev Tasks | `/api/dev-tasks/*` | 외부 개발 작업 요청 |
| Dynamic Endpoints | `/api/*` | DB 등록 SQL 쿼리 실행 |

---

## 5. 공통 모듈 (@jabis/*)

jabis-common을 각 프론트엔드 프로젝트에서 참조합니다.

| 패키지 | 역할 |
|--------|------|
| @jabis/ui | shadcn/ui 기반 공유 UI 컴포넌트 |
| @jabis/layout | DashboardLayout, Header, Sidebar |
| @jabis/auth | LoginPage, CallbackPage, useAuthStore |
| @jabis/core | ThemeProvider, ThemeToggle, sites |

- 부서 리포는 공통 패키지 수정 금지 — `@jabis/*` import만 허용
- 메뉴는 각 부서 리포에서 정적 JSON으로 관리

---

## 6. 배포 파이프라인

```
Bitbucket Push → Pipeline 트리거
→ jabis-common clone → Docker Build → Harbor Push
→ jabis-helm values 업데이트 → Helm Package → Harbor Push
→ ArgoCD 자동 감지 → K3S 배포
→ Teams 알림
```

---

## 7. K8S 내부 서비스 통신

Pod 간 직접 통신은 내부 서비스 URL 사용 (외부 도메인 금지):

```
http://{서비스명}-prod-service.jabis-prod:{포트}
```

| 서비스 | 내부 URL |
|--------|----------|
| jabis-cert | `http://jabis-cert-prod-service.jabis-prod:3000` |
| jabis-api-gateway | `http://jabis-api-gateway-prod-service.jabis-prod:3100` |
