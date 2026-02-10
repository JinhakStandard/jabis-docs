# jabis-api-gateway

> SQL 쿼리 실행 전용 API Gateway (GET/POST만 허용)

---

## 기술 스택
- Fastify 4.28.1
- TypeScript 5.5.3
- Node.js 20
- PostgreSQL 16
- Redis 7.x
- Zod
- Axios
- Winston
- OpenTelemetry (선택)
- Prometheus (메트릭)

## 프로젝트 구조
```
src/
├── server.ts              # Fastify 진입점
├── app.ts                 # Fastify 설정
├── routes/
│   ├── api.ts             # API 엔드포인트 라우트
│   └── health.ts
├── services/
│   ├── queryService.ts    # SQL 쿼리 실행 + 파라미터 검증
│   ├── certService.ts     # jabis-cert 인증 검증
│   └── cacheService.ts    # Redis 캐싱
├── controllers/
├── middleware/
│   ├── auth.ts
│   ├── rateLimit.ts
│   └── errorHandler.ts
├── repositories/
│   └── endpointRepository.ts
├── types/
├── database/
│   ├── pool.ts
│   └── migrate.js
└── docs/openapi/          # OpenAPI 스펙
```

## 주요 명령어
```bash
npm run dev            # 개발 서버
npm run build          # 프로덕션 빌드
npm start              # 프로덕션 실행
npm test               # 테스트
npm run lint           # ESLint 검사
npm run typecheck      # 타입 체크
npm run migrate        # DB 마이그레이션
npm run openapi:lint   # OpenAPI 스펙 검증
npm run openapi:preview  # OpenAPI 미리보기
```

## 포트
- 3100

## 환경변수 (Dockerfile)
- NODE_ENV
- PORT
- HOST
- LOG_LEVEL
- DATABASE_HOST
- DATABASE_PORT
- DATABASE_NAME
- DATABASE_USER
- DATABASE_POOL_MIN
- DATABASE_POOL_MAX
- REDIS_HOST
- REDIS_PORT
- CERT_SERVER_URL
- CERT_CLIENT_ID
- CERT_CLIENT_SECRET
- TOKEN_CACHE_TTL
- DEFAULT_RATE_LIMIT
- RATE_LIMIT_WINDOW_MS
- IP_WHITELIST
- DEV_TASKS_API_KEYS
- DB_CONSOLE_ENABLED

## Dockerfile
- node:20-alpine multi-stage 빌드
- tsc + OpenAPI bundling
- non-root user (nodejs:1001)
- Vault /keys/vault-key.json 마운트
- health check: localhost:3100/health

## Bitbucket Pipeline
- Pattern A
- main/master → prod 배포
- alpha → alpha 배포
- manual pipelines 사용 가능

## Skills
- apply-standard
- commit
- review-pr
- session-start
- test

## 프로젝트 고유 규칙
- PUT/DELETE 사용 금지 -- GET/POST만 허용
- parameters JSONB 필드로 파라미터 정의: `[{name, type, required, default, description}]`
- validateParams() 함수로 필수값 + 타입 검증
- SQL Injection 방지: 파라미터 바인딩 필수 ($1, $2)
- DB 에러 처리: 클라이언트에 일반 메시지, 서버 로그에 원본
- API 추가/수정 시 docs/openapi/ 업데이트 필수
- jabis-cert 연동: CERT_SERVER_URL로 /api/internal/token/introspect 호출
- CDN, 분산 트레이싱, GeoIP, PagerDuty 등 구현 금지 (인프라 없음)
