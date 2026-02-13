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
Bitbucket Push → Pipeline 트리거
→ Docker Build → Harbor Push
→ jabis-helm values 업데이트 → Helm Package → Harbor Push
→ ArgoCD 자동 감지 → K3S 배포
→ Teams 알림
```

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
