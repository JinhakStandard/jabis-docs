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

| 브랜치 | 용도 | 비고 |
|--------|------|------|
| `main` | 운영 배포 | Pipeline 자동 트리거 |
| `alpha` | 알파 배포 | Pipeline 자동 트리거 |
| `feature/*` | 기능 개발 | main에서 분기 |
| `fix/*` | 버그 수정 | main에서 분기 |

---

## 3. 규칙

### 커밋 생성 절차 (AI 협업 시)

Claude Code에서 `/commit` 스킬을 사용하면 아래 절차가 자동 수행됩니다:

1. `git status`로 변경/추가/삭제된 파일 확인
2. `git diff`로 변경 내용 상세 분석
3. `git log --oneline -5`로 최근 커밋 스타일 확인
4. 관련 파일만 개별 스테이징 (`git add -A` 지양)
5. HEREDOC 형식으로 커밋 메시지 작성
6. `git status`로 커밋 성공 확인
7. `.ai/CURRENT_SPRINT.md` 및 `.ai/SESSION_LOG.md` 업데이트

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

> 위 금지 명령은 `.claude/settings.json`의 `deny` 규칙으로 **물리적 차단**됩니다. 개인 설정으로 우회 불가합니다.

---

## 4. Bitbucket Pipeline 패턴

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
> 모든 jabis 프로젝트(jabis, jabis-dev, jabis-producer, jabis-hr 등)에서 동일한 토큰을 직접 기입합니다.
> 신규 프로젝트 생성 시 기존 프로젝트의 `bitbucket-pipelines.yml`에서 토큰 값을 복사하여 사용하세요.

### jabis-helm Clone 토큰

Pipeline의 Update Helm 단계에서 `jabis-helm`을 clone할 때도 동일하게 **토큰을 YAML에 직접 기입**하는 방식을 사용합니다.

```yaml
# Update Helm step 내부
- git clone https://x-token-auth:{토큰}@bitbucket.org/jinhaksa/jabis-helm.git
- git remote set-url origin https://x-token-auth:{토큰}@bitbucket.org/jinhaksa/jabis-helm.git
```

> **주의**: `${JABIS_HELM_TOKEN}` 같은 환경변수 참조 방식을 사용하지 않습니다.
> jabis-common과 마찬가지로 모든 프로젝트에서 동일한 토큰을 직접 기입합니다.
> `git clone`과 `git remote set-url origin` 두 곳 모두 토큰을 기입해야 합니다.

> **App Password 폐지 예정**: Bitbucket이 2026-06-09에 기존 App Password를 비활성화합니다. 그 전에 API Token으로 교체가 필요합니다.

### Teams Webhook URL (배포 알림)

Pipeline의 Notify Teams 단계에서 Microsoft Teams로 배포 완료 알림을 보낼 때, **Webhook URL을 YAML에 직접 기입**하는 방식을 사용합니다.

```yaml
# Notify Teams step 내부
curl -H "Content-Type: application/json" -d '{...}' "{Webhook URL}"
```

> **주의**: `$TEAMS_WEBHOOK_URL` 같은 환경변수 참조 방식을 사용하지 않습니다.
> 모든 jabis 프로젝트에서 동일한 Webhook URL을 직접 기입합니다.
> 신규 프로젝트 생성 시 기존 프로젝트의 `bitbucket-pipelines.yml`에서 URL을 복사하여 사용하세요.
> Webhook URL이 변경될 경우, 전체 프로젝트의 `bitbucket-pipelines.yml`을 일괄 수정해야 합니다.
