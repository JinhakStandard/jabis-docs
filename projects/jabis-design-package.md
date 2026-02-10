# jabis-design-package

> @jabis/ui 디자인 토큰 추출 패키지 (npm 배포용)

---

## 기술 스택

- React 18.2.0 (peer dependency)
- Radix UI (15개 의존성)
- Tailwind CSS 3.4.0
- CVA (class-variance-authority)
- tailwind-merge
- lucide-react
- TypeScript 5.3.3
- vitest 1.1.0

## 프로젝트 구조

```
jabis-design-package/
├── src/
│   ├── components/ui/    # 30+ 컴포넌트 (tsx)
│   ├── hooks/use-toast.tsx
│   ├── lib/utils.ts
│   ├── styles/globals.css
│   ├── test/setup.ts
│   └── index.ts
├── tailwind.preset.js
├── vitest.config.ts
└── package.json
```

## 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `npm run test` | vitest 테스트 실행 |
| `npm run test:watch` | vitest watch 모드 |
| `npm run typecheck` | TypeScript 타입 체크 |

## 포트

N/A (npm 패키지)

## 환경변수 (Dockerfile)

N/A

## Dockerfile

없음

## Bitbucket Pipeline

없음 (npm 배포)

## Skills

없음

## 프로젝트 고유 규칙

- 소스 전용 배포 (TypeScript 소스 그대로, 컴파일하지 않음)
- 컴포넌트는 tsx 형식 (타입 안전성 보장)
- vitest + React Testing Library로 테스트
- `tailwind-preset`, `globals.css`는 별도 export entry point로 제공
- peer dependency: React 18.2.0
