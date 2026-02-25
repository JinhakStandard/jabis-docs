# 나이트모드(NightExecutor) 실행 정책

> jabis-maker의 NightExecutor를 통해 대량 작업을 배치로 실행하는 정책입니다.
> "night mode로 작업" = NightExecutor로 태스크를 실행한다는 의미입니다.

---

## 1. 개요

나이트모드는 jabis-maker의 NightExecutor가 Claude 프로세스를 순차적으로 실행하여 여러 태스크를 자동 처리하는 시스템입니다.

```
사용자/AI → WebSocket night_start → jabis-maker NightExecutor
                                         ↓
                          Claude 프로세스 (태스크별 순차 실행)
                                         ↓
                          코드 수정 → 커밋 → push (자동 배포)
```

### 1.1 실행 방식

| 방식 | 설명 |
|------|------|
| **jabis-maker 웹 UI** | 세션에서 직접 태스크 JSON 입력 후 실행 |
| **프로그래밍 제출 (CLI)** | JWT 토큰 생성 → REST 세션 생성 → WebSocket `night_start` 전송 |

---

## 2. 태스크 작성 규칙

### 2.1 태스크 구조

```json
{
  "description": "태스크 설명 (자연어, 가능한 상세하게)",
  "project": "대상 프로젝트명 (jabis-api-gateway, jabis-common 등)",
  "dependsOn": []
}
```

### 2.2 태스크 작성 원칙

1. **구체적으로 작성**: 파일 경로, 행 번호, 현재 동작, 기대 동작을 명시
2. **검증 기준 포함**: 수정 후 어떻게 확인할 수 있는지 기술
3. **의존성 명시**: `dependsOn`에 선행 태스크 인덱스를 배열로 지정 (없으면 빈 배열)
4. **프로젝트 지정**: `project` 필드로 작업 대상 프로젝트를 명시
5. **submodule 동기화 포함**: jabis-common 수정 태스크에는 소비 프로젝트 동기화 지시를 반드시 포함

### 2.3 jabis-common 수정 시 필수 지시문

jabis-common을 수정하는 태스크에는 아래 문구를 반드시 포함:

```
수정 후 jabis-common을 커밋/push하고, 모든 소비 프로젝트
(jabis, jabis-hr, jabis-dev, jabis-producer, jabis-design-system, jabis-finance)에서
git submodule update --remote packages && git add packages로 동기화 → 커밋 → push하라.
jabis-lab, jabis-template은 제외.
프로젝트별 push 간격은 30초 이상 유지하라.
```

### 2.4 태스크 예시

```json
[
  {
    "description": "financeRepository.ts의 getUserCompany() 반환값에서 ㈜ 접두사를 제거하라. ...(상세 설명)... 수정 후 jabis-api-gateway를 커밋/push하라.",
    "project": "jabis-api-gateway",
    "dependsOn": []
  },
  {
    "description": "ApprovalFormPage.jsx의 이중 submit 버그를 수정하라. ...(상세 설명)... 수정 후 jabis-common 커밋/push + 소비 프로젝트 동기화.",
    "project": "jabis-common",
    "dependsOn": []
  }
]
```

---

## 3. 프로그래밍 방식 제출 (Claude Code에서 실행)

Claude Code 세션에서 나이트모드를 직접 제출할 수 있습니다.

### 3.1 사전 조건

- jabis-maker가 PM2로 실행 중 (`pm2 list`로 확인)
- jabis-maker의 포트: `3200` (로컬)

### 3.2 제출 플로우

```
1. JWT 토큰 생성 (jabis-maker config에서 시크릿 로드)
2. REST API로 세션 생성
3. WebSocket으로 night_start 메시지 전송
```

### 3.3 Step 1: JWT 토큰 생성

jabis-maker 프로젝트 디렉토리에서 실행:

```typescript
// gen-token.ts (jabis-maker/ 에서 실행)
import * as jose from 'jose';
import { loadConfig } from './src/config/loader.js';

async function main() {
  const config = loadConfig();
  const secret = config.auth.jwt.secret;
  const secretKey = new TextEncoder().encode(secret);
  const token = await new jose.SignJWT({
    sub: 'bb6f469a-d266-4b3e-a22a-403cd3043c22',  // navskh user ID
    username: 'navskh',
    role: 'admin'
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuer(config.auth.jwt.issuer || 'jabis-maker')
    .setExpirationTime('1d')
    .setIssuedAt()
    .sign(secretKey);
  console.log(token);
}
main();
```

```bash
cd /home/jinhak/Projects/jabis-maker
npx tsx gen-token.ts
# → eyJhbGc... (JWT 토큰 출력)
```

### 3.4 Step 2: REST API로 세션 생성

```bash
SESSION_ID=$(curl -s -X POST http://localhost:3200/api/sessions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"action":"create","projectName":"__global__"}' \
  | node -e "let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{console.log(JSON.parse(d).data?.id)})")
```

### 3.5 Step 3: WebSocket으로 night_start 전송

```javascript
// night-submit.cjs (jabis-maker/ 에서 실행 — ws 모듈 필요)
const WebSocket = require('ws');
const TOKEN = process.argv[2];
const SESSION_ID = process.argv[3];

const tasks = [/* 태스크 배열 */];
const ws = new WebSocket('ws://localhost:3200/ws');

ws.on('open', () => {
  ws.send(JSON.stringify({ type: 'auth', token: TOKEN }));
});

ws.on('message', (data) => {
  const msg = JSON.parse(data.toString());
  if (msg.type === 'auth_ok') {
    ws.send(JSON.stringify({
      type: 'night_start',
      sessionId: SESSION_ID,
      tasks: tasks
    }));
  } else if (msg.type === 'night_progress') {
    console.log('Night mode started:', msg.runId);
    setTimeout(() => ws.close(), 1000);
  } else if (msg.type === 'error') {
    console.error('ERROR:', msg.message);
    ws.close();
  }
});
```

> **주의**: `.cjs` 확장자 사용 필수 (jabis-maker가 ESM 프로젝트이므로 `.js`는 ESM으로 해석됨)

### 3.6 임시 파일 정리

제출 완료 후 생성한 `gen-token.ts`, `night-submit.cjs` 파일은 반드시 삭제합니다.

---

## 4. WebSocket 메시지 타입

| 타입 | 방향 | 설명 |
|------|------|------|
| `auth` | → 서버 | JWT 토큰으로 인증 `{ type: 'auth', token: '...' }` |
| `auth_ok` | ← 서버 | 인증 성공 |
| `night_start` | → 서버 | 나이트모드 시작 `{ type: 'night_start', sessionId, tasks }` |
| `night_status` | → 서버 | 진행 상태 조회 `{ type: 'night_status', sessionId }` |
| `night_abort` | → 서버 | 나이트모드 중단 `{ type: 'night_abort', sessionId }` |
| `night_progress` | ← 서버 | 진행 상태 응답 `{ type: 'night_progress', runId, state }` |
| `error` | ← 서버 | 에러 `{ type: 'error', message: '...' }` |

---

## 5. 진행 상태 모니터링

### 5.1 로그 기반 모니터링

```bash
# Night mode 이벤트 확인
grep "Night" /home/jinhak/Projects/jabis-maker/logs/jabis-maker-$(date +%Y-%m-%d).log \
  | grep -v "Starting Claude" | tail -20

# 특정 세션의 프롬프트 활동 확인
grep "${SESSION_ID}" /home/jinhak/Projects/jabis-maker/logs/jabis-maker-$(date +%Y-%m-%d).log \
  | grep "Prompt sent" | tail -10

# 최신 로그 확인
tail -5 /home/jinhak/Projects/jabis-maker/logs/jabis-maker-$(date +%Y-%m-%d).log
```

### 5.2 Git 커밋으로 결과 확인

```bash
# 특정 시간 이후 커밋 확인
for proj in jabis-api-gateway jabis-common jabis jabis-hr jabis-dev jabis-producer jabis-design-system jabis-finance; do
  echo "=== $proj ==="
  cd /home/jinhak/Projects/$proj 2>/dev/null && \
    git log --oneline --since="$(date +%Y-%m-%d) 23:00" --format="%h %ai %s" | head -5
done
```

### 5.3 push 상태 확인

```bash
# remote와 동기화 여부 확인 (ahead/behind 없으면 push 완료)
cd /home/jinhak/Projects/${PROJECT} && git status -sb | head -1
```

---

## 6. 주의사항

### 6.1 글로벌 잠금

- 나이트모드는 **한 번에 하나만** 실행 가능 (글로벌 잠금)
- 새 나이트모드를 시작하려면 기존 실행이 완료되거나 abort되어야 함
- abort 후 약 5초 대기 후 새 나이트모드 시작 가능

### 6.2 Abort 처리

```javascript
// abort는 응답을 반환하지 않음 — 전송 후 2초 대기 후 연결 종료
ws.send(JSON.stringify({ type: 'night_abort', sessionId: '...' }));
setTimeout(() => ws.close(), 2000);
```

### 6.3 세션 재사용 불가

- 이미 나이트모드가 실행된(혹은 실행 중인) 세션에 다시 `night_start`를 보내면 "Night mode is already running on this session" 에러
- **항상 새 세션을 생성**하여 나이트모드를 시작해야 함

### 6.4 push = 배포

- Bitbucket에 push하면 Pipeline이 **자동으로 운영 배포** 실행
- 여러 프로젝트 push 시 **30초 간격** 필수 (Helm 충돌 방지)
- 나이트모드 태스크에 이 규칙을 명시해야 함

### 6.5 나이트모드 hang 감지

- 마지막 프롬프트 전송 후 **30분 이상** 활동이 없으면 hang으로 판단
- abort 후 재시작 권장

---

## 7. 핵심 참조 정보

| 항목 | 값 |
|------|---|
| jabis-maker 포트 | `3200` (로컬) |
| jabis-maker PM2 이름 | `jabis-maker` |
| 로그 경로 | `/home/jinhak/Projects/jabis-maker/logs/jabis-maker-YYYY-MM-DD.log` |
| navskh user ID | `bb6f469a-d266-4b3e-a22a-403cd3043c22` |
| JWT 알고리즘 | HS256 |
| JWT issuer | `jabis-maker` |
| JWT secret 소스 | `loadConfig().auth.jwt.secret` (환경변수 `JABIS_AUTH__JWT__SECRET`) |
| DB | SQLite — `/home/jinhak/Projects/jabis-maker/data/jabis-maker.db` |
| WebSocket 경로 | `ws://localhost:3200/ws` |
| 세션 생성 API | `POST /api/sessions` `{"action":"create","projectName":"__global__"}` |

---

## 8. 관련 문서

| 문서 | 설명 |
|------|------|
| `projects/jabis-night-builder.md` | jabis-night-builder (폴링 기반 자동 빌더) — 별도 시스템 |
| `api-specs/night-builder-api.md` | night-builder API 스펙 |
| `policies/deployment.md` | 배포 정책 (push=배포, 30초 간격 등) |
| `policies/ai-collaboration.md` | AI 협업 전반 정책 |

> **참고**: jabis-night-builder와 jabis-maker NightExecutor는 별개 시스템입니다.
> - jabis-night-builder: API Gateway의 feature request를 폴링하여 자동 빌드/PR 생성
> - jabis-maker NightExecutor: 사용자가 직접 정의한 태스크를 배치 실행 (이 문서의 대상)
