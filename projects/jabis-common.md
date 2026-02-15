# jabis-common

> 공유 UI 라이브러리 (@jabis/ui, @jabis/layout, @jabis/auth, @jabis/core, @jabis/menu, @jabis/shared-pages)

---

## 기술 스택
- pnpm monorepo
- React 18.2.0
- Radix UI
- Tailwind CSS
- TypeScript (일부)
- JavaScript

## 프로젝트 구조
```
jabis-common/
├── ui/                 # @jabis/ui (30+ shadcn/ui 컴포넌트)
│   ├── src/components/
│   ├── src/styles/globals.css
│   ├── src/index.js
│   └── tailwind.preset.js
├── layout/             # @jabis/layout (Header, Sidebar)
├── auth/               # @jabis/auth (OAuth, session)
│   ├── authConfig.js
│   ├── authService.js
│   ├── SessionTimeoutProvider.jsx
│   └── LoginPage.jsx
├── core/               # @jabis/core (Theme, providers)
│   ├── ThemeProvider.jsx
│   └── ThemeToggle.jsx
├── menu/               # @jabis/menu (공통 메뉴, 역할 데이터, 빌더)
│   └── src/
│       ├── commonGroups.js   # createCommonGroups() — 5개 공통 메뉴 그룹
│       ├── roles.js          # ROLES(17개), roleLabels, defaultUserNames
│       ├── buildMenu.js      # buildMenu(), staticMenu() 유틸리티
│       └── index.js
├── shared-pages/       # @jabis/shared-pages (공통 페이지 컴포넌트)
│   └── src/
│       ├── index.js          # 101개 export (페이지 25 + 컴포넌트 30+ + 스토어 10 + 유틸)
│       ├── pages/            # 25개 공통 페이지 컴포넌트
│       │   ├── TodoPage.jsx             # 할일 관리
│       │   ├── DocumentPage.jsx         # 문서 편집 (BlockNote)
│       │   ├── GoalPage.jsx             # 목표 관리 (팀/부서/본부)
│       │   ├── SchedulePage.jsx         # 일정 관리
│       │   ├── ApprovalPage.jsx         # 결재 문서 목록 (8종 문서함)
│       │   ├── ApprovalFormPage.jsx     # 결재 기안 (양식 선택, 동적 필드, 결재선)
│       │   ├── ApprovalDetailPage.jsx   # 결재 상세 (승인/반려/회수, 후처리)
│       │   ├── MessengerPage.jsx        # 메신저
│       │   ├── MailPage.jsx             # 메일 (AI 작성)
│       │   ├── MeetingRoomPage.jsx      # 회의실 예약
│       │   ├── TreasureBlogPage.jsx     # 보물블로그
│       │   ├── BreakfastPage.jsx        # 조식 메뉴
│       │   ├── NewsMonitoringPage.jsx   # 뉴스 모니터링
│       │   ├── RestaurantPage.jsx       # 맛집 추천
│       │   ├── MemoryPage.jsx           # 추억 공유
│       │   ├── OrganizationPage.jsx     # 조직도
│       │   ├── UniversityPage.jsx       # 대학 목록
│       │   ├── UniversityDetailPage.jsx # 대학 상세
│       │   ├── ReportPage.jsx           # 리포트
│       │   ├── VoucherPage.jsx          # 전표 관리
│       │   ├── SitesPage.jsx            # 사이트 현황
│       │   ├── ProjectPage.jsx          # 프로젝트 관리
│       │   ├── MyHRPage.jsx             # 나의 인사정보
│       │   ├── AiWorkReportPage.jsx     # AI 업무 보고서
│       │   └── AiTeamReportPage.jsx     # AI 팀 보고서
│       ├── components/       # 공유 컴포넌트
│       │   ├── approval/     # 결재 관련 (21개 파일)
│       │   │   ├── ApprovalLineEditor.jsx     # 결재선 편집 (추가/삭제/순서변경/유형변경)
│       │   │   ├── ApprovalLineTemplates.jsx  # 결재선 템플릿
│       │   │   ├── SortableApproverItem.jsx   # 드래그 가능한 결재자 아이템
│       │   │   ├── OrgPickerDialog.jsx        # 조직도 기반 결재자 선택
│       │   │   ├── ReferenceEditor.jsx        # 참조자 편집
│       │   │   ├── RelatedDocSelector.jsx     # 관련문서 검색/연결
│       │   │   ├── ApprovalTimeline.jsx       # 결재 진행 타임라인
│       │   │   ├── ProcessingTimeline.jsx     # 후처리 부서 타임라인
│       │   │   ├── ReferenceSection.jsx       # 참조자 표시
│       │   │   ├── RelatedDocsSection.jsx     # 관련문서 링크
│       │   │   ├── FormFieldRenderer.jsx      # 동적 필드 렌더러
│       │   │   ├── FormStepIndicator.jsx      # 기안 단계 표시
│       │   │   ├── TemplateSelectorSection.jsx # 양식 선택
│       │   │   ├── DocumentContentSection.jsx # 문서 내용
│       │   │   ├── SubmitConfirmDialog.jsx    # 제출 확인
│       │   │   ├── DeptProcessingPanel.jsx    # 부서 처리 패널
│       │   │   ├── ApprovalActionDialogs.jsx  # 승인/반려/회수/처리 다이얼로그 5종
│       │   │   ├── ApprovalBoxTabs.jsx        # 문서함 탭
│       │   │   ├── ApprovalFilters.jsx        # 필터
│       │   │   ├── ApprovalStatsCards.jsx     # 통계 카드
│       │   │   └── ApprovalDocumentItem.jsx   # 문서 목록 아이템
│       │   ├── document/     # 문서 편집 (7개)
│       │   │   ├── BlockNoteEditor.jsx        # BlockNote 에디터
│       │   │   ├── DocumentEditor.jsx         # 문서 편집기
│       │   │   ├── Block.jsx                  # 블록 컴포넌트
│       │   │   ├── BlockMenu.jsx              # 블록 메뉴
│       │   │   ├── CollaborationStatus.jsx    # 협업 상태
│       │   │   ├── TemplateSelector.jsx       # 템플릿 선택
│       │   │   └── VersionHistory.jsx         # 버전 이력
│       │   ├── goals/        # 목표 관리 (1개 — 5종 다이얼로그)
│       │   │   └── GoalDialogs.jsx            # Team/Member/Execution/Dept/Division
│       │   ├── organization/ # 조직도 (5개)
│       │   │   ├── OrgChartView.jsx           # 차트 뷰
│       │   │   ├── OrgTreeView.jsx            # 트리 뷰
│       │   │   ├── OrgTableView.jsx           # 테이블 뷰
│       │   │   ├── DetailPanel.jsx            # 상세 패널
│       │   │   └── DepartmentFormDialog.jsx   # 부서 생성/삭제/이동 다이얼로그
│       │   └── AiMailComposer.jsx             # AI 메일 작성
│       ├── hooks/            # 커스텀 훅
│       │   └── useApprovalPermissions.js      # 결재 권한 판단
│       ├── utils/            # 유틸리티
│       │   └── approvalDetailUtils.jsx        # 날짜/금액 포맷, 필드 렌더링
│       ├── stores/           # Zustand 스토어 (10개)
│       │   ├── approvalStore.js       # 결재 문서 CRUD, 승인/반려, 참조자/관련문서
│       │   ├── documentStore.js       # 문서 CRUD
│       │   ├── goalStore.js           # 목표 관리
│       │   ├── messengerStore.js      # 메신저
│       │   ├── meetingRoomStore.js    # 회의실
│       │   ├── newsStore.js           # 뉴스 모니터링
│       │   ├── restaurantStore.js     # 맛집
│       │   ├── memoryStore.js         # 추억
│       │   ├── organizationStore.js   # 조직도
│       │   └── voucherStore.js        # 전표
│       └── data/             # 공유 Mock 데이터 (10개)
│           ├── approval.js        # 결재 양식 템플릿, 카테고리, 상태 매핑
│           ├── documentTypes.js   # 문서 유형 정의
│           ├── goals.js           # 목표 Mock 데이터
│           ├── organization.js    # 조직 Mock 데이터
│           ├── projects.js        # 프로젝트 Mock 데이터
│           ├── sales.js           # 영업 Mock 데이터
│           ├── schedules.js       # 일정 Mock 데이터
│           ├── sites.js           # 사이트 Mock 데이터
│           ├── todos.js           # 할일 Mock 데이터
│           └── universities.js    # 대학 Mock 데이터
├── package.json
└── pnpm-workspace.yaml
```

## 주요 명령어
없음 (워크스페이스/라이브러리 전용)

## 포트
N/A (라이브러리)

## 환경변수 (Dockerfile)
N/A

## Dockerfile
없음 (컨테이너화 안 됨)

## Bitbucket Pipeline
없음

## Skills
없음

## @jabis/menu 사용법

```js
import { buildMenu, staticMenu, roleLabels, defaultUserNames } from '@jabis/menu'

// 공통 그룹 + 역할 전용 메뉴 조합
export const myMenu = buildMenu('hr', (common, { extend }) => [
  { icon: 'LayoutDashboard', label: '대시보드', path: '/hr' },
  common.work,                    // 공통 그룹 그대로 사용
  extend(common.work, [           // 공통 그룹에 아이템 추가 (뒤에)
    { icon: 'FolderKanban', label: '프로젝트', path: '/hr/projects' },
  ]),
  extend(common.organization, {   // 앞뒤로 아이템 추가
    prepend: [{ icon: 'Users', label: '팀 관리', path: '/hr/teams' }],
    append: [{ icon: 'BarChart3', label: '리포트', path: '/hr/reports' }],
  }),
  common.communication,
  common.life,
  common.etc,
])

// 공통 그룹 없이 정적 메뉴
export const portalMenu = staticMenu([
  { icon: 'LayoutDashboard', label: '홈', path: '/portal' },
])
```

## @jabis/layout 역할 전환 기능

Header 컴포넌트에서 역할 배지 클릭 시 역할 전환 드롭다운을 표시할 수 있다.

```jsx
<DashboardLayout
  // ...기존 props...
  availableRoles={[
    { id: 'developer', label: '개발자' },
    { id: 'operation', label: '운영자' },
    // ...
  ]}
  onRoleChange={(roleId) => {
    localStorage.setItem('jabis_role', roleId)
    navigate(`/${roleId}`)
  }}
>
```

- `availableRoles`: `[{ id, label }]` 배열. `@jabis/menu`의 `ROLES`와 `roleLabels`로 생성 가능.
- `onRoleChange`: 역할 변경 시 호출. localStorage 업데이트 + 페이지 이동 처리.
- 두 prop이 모두 없으면 기존 정적 Badge로 폴백 (하위 호환).

## @jabis/shared-pages 라우팅 통합 현황

모든 프론트엔드 프로젝트에서 `@jabis/shared-pages`를 `workspace:*`로 의존성 등록하고 라우팅에 통합 완료 (2026-02-16 검증).

| 소비 프로젝트 | 사용 페이지 수 | 라우팅 패턴 | role prop |
|-------------|-------------|-----------|-----------|
| jabis | 25개 | `/:role/:page` | 동적 (17개 역할) |
| jabis-producer | 18개 | `/producer/:page` | `"producer"` 고정 |
| jabis-dev | 23개 | `/dev/:page` | `"developer"` 고정 |
| jabis-hr | 16개 | `/hr/:page` | `"hr"` 고정 |

### 사용법

```jsx
import { TodoPage, DocumentPage, ApprovalPage } from '@jabis/shared-pages'

// 라우트 등록
<Route path="todos" element={<TodoPage role="developer" />} />
<Route path="documents" element={<DocumentPage role="developer" />} />
<Route path="approval" element={<ApprovalPage role="developer" />} />
```

### Export 구성 (101개)

| 카테고리 | 수량 | 주요 항목 |
|---------|------|---------|
| 페이지 (업무관리) | 7 | TodoPage, DocumentPage, GoalPage, SchedulePage, Approval 3종 |
| 페이지 (소통) | 3 | MessengerPage, MailPage, MeetingRoomPage |
| 페이지 (사내생활) | 5 | TreasureBlogPage, BreakfastPage, NewsMonitoringPage, RestaurantPage, MemoryPage |
| 페이지 (조직/관리) | 4 | OrganizationPage, UniversityPage, UniversityDetailPage, ReportPage |
| 페이지 (기타) | 6 | VoucherPage, SitesPage, ProjectPage, MyHRPage, AiWorkReportPage, AiTeamReportPage |
| 컴포넌트 (결재) | 20+ | ApprovalLineEditor, ApprovalTimeline, ActionDialogs 5종 등 |
| 컴포넌트 (문서) | 7 | BlockNoteEditor, DocumentEditor, Block, BlockMenu 등 |
| 컴포넌트 (목표) | 5 | TeamTaskDialog, MemberGoalDialog 등 |
| 컴포넌트 (조직) | 8 | OrgChartView, OrgTreeView, OrgTableView, DetailPanel 등 |
| 스토어 | 10 | useApprovalStore, useDocumentStore 등 |
| 훅/유틸 | 6 | useApprovalPermissions, formatDate 등 |

## 프로젝트 고유 규칙
- pnpm 필수 (npm 사용 금지)
- 모든 UI 컴포넌트는 @jabis/ui로 export
- 메뉴 공통 그룹/역할 데이터는 @jabis/menu로 export
- 공통 페이지 컴포넌트는 @jabis/shared-pages로 export
- tailwind.preset.js로 디자인 토큰 통일
- Git submodule로 jabis, jabis-template, jabis-design-system에서 참조
- 부서 리포에서 수정 금지 — import만 허용
