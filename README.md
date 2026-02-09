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
└── templates/                 # 프로젝트 템플릿
    └── (예정)
```

---

## 문서 관리 원칙

### 단일 소스 (Single Source of Truth)
- 각 문서는 이 저장소에 **하나만** 존재합니다
- 개별 프로젝트에서 동일 문서를 복제하지 않습니다
- 개별 프로젝트 CLAUDE.md에는 프로젝트 고유 정보만 보관합니다

### 마스터 문서 출처
| 문서 | 원본 프로젝트 |
|------|-------------|
| DESIGN-SYSTEM.md | jabis-common |
| API 명세서 | jabis-api-gateway |
| AI 프롬프트 | jabis-lab, jabis-night-builder |
| DB 스키마 | 각 프로젝트 |
| 정책 문서 | .claude/rules/ (통합) |

### 수정 불가 원본
다음 프로젝트의 문서는 **읽기 전용**으로 취급합니다 (복제만 허용):
- `jabis-lab` - 부장님 관리
- `jabis-night-builder` - 부장님 관리
