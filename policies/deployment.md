# JABIS 배포 및 인프라 정책

> 이 문서는 JABIS 전체 프로젝트에 적용되는 배포 및 인프라 정책입니다.

---

## 1. 환경 구분

| 환경 | NODE_ENV | 도메인 | Harbor |
|------|----------|--------|--------|
| 로컬 | local | localhost | - |
| 알파 | alpha | *-alpha.jinhakapply.com | harbor-apply.jinhaksa.com/{ns}-alpha |
| 운영 | production | *.jinhakapply.com | harbor-apply.jinhaksa.com/{ns} |

---

## 2. 배포 파이프라인

```
Feature 브랜치 작업 → Push → Bitbucket PR 생성
→ Reviewer(navskh) 승인 → Merge to main
→ Pipeline 트리거 → Docker Build → Harbor Push
→ jabis-helm values 업데이트 → Helm Package → Harbor Push
→ ArgoCD 자동 감지 → K3S 배포
→ Teams 알림
```

> **main 직접 push 금지**: 모든 변경은 feature 브랜치 → PR → 승인 → merge 플로우를 따릅니다.
> 상세 정책은 `policies/git-workflow.md`의 "2.3 main 브랜치 보호 정책" 및 "3. Pull Request 워크플로우" 참조.

### ArgoCD 설정 주의사항
- VALUES FILES 순서: `values.yaml` → `values-{env}.yaml` (순서 중요!)
- common-helm 의존성 사용 (현재 1.1.0)

---

## 3. Helm 설정

### Application Values 구조
```yaml
common-helm:
  namespace: jabis
  stage: "prod"
  applications:
    "서비스명":
      enabled: true
      replicaCount: 1
      image:
        repository: "harbor-apply.jinhaksa.com/jabis/서비스명"
        tag: v1.0.0-prod.N
        pullPolicy: IfNotPresent
        containerPort: 3000
      service:
        type: ClusterIP
        port: 3000
```

### Vault 연동 설정
```yaml
vaultconfig:
  enabled: true
  mountPath: /app/keys
  approle: "vault-agent-role"
hostAliases:
  - ip: "192.168.50.250"
    hostnames:
      - "vault-apply.jinhaksa.com"
```

### 환경변수 설정 방법 (중요)

**환경변수는 Helm values가 아닌 Dockerfile에서 설정한다.**

common-helm 서브차트(v1.0.4)의 deployment 템플릿은 `extraEnvVars` 키만 인식하며, `env` 키는 무시된다. 현재 운영 환경에서는 **Dockerfile의 `ENV` 구문이 실제 환경변수를 설정**한다.

| 설정 위치 | 키 | 동작 여부 |
|-----------|-----|----------|
| Dockerfile `ENV` | - | **동작함** (실제 환경변수) |
| Helm values `env` | - | **무시됨** (템플릿에서 미사용) |
| Helm values `extraEnvVars` | - | 동작함 (템플릿에서 사용) |

**새 환경변수 추가 시:**
1. 해당 프로젝트의 `Dockerfile`에 `ENV` 구문 추가
2. 로컬 개발용 `.env` 파일에도 동일하게 추가
3. Helm values의 `env` 목록은 참고용으로만 관리 (실제 동작하지 않음)

> **주의:** Helm values에 `env`로 넣으면 동작하는 것처럼 보이지만 실제 Pod에는 반영되지 않는다. 반드시 Dockerfile을 확인할 것.

### 신규 프로젝트 배포 순서 (중요)

새 프로젝트를 처음 배포할 때는 **Helm 설정이 먼저, 프로젝트 파이프라인이 나중** 순서를 따른다.

```
1. jabis-helm에 앱/ingress 설정 추가 → 커밋 → push
   (초기 이미지 태그는 v1.0.0-prod.1 등 플레이스홀더)
2. 프로젝트(jabis-finance 등) Pipeline 실행
   → Docker 이미지 빌드 → Harbor push
   → jabis-helm의 image.tag를 실제 빌드 버전으로 업데이트
   → Helm 패키지 → Harbor push
3. ArgoCD가 감지 → K3S 배포
```

**이유:** 파이프라인의 `update-helm` 단계가 `yq`로 jabis-helm values의 이미지 태그를 업데이트하는데, 해당 앱 키(`jabis-finance`)가 values 파일에 이미 존재해야 업데이트가 성공한다. Helm 설정 없이 파이프라인만 돌리면 태그 업데이트가 실패한다.

> **요약:** Helm에 자리를 먼저 만들고 → 프로젝트 파이프라인이 그 자리에 버전을 채우는 방식.

### 순차 배포 정책 (중요)

**여러 프로젝트를 동시에 배포하면 Helm 버전 충돌이 발생할 수 있다.**

각 파이프라인이 jabis-helm의 이미지 태그를 업데이트하고 Helm 패키지를 Harbor에 푸시하는데, 동시에 실행되면 나중에 완료된 파이프라인이 먼저 완료된 파이프라인의 태그 변경을 덮어쓴다.

**규칙:**
1. **main/master 브랜치** push 시, 프로젝트 간 **최소 30초 간격**을 둔다 (`sleep 30`)
2. 의존 관계가 있으면 하위 서비스부터 배포한다 (예: gateway → frontend)
3. feature 브랜치 push는 파이프라인이 트리거되지 않으므로 간격 불필요

```bash
# 올바른 배포 순서 예시 (main 브랜치)
cd jabis-api-gateway && git push
sleep 30
cd ../jabis-hr && git push
sleep 30
cd ../jabis && git push
```

> **주의:** Claude Code로 submodule 동기화 등 다수 프로젝트를 연속 push할 때 반드시 30초 간격을 지킬 것. 간격 없이 push하면 Helm 버전 충돌로 이전 프로젝트의 이미지 태그가 덮어씌워질 수 있다.

---

## 4. PostgreSQL (JABIS DB)

| 항목 | 개발 | 운영 |
|------|------|------|
| Host | postgresql-helm.database-dev | postgresql-helm.database |
| Port | 5432 | 5432 |
| Database | jabis | jabis |
| User | jabis_admin | jabis_admin |
| NodePort | 30432 (활성화) | 비활성화 |

- DB 쿼리 작성 전 반드시 `JABIS-DB-SCHEMA.md` 참조

---

## 5. K8S 내부 서비스 호출

Pod 간 직접 통신 시 **내부 서비스 URL** 사용 (외부 도메인 사용 금지)

### URL 형식
```
http://{서비스명}-prod-service.{네임스페이스}:{포트}
```

### 등록된 서비스 목록
| 서비스 | 내부 URL | 외부 도메인 |
|--------|----------|-------------|
| jabis-cert | `http://jabis-cert-prod-service.jabis-prod:3000` | jabis-cert.jinhakapply.com |
| jabis-api-gateway | `http://jabis-api-gateway-prod-service.jabis-prod:3100` | jabis-gateway.jinhakapply.com |

### 주의사항
- 외부 도메인(`https://xxx.jinhakapply.com`)으로 호출하면 Ingress를 거쳐 외부 IP로 인식됨
- 내부 API(`/api/internal/*`)는 내부 네트워크에서만 접근 가능하므로 반드시 내부 URL 사용
- 내부 URL은 http 사용 (TLS 불필요)

---

## 6. 디버깅 제약

- **kubectl 직접 접근 불가** (K8S Pod 로그 사용 불가)
- 대안:
  - DB 쿼리 결과 확인
  - curl API 응답 확인
  - 로컬 환경 테스트

---

## 7. 미리보기 빌드 (Preview Build)

jabis-maker(포트 3200)가 각 프로젝트의 `dist/`를 `/preview/{프로젝트명}/` 경로로 서빙한다.

### build vs build:preview

| 스크립트 | vite mode | base path | 용도 |
|---------|-----------|-----------|------|
| `pnpm build` | production | `/` 또는 `./` | 운영 배포 (K3S) |
| `pnpm build:preview` | preview | `/preview/{프로젝트명}/` | 미리보기 (jabis-maker) |

**운영 빌드(`pnpm build`)로 미리보기를 띄우면 base path 불일치로 흰 화면이 발생한다.**

### 프로젝트별 미리보기 빌드 명령

| 프로젝트 | 미리보기 빌드 | preview base path |
|---------|-------------|-------------------|
| jabis | `pnpm build:preview` | `/preview/jabis/` |
| jabis-design-system | `pnpm build:preview` | `/preview/jabis-design-system/` |
| jabis-dev | `pnpm build:preview` | `/preview/jabis-dev/` |
| jabis-producer | `pnpm build:preview` | `/preview/jabis-producer/` |
| jabis-maker-admin | `pnpm build:preview` | `/preview/jabis-maker-admin/` |
| jabis-hr | `pnpm build:preview` | `/preview/jabis-hr/` |
| jabis-finance | `pnpm build:preview` | `/preview/jabis-finance/` |

### 구현 방식

### 모노레포 프로젝트 심볼릭 링크 (필수)

jabis-maker는 `{프로젝트루트}/dist/`에서 파일을 서빙한다. 모노레포 구조(`apps/{앱이름}/`)를 사용하는 프로젝트는 빌드 결과가 `apps/{앱이름}/dist/`에 생성되므로, 프로젝트 루트에 심볼릭 링크가 필요하다.

```bash
ln -s apps/{앱이름}/dist dist
```

| 프로젝트 | 심볼릭 링크 |
|---------|------------|
| jabis-hr | `dist -> apps/hr/dist` |
| jabis-dev | `dist -> apps/dev/dist` |
| jabis-producer | `dist -> apps/producer/dist` |
| jabis-finance | `dist -> apps/finance/dist` |

> 심볼릭 링크가 없으면 jabis-maker에서 "No build output found" 오류가 발생한다.

### 구현 방식

각 프로젝트의 `vite.config.js`에서 mode에 따라 base를 분기:
```js
base: mode === 'preview' ? '/preview/{프로젝트명}/' : '/',
```

앱 `package.json`에 `build:preview` 스크립트 등록:
```json
"build:preview": "vite build --mode preview"
```

루트 `package.json`에서 앱으로 위임:
```json
"build:preview": "pnpm --filter {앱이름} build:preview"
```

---

## 8. 개발 서버 실행 규칙

- 실행 전 해당 포트를 사용 중인 프로세스 kill 후 실행
- 예시: `lsof -t -i :3000 | xargs kill -9 2>/dev/null; npm run dev`

### 서비스별 기본 포트
| 서비스 | 포트 |
|--------|------|
| jabis, jabis-template, jabis-lab | 3000 |
| jabis-cert | 3000 |
| jabis-api-gateway | 3100 |
| jabis-bitbucket-sync | 3010 |
| jabis-emergency-console | 3005 |

---

## 9. 프론트엔드 API 환경별 설정

프론트엔드에서 API 호출 시, **개발 / 미리보기 / 운영** 3가지 환경에 따라 설정이 달라진다.

### 환경별 API 동작 방식

| 환경 | API 경로 | OAuth | 인증 방식 |
|------|---------|-------|----------|
| **개발** (`pnpm dev`) | Vite proxy (`/gateway-api` → localhost:3100) | OFF | 없음 (프록시 우회) |
| **미리보기** (`pnpm build:preview`) | `VITE_GATEWAY_URL` 직접 호출 | OFF | `X-Dev-Token` 헤더 |
| **운영** (`pnpm build`) | `VITE_GATEWAY_URL` 직접 호출 | ON | OAuth JWT 쿠키 |

### 환경변수 (.env 파일)

| 파일 | VITE_OAUTH_ENABLED | VITE_GATEWAY_URL | VITE_DEV_TOKEN |
|------|-------------------|------------------|----------------|
| `.env.development` | `false` | 불필요 (Vite proxy) | 불필요 |
| `.env.preview` | `false` | `https://jabis-gateway.jinhakapply.com` | 발급된 토큰 |
| `.env.production` | `true` | `https://jabis-gateway.jinhakapply.com` | 불필요 |

### API 클라이언트 패턴 — 2가지 방식

#### 패턴 A: 앱 자체 API 모듈 (jabis, jabis-producer)

앱 내부에 API 호출 모듈이 있어, 환경별 분기를 직접 처리한다.

```javascript
// src/lib/api.js 또는 src/services/gatewayApi.js
const GATEWAY_BASE = import.meta.env.DEV
  ? '/gateway-api'                              // 개발: Vite proxy
  : (import.meta.env.VITE_GATEWAY_URL || '')    // 미리보기/운영: 환경변수

// fetch 또는 axios로 GATEWAY_BASE + path 호출
```

| 프로젝트 | API 모듈 위치 | HTTP 라이브러리 |
|---------|-------------|---------------|
| jabis | `apps/prototype/src/lib/api.js` | fetch |
| jabis-producer | `apps/producer/src/services/gatewayApi.js` | axios |

#### 패턴 B: shared-pages API 클라이언트 주입 (jabis-hr 등)

shared-pages 공통 라이브러리의 store가 API를 호출하므로, **소비 프로젝트의 main.jsx에서 API 클라이언트를 주입**해야 한다.

```javascript
// main.jsx
import { setApiClient as setOrgApiClient, setApprovalApiClient } from '@jabis/shared-pages'

const GATEWAY_BASE = import.meta.env.DEV ? '' : (import.meta.env.VITE_GATEWAY_URL || '')
const DEV_TOKEN = import.meta.env.VITE_DEV_TOKEN || ''

function makeApiGet(path) {
  const headers = {}
  if (DEV_TOKEN) headers['X-Dev-Token'] = DEV_TOKEN
  return fetch(`${GATEWAY_BASE}${path}`, { credentials: 'include', headers })
    .then(res => { if (!res.ok) throw new Error(`API 요청 실패`); return res.json() })
    .then(json => { if (!json.success) throw new Error(json.error?.message); return json.data })
}

function makeApiPost(path, body) {
  const headers = { 'Content-Type': 'application/json' }
  if (DEV_TOKEN) headers['X-Dev-Token'] = DEV_TOKEN
  return fetch(`${GATEWAY_BASE}${path}`, { method: 'POST', credentials: 'include', headers, body: JSON.stringify(body) })
    .then(res => { if (!res.ok) throw new Error(`API 요청 실패`); return res.json() })
    .then(json => { if (!json.success) throw new Error(json.error?.message); return json.data })
}

setOrgApiClient({ apiGet: makeApiGet, apiPost: makeApiPost })
setApprovalApiClient({ apiGet: makeApiGet, apiPost: makeApiPost })
```

**주입 가능한 shared-pages API 클라이언트:**

| 함수 | 대상 store | API 엔드포인트 |
|------|----------|--------------|
| `setApiClient` | organizationStore | `/api/organization/*` |
| `setApprovalApiClient` | approvalStore | `/api/approval/*` |

> **중요:** shared-pages를 사용하는 프로젝트에서 이 주입을 빠뜨리면, store의 기본 fetch가 상대경로(`/api/...`)로 호출하여 **미리보기 환경에서 API가 동작하지 않는다.** 새 프로젝트 생성 시 반드시 main.jsx에 API 클라이언트 주입 코드를 추가할 것.

### Vite proxy 설정 (개발 환경 전용)

`vite.config.js`에서 개발 서버용 proxy를 설정한다.

```javascript
server: {
  proxy: {
    '/gateway-api': {
      target: 'http://localhost:3100',   // jabis-api-gateway
      changeOrigin: true,
      rewrite: path => path.replace(/^\/gateway-api/, ''),
    },
    '/auth-api': {
      target: 'http://localhost:3000',   // jabis-cert
      changeOrigin: true,
      rewrite: path => path.replace(/^\/auth-api/, ''),
    },
  },
}
```

### 프로젝트별 설정 현황

| 프로젝트 | API 패턴 | Vite proxy | shared-pages 주입 | 비고 |
|---------|---------|-----------|------------------|------|
| jabis | A + B | O | **적용** | 자체 `api.js` + shared-pages 주입 |
| jabis-producer | A (자체 모듈) | O | 불필요 | axios 기반 |
| jabis-hr | B (주입) | O | **적용** | `setOrgApiClient` + `setApprovalApiClient` |
| jabis-dev | B (주입) | O | **적용** | `setOrgApiClient` + `setApprovalApiClient` |
| jabis-design-system | 없음 | X | 미사용 | UI 쇼케이스 전용 |

### 신규 프로젝트 체크리스트

shared-pages를 사용하는 새 프로젝트를 만들 때:

1. `.env.preview`에 `VITE_GATEWAY_URL`, `VITE_DEV_TOKEN` 설정
2. `.env.production`에 `VITE_GATEWAY_URL`, `VITE_OAUTH_*` 설정
3. `main.jsx`에서 shared-pages API 클라이언트 주입 (`setOrgApiClient`, `setApprovalApiClient`)
4. `vite.config.js`에 개발용 proxy 설정 (`/gateway-api`, `/auth-api`)
5. 모노레포 구조인 경우 루트에 `ln -s apps/{앱이름}/dist dist` 심볼릭 링크 생성
6. `pnpm build:preview`로 미리보기 빌드 후 API 호출 정상 동작 확인
