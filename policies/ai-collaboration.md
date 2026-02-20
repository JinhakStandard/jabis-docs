# JABIS AI 협업 정책

> 이 문서는 JABIS 프로젝트에서 Claude Code와 협업할 때 따라야 하는 정책입니다.
> 전사 AI 개발 표준(ai-vibecoding)을 JABIS 에코시스템에 맞게 적용한 내용입니다.

---

## 1. 전사 AI 개발 표준

JABIS 전체 프로젝트는 **JINHAK AI 개발 표준 v1.6**을 따릅니다.

- **표준 저장소**: https://github.com/JinhakStandard/ai-vibecoding
- **표준 문서**: `CLAUDE.md` (전사 핵심 원칙), `VIBE_CODING_GUIDE.md` (방법론)
- **버전 관리**: 각 프로젝트의 `CLAUDE.md`에 메타정보로 적용 버전이 기록됨

### 1.1 적용 버전 메타정보

각 프로젝트의 `CLAUDE.md` 하단에 HTML 주석으로 다음이 포함됩니다:

```html
<!-- jinhak_standard_version: 1.6 -->
<!-- jinhak_standard_repo: https://github.com/JinhakStandard/ai-vibecoding -->
<!-- applied_date: 2026-02-12 -->
```

`/session-start` 실행 시 표준 저장소의 `CHANGELOG.md`와 비교하여 업데이트 여부를 자동 안내합니다.

---

## 2. 프로젝트 필수 구조

### 2.1 Claude Code 설정 파일

모든 JABIS 프로젝트는 다음 구조를 갖추어야 합니다:

```
프로젝트루트/
├── CLAUDE.md              # Claude Code 메인 설정 파일 (필수)
├── CLAUDE.local.md        # 로컬 개발자 설정 (git 제외, 선택)
├── .claude/               # Claude Code 설정 폴더
│   ├── settings.json      # 권한, 환경변수, hooks 설정
│   ├── settings.local.json # 로컬 설정 (git 제외)
│   └── skills/            # 슬래시 명령어 정의
│       ├── apply-standard/SKILL.md
│       ├── commit/SKILL.md
│       ├── review-pr/SKILL.md
│       ├── session-start/SKILL.md
│       └── test/SKILL.md
└── .ai/                   # 세션 관리 폴더
    ├── SESSION_LOG.md     # 세션별 작업 기록
    ├── CURRENT_SPRINT.md  # 현재 진행/대기 작업
    ├── DECISIONS.md       # 기술 의사결정 기록 (ADR)
    ├── ARCHITECTURE.md    # 시스템 아키텍처
    └── CONVENTIONS.md     # 코딩 컨벤션
```

### 2.2 .gitignore 필수 항목

```
CLAUDE.local.md
.claude/settings.local.json
.env
.env.*
```

### 2.3 CLAUDE.local.md 오버라이드

개인 개발자가 로컬 환경에 맞게 Claude Code 동작을 조정할 수 있습니다.

**우선순위:**
```
전사 표준 CLAUDE.md < 프로젝트 CLAUDE.md < CLAUDE.local.md
```

**오버라이드 불가 항목:**
- 보안 규칙 (deny 규칙, ISMS 보안)
- 핵심 품질 규칙 (`any` 타입 금지, `console.log` 금지 등)

---

## 3. Claude Code 권한 설정 (settings.json)

### 3.1 표준 permissions

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(pnpm *)",
      "Bash(npx *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git add *)",
      "Bash(git * commit *)",
      "Bash(git commit *)",
      "Bash(git * push *)",
      "Bash(git push *)",
      "Bash(git checkout *)",
      "Bash(git branch *)",
      "Bash(git fetch *)",
      "Bash(ls *)",
      "Bash(mkdir *)",
      "Read", "Glob", "Grep"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(git clean -f*)",
      "Bash(git config *)",
      "Bash(*--no-verify*)"
    ]
  }
}
```

### 3.2 deny 규칙 설명

| deny 규칙 | 차단 대상 | 이유 |
|-----------|----------|------|
| `rm -rf *` | 재귀 강제 삭제 | 복구 불가능한 파일 손실 |
| `git push --force*` | 강제 푸시 | 원격 히스토리 덮어쓰기, 팀원 작업 손실 |
| `git reset --hard*` | 하드 리셋 | 커밋되지 않은 변경사항 영구 삭제 |
| `git clean -f*` | 추적 안 되는 파일 강제 삭제 | 의도치 않은 파일 삭제 |
| `git config *` | Git 설정 변경 | 사용자 정보, hooks 경로 등 임의 변경 방지 |
| `*--no-verify*` | Hook 건너뛰기 | pre-commit, pre-push 등 품질 검증 우회 방지 |

> **중요**: `deny` 규칙은 프로젝트 전체에 **강제 적용**됩니다. `settings.local.json`이나 `~/.claude/settings.json`으로 우회할 수 없습니다. `deny`가 `allow`보다 우선합니다.

---

## 4. Hooks 시스템

`.claude/settings.json`에서 이벤트 기반 자동화를 설정합니다.

### 4.1 Hook 이벤트

| 이벤트 | 설명 | 주요 변수 |
|--------|------|----------|
| `UserPromptSubmit` | 사용자 프롬프트 제출 시 | - |
| `PreToolUse` | 도구 실행 전 | `${tool}`, `${command}`, `${file}` |
| `PostToolUse` | 도구 실행 후 | `${tool}`, `${file}` |
| `Stop` | Claude 응답 완료 시 | - |
| `SubagentStart` | 서브에이전트 생성 시 | - |

### 4.2 JABIS 표준 Hooks 설정

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat .ai/CURRENT_SPRINT.md 2>/dev/null | head -50 || echo ''",
            "once": true
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo [SECURITY] 파일 수정 감지: ${file}"
          }
        ]
      }
    ],
    "SubagentStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo [SECURITY] 서브에이전트 시작 - deny 규칙이 상속됩니다"
          }
        ]
      }
    ]
  }
}
```

**Hook 동작 설명:**

| Hook | 동작 | 목적 |
|------|------|------|
| UserPromptSubmit (`once: true`) | 세션 시작 시 1회, `CURRENT_SPRINT.md` 내용을 Claude에게 주입 | 진행 중인 작업 컨텍스트 자동 로드 |
| PreToolUse (Edit\|Write) | 파일 수정 직전 보안 경고 출력 | 파일 변경 감사 추적 |
| SubagentStart | 서브에이전트 시작 시 알림 | deny 규칙 상속 확인 |

> `once: true` 옵션은 세션 중 1회만 실행하여 토큰을 절약합니다.

---

## 5. Skills (슬래시 명령어)

`.claude/skills/` 디렉토리에 반복 작업을 자동화하는 명령어를 정의합니다.

| 명령어 | 용도 | 설명 |
|--------|------|------|
| `/apply-standard` | 표준 적용/업데이트 | ai-vibecoding 표준을 프로젝트에 적용 |
| `/session-start` | 세션 시작 | `.ai/` 파일 읽기 → 표준 버전 체크 → 브리핑 |
| `/commit` | 커밋 생성 | 변경사항 분석 → 표준 형식 커밋 → `.ai/` 업데이트 |
| `/review-pr <번호>` | PR 리뷰 | 표준 기준으로 코드 품질/보안 리뷰 |
| `/test` | 테스트 실행 | 테스트 실행 + 타입 체크 + 결과 분석 |

### 5.1 세션 워크플로우

```
세션 시작 → /session-start (브리핑)
    ↓
feature 브랜치 생성 (feature/{작업명})
    ↓
작업 수행 (hooks가 자동으로 보안 감시)
    ↓
커밋 → /commit (표준 형식 + .ai/ 자동 업데이트)
    ↓
feature 브랜치 push → Bitbucket PR 생성 (API)
    ↓
navskh 리뷰/승인 → Merge → 자동 배포
```

> **중요**: main에 직접 push하지 않습니다. 모든 작업은 feature 브랜치 → PR → 승인 → merge 플로우를 따릅니다.
> 상세 정책은 `policies/git-workflow.md`의 "3. PR 워크플로우" 및 "4. Claude Code PR 자동 생성" 참조.

---

## 6. 세션 관리 규칙

### 6.1 Agent Memory 활용

Claude Code는 프로젝트별 메모리를 자동으로 기록/회상합니다. `.ai/` 파일은 Agent Memory의 보조 수단으로, **팀원 간 공유와 Git 추적이 필요한 핵심 사항**만 기록합니다.

### 6.2 세션 시작 시

1. `.ai/CURRENT_SPRINT.md` 읽어 진행 중인 작업 파악
2. `.ai/SESSION_LOG.md`에서 최근 작업 확인
3. UserPromptSubmit hook이 자동으로 CURRENT_SPRINT.md를 Claude에게 주입

### 6.3 작업 완료 후 (필수)

1. `.ai/CURRENT_SPRINT.md` 진행 상태 업데이트
2. `.ai/SESSION_LOG.md`에 요약 기록 (핵심 변경사항 위주)
3. 중요 기술 결정 시 `.ai/DECISIONS.md` 업데이트

### 6.4 SESSION_LOG.md 기록 형식

```markdown
## YYYY-MM-DD

### 세션 요약
- 작업 1 설명
- 작업 2 설명

### 주요 변경
- `파일경로` - 변경 내용

### 커밋
- `해시` 커밋 메시지
```

---

## 7. 3중 보안 방어 구조

AI 협업 시 보안을 3개 레이어로 방어합니다:

```
┌─────────────────────────────────────────────┐
│ 1층: CLAUDE.md 자연어 규칙                    │
│   → 안티패턴 감지 + 경고 + 대안 제시          │
├─────────────────────────────────────────────┤
│ 2층: settings.json deny 규칙 (하드 블로킹)    │
│   → rm -rf, push --force, --no-verify 등     │
│     물리적 차단 (우회 불가)                    │
├─────────────────────────────────────────────┤
│ 3층: hooks 보조 경고 (소프트 알림)             │
│   → 파일 수정 시 보안 경고 주입               │
│   → 서브에이전트 시작 시 deny 규칙 상속 알림   │
└─────────────────────────────────────────────┘
```

### 7.1 AI 안티패턴 자동 감지

Claude는 다음 안티패턴을 감지하면 즉시 경고하고 대안을 제시합니다:

| 분류 | 안티패턴 | Claude 대응 |
|------|---------|------------|
| 위험한 요청 | `push --force`, `reset --hard`, `--no-verify` | 실행 거부 + 안전한 대안 제시 |
| 위험한 요청 | 프로덕션 DB 직접 조작 요청 | 거부 + 스테이징 환경 사용 안내 |
| 민감 정보 | 프롬프트에 비밀번호, API Key, 개인정보 포함 | 경고 + 마스킹/가명화 요청 |
| 민감 정보 | `.env` 파일 내용 공유 요청 | 거부 + Vault 사용 안내 |
| 품질 저하 | "전체를 처음부터 다시 작성해줘" | 부분 수정 제안 |
| 품질 저하 | 순차 의존성 있는 5개 이상 기능 동시 요청 | 단계별 분할 제안 |

---

## 8. Agent Teams (멀티 에이전트 협업)

여러 에이전트를 팀으로 구성하여 병렬 작업을 수행할 수 있습니다.

### 8.1 활성화

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 8.2 적합한 작업

- 코드베이스 전체 리뷰 (독립적인 파일/모듈별 분석)
- 다중 파일 동시 분석/검색
- 독립적인 기능의 병렬 구현
- 대규모 리팩토링의 사전 조사

### 8.3 부적합한 작업

- 순차 의존성이 높은 작업 (A 결과가 B 입력이 되는 경우)
- 같은 파일을 동시에 수정하는 작업
- 단일 파일의 간단한 수정

### 8.4 Agent Teams 보안

| 항목 | 규칙 |
|------|------|
| deny 규칙 상속 | 서브에이전트는 메인 에이전트의 deny 규칙을 자동 상속 |
| SubagentStart 알림 | 서브에이전트 시작 시 보안 규칙 전파 확인 hook 설정 |
| 데이터 격리 | 서브에이전트 간 민감 데이터 직접 전달 금지 |
| 병렬 작업 범위 | 서브에이전트에게 프로덕션 환경 접근 명령 위임 금지 |

---

## 9. Plan 모드 & Effort 설정

### 9.1 Plan 모드

복잡한 구현 전에 Plan 모드를 활용하여 설계를 먼저 검토합니다:

- 3개 이상 파일 수정이 예상되는 작업
- 아키텍처 결정이 필요한 신규 기능
- `.ai/DECISIONS.md`에 기록할 수준의 기술 결정

### 9.2 Effort 설정

| Effort | 용도 | 비용/속도 |
|--------|------|----------|
| `low` | 간단한 포맷팅, 이름 변경, 오타 수정 | 최저 비용, 최고 속도 |
| `medium` | 일반 CRUD 구현, 단순 컴포넌트 작업 | 균형 |
| `high` (기본) | 복잡한 비즈니스 로직, 버그 분석 | 높은 품질 |
| `max` | 아키텍처 설계, 보안 감사, 복잡한 알고리즘 | 최고 품질 |

---

## 10. AI에게 기대하는 행동

1. **코드를 읽기 전에 수정하지 않는다** — 반드시 기존 코드를 먼저 이해한 후 변경 제안
2. **과도한 엔지니어링을 피한다** — 요청된 범위 내에서만 작업, 불필요한 추상화 금지
3. **보안을 최우선으로 한다** — OWASP Top 10 취약점 방지
4. **기존 패턴을 따른다** — 프로젝트의 기존 컨벤션과 패턴을 존중
5. **한국어로 소통한다** — 모든 응답, 주석 설명, 커밋 메시지는 한국어
6. **jabis-docs를 참조/반영한다** — 작업 전 관련 문서 확인, 변경 시 문서 업데이트

---

## 11. 참조 문서

| 문서 | 위치 | 설명 |
|------|------|------|
| AI 표준 세팅 가이드 | `onboarding/ai-standard-setup.md` | 프로젝트에 표준 적용하는 방법 |
| 코딩 스타일 정책 | `policies/coding-style.md` | 네이밍, HTTP, 로깅 규칙 |
| Git 워크플로우 | `policies/git-workflow.md` | 커밋 규칙, 브랜치 전략 |
| 보안 정책 | `policies/security.md` | 시크릿, 인증, DB 보안 |
| 바이브 코딩 가이드 | `onboarding/vibe-coding-guide.md` | JABIS 바이브 코딩 대전제 |
| 전사 AI 표준 저장소 | https://github.com/JinhakStandard/ai-vibecoding | CLAUDE.md, Skills, templates |
