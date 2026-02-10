# jabis-design-system

> 디자인 시스템 문서 (컴포넌트 쇼케이스 + 가이드)

---

## 기술 스택

- Vite 5 + React 18.2
- pnpm monorepo
- Tailwind CSS 3.3.5
- react-router-dom 6
- react-syntax-highlighter
- recharts
- lucide-react

## 프로젝트 구조

```
jabis-design-system/
├── pnpm-workspace.yaml
├── packages/
│   ├── ui/           # jabis-common에서 clone
│   ├── layout/
│   ├── auth/
│   └── core/
├── apps/
│   └── design/
│       ├── src/
│       ├── public/
│       │   ├── guide.html
│       │   ├── architecture.html
│       │   └── serve.json
│       ├── vite.config.js
│       └── tailwind.config.js
└── Dockerfile
```

## 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `pnpm dev` | Vite 개발 서버 (5173) |
| `pnpm build` | 프로덕션 빌드 |
| `pnpm preview` | 빌드 결과 미리보기 |

## 포트

| 환경 | 포트 |
|------|------|
| 개발 (Vite dev) | 5173 |
| 운영 (serve) | 3000 |

## 환경변수 (Dockerfile)

| 변수 | 값 | 설명 |
|------|-----|------|
| `NODE_ENV` | production | 실행 환경 |
| `NODE_OPTIONS` | --max-old-space-size=8192 | Node.js 힙 메모리 8GB |
| `VITE_BUILD_MINIFY` | esbuild | 빌드 최적화 |

## Dockerfile

- node:20-alpine 기반 multi-stage 빌드
- pnpm 사용
- jabis-common을 Docker build 중 git clone
- `serve dist -s -l 3000`으로 정적 파일 서빙

## Bitbucket Pipeline

- Pattern A
- master 브랜치 only
- jabis-common을 Docker build 중 git clone

## Skills

없음

## 프로젝트 고유 규칙

- `serve.json`에 rewrites 설정 필수 (정적 HTML 파일이 SPA 라우팅과 충돌하지 않도록)
- `guide.html`, `architecture.html`은 jabis 프로젝트에서 이관된 파일
- 빌드 시 높은 메모리 필요 (8GB Node.js heap) -- `NODE_OPTIONS=--max-old-space-size=8192`
