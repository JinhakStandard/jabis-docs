# JABIS Documentation Hub

> JABIS(진학 통합 시스템) 중앙 문서 저장소

---

## 구조

```
jabis-docs/
├── policies/                  # 전사 정책 (모든 프로젝트 공통)
│   ├── coding-style.md        # 코딩 스타일 정책
│   ├── security.md            # 보안 정책
│   ├── git-workflow.md        # Git 워크플로우
│   └── deployment.md          # 배포 및 인프라 정책
│
├── architecture/              # 시스템 아키텍처
│   └── system-overview.md     # JABIS 전체 구조도
│
├── api-specs/                 # API 명세서 (단일 소스)
│   ├── gateway-api.md         # AI Orchestra API 규약
│   ├── dashboard-api.md       # 대시보드 API 규약
│   ├── night-builder-api.md   # Night Builder API 규약
│   ├── dev-tasks-api.md       # Dev Tasks API 가이드
│   └── frontend-integration.md # 프론트엔드 연동 가이드
│
├── design-system/             # 디자인 시스템 (단일 소스)
│   └── DESIGN-SYSTEM.md       # @jabis/ui 컴포넌트 가이드
│
├── ai-prompts/                # AI 프롬프트 (단일 소스)
│   ├── prompt-templates.md    # 진학어플라이 AI 프롬프트 템플릿
│   ├── night-builder-prompts.md    # Night Builder 프롬프트
│   ├── auto-debugger-prompts.md    # Auto Debugger 프롬프트
│   └── auto-debugger-architecture.md # Auto Debugger 아키텍처
│
├── onboarding/                # 온보딩/가이드
│   ├── project-map.md         # 13개 프로젝트 맵
│   ├── vibe-coding-guide.md   # 바이브 코딩 대전제
│   └── oauth-guide.md         # OAuth 구현 가이드
│
├── db-schemas/                # DB 스키마
│   ├── bitbucket-schema.md    # Bitbucket Sync 스키마
│   └── night-builder-schema.md # Night Builder 스키마
│
├── projects/                  # 프로젝트별 고유 정보
│   ├── jabis-cert.md          # 통합인증 서버
│   ├── jabis-api-gateway.md   # API Gateway
│   ├── jabis.md               # 메인 운영 앱
│   ├── jabis-template.md      # 템플릿/데모 앱
│   ├── jabis-lab.md           # 실험 플랫폼 (읽기전용 복제)
│   ├── jabis-dev.md           # 개발부서 전용
│   ├── jabis-night-builder.md # Night Builder (읽기전용 복제)
│   ├── jabis-bitbucket-sync.md # Bitbucket 동기화
│   ├── jabis-emergency-console.md # 긴급 DB 콘솔
│   ├── jabis-common.md        # 공유 UI 라이브러리
│   ├── jabis-design-system.md # 디자인 시스템 문서
│   ├── jabis-design-package.md # 디자인 토큰 추출
│   ├── jabis-helm.md          # Helm 배포 관리
│   └── jabis-hr.md            # 인사관리 전용 앱
│
└── templates/                 # 프로젝트 템플릿
    └── (예정)
```

---

## 문서 관리 원칙

### 단일 소스 (Single Source of Truth)
- 각 문서는 이 저장소에 **하나만** 존재합니다
- 개별 프로젝트에서 동일 문서를 복제하지 않습니다
- 프로젝트별 고유 정보(구조, 명령어, 스택, 환경변수 등)도 `projects/`에서 중앙 관리합니다

### 마스터 문서 출처
| 문서 | 원본 프로젝트 |
|------|-------------|
| DESIGN-SYSTEM.md | jabis-common |
| API 명세서 | jabis-api-gateway |
| AI 프롬프트 | jabis-lab, jabis-night-builder |
| DB 스키마 | 각 프로젝트 |
| 정책 문서 | .claude/rules/ (통합) |

### 프로젝트 추가 시 필수 업데이트
새 프로젝트가 추가되면 **반드시** 다음을 함께 업데이트합니다:
1. `projects/{프로젝트명}.md` 문서 생성
2. 이 README의 구조 트리에 추가
3. `onboarding/project-map.md`에 프로젝트 정보 추가

### 수정 불가 원본
다음 프로젝트의 문서는 **읽기 전용**으로 취급합니다 (복제만 허용):
- `jabis-lab` - 부장님 관리
- `jabis-night-builder` - 부장님 관리
