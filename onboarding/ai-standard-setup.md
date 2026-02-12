# JABIS AI 표준 세팅 가이드

> 새 프로젝트 또는 기존 프로젝트에 JINHAK AI 개발 표준을 적용하는 가이드입니다.
> 적용 후 Claude Code와의 협업 품질, 보안, 일관성이 확보됩니다.

---

## 1. 표준 저장소

- **URL**: https://github.com/JinhakStandard/ai-vibecoding
- **현재 버전**: v1.6
- **변경 이력**: 저장소의 `CHANGELOG.md` 참조

---

## 2. 적용 방법

### 방법 0: 글로벌 Hook 설치 (최초 1회, 권장)

글로벌 Hook을 설치하면 **모든 프로젝트**에서 Claude Code 세션 시작 시 표준 적용 여부가 자동 감지됩니다.

```bash
# 표준 저장소 클론
git clone https://github.com/JinhakStandard/ai-vibecoding.git /tmp/jinhak-standards

# 글로벌 Hook 설치
node /tmp/jinhak-standards/scripts/install-global-hook.js

# 제거 시
node /tmp/jinhak-standards/scripts/install-global-hook.js --remove
```

설치 후 아무 프로젝트에서 Claude Code를 열면:

| 상황 | 자동 안내 |
|------|---------|
| 표준 미적용 프로젝트 | `/apply-standard` 안내 표시 |
| 표준 적용된 프로젝트 | `/session-start` 안내 표시 |
| CLAUDE.md는 있으나 메타정보 없음 | `/apply-standard` 안내 표시 |

**동작 원리:**
- `~/.claude/settings.json`에 `UserPromptSubmit` hook 추가
- `once: true`로 세션당 1회만 실행 (토큰 절약)
- 기존 설정을 보존하면서 추가, 백업 자동 생성

### 방법 1: 빠른 적용 프롬프트 (권장)

Claude Code에서 아래 프롬프트를 **그대로 복사-붙여넣기**합니다:

```
이 프로젝트에 JINHAK AI 개발 표준을 적용해줘.

표준 저장소: https://github.com/JinhakStandard/ai-vibecoding (master 브랜치)

## 적용 방법

아래 순서대로 진행해줘. 단, 레포의 각 파일을 웹에서 하나씩 가져오지 말고,
먼저 레포 전체를 로컬에 클론한 다음 로컬 파일을 참고해서 작업해.

### 0단계: 표준 레포 로컬 클론
git clone https://github.com/JinhakStandard/ai-vibecoding.git /tmp/jinhak-standards
만약 이미 /tmp/jinhak-standards가 있으면 git pull로 최신화만 해.
참고: /tmp/jinhak-standards는 참고용일 뿐, 현재 프로젝트의 git에 포함시키지 마.

### 1단계: 현재 프로젝트 분석
- package.json, tsconfig.json, build 설정 등을 읽어서 기술 스택 파악
- 이미 CLAUDE.md나 .claude/ 폴더가 있는지 확인
- 이미 적용된 경우 jinhak_standard_version을 비교해서 업데이트만 수행

### 2단계: CLAUDE.md 생성
- /tmp/jinhak-standards/templates/project-claude.md 템플릿을 기반으로 생성
- /tmp/jinhak-standards/CLAUDE.md 의 핵심 원칙을 포함
- 1단계에서 파악한 프로젝트 정보를 채워넣기

### 3단계: .ai/ 폴더 생성
mkdir -p .ai
- .ai/SESSION_LOG.md, CURRENT_SPRINT.md, DECISIONS.md, ARCHITECTURE.md, CONVENTIONS.md 생성

### 4단계: .claude/ 설정 복사
- /tmp/jinhak-standards/.claude/settings.json → .claude/settings.json
- /tmp/jinhak-standards/.claude/skills/ 폴더 전체를 .claude/skills/ 로 복사

### 5단계: .gitignore 업데이트
CLAUDE.local.md, .claude/settings.local.json, .env, .env.* 추가

### 6단계: 적용 결과 요약
```

> 신규/기존 프로젝트 모두 동일한 프롬프트를 사용합니다.
> 두 번째 프로젝트부터는 `/tmp/jinhak-standards`가 이미 있으므로 더 빠릅니다.

### 방법 2: /apply-standard 스킬

표준이 이미 적용된 프로젝트에서는 스킬로 바로 실행할 수 있습니다:

```
/apply-standard
```

### 방법 3: URL 전달

```
https://github.com/JinhakStandard/ai-vibecoding 여기를 참고해서 프로젝트에 적용해줘
```

---

## 3. 적용 후 생성/수정되는 파일

| 파일 | 역할 |
|------|------|
| `CLAUDE.md` | 프로젝트 AI 설정 파일 (기술 스택, 규칙, 메타정보) |
| `.ai/SESSION_LOG.md` | 세션별 작업 기록 |
| `.ai/CURRENT_SPRINT.md` | 현재 진행/대기 작업 현황 |
| `.ai/DECISIONS.md` | 기술 의사결정 기록 (ADR) |
| `.ai/ARCHITECTURE.md` | 프로젝트 아키텍처 |
| `.ai/CONVENTIONS.md` | 프로젝트별 코딩 컨벤션 |
| `.claude/settings.json` | 권한(allow/deny), hooks, env 설정 |
| `.claude/skills/*/SKILL.md` | 슬래시 명령어 5개 |
| `.gitignore` | CLAUDE.local.md, .env 등 추가 |

---

## 4. 적용 후 검증 체크리스트

### 파일 생성 검증

- [ ] `CLAUDE.md`에 프로젝트 정보가 정확한가
- [ ] `CLAUDE.md`에 `jinhak_standard_version` 메타정보가 있는가
- [ ] `.ai/` 폴더와 5개 파일이 모두 생성되었는가
- [ ] `.claude/settings.json`이 올바르게 설정되었는가
- [ ] `.claude/skills/`에 5개 스킬이 모두 있는가
- [ ] `.gitignore`에 `CLAUDE.local.md`, `.claude/settings.local.json`, `.env`가 포함되었는가

### 기능 동작 검증

- [ ] `/session-start` 명령이 정상 실행되는가
- [ ] UserPromptSubmit hook이 CURRENT_SPRINT.md를 출력하는가
- [ ] PreToolUse hook이 파일 수정 시 보안 경고를 출력하는가

### 보안 검증

- [ ] deny 규칙에 `--no-verify`, `push --force`, `reset --hard`, `clean -f`, `rm -rf`, `git config` 포함되었는가
- [ ] `.env` 파일이 `.gitignore`에 포함되었는가

---

## 5. 적용 후 사용 가능한 명령어

```
/session-start          # 세션 시작 (이전 작업 확인 + 표준 버전 체크)
/commit                 # 표준에 맞는 커밋 생성
/review-pr 123          # PR을 표준 기준으로 리뷰
/test                   # 테스트 실행 및 결과 분석
/apply-standard         # 표준 재적용/업데이트
```

---

## 6. 표준 업데이트

### 자동 감지

`/session-start` 실행 시 자동으로 업데이트 여부를 확인합니다:

1. 프로젝트 `CLAUDE.md`의 `jinhak_standard_version` 확인
2. 표준 저장소의 `CHANGELOG.md`에서 최신 버전 확인
3. 새 버전이 있으면 변경 내역을 요약하여 안내
4. 사용자 승인 시 업데이트 적용

### 수동 업데이트

```
/apply-standard
```

업데이트 시 변경되는 항목:
- `CLAUDE.md`의 변경된 규칙 반영
- 새로 추가된 스킬 파일 복사
- `settings.json` 규칙 업데이트
- `jinhak_standard_version` 메타정보 업데이트

---

## 7. JABIS 프로젝트별 적용 현황

> 각 프로젝트의 `CLAUDE.md` 내 `jinhak_standard_version`으로 확인 가능합니다.

| 프로젝트 | 표준 적용 | 비고 |
|---------|----------|------|
| jabis | O | 메인 프론트엔드 |
| jabis-api-gateway | O | API 게이트웨이 |
| jabis-cert | O | 인증 서버 |
| jabis-common | O | 공유 라이브러리 |
| jabis-night-builder | O | AI 자동화 |
| jabis-bitbucket-sync | O | Bitbucket 동기화 |
| jabis-emergency-console | O | 긴급 콘솔 |
| jabis-helm | - | Helm Charts (코드 프로젝트 아님) |
| jabis-design-system | O | 디자인 시스템 |
| jabis-design-package | O | 디자인 패키지 |
| jabis-dev | O | 개발 유틸리티 |
| jabis-producer | O | 프로듀서 앱 |
| jabis-lab | O | 실험 환경 |
| jabis-template | O | 프로젝트 템플릿 |

---

## 참조

- [AI 협업 정책](../policies/ai-collaboration.md) — Claude Code 설정 상세
- [코딩 스타일 정책](../policies/coding-style.md) — 코드 품질 규칙
- [보안 정책](../policies/security.md) — 시크릿 관리, ISMS 보안
- [전사 AI 표준 저장소](https://github.com/JinhakStandard/ai-vibecoding) — 원본 표준 문서
