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
│   ├── api.ts             # API 엔드포인트 라우트 (와일드카드)
│   ├── organization.ts    # 조직도 + 권한 관리 API
│   └── health.ts
├── services/
│   ├── queryService.ts    # SQL 쿼리 실행 + 파라미터 검증
│   ├── certService.ts     # jabis-cert 인증 검증
│   ├── organizationService.ts  # 조직 관리 비즈니스 로직
│   └── cacheService.ts    # Redis 캐싱
├── controllers/
├── middleware/
│   ├── auth.ts
│   ├── rateLimit.ts
│   └── errorHandler.ts
├── repositories/
│   ├── endpointRepository.ts
│   └── organizationRepository.ts  # 조직 관리 DB 접근
├── types/
│   └── organization.ts    # 조직 관리 타입 정의
├── database/
│   ├── pool.ts
│   └── migrate.js
├── docs/openapi/          # OpenAPI 스펙
└── sql/
    ├── organization-schema.sql     # 조직 DB 스키마
    ├── organization-seed.sql       # 개발용 시드 데이터
    ├── organization-seed-real.sql  # 실제 조직도 데이터 (57부서, 202명)
    └── organization-deploy.sql     # 배포용 통합 스크립트
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

## 조직 관리 시스템 (Organization)

2026-02-11 구현. 부서/직원/사용자/권한 통합 관리.

### API 엔드포인트
- `GET /api/organization/departments` — 부서 목록
- `GET /api/organization/employees` — 직원 목록
- `GET /api/organization/employees/search?name=` — 직원 검색
- `GET /api/organization/me` — 현재 사용자 정보 + 권한 (JWT 필수)
- `GET /api/organization/users` — 사용자 목록
- `GET /api/organization/permissions` — 권한 목록
- `GET /api/organization/users/:userId/permissions` — 특정 사용자 권한
- `POST /api/organization/departments` — 부서 CRUD (action: create/update/delete)
- `POST /api/organization/employees` — 직원 CRUD (action: create/update/delete)
- `POST /api/organization/me/map-employee` — 직원 매칭
- `POST /api/organization/users/:userId/permissions` — 권한 관리 (action: grant/revoke/set/set-super-admin)

### DB 스키마
- 스키마: `organization` (5개 테이블)
- 상세: [db-schemas/organization-schema.md](../db-schemas/organization-schema.md)
- API 상세: [api-specs/organization-api.md](../api-specs/organization-api.md)

### 핵심 규칙
- 사용자 ID: JWT sub/email에서 `@` 앞부분만 추출 (절대 규칙)
- 슈퍼관리자(`is_super_admin=TRUE`): 모든 권한 자동 부여
- DDL 관리: DBA 수동 실행 (앱에서 자동 DDL 금지)
- 배포: `psql -f sql/organization-deploy.sql`
