# jabis-lab

> 실험적 개발 플랫폼 (읽기전용 -- 부장님 관리)

---

## 기술 스택
- React 18.2 (JSX)
- Vite 5
- pnpm monorepo
- Zustand
- React Router v6
- Tailwind CSS 3.3 + CVA
- @jabis/ui

## 프로젝트 구조
```
jabis-lab/
├── pnpm-workspace.yaml
├── packages/
│   ├── ui/
│   ├── layout/
│   ├── auth/
│   └── core/
├── apps/
│   └── prototype/
│       ├── src/
│       └── tailwind.config.js
├── .claude/
│   ├── settings.json
│   └── skills/            # 5개
└── .ai/                   # 세션 로깅
```

## 주요 명령어
```bash
pnpm dev                  # 개발 서버 (--filter prototype)
pnpm build                # 프로덕션 빌드
pnpm preview              # 빌드 결과 미리보기
```

## 포트
- 3000 (serve)

## 환경변수 (Dockerfile)
- NODE_OPTIONS
- VITE_BUILD_MINIFY
- VITE_OAUTH_ENABLED
- VITE_OAUTH_CLIENT_ID
- VITE_OAUTH_REDIRECT_URI
- VITE_OAUTH_SCOPES
- VITE_OAUTH_AUTH_SERVER

## Dockerfile
- node:20-alpine multi-stage pnpm 빌드
- serve -s dist -l 3000

## Bitbucket Pipeline
- Pattern A
- dev/alpha/prod 환경별 배포

## Skills
- apply-standard
- commit
- review-pr
- session-start
- test

## 프로젝트 고유 규칙
- 모의 데이터는 apps/prototype/src/data/ 관리
- .ai/ 폴더 세션 기록 필수
- OAuth 설정은 빌드 시 번들에 포함 (환경변수)

> **주의**: 이 프로젝트는 부장님이 직접 관리합니다. 다른 팀원은 소스 코드를 수정하지 마세요.
