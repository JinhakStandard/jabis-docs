# jabis-bitbucket-sync

> Bitbucket Cloud 리포/브랜치/의존성 정보를 스케줄링으로 수집하여 JABIS 모니터링 대시보드에 제공

---

## 기술 스택
- Node.js 20+
- TypeScript
- Express 5.2.1
- axios
- node-cron
- fast-xml-parser
- semver
- pino (로깅)

## 프로젝트 구조
```
jabis-bitbucket-sync/
├── src/
│   ├── index.ts
│   ├── cli.ts             # CLI 모드
│   ├── config/
│   ├── clients/           # Bitbucket, API Gateway 클라이언트
│   ├── services/          # sync, dependency, version-check
│   ├── models/
│   ├── types/
│   ├── utils/
│   └── routes/            # Express 라우트
├── sql/create-tables.sql
├── keys/                  # vault-key.json
└── .env.example
```

## 주요 명령어
```bash
npm run dev               # 개발 실행 (tsx watch)
npm run build             # 빌드 (tsc)
npm start                 # 운영 실행
npm run cli               # CLI 모드
npm run sync              # 동기화 실행
```

## 포트
- 3010

## 환경변수 (Dockerfile)
- NODE_ENV
- PORT
- LOG_LEVEL
- BITBUCKET_CLIENT_ID
- BITBUCKET_CLIENT_SECRET
- BITBUCKET_WORKSPACE
- API_GATEWAY_URL
- API_GATEWAY_TOKEN
- SCHEDULER_ENABLED
- SYNC_FULL_CRON
- SYNC_INCREMENTAL_CRON
- SYNC_VERSION_CHECK_CRON
- SYNC_STATUS_UPDATE_CRON

## Dockerfile
- node:20-alpine multi-stage
- tsc 빌드
- non-root user (nodejs:1001)
- Vault /keys/vault-key.json
- health check /health

## Bitbucket Pipeline
- Pattern B
- prod only

## Skills
없음

## 프로젝트 고유 규칙
- Bitbucket OAuth 2.0 Client Secret은 Vault에서 주입
- 크론 스케줄러: full sync, incremental sync, version check, status update
- Bitbucket API rate limit 대응: 60초 대기 반복
- K8S 내부 URL 사용: http://jabis-api-gateway-prod-service.jabis-prod:3100
- full sync 순서: repository → branch → dependency
