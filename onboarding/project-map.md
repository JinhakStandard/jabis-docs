# JABIS 프로젝트 맵

> JABIS 에코시스템의 13개 프로젝트와 역할을 정리합니다.

---

## 프로젝트 전체 목록

| # | 프로젝트 | Bitbucket | 역할 | 기술 스택 |
|---|----------|-----------|------|-----------|
| 1 | jabis-cert | jinhaksa/jabis-cert | 통합인증 서버 | Next.js 14, TypeScript |
| 2 | jabis-api-gateway | jinhaksa/jabis-api-gateway | SQL 쿼리 전용 API Gateway | Fastify, TypeScript |
| 3 | jabis | jinhaksa/jabis | 메인 운영 앱 | Vite, React, Zustand |
| 4 | jabis-template | jinhaksa/jabis-template | 템플릿/데모 앱 | Vite, React, Zustand |
| 5 | jabis-lab | jinhaksa/jabis-lab | 실험적 개발 플랫폼 | Vite, React, Zustand |
| 6 | jabis-dev | jinhaksa/jabis-dev | 개발부서 전용 | Vite, React, Zustand |
| 7 | jabis-night-builder | jinhaksa/jabis-night-builder | 24시간 자동 구현 콘솔 | Node.js, TypeScript |
| 8 | jabis-bitbucket-sync | jinhaksa/jabis-bitbucket-sync | Bitbucket 동기화 | Node.js, TypeScript |
| 9 | jabis-emergency-console | jinhaksa/jabis-emergency-console | 긴급 DB 콘솔 | Node.js, TypeScript |
| 10 | jabis-common | jinhaksa/jabis-common | 공유 UI 라이브러리 | pnpm monorepo, JavaScript |
| 11 | jabis-design-system | jinhaksa/jabis-design-system | 디자인 시스템 문서 | Vite, React |
| 12 | jabis-design-package | jinhaksa/jabis-design-package | 디자인 토큰 추출 | Node.js |
| 13 | jabis-helm | jinhaksa/jabis-helm | Helm 배포 관리 | Helm, YAML |

---

## 프로젝트 관계도

```
jabis-common (공유 라이브러리)
  ├── jabis (메인 앱)
  ├── jabis-template (템플릿)
  ├── jabis-lab (실험)
  ├── jabis-dev (개발부서)
  └── jabis-design-system (디자인 문서)

jabis-cert (인증)
  └── 모든 프론트엔드 앱에서 OAuth 인증

jabis-api-gateway (API)
  ├── 모든 프론트엔드 앱에서 데이터 조회
  └── jabis-night-builder가 폴링하여 작업 처리

jabis-helm (배포)
  └── 모든 서비스의 K8S 배포 관리
```

---

## 로컬 Clone URL 패턴

```bash
git clone https://navskh@bitbucket.org/jinhaksa/{프로젝트명}.git
```

---

## 운영 도메인

| 서비스 | URL |
|--------|-----|
| 메인 앱 | https://jabis.jinhakapply.com |
| 템플릿 | https://jabis-template.jinhakapply.com |
| 실험 | https://jabis-lab.jinhakapply.com |
| 통합인증 | https://jabis-cert.jinhakapply.com |
| API Gateway | https://jabis-gateway.jinhakapply.com |
| 긴급 콘솔 | https://jabis-emergency.jinhakapply.com |
| 디자인 시스템 | https://jabis-design.jinhakapply.com |
| Bitbucket 동기화 | https://jabis-bitbucket-sync.jinhakapply.com |
