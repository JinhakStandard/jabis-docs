# jabis-night-builder

> 24시간 자동 구현 콘솔 — Night Builder & Auto Debugger (읽기전용 — 부장님 관리)

---

## 기술 스택
- Node.js 20+
- TypeScript (strict mode)
- js-yaml + zod (YAML 설정 + 검증)
- winston (로깅)
- vitest (테스트)
- pm2 (프로세스 관리)

## 프로젝트 구조
```
jabis-night-builder/
├── pm2.config.js
├── config/
│   ├── default.yaml
│   ├── development.yaml
│   └── production.yaml
├── src/
│   ├── index.ts           # 진입점
│   ├── app.ts             # App 클래스
│   ├── config/            # Zod 스키마, loader, types
│   ├── core/              # lifecycle, scheduler, logger
│   ├── clients/           # api-gateway, git, orchestra
│   ├── night-builder/     # builder.service, request-processor, test-runner, pr-creator
│   ├── auto-debugger/     # debugger.service, task-processor
│   ├── shared/            # retry, platform, process-runner, errors
│   └── types/
├── .claude/skills/        # 5개
└── .ai/                   # 세션 기록
```

## 주요 명령어
```bash
npm run dev               # 개발 실행 (tsx src/index.ts)
npm run build             # 빌드 (tsc)
npm start                 # 운영 실행
npm test                  # 테스트 (vitest)
pm2 start pm2.config.js   # PM2 프로세스 관리
```

## 포트
- 없음 (데몬/CLI 도구)

## 환경변수 (Dockerfile)
- NODE_ENV
- JABIS_API_GATEWAY__AUTH_TOKEN
- CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
- 모든 설정은 JABIS_ 접두사 + __ 구분자로 오버라이드 가능

## Dockerfile
없음 (CI/CD 불필요, 내부 크론/데몬)

## Bitbucket Pipeline
없음

## Skills
- apply-standard
- commit
- review-pr
- session-start
- test

## 프로젝트 고유 규칙
- 3중 방어 구조: CLAUDE.md (규칙) → settings.json (권한) → Hooks (감시)
- 크로스플랫폼(Windows/Linux) 호환 필수 (shared/platform.ts)
- 모든 외부 프로세스는 shared/process-runner.ts 경유
- 에러는 shared/errors.ts 커스텀 에러 클래스 사용
- 외부 입력은 Zod 스키마로 검증
- 비동기 에러는 반드시 catch 처리
- 설정은 config/default.yaml 사용 (하드코딩 금지)

> **주의**: 이 프로젝트는 부장님이 직접 관리합니다. 다른 팀원은 소스 코드를 수정하지 마세요.
