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

| 브랜치 | 용도 | 비고 |
|--------|------|------|
| `main` | 운영 배포 | Pipeline 자동 트리거, **직접 push 금지** |
| `alpha` | 알파 배포 | Pipeline 자동 트리거 |
| `feature/*` | 기능 개발 | main에서 분기 |
| `fix/*` | 버그 수정 | main에서 분기 |
| `chore/*` | 빌드/설정 변경 | main에서 분기 |
| `refactor/*` | 리팩토링 | main에서 분기 |

### 2.2 브랜치 네이밍 규칙

```
{prefix}/{간결한-설명}
```

| Prefix | 용도 | 예시 |
|--------|------|------|
| `feature/` | 새 기능 | `feature/profile-dialog` |
| `fix/` | 버그 수정 | `fix/oauth-redirect` |
| `chore/` | 빌드/설정 | `chore/helm-upgrade` |
| `refactor/` | 리팩토링 | `refactor/menu-config` |

- 설명은 영문 소문자 + 하이픈 (kebab-case)
- 한국어 사용 금지 (브랜치명에서)

### 2.3 main 브랜치 보호 정책

**모든 JABIS 저장소**에 적용합니다:

| 설정 | 값 | 설명 |
|------|-----|------|
| main 직접 push | **금지** | PR을 통해서만 merge 가능 |
| 최소 승인 수 | 1 (navskh) | 리뷰어가 Approve해야 merge 가능 |
| Default reviewer | navskh | PR 생성 시 자동으로 reviewer 지정 |

**적용 대상 저장소** (전체):

| 저장소 | 비고 |
|--------|------|
| jabis | 메인 프론트엔드 |
| jabis-common | 공유 라이브러리 (submodule 원본) |
| jabis-api-gateway | API 게이트웨이 |
| jabis-cert | 인증 서버 |
| jabis-hr | 인사담당자 프로젝트 |
| jabis-dev | 개발 유틸리티 |
| jabis-producer | 프로듀서 앱 |
| jabis-design-system | 디자인 시스템 쇼케이스 |
| jabis-design-package | @jabis/ui NPM 패키지 |
| jabis-bitbucket-sync | Bitbucket 동기화 |
| jabis-night-builder | AI 자동화 서비스 |
| jabis-emergency-console | DB 긴급 관리 콘솔 |
| jabis-helm | K3S Kubernetes 배포 설정 |
| jabis-lab | 실험 환경 (submodule 불안정 — 예외 가능) |
| jabis-template | 템플릿 (당분간 미사용 — 예외 가능) |
| jabis-maker | 통합 빌드/미리보기 서버 |
| jabis-maker-admin | Maker 관리자 UI |

> **Bitbucket 설정 방법**: 각 저장소 → Repository settings → Branch permissions → Add a branch permission
> - Branch: `main`
> - Write access: "No one" (직접 push 차단)
> - Merge via pull request: 체크
> - Minimum approvals: 1

---

## 3. Pull Request (PR) 워크플로우

### 3.1 기본 플로우

```
main에서 분기 → feature 브랜치 작업 → commit → push
    → Bitbucket PR 생성 → navskh 메일 알림
    → 리뷰/승인 → Merge → main Pipeline 자동 배포
```

### 3.2 PR 규칙

| 항목 | 규칙 |
|------|------|
| PR 제목 | 커밋 prefix와 동일한 형식 (`feat:`, `fix:` 등) |
| 설명 | 변경 내용 요약 + 테스트 방법 |
| Reviewer | navskh (자동 지정) |
| Merge 방식 | **Merge commit** (히스토리 보존) |
| 브랜치 삭제 | Merge 후 feature 브랜치 **자동 삭제** |

### 3.3 PR 제목 형식

```
{prefix}: {설명} ({프로젝트명})
```

예시:
- `feat: 프로필 다이얼로그 추가 (jabis-common)`
- `fix: OAuth 리다이렉트 오류 수정 (jabis-cert)`
- `chore: Helm 차트 버전 업그레이드 (jabis-helm)`

### 3.4 PR 설명(Body) 템플릿

```markdown
## 변경 내용
- (변경사항 요약)

## 수정된 파일
- `경로/파일명` — 변경 내용

## 테스트
- [ ] 로컬 빌드 확인
- [ ] 미리보기 확인 (해당 시)

## 관련 작업
- (관련 이슈나 작업 설명)
```

---

## 4. Claude Code PR 자동 생성

### 4.1 워크플로우

Claude Code가 작업을 완료하면 다음 플로우를 따릅니다:

```
1. feature 브랜치 생성
   git checkout -b feature/{작업명}

2. 작업 수행 + 커밋

3. feature 브랜치에 push
   git push -u origin feature/{작업명}

4. Bitbucket PR 생성 (Bitbucket REST API 사용)
   → navskh가 default reviewer로 자동 지정
   → navskh@naver.com으로 메일 알림 발송

5. 사용자(navskh)가 Bitbucket 웹에서 리뷰 후 Merge
   → main Pipeline 자동 배포
```

### 4.2 Bitbucket PR 생성 API

Claude Code는 다음 curl 명령으로 PR을 생성합니다:

```bash
curl -X POST \
  -u "navskh:{app-password}" \
  -H "Content-Type: application/json" \
  "https://api.bitbucket.org/2.0/repositories/jinhaksa/{repo-slug}/pullrequests" \
  -d '{
    "title": "feat: 프로필 다이얼로그 추가",
    "source": {
      "branch": { "name": "feature/profile-dialog" }
    },
    "destination": {
      "branch": { "name": "main" }
    },
    "description": "## 변경 내용\n- ProfileDialog 컴포넌트 추가\n- DashboardLayout에 연동",
    "reviewers": [
      { "account_id": "navskh" }
    ],
    "close_source_branch": true
  }'
```

> **인증**: Bitbucket App Password 사용 (navskh 계정)
> **close_source_branch**: merge 후 feature 브랜치 자동 삭제

### 4.3 Claude Code 작업 시 전체 절차

```bash
# 1. 최신 main 가져오기
git checkout main
git pull

# 2. feature 브랜치 생성
git checkout -b feature/{작업명}

# 3. 작업 수행 + 커밋
git add {파일들}
git commit -m "feat: ..."

# 4. push
git push -u origin feature/{작업명}

# 5. Bitbucket API로 PR 생성
curl -X POST ... (위 4.2 참조)

# 6. 사용자에게 PR URL 안내
```

### 4.4 submodule 포함 작업 시 PR 순서

jabis-common을 수정하고 소비 프로젝트에 반영할 때:

```
1. jabis-common에서 feature 브랜치 → 작업 → PR 생성
2. navskh가 jabis-common PR Merge
3. 소비 프로젝트에서 feature 브랜치 생성
4. git submodule update --remote packages
5. git add packages → 커밋 → push → PR 생성
6. navskh가 소비 프로젝트 PR Merge
```

> **중요**: jabis-common PR이 merge되기 전에 소비 프로젝트 PR을 만들면 안 됨 (submodule이 이전 커밋을 참조하게 됨)

---

## 5. NightExecutor PR 워크플로우

### 5.1 기존 vs 변경

| 항목 | 기존 | 변경 |
|------|------|------|
| 작업 결과 반영 | main에 직접 push | feature 브랜치 → PR 생성 |
| 리뷰 | 없음 (자동 배포) | navskh가 다음 날 아침 리뷰 후 Merge |
| 롤백 | 어려움 | PR 단위로 Merge/Reject 가능 |

### 5.2 NightExecutor 작업 플로우

```
1. NightExecutor가 작업 시작
2. feature 브랜치 생성 (feature/night-{작업ID} 또는 feature/night-{설명})
3. 작업 수행 + 커밋
4. feature 브랜치에 push
5. Bitbucket API로 PR 생성 (reviewer: navskh)
6. 다음 날 아침 navskh가 PR 리뷰
   - 승인 → Merge → 자동 배포
   - 거부 → PR Close (코드 폐기)
```

### 5.3 NightExecutor PR 제목 형식

```
[Night] {prefix}: {설명} ({프로젝트명})
```

예시:
- `[Night] feat: 대시보드 위젯 추가 (jabis)`
- `[Night] fix: API 타임아웃 수정 (jabis-api-gateway)`

이렇게 `[Night]` 접두어를 붙여 야간 자동 작업임을 명시합니다.

---

## 6. 규칙

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

## 7. Bitbucket Pipeline 패턴

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

---

## 8. Bitbucket Branch Permission 설정 절차

전체 JABIS 저장소에 main 브랜치 보호를 설정하는 절차입니다.

### 8.1 저장소별 설정 (Bitbucket 웹)

각 저장소에 대해:

1. **Repository settings** → **Branch permissions** → **Add a branch permission**
2. Branch: `main` 선택
3. 설정:
   - **Prevent all changes** (모든 직접 push 차단): 사용
   - **Allow changes via pull request**: 체크
   - **Minimum approvals**: 1
4. **Default reviewers** → navskh 추가
5. **Merge checks** → "Require at least 1 approval" 활성화

### 8.2 알림 설정

Bitbucket 기본 알림으로 충분합니다:
- PR 생성 시 → reviewer(navskh)에게 이메일 발송
- PR 코멘트 시 → 관련자에게 이메일 발송
- PR merge 시 → 관련자에게 이메일 발송

개인 알림 설정: **Bitbucket** → **Personal settings** → **Notifications** → Email 활성화 확인

### 8.3 Pipeline 트리거 변경 없음

기존 Pipeline 설정은 변경 불필요합니다:
- Pipeline은 `branches: main`에 설정되어 있음
- PR merge 시 main에 커밋이 생성되므로 **Pipeline 자동 트리거** (기존과 동일)
- feature 브랜치 push 시에는 Pipeline이 실행되지 않음 (의도된 동작)

---

## 9. 전환 계획 (TODO)

### Phase 1: 정책 수립 (완료)
- [x] git-workflow.md 정책 문서 작성

### Phase 2: Bitbucket 설정
- [ ] 전체 저장소 Branch permission 설정
- [ ] Default reviewer 설정
- [ ] 알림 동작 확인

### Phase 3: 도구 적용
- [ ] Claude Code에서 feature branch + PR 생성 플로우 테스트
- [ ] NightExecutor에 PR 생성 로직 추가 (jabis-night-builder)

### Phase 4: 운영
- [ ] 전체 프로젝트에서 PR 플로우 정상 동작 확인
- [ ] main 직접 push 차단 확인
