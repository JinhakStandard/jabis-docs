# JABIS Git 워크플로우

> 이 문서는 JABIS 전체 프로젝트에 적용되는 Git 워크플로우 정책입니다.

---

## 1. 커밋 메시지

Conventional Commits 형식을 따릅니다. 본문은 한국어 허용.

| Prefix | 용도 | 예시 |
|--------|------|------|
| `feat:` | 새 기능 | `feat: 대시보드 위젯 추가` |
| `fix:` | 버그 수정 | `fix: OAuth 리다이렉트 오류 수정` |
| `refactor:` | 리팩토링 | `refactor: proxy 기능 제거` |
| `chore:` | 빌드/설정 변경 | `chore: common-helm 버전 업그레이드` |
| `remove:` | 코드/파일 삭제 | `remove: 미사용 pay 모듈 삭제` |
| `config:` | 환경설정 변경 | `config: 환경별 SMTP 설정 추가` |
| `docs:` | 문서 변경 | `docs: API 명세 업데이트` |

### Co-Authored-By
AI와 함께 작업한 커밋에는 다음을 추가합니다:
```
Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## 2. 브랜치 전략

### 2.1 브랜치 종류

| 브랜치 | 용도 | Pipeline |
|--------|------|----------|
| `main` | 운영 배포 | push 시 자동 트리거 |
| `alpha` | Alpha(스테이징) 배포 | push 시 자동 트리거 |

> feature 브랜치는 사용하지 않습니다. **alpha 브랜치에서 직접 작업**합니다.

---

## 3. 배포 플로우

### 3.1 일반 배포 (권장)

```
alpha 브랜치에서 작업 → 커밋 → push
→ Bitbucket Pipeline → Alpha 환경 자동 배포
→ Alpha 환경에서 테스트/검증 (*-alpha.jinhakapply.com)
→ alpha → main merge + push
→ Bitbucket Pipeline → 운영 환경 자동 배포
```

**절차:**

```bash
# 1. alpha 브랜치에서 작업
git checkout alpha && git pull
# 작업 수행...
git add {파일들}
git commit -m "feat: ..."
git push origin alpha
# → Alpha 환경에 자동 배포 (약 5분 소요)

# 2. Alpha 환경에서 테스트
# https://*-alpha.jinhakapply.com 접속하여 확인

# 3. 검증 완료 → 운영 배포
git checkout main && git pull
git merge alpha
git push origin main
# → 운영 환경에 자동 배포 (약 5분 소요)

# 4. alpha 브랜치로 복귀
git checkout alpha
```

### 3.2 긴급 핫픽스

운영에서 긴급 수정이 필요하고 Alpha 검증을 거칠 시간이 없을 때:

```bash
# 1. main에서 직접 수정
git checkout main && git pull
# 긴급 수정...
git add {파일들}
git commit -m "fix: ..."
git push origin main
# → 운영 환경에 즉시 배포

# 2. alpha에 역동기화 (필수)
git checkout alpha && git pull
git merge main
git push origin alpha
```

> **핫픽스 후 반드시 alpha에 역merge** 해야 합니다. 안 하면 다음 alpha → main merge 시 충돌이 발생합니다.

### 3.3 jabis-common submodule 동기화

jabis-common은 **master 브랜치만 사용**합니다 (Pipeline이 없는 순수 소스 저장소).

```bash
# 1. jabis-common에서 수정 → master push
cd jabis-common
# 작업...
git add . && git commit -m "feat: ..." && git push

# 2. 소비 프로젝트의 alpha 브랜치에서 submodule 업데이트
cd jabis-hr  # (또는 jabis, jabis-dev, jabis-producer 등)
git checkout alpha && git pull
git submodule update --remote packages
git add packages
git commit -m "chore: jabis-common submodule 업데이트"
git push origin alpha
# → Alpha 환경 배포 → 테스트

# 3. 검증 후 운영 배포
git checkout main && git pull
git merge alpha
git push origin main
```

**핵심:**
- submodule 참조 커밋이 잠금장치 역할 — main 브랜치는 merge 전까지 이전 커밋 참조 유지
- alpha → main merge 시 submodule 참조 커밋도 함께 merge됨

### 3.4 NightExecutor 배포

NightExecutor는 **alpha 브랜치**에서 작업합니다.

```
NightExecutor가 alpha에서 작업 → 커밋 → push
→ Alpha 환경 자동 배포
→ 다음 날 아침 사용자가 Alpha에서 확인
→ 문제 없으면 자비스 패키지에서 alpha → main 승격
```

### 3.5 자비스 패키지 (배포 관리 도구)

alpha → main 승격은 **자비스 패키지** UI에서 수행합니다.

- 프로젝트별 alpha/main 커밋 비교 시각화
- "운영 배포" 버튼 → alpha → main merge + push
- 핫픽스 후 main → alpha 역동기화
- 접근: jabis-maker의 `/package/` 경로

> **상세**: `projects/jabis-maker.md`의 "자비스 패키지" 절 참조.

---

## 4. 규칙

### 커밋 생성 절차 (AI 협업 시)

1. `git status`로 변경/추가/삭제된 파일 확인
2. `git diff`로 변경 내용 상세 분석
3. `git log --oneline -5`로 최근 커밋 스타일 확인
4. 관련 파일만 개별 스테이징 (`git add -A` 지양)
5. HEREDOC 형식으로 커밋 메시지 작성
6. `git status`로 커밋 성공 확인

### 필수
- 커밋 전 반드시 `git diff`로 변경 내용 확인
- 민감 정보(`.env`, credentials) 커밋 금지
- Teams 배포 알림 webhook step 포함 (`bitbucket-pipelines.yml`)
- pre-commit hook 실패 시 **새 커밋** 생성 (`--amend` 금지)

### 금지
- `git push --force` 금지 (특히 main/master)
- `git reset --hard` 금지 (확인 없이)
- `--no-verify` 옵션 금지
- `git config` 직접 수정 금지

> 위 금지 명령은 `.claude/settings.json`의 `deny` 규칙으로 **물리적 차단**됩니다.

---

## 5. Bitbucket Pipeline 패턴

JABIS 프로젝트는 두 가지 Pipeline 패턴을 사용합니다:

### Pattern A: Import 방식 (권장)
```yaml
# definitions에서 정의 후 import
definitions:
  pipelines:
    deploy-pipeline:
      - step: *set-common-variables
      - step: *set-prod-variables
      - step: *build-docker-image
      - step: *update-helm
      - step: *notify-teams

pipelines:
  branches:
    main:
      import: {repo-name}:main:deploy-pipeline
    alpha:
      import: {repo-name}:alpha:deploy-alpha-pipeline
```

### Pattern B: Direct Reference 방식
```yaml
pipelines:
  branches:
    main:
      - step: *set-common-variables
      - step: *set-prod-variables
      - step: *build-docker-image
      - step: *update-helm
      - step: *notify-teams
    alpha:
      - step: *set-common-variables
      - step: *set-alpha-variables
      - step: *build-docker-image
      - step: *update-helm
      - step: *notify-teams
```

### 공통 Step 구성
1. **Set Common Variables** — REPO_SLUG, NAMESPACE, BASE_PACKAGE_VERSION
2. **Set Environment Variables** — HARBOR_DOMAIN, TAG, IMAGE_NAME
3. **Build Docker Image** — jabis-common clone + Docker build + Harbor push
4. **Update Helm** — jabis-helm clone + values 업데이트 + Helm package + push
5. **Notify Teams** — 배포 완료 알림 (Adaptive Card)

### jabis-common Clone 토큰

Pipeline의 Build Docker Image 단계에서 `jabis-common`을 clone할 때, **Bitbucket Repository Access Token을 YAML에 직접 기입**하는 방식을 사용합니다.

```yaml
# Build Docker Image step 내부
- rm -rf packages
- apk add --no-cache git
- git clone https://x-token-auth:{토큰}@bitbucket.org/jinhaksa/jabis-common.git packages
```

> **주의**: `${JABIS_COMMON_TOKEN}` 같은 환경변수 참조 방식을 사용하지 않습니다.
> 모든 jabis 프로젝트에서 동일한 토큰을 직접 기입합니다.
> 신규 프로젝트 생성 시 기존 프로젝트의 `bitbucket-pipelines.yml`에서 토큰 값을 복사하여 사용하세요.

### jabis-helm Clone 토큰

Pipeline의 Update Helm 단계에서 `jabis-helm`을 clone할 때도 동일하게 **토큰을 YAML에 직접 기입**하는 방식을 사용합니다.

```yaml
# Update Helm step 내부
- git clone https://x-token-auth:{토큰}@bitbucket.org/jinhaksa/jabis-helm.git
- git remote set-url origin https://x-token-auth:{토큰}@bitbucket.org/jinhaksa/jabis-helm.git
```

> **주의**: `${JABIS_HELM_TOKEN}` 같은 환경변수 참조 방식을 사용하지 않습니다.
> `git clone`과 `git remote set-url origin` 두 곳 모두 토큰을 기입해야 합니다.

> **App Password 폐지 예정**: Bitbucket이 2026-06-09에 기존 App Password를 비활성화합니다. 그 전에 API Token으로 교체가 필요합니다.

### Teams Webhook URL (배포 알림)

Pipeline의 Notify Teams 단계에서 Microsoft Teams로 배포 완료 알림을 보낼 때, **Webhook URL을 YAML에 직접 기입**하는 방식을 사용합니다.

```yaml
# Notify Teams step 내부
curl -H "Content-Type: application/json" -d '{...}' "{Webhook URL}"
```

> **주의**: `$TEAMS_WEBHOOK_URL` 같은 환경변수 참조 방식을 사용하지 않습니다.
> 신규 프로젝트 생성 시 기존 프로젝트의 `bitbucket-pipelines.yml`에서 URL을 복사하여 사용하세요.
