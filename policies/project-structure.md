# 프로젝트 구조 정책

> 모든 JABIS 프로젝트의 디렉토리 구조와 파일 배치 규칙을 정의한다.
> 이 정책을 위반하면 submodule 불일치, 빌드 실패, 메뉴/UI 누락 등 연쇄 장애가 발생한다.

---

## 1. jabis-common (공유 라이브러리) — 가장 중요

jabis-common은 모든 프론트엔드 프로젝트가 submodule로 참조하는 **단일 원본(single source of truth)**이다.

### 1.1 디렉토리 구조 (확정)

```
jabis-common/
├── ui/                 # @jabis/ui
├── layout/             # @jabis/layout
├── auth/               # @jabis/auth
├── core/               # @jabis/core
├── menu/               # @jabis/menu
├── shared-pages/       # @jabis/shared-pages
├── tools/              # 내부 도구
├── docs/               # 문서
├── package.json
├── pnpm-workspace.yaml # ← 패키지 등록 유일 원본
└── pnpm-lock.yaml
```

### 1.2 절대 규칙

| 규칙 | 설명 |
|------|------|
| **패키지는 루트에만 생성** | `ui/`, `menu/`, `shared-pages/` 등 모든 패키지는 jabis-common **루트 레벨**에 위치. `packages/` 하위 디렉토리에 절대 생성 금지 |
| **pnpm-workspace.yaml이 유일한 등록처** | 새 패키지를 만들면 반드시 여기에 등록. 여기에 없으면 패키지가 아님 |
| **동일 이름 폴더 중복 금지** | 같은 이름의 패키지 폴더가 2개 이상 존재하면 안 됨 (예: `/menu/`와 `/packages/menu/` 동시 존재 금지) |
| **packages/ 디렉토리 사용 금지** | jabis-common 내에 `packages/` 디렉토리를 만들지 않음. 과거 리팩토링의 유물이며, 존재 시 삭제 대상 |
| **수정은 jabis-common에서만** | 소비 프로젝트(jabis-hr 등)에서 직접 수정 금지 — submodule 동기화 시 유실됨 |

### 1.3 새 패키지 추가 절차

```
1. jabis-common 루트에 폴더 생성 (예: new-package/)
2. pnpm-workspace.yaml에 'new-package' 추가
3. package.json에 "name": "@jabis/new-package" 설정
4. 커밋 → push
5. 모든 소비 프로젝트에서 submodule update → 커밋 → push
```

### 1.4 수정 → 소비 프로젝트 반영 플로우

```
1. jabis-common에서 수정 → 커밋 → push
2. 소비 프로젝트에서:
   git submodule update --remote packages
   git add packages
   git commit -m "chore: update jabis-common submodule"
   git push
3. pnpm build:preview로 검증
```

소비 프로젝트 목록 (jabis-lab, jabis-template 제외):
- jabis, jabis-hr, jabis-dev, jabis-producer, jabis-design-system, jabis-finance

---

## 2. dept 프로젝트 (jabis-hr, jabis-dev, jabis-producer, jabis-finance 등)

모든 dept 프로젝트는 **동일한 pnpm 모노레포 구조**를 따른다.

### 2.1 디렉토리 구조 (템플릿)

```
jabis-{role}/
├── .gitmodules                    # packages = jabis-common submodule
├── pnpm-workspace.yaml
├── package.json                   # root: pnpm --filter {role} 스크립트
├── pnpm-lock.yaml
├── Dockerfile
├── bitbucket-pipelines.yml
├── dist -> apps/{role}/dist       # 심볼릭 링크 (미리보기 필수)
├── packages/                      # git submodule → jabis-common
│   ├── ui/
│   ├── layout/
│   ├── auth/
│   ├── core/
│   ├── menu/
│   └── shared-pages/
└── apps/
    └── {role}/                    # 실제 앱 (1개만)
        ├── src/
        │   ├── App.jsx            # 라우팅 + 메뉴 설정
        │   ├── main.jsx           # 진입점
        │   ├── pages/             # 역할 전용 페이지
        │   ├── components/        # (선택) 역할 전용 컴포넌트
        │   ├── stores/            # (선택) Zustand 스토어
        │   └── data/              # (선택) 모의 데이터
        ├── package.json           # @jabis/* workspace:* 의존성
        ├── vite.config.js
        ├── tailwind.config.js     # @jabis/ui 프리셋 + shared-pages content 경로
        ├── .env.local
        ├── .env.preview
        └── .env.production
```

### 2.2 절대 규칙

| 규칙 | 설명 |
|------|------|
| **packages/ 수정 금지** | submodule이므로 jabis-common에서만 수정 |
| **앱은 apps/{role}/ 하나만** | 역할당 1개 앱. 루트에 src/ 두지 않음 |
| **dist 심볼릭 링크 필수** | `dist -> apps/{role}/dist` — 없으면 미리보기 불가 |
| **공통 메뉴는 @jabis/menu 사용** | `buildMenu({ standalone: true })` 필수. 하드코딩 금지 |
| **역할 데이터는 @jabis/menu에서 import** | `availableRoles`, `getRoleUrl` 하드코딩 금지 |
| **tailwind content에 shared-pages 포함** | `../../packages/shared-pages/src/**/*.{js,jsx}` 필수 |

### 2.3 새 dept 프로젝트 생성 체크리스트

- [ ] `jabis-{role}/` 디렉토리 생성
- [ ] `.gitmodules`에 jabis-common submodule 등록
- [ ] `pnpm-workspace.yaml` 작성 (6개 공유 패키지 + apps/*)
- [ ] `apps/{role}/` 앱 생성
- [ ] `dist -> apps/{role}/dist` 심볼릭 링크 생성
- [ ] Dockerfile, bitbucket-pipelines.yml 작성
- [ ] `.env.preview`, `.env.production` 환경변수 설정
- [ ] jabis-docs/projects/jabis-{role}.md 문서 작성

---

## 3. jabis (메인 프론트엔드)

jabis 메인은 dept 프로젝트와 동일한 모노레포 구조이나, 앱 이름이 `prototype`이다.

### 3.1 디렉토리 구조

```
jabis/
├── .gitmodules
├── pnpm-workspace.yaml
├── package.json
├── dist -> apps/prototype/dist
├── packages/                      # submodule → jabis-common
├── apps/
│   └── prototype/                 # 17개 역할 대시보드
│       ├── src/
│       │   ├── App.jsx
│       │   ├── pages/
│       │   ├── components/
│       │   ├── data/              # menuConfig.js 등
│       │   ├── stores/
│       │   ├── hooks/
│       │   └── lib/
│       ├── party/                 # PartyKit 협업
│       └── ...
└── docs/
```

### 3.2 jabis만의 규칙

- `menuConfig.js`에서 `buildMenu()` 사용 (standalone 없이 — role prefix 포함)
- 17개 역할 각각의 메뉴를 하나의 파일에서 관리
- `staticMenu()`은 portal 등 공통 그룹 불필요 역할에만 사용

---

## 4. 백엔드 프로젝트 (jabis-api-gateway, jabis-maker 등)

### 4.1 공통 디렉토리 구조

```
jabis-{service}/
├── src/
│   ├── server.ts                  # 진입점
│   ├── config/                    # 환경변수 검증 (Zod)
│   │   ├── index.ts
│   │   ├── schema.ts
│   │   └── types.ts
│   ├── routes/                    # API 엔드포인트
│   ├── services/                  # 비즈니스 로직
│   ├── middleware/                 # 인증, 로깅 등
│   ├── types/                     # TypeScript 타입
│   └── utils/                     # 유틸리티
├── config/                        # YAML 런타임 설정 (선택)
├── sql/                           # DB 마이그레이션 (선택)
├── tests/                         # 테스트 (선택)
├── package.json                   # npm 프로젝트
├── package-lock.json
├── tsconfig.json
├── Dockerfile
└── bitbucket-pipelines.yml
```

### 4.2 절대 규칙

| 규칙 | 설명 |
|------|------|
| **npm 사용** | 백엔드는 npm (pnpm 아님) |
| **TypeScript strict** | tsconfig.json strict: true |
| **Winston 로깅** | console.log 금지 |
| **Zod 검증** | 환경변수, API 입력 모두 Zod 스키마 |
| **config/ 디렉토리** | YAML 설정 파일 위치 |
| **sql/ 디렉토리** | DB 마이그레이션 쿼리 위치 (api-gateway) |

---

## 5. 금지 사항 (전체 프로젝트 공통)

1. **같은 이름의 디렉토리/패키지를 2곳에 생성하지 않는다**
2. **jabis-common 내에 `packages/` 하위 디렉토리를 만들지 않는다**
3. **소비 프로젝트에서 packages/ (submodule) 내부를 직접 수정하지 않는다**
4. **pnpm-workspace.yaml 또는 package.json에 등록하지 않은 패키지를 사용하지 않는다**
5. **환경변수를 하드코딩하지 않는다** — .env 파일 또는 config/ YAML 사용
6. **K3S 서비스 URL을 추정으로 작성하지 않는다** — deployment.md 참조 필수
