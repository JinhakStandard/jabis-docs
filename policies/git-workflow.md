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

### 필수
- 커밋 전 반드시 `git diff`로 변경 내용 확인
- 민감 정보(`.env`, credentials) 커밋 금지
- Teams 배포 알림 webhook step 포함 (`bitbucket-pipelines.yml`)

### 금지
- `git push --force` 금지 (특히 main/master)
- `git reset --hard` 금지 (확인 없이)
- `--no-verify` 옵션 금지

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
