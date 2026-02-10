# jabis-template

> 템플릿 데모 프로젝트 — jabis-{dept} 부서별 앱 생성 시 복제 기반이 되는 템플릿 쇼케이스

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
jabis-template/
├── pnpm-workspace.yaml
├── packages/
│   ├── ui/            # @jabis/ui
│   ├── layout/
│   ├── auth/
│   └── core/
├── apps/
│   └── prototype/
│       ├── src/
│       │   ├── components/
│       │   ├── pages/
│       │   ├── stores/
│       │   ├── data/
│       │   ├── App.jsx
│       │   └── main.jsx
│       └── tailwind.config.js
└── public/
    ├── guide.html
    ├── architecture.html
    └── serve.json
```

## 주요 명령어
```bash
pnpm install              # 의존성 설치
pnpm dev                  # 개발 서버
pnpm build                # 프로덕션 빌드
pnpm preview              # 빌드 결과 미리보기
```

## 포트
- 3000 (serve)
- 5173 (Vite dev)

## 환경변수 (Dockerfile)
- NODE_OPTIONS (--max-old-space-size=8192)
- VITE_BUILD_MINIFY (esbuild)

## Dockerfile
- jabis와 동일 (node:20-alpine multi-stage pnpm, serve SPA)

## Bitbucket Pipeline
- Pattern A
- master/alpha → 수동 deploy
- jabis-common git clone

## Skills
없음

## 프로젝트 고유 규칙
- jabis와 동일 구조/규칙
- 부서별 앱(jabis-{dept}) 생성 시 이 프로젝트를 복제하여 시작
- 템플릿 컴포넌트/페이지 쇼케이스 및 데모 용도
