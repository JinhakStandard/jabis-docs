# jabis-cert

> 통합인증 서버 (Microsoft OAuth + PKCE, LDAP, TOTP)

---

## 기술 스택
- Next.js 14.2.15 (App Router)
- TypeScript
- React 18.3.1
- PostgreSQL 16
- Redis (optional)
- Azure AD MSAL Node
- LDAP (ldapjs)
- JWT (jose)
- TOTP (otpauth/otplib)
- Winston

## 프로젝트 구조
```
src/
├── app/                    # Next.js App Router
│   ├── api/               # API routes
│   │   ├── admin/         # 관리자 전용 API
│   │   ├── auth/          # 인증 API (Microsoft, LDAP, TOTP)
│   │   ├── clients/       # 클라이언트 관리 API
│   │   └── internal/      # K3S 내부 API 전용
│   ├── dashboard/
│   └── login/
├── components/
├── lib/
│   ├── auth/              # Microsoft PKCE, LDAP, crypto
│   ├── clients.ts
│   ├── database.ts
│   ├── tokens.ts
│   ├── rate-limit.ts
│   └── logger.ts
└── types/
```

## 주요 명령어
```bash
npm run dev       # 개발 서버
npm run build     # 프로덕션 빌드
npm start         # 프로덕션 실행
npm run lint      # ESLint 검사
```

## 포트
- 3000

## 환경변수 (Dockerfile)
- NODE_ENV
- PORT
- HOSTNAME
- BASE_URL
- POSTGRES_HOST
- POSTGRES_PORT
- POSTGRES_DATABASE
- POSTGRES_SCHEMA
- POSTGRES_USER
- POSTGRES_SSL
- JWT_EXPIRES_IN
- JWT_REFRESH_EXPIRES_IN
- AZURE_CLIENT_ID
- AZURE_TENANT_ID
- AZURE_REDIRECT_URI
- LDAP_URL
- LDAP_BASE_DN
- ALLOWED_DOMAINS
- K3S_INTERNAL_CIDRS
- INTERNAL_NETWORK_CIDRS
- TOTP_ENABLED
- TOTP_ISSUER
- REDIS_ENABLED

## Dockerfile
- node:20-alpine multi-stage 빌드
- Next.js standalone 빌드
- non-root user (nextjs:1001)
- Vault Agent /keys/vault-key.json 마운트
- health check: /api/health

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
- Next.js 14.2.15 고정 (15/16 사용 금지 - SSR 이슈)
- React 18.3.1 고정 (19 사용 금지)
- 클라이언트/서버 컴포넌트 명확 분리 (`use client`)
- 동적 렌더링: usePathname/useSearchParams 시 `export const dynamic = 'force-dynamic'`
- API 응답 형식: `{ success: true, data }` 또는 `{ success: false, error }`
- RoleLevel enum (USER=0, DEVELOPER=1, APPROVER=2, ADMIN=3)
- 휴면 계정 정책: 90일 미로그인 시 자동 휴면
- Open Redirect 방지: 콜백 URL 검증 필수
- 클라이언트 시크릿 해싱: bcrypt 사용
- Rate Limiting: checkRateLimit 미들웨어
- 내부 API: /api/internal/token/introspect (Basic Auth, CIDR 제한)
- 빌드 오류 시: .next 폴더 삭제 후 재빌드
