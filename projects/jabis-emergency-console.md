# jabis-emergency-console

> 긴급 PostgreSQL DB 콘솔 (Full-stack)

---

## 기술 스택
- Express.js + TypeScript (백엔드)
- React + Vite (프론트엔드, public/app/)
- PostgreSQL (pg)
- LDAP (ldapts)
- JWT
- Helmet
- CORS
- express-rate-limit
- speakeasy (2FA)
- Winston (로깅)

## 프로젝트 구조
```
jabis-emergency-console/
├── src/              # 백엔드 TypeScript
├── client/           # 프론트엔드 React
├── public/app/       # 빌드된 프론트엔드
├── dist/             # 컴파일된 백엔드
├── logs/
└── Dockerfile
```

## 주요 명령어
```bash
npm run dev               # 개발 실행
npm run build             # 백엔드 빌드
npm run build:client      # 프론트엔드 빌드
npm run build:all         # 전체 빌드
npm start                 # 운영 실행
npm run start:alpha       # 알파 환경 실행
```

## 포트
- 3005

## 환경변수 (Dockerfile)
- NODE_ENV
- PORT
- DATABASE_URL
- JWT_SECRET
- SESSION_SECRET
- LDAP_SERVER
- LDAP_BIND_DN
- LDAP_BIND_PASSWORD

## Dockerfile
- node:20-alpine multi-stage
- 프론트엔드 + 백엔드 빌드
- .env.production 복사
- health check /health

## Bitbucket Pipeline
- Pattern A
- main/master/alpha + custom pipelines

## Skills
없음

## 프로젝트 고유 규칙
- 백엔드 + 프론트엔드가 하나의 프로젝트에 포함
- LDAP 인증 + 2FA (QR 코드 생성)
- .env.production이 커밋됨 (내부 전용 툴 예외)
- Vault Agent 시크릿 주입 지원
