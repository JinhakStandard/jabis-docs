# JABIS 환경 정의

> JABIS 프로젝트의 세 가지 환경(미리보기 / Alpha / 운영)을 정의합니다.
> 각 환경은 E2E 테스트(ai-e2e-tester)의 대상이 됩니다.

---

## 1. 환경 개요

| 환경 | 목적 | 빌드 명령 | 접근 방식 | E2E 테스트 |
|------|------|----------|----------|-----------|
| **미리보기 (preview)** | 빌드 결과물 검증 | `pnpm build:preview` | jabis-maker 서빙 | O (미리보기 테스트) |
| **Alpha (staging)** | 운영 배포 전 통합 검증 | `pnpm build` | K3S jabis-alpha 배포 | O (alpha 테스트) |
| **운영 (production)** | 실제 서비스 | `pnpm build` | K3S jabis-prod 배포 | O (운영 테스트) |

---

## 2. 미리보기 환경 (Preview)

### 개요
`pnpm build:preview`로 빌드한 정적 파일을 jabis-maker가 서빙하는 환경.
**빌드 결과물이 실제로 잘 동작하는지 검증**하는 것이 목적이며, 커밋/배포 전 E2E 테스트의 대상이 된다.

### 특성
| 항목 | 값 |
|------|-----|
| 빌드 | `pnpm build:preview` (`vite build --mode preview`) |
| 서빙 | jabis-maker (포트 3200) |
| URL | `https://jabis-maker.ngrok-free.dev/preview/{프로젝트명}/` |
| Vite mode | `preview` |
| base path | `/preview/{프로젝트명}/` |
| OAuth | OFF |
| API 호출 | `VITE_GATEWAY_URL` 직접 호출 (운영 gateway) |
| 인증 | `X-Dev-Token` 헤더 (개발용 토큰) |

### 미리보기 URL 목록

| 프로젝트 | URL |
|---------|-----|
| jabis | `https://jabis-maker.ngrok-free.dev/preview/jabis/` |
| jabis-hr | `https://jabis-maker.ngrok-free.dev/preview/jabis-hr/` |
| jabis-dev | `https://jabis-maker.ngrok-free.dev/preview/jabis-dev/` |
| jabis-producer | `https://jabis-maker.ngrok-free.dev/preview/jabis-producer/` |
| jabis-finance | `https://jabis-maker.ngrok-free.dev/preview/jabis-finance/` |
| jabis-design-system | `https://jabis-maker.ngrok-free.dev/preview/jabis-design-system/` |
| jabis-maker-admin | `https://jabis-maker.ngrok-free.dev/preview/jabis-maker-admin/` |

### 환경변수 (.env.preview)

```
VITE_OAUTH_ENABLED=false
VITE_GATEWAY_URL=https://jabis-gateway.jinhakapply.com
VITE_DEV_TOKEN={발급된 개발용 토큰}
```

### 미리보기의 한계

- **로그인 플로우 테스트 불가** — OAuth가 비활성화되어 있으므로 인증 관련 기능은 검증할 수 없음
- **base path 차이** — 운영과 경로가 다르므로, 절대경로로 하드코딩된 링크는 미리보기에서 깨질 수 있음
- **쿠키 기반 기능 제한** — 운영에서 쿠키로 전달되는 사용자 정보가 미리보기에서는 없음

### 용도
- 빌드 결과물 검증 (흰 화면, 라우팅 등)
- E2E 테스트 — **미리보기 테스트** (커밋 전 검증)
- UI/메뉴/네비게이션 동작 확인

---

## 3. Alpha 환경 (Staging)

### 개요
운영 배포 전 통합 검증을 위한 스테이징 환경. K3S `jabis-alpha` 네임스페이스에 배포되며, alpha 브랜치 push로 자동 배포됩니다. 운영과 동일한 빌드(`pnpm build`)를 사용하되, 환경변수와 DB가 분리되어 있습니다.

### 특성
| 항목 | 값 |
|------|-----|
| 빌드 | `pnpm build` (운영과 동일) |
| 서빙 | K3S jabis-alpha 네임스페이스 |
| 도메인 | `*-alpha.jinhakapply.com` |
| Vite mode | `production` |
| base path | `/` 또는 `/{dept}/` (운영과 동일) |
| OAuth | ON (운영 jabis-cert 공유, alpha 전용 client_id) |
| API 호출 | alpha gateway (`jabis-gateway-alpha.jinhakapply.com`) |
| 인증 | OAuth JWT 쿠키 (운영 jabis-cert 발급) |

### Alpha URL 목록

| 프로젝트 | URL | 비고 |
|---------|-----|------|
| jabis | `https://jabis-alpha.jinhakapply.com` | 메인 앱 |
| jabis-hr | `https://jabis-alpha.jinhakapply.com/hr` | dept 프로젝트 |
| jabis-dev | `https://jabis-alpha.jinhakapply.com/dev` | dept 프로젝트 |
| jabis-producer | `https://jabis-alpha.jinhakapply.com/producer` | dept 프로젝트 |
| jabis-finance | `https://jabis-alpha.jinhakapply.com/finance` | dept 프로젝트 |
| jabis-api-gateway | `https://jabis-gateway-alpha.jinhakapply.com` | API 게이트웨이 |

### 환경변수 (.env.production — alpha Dockerfile에서 오버라이드)

```
VITE_OAUTH_ENABLED=true
VITE_GATEWAY_URL=https://jabis-gateway-alpha.jinhakapply.com
VITE_OAUTH_CLIENT_ID={alpha 전용 클라이언트 ID}
VITE_OAUTH_REDIRECT_URI=https://jabis-alpha.jinhakapply.com/{경로}/auth/callback
VITE_OAUTH_AUTH_SERVER=https://jabis-cert.jinhakapply.com
```

### 공유 인프라

Alpha 환경은 일부 인프라를 운영과 공유합니다.

| 인프라 | 공유 방식 | 격리 방법 |
|--------|----------|----------|
| jabis-cert | **운영 공유** | alpha 전용 client_id 별도 발급 |
| PostgreSQL | **운영 공유** (같은 서버) | `jabis_alpha` DB 별도 사용 |
| Redis | **운영 공유** (같은 인스턴스) | DB 번호 분리 (운영 DB 0, alpha DB 1) |
| MinIO | **운영 공유** (같은 버킷) | `alpha/` prefix로 경로 분리 |

### 배포 플로우

```
alpha 브랜치 push → Bitbucket Pipeline → Docker Build
→ Harbor Push (jabis-alpha 네임스페이스)
→ jabis-helm alpha values 업데이트 → Helm Package → Harbor Push
→ ArgoCD 감지 → K3S jabis-alpha 배포
```

### 용도
- 운영 배포 전 통합 테스트
- E2E 테스트 — **alpha 테스트** (alpha → main 머지 전 검증)
- 여러 feature 브랜치의 통합 검증

> **상세 정책**: `policies/staging.md` 참조.

---

## 4. 운영 환경 (Production)

### 개요
`pnpm build`로 빌드한 정적 파일을 K3S 클러스터에서 서비스하는 실제 운영 환경.
Bitbucket push → Pipeline → Docker → Harbor → ArgoCD → K3S 자동 배포.
배포 후 E2E 테스트(**운영 테스트**)로 실서비스 정상 동작을 검증한다.

### 특성
| 항목 | 값 |
|------|-----|
| 빌드 | `pnpm build` |
| 서빙 | K3S (Nginx 컨테이너) |
| 도메인 | `*.jinhakapply.com` |
| Vite mode | `production` |
| base path | `/` 또는 `/{dept}/` |
| OAuth | ON (Microsoft OAuth + PKCE) |
| API 호출 | `VITE_GATEWAY_URL` 직접 호출 |
| 인증 | OAuth JWT 쿠키 (jabis-cert 발급) |

### 운영 URL 목록

| 프로젝트 | URL | 비고 |
|---------|-----|------|
| jabis | `https://jabis.jinhakapply.com` | 메인 앱 |
| jabis-hr | `https://jabis.jinhakapply.com/hr` | dept 프로젝트 (같은 도메인) |
| jabis-dev | `https://jabis.jinhakapply.com/dev` | dept 프로젝트 |
| jabis-producer | `https://jabis.jinhakapply.com/producer` | dept 프로젝트 |
| jabis-finance | `https://jabis.jinhakapply.com/finance` | dept 프로젝트 |
| jabis-design-system | `https://jabis-design.jinhakapply.com` | 별도 도메인 |
| jabis-cert | `https://jabis-cert.jinhakapply.com` | 인증 서버 |
| jabis-api-gateway | `https://jabis-gateway.jinhakapply.com` | API 게이트웨이 |

### 환경변수 (.env.production)

```
VITE_OAUTH_ENABLED=true
VITE_GATEWAY_URL=https://jabis-gateway.jinhakapply.com
VITE_OAUTH_CLIENT_ID={클라이언트 ID}
VITE_OAUTH_REDIRECT_URI=https://jabis.jinhakapply.com/{경로}/auth/callback
VITE_OAUTH_AUTH_SERVER=https://jabis-cert.jinhakapply.com
```

### 배포 플로우

```
코드 push → Bitbucket Pipeline → Docker Build → Harbor Push
→ jabis-helm values 업데이트 → Helm Package → Harbor Push
→ ArgoCD 감지 → K3S 배포
```

### 용도
- 실제 사용자 서비스
- E2E 테스트 — **운영 테스트** (배포 후 검증)

---

## 5. 환경 비교

| 항목 | 미리보기 | Alpha | 운영 |
|------|---------|-------|------|
| **빌드** | `build:preview` | `build` | `build` |
| **서빙** | jabis-maker | K3S (jabis-alpha) | K3S (jabis-prod) |
| **도메인** | jabis-maker.ngrok-free.dev | *-alpha.jinhakapply.com | *.jinhakapply.com |
| **base path** | `/preview/{name}/` | `/` or `/{dept}/` | `/` or `/{dept}/` |
| **OAuth** | OFF | ON (alpha client_id) | ON |
| **API 인증** | X-Dev-Token | JWT 쿠키 | JWT 쿠키 |
| **로그인** | 불필요 | 필수 (Microsoft OAuth) | 필수 (Microsoft OAuth) |
| **데이터** | 운영 DB | Alpha DB (`jabis_alpha`) | 운영 DB (`jabis`) |
| **E2E 테스트** | 미리보기 테스트 | alpha 테스트 | 운영 테스트 |
| **시점** | 커밋 전 | alpha 브랜치 push 후 | main 머지 후 |

---

## 6. 관련 문서

| 문서 | 내용 |
|------|------|
| `policies/deployment.md` | 배포 파이프라인, Helm, DB 마이그레이션 |
| `policies/staging.md` | Alpha 환경 구축 계획, 인프라 상세 |
| `policies/git-workflow.md` | Git 워크플로우, Alpha 브랜치 전략 |
| `policies/e2e-testing.md` | E2E 테스트 정책 (ai-e2e-tester) |
| `projects/jabis-maker.md` | jabis-maker 상세 (미리보기 서빙 포함) |
