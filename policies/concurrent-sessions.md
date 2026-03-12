# 동시 작업 세션 정책

> 여러 Claude 세션이 동시에 JABIS 프로젝트를 작업할 때의 정책입니다.
> jabis-maker의 세션 관리 로직은 이 정책을 기반으로 구현합니다.

---

## 1. 세션 디렉토리 격리

여러 세션이 같은 프로젝트를 동시에 작업하면 로컬 파일 충돌이 발생합니다. 이를 방지하기 위해 **세션별 독립 디렉토리**에서 작업합니다.

### 1.1 디렉토리 구조

```
/home/jinhak/Projects/                         ← 원본 (NightExecutor 전용)
/home/jinhak/Projects/.sessions/
  ├── <session-id>/
  │   └── <project-name>/                      ← git clone
  ├── <session-id>/
  │   └── <project-name>/
  └── ...
```

### 1.2 세션 생성 시

jabis-maker가 세션을 생성할 때 다음을 자동 수행합니다:

```bash
# 1. 세션 디렉토리 생성 + 프로젝트 clone
git clone <repo-url> .sessions/<session-id>/<project-name>/

# 2. submodule 초기화 (jabis-common 등)
cd .sessions/<session-id>/<project-name>/
git submodule update --init --recursive

# 3. 최신 submodule 반영
git submodule update --remote packages
```

### 1.3 원본 디렉토리 보호

- `/home/jinhak/Projects/<project>/` 원본 디렉토리는 **NightExecutor 전용**
- 일반 세션은 원본에 직접 접근하지 않음
- NightExecutor는 기존 방식 유지 (원본 디렉토리 + 글로벌 잠금)

---

## 2. 머지 충돌 처리

세션 디렉토리에서 작업하면 파일 충돌은 발생하지 않지만, **push 시점에 git 충돌**이 발생할 수 있습니다.

### 2.1 기본 전략

```
push 실패 (non-fast-forward)
  → git pull --rebase
  → 충돌 발생 시 Claude가 코드를 이해하고 해결
  → 재 push
```

### 2.2 충돌 해결 불가 시

- Claude가 자동 해결할 수 없는 복잡한 충돌 → 사용자에게 알림
- 세션을 중단하고 수동 개입 요청

---

## 3. 프로젝트 간 의존성 (submodule 동기화)

jabis-common을 submodule로 사용하는 소비 프로젝트(jabis, jabis-hr, jabis-dev 등)는 최신 상태 동기화가 필요합니다.

### 3.1 동기화 시점

| 시점 | 동작 |
|------|------|
| 세션 생성 시 | `git submodule update --init --recursive && git submodule update --remote packages` |
| push 직전 | `git submodule update --remote packages && git add packages` (변경 있으면 자동 커밋) |

### 3.2 push 직전 자동 동기화

```bash
cd .sessions/<session-id>/<project-name>/
git submodule update --remote packages
git add packages
# submodule이 변경되었으면 커밋
git diff --cached --quiet packages || git commit -m "chore: sync submodule"
git push origin master
```

### 3.3 원칙

- 각 세션은 **자기 시작 시점 + push 직전**에 최신화
- 작업 도중 jabis-common이 변경되어도 중간 동기화는 하지 않음
- push 직전 동기화로 최종 반영

---

## 4. 세션 종료 및 정리

### 4.1 정상 종료

- 작업 완료 → push → 세션 디렉토리 삭제

### 4.2 비정상 종료 (세션 사망)

- **감지**: WebSocket `close` 이벤트
- **처리**: 세션 디렉토리 삭제 (uncommitted 변경 포함)
- **heartbeat 사용 안 함** — WebSocket 연결 상태로만 판단

```js
ws.on('close', () => {
  cleanupSessionDirectory(sessionId);
});
```

### 4.3 jabis-maker 재시작 시

- jabis-maker가 재시작되면 모든 WebSocket 연결이 끊어짐
- **시작 시 `.sessions/` 디렉토리 전체 삭제**

```js
async function onStartup() {
  await fs.rm('.sessions/', { recursive: true, force: true });
  await fs.mkdir('.sessions/');
}
```

---

## 5. 현황 모니터링

### 5.1 세션 메타데이터

jabis-maker 세션에 다음 정보를 추가로 관리합니다:

| 필드 | 설명 |
|------|------|
| `projectName` | 작업 대상 프로젝트 |
| `sessionDir` | 세션 디렉토리 경로 |
| `startedAt` | 작업 시작 시간 |
| `status` | 활성/유휴/종료 |

### 5.2 조회 API

`GET /api/sessions` 응답에 세션 디렉토리 정보 포함:

```json
{
  "sessions": [
    {
      "id": "session-abc",
      "projectName": "jabis-common",
      "sessionDir": ".sessions/session-abc/jabis-common",
      "startedAt": "2026-03-11T14:30:00Z",
      "status": "active"
    }
  ]
}
```

---

## 6. NightExecutor 연동

### 6.1 NightExecutor는 현행 유지

- 원본 디렉토리(`/home/jinhak/Projects/`)에서 직접 작업
- 글로벌 잠금 (한 번에 하나만 실행)
- 세션 디렉토리 방식 미적용

### 6.2 동시 실행 주의

- NightExecutor 실행 중 일반 세션이 같은 프로젝트를 작업하는 것은 허용
- 일반 세션은 세션 디렉토리에서 작업하므로 NightExecutor와 파일 충돌 없음
- push 시점의 git 충돌은 2절(머지 충돌 처리)에 따라 해결

---

## 7. 구현 체크리스트

jabis-maker에 구현해야 할 항목:

- [x] 세션 생성 시 `.sessions/<session-id>/<project>/` 디렉토리 자동 생성 (git clone)
- [x] submodule 초기화 + 최신화 자동 실행
- [x] push 직전 submodule 재동기화 로직 (pre-push hook 자동 설치)
- [x] push 실패 시 rebase + 재시도 — Claude Code가 자체 판단하여 처리 (별도 구현 불필요)
- [x] WebSocket `close` 시 세션 디렉토리 삭제 (killSession 경유)
- [x] jabis-maker 시작 시 `.sessions/` 전체 정리
- [x] 세션 메타데이터 (projectName, sessionDir, startedAt) 관리
- [x] `GET /api/sessions` 응답에 세션 디렉토리 정보 포함

---

## 참조 문서

| 문서 | 설명 |
|------|------|
| `projects/jabis-maker.md` | jabis-maker 프로젝트 문서 |
| `policies/night-mode.md` | NightExecutor 정책 |
| `policies/git-workflow.md` | Git 워크플로우 정책 |
| `policies/ai-collaboration.md` | AI 협업 정책 |
