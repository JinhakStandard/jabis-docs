# 법인카드 불출 시스템 - Night Builder 태스크 정의서

> 생성일: 2026-02-22
> 대상 프로젝트: jabis-finance, jabis-common (packages submodule)
> ref 파일: /home/jinhak/Projects/ref/card-system/*.html

---

## 전체 구조 요약

```
[일반 직원 — 모든 역할]                    [재무 관리자 — finance 역할]
공통메뉴 > 업무 관리 > 법인카드 신청        jabis-finance > 법인카드 관리 (새 그룹)
  ├ 불출 기안서 작성                         ├ 카드 현황 대시보드
  ├ 내 기안서 목록 (본인 것만)               ├ 불출/반납 승인 (결재)
  └ 반납 기안서 작성                         └ 카드 설정 (마스터 관리)
```

**실행 순서**: Task 1 → 2 → 3 → 4 → 5 → 6 (순차 의존성 있음)

---

## Task 1: 법인카드 관리 메뉴 그룹 추가 + 라우트 세팅

```json
{
  "id": "req-20260222-001",
  "title": "jabis-finance 법인카드 관리 메뉴 그룹 및 라우트 추가",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 10
}
```

### prompt

```
jabis-finance 프로젝트에 "법인카드 관리" 메뉴 그룹을 추가하고 라우트를 세팅해줘.

## 작업 대상 파일
- /home/jinhak/Projects/jabis-finance/apps/finance/src/App.jsx

## 요구사항

### 1. 메뉴 그룹 추가
App.jsx의 buildMenu 호출 부분에 새 메뉴 그룹을 추가해줘.
기존 재무 전용 메뉴 그룹들 (결산/재무보고, 예산 관리, ..., 자산 관리) 뒤에 추가.

```javascript
{
  label: '법인카드 관리',
  icon: 'CreditCard',
  items: [
    { icon: 'LayoutDashboard', label: '카드 현황', path: '/card/status' },
    { icon: 'FileCheck', label: '불출/반납 승인', path: '/card/approval' },
    { icon: 'Settings', label: '카드 설정', path: '/card/settings' },
  ],
}
```

### 2. 라우트 추가
Routes 섹션에 다음 라우트를 추가해줘.
일단 모든 페이지는 ComingSoonPage를 사용 (추후 실제 페이지로 교체).

```jsx
{/* 법인카드 관리 */}
<Route path="card/status" element={<ComingSoonPage />} />
<Route path="card/approval" element={<ComingSoonPage />} />
<Route path="card/settings" element={<ComingSoonPage />} />
```

### 3. lucide-react import 확인
CreditCard, LayoutDashboard, FileCheck, Settings 아이콘이
@jabis/ui 또는 lucide-react에서 사용 가능한지 확인.
buildMenu에서는 문자열로 아이콘명을 전달하므로 import는 불필요할 수 있음.
기존 메뉴 아이콘 패턴을 따를 것.

## 참조
- 기존 메뉴 그룹 패턴: App.jsx 32-102 라인
- 기존 라우트 패턴: App.jsx 138-189 라인
- ComingSoonPage: /apps/finance/src/pages/ComingSoonPage.jsx
```

---

## Task 2: 카드 현황 대시보드 페이지

```json
{
  "id": "req-20260222-002",
  "title": "법인카드 카드 현황 대시보드 페이지 구현",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 9
}
```

### prompt

```
jabis-finance에 법인카드 카드 현황 대시보드 페이지를 구현해줘.

## 작업 대상
- 새 파일: /home/jinhak/Projects/jabis-finance/apps/finance/src/pages/CardStatusPage.jsx
- 수정: /home/jinhak/Projects/jabis-finance/apps/finance/src/App.jsx (라우트 연결)

## 기능 요구사항

### 1. 통계 요약 카드 (상단)
StatCard 컴포넌트 사용. 5개 카드:
- 총 카드 수
- 불출 가능
- 불출 중
- 미결 불출 기안서
- 미결 반납 기안서

### 2. 보유 법인카드 현황 (중단)
법인별 그룹화 표시 (진학사 / 진학어플라이).
각 카드마다:
- 카드 뒷자리 4자리 표시
- 불출 가능/불출 중 상태 배지
- 불출 중인 경우 불출자 이름 표시
- 카드 아이콘 또는 컬러로 시각적 구분

### 3. 최근 기안서 내역 (하단)
최근 10건의 불출/반납 기안서를 테이블로 표시.
컬럼: 기안번호, 카드번호, 불출자, 상태, 작성일

## UI 패턴
- PageHeader 사용: icon={CreditCard}, title="카드 현황", description="법인카드 보유 현황 및 불출 상태를 확인합니다"
- 레이아웃: h-full flex flex-col + flex-1 overflow-auto p-4 space-y-6
- @jabis/ui 컴포넌트: Card, CardContent, Badge, StatCard
- Tailwind CSS 클래스 사용

## 데이터
현재 API가 없으므로 더미 데이터로 구현.
페이지 상단에 더미 데이터 상수를 정의해서 사용.

```javascript
// 더미 데이터 예시
const DUMMY_CARDS = [
  { number: '1234', company: '진학사', status: 'available' },
  { number: '4567', company: '진학사', status: 'in_use', holder: '김철수' },
  { number: '7890', company: '진학사', status: 'available' },
  { number: '1313', company: '진학어플라이', status: 'in_use', holder: '이영희' },
  { number: '1414', company: '진학어플라이', status: 'available' },
  { number: '1515', company: '진학어플라이', status: 'available' },
]

const DUMMY_PROPOSALS = [
  { id: 1, sequence: '00000001', cardNumber: '4567', userName: '김철수', status: 'approved', type: 'checkout', createdDate: '2026.02.20' },
  { id: 2, sequence: '00000002', cardNumber: '1313', userName: '이영희', status: 'pending', type: 'checkout', createdDate: '2026.02.21' },
  // ...
]
```

## 참조
- ref HTML: /home/jinhak/Projects/ref/card-system/법인카드_불출관리.html (통계요약, 카드현황 섹션)
- 기존 페이지 패턴: /home/jinhak/Projects/jabis-finance/apps/finance/src/pages/DashboardPage.jsx
- @jabis/ui 컴포넌트: /home/jinhak/Projects/jabis-finance/packages/ui/src/components/

## App.jsx 수정
ComingSoonPage → CardStatusPage로 교체:
```jsx
import CardStatusPage from './pages/CardStatusPage'
// ...
<Route path="card/status" element={<CardStatusPage />} />
```
```

---

## Task 3: 불출/반납 승인 페이지

```json
{
  "id": "req-20260222-003",
  "title": "법인카드 불출/반납 승인 관리 페이지 구현",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 8
}
```

### prompt

```
jabis-finance에 법인카드 불출/반납 승인 관리 페이지를 구현해줘.

## 작업 대상
- 새 파일: /home/jinhak/Projects/jabis-finance/apps/finance/src/pages/CardApprovalPage.jsx
- 수정: /home/jinhak/Projects/jabis-finance/apps/finance/src/App.jsx (라우트 연결)

## 기능 요구사항

### 1. 탭 구성
Tabs 컴포넌트로 2개 탭:
- 불출 기안서 (기본 탭)
- 반납 기안서

### 2. 필터 영역
각 탭 상단에 필터:
- 카드번호 (Select)
- 사용자 이름 (Input, 텍스트 검색)
- 본부 (Select)
- 작성월 (month input)
- 상태 (Select: 전체/결재 전/불출 중/반납 결재 중/반납 완료)
- 필터 초기화 버튼

### 3. 기안서 목록
각 기안서 항목 표시:
- 기안번호 (Badge, 파란색)
- 카드번호, 불출자, 본부, 작성일, 예정금액
- 상태 배지 (결재 전=주황, 불출 중=초록, 반납 결재 중=파랑, 반납 완료=회색)
- 본문 내용 (접힘/펼침 가능)

### 4. 승인/반려 액션 (관리자)
결재 전 상태의 기안서에:
- "승인" 버튼 (초록) → 확인 다이얼로그 후 상태 변경
- "삭제" 버튼 (빨강) → 확인 다이얼로그 후 삭제

반납 기안서 결재 전 상태:
- "반납 승인" 버튼 → 승인 시 카드가 불출 가능 상태로 변경

### 5. 상세보기
기안서 클릭 또는 "상세보기" 버튼 → Dialog 모달:
- 불출 기안서: 기안번호, 카드번호, 불출자, 작성일, 예정금액, 본문, 결재선
- 반납 기안서: 기안번호, 카드번호, 불출자, 작성일, 사용 내역, 첨부 영수증 목록

## 비즈니스 로직 (더미 구현)
```javascript
// 상태 흐름
// 불출: pending → approved (승인) → returned (반납완료)
// 반납: pending → approved (반납승인)

// 더미 데이터
const DUMMY_CHECKOUT_PROPOSALS = [
  {
    id: 1, sequence: '00000001', cardNumber: '4567', userName: '김철수',
    userDepartment: '서비스본부', content: '거래처 미팅 식대 및 교통비',
    expectedAmount: 200000, status: 'pending', type: 'checkout',
    createdDate: '2026.02.20 14:30', returnProposalId: null
  },
  {
    id: 2, sequence: '00000002', cardNumber: '1313', userName: '이영희',
    userDepartment: '기획본부', content: '외부 세미나 참석 등록비',
    expectedAmount: 500000, status: 'approved', type: 'checkout',
    createdDate: '2026.02.19 09:15', returnProposalId: 101
  },
]

const DUMMY_RETURN_PROPOSALS = [
  {
    id: 101, sequence: '00000002', cardNumber: '1313', userName: '이영희',
    userDepartment: '기획본부', content: '2026.02.19 세미나 등록비: 500,000원\n2026.02.19 점심 식대: 45,000원',
    status: 'pending', type: 'return', originalProposalId: 2,
    receipts: [{ name: '영수증_세미나.pdf', size: 245000 }, { name: '영수증_식대.jpg', size: 1200000 }],
    createdDate: '2026.02.21 16:00'
  },
]
```

## UI 패턴
- PageHeader: icon={FileCheck}, title="불출/반납 승인", description="법인카드 불출 및 반납 기안서를 관리합니다"
- @jabis/ui: Card, Tabs, TabsList, TabsTrigger, TabsContent, Badge, Button, Dialog, Input, Select
- 상태 배지 스타일:
  - 결재 전: bg-orange-500/10 text-orange-600
  - 불출 중: bg-green-500/10 text-green-600
  - 반납 결재 중: bg-blue-500/10 text-blue-600
  - 반납 완료: bg-gray-500/10 text-gray-600

## 참조
- ref HTML: /home/jinhak/Projects/ref/card-system/법인카드_불출관리.html (기안서 목록, 필터, 상세보기 섹션)
- 기존 유사 패턴: /home/jinhak/Projects/jabis-finance/packages/shared-pages/src/pages/ApprovalPage.jsx
- App.jsx 라우트: ComingSoonPage → CardApprovalPage로 교체
```

---

## Task 4: 카드 설정 페이지 (마스터 관리)

```json
{
  "id": "req-20260222-004",
  "title": "법인카드 설정 관리 페이지 구현 (카드/본부/사용자 마스터)",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 7
}
```

### prompt

```
jabis-finance에 법인카드 설정 관리 페이지를 구현해줘. 관리자 전용 마스터 데이터 관리 화면.

## 작업 대상
- 새 파일: /home/jinhak/Projects/jabis-finance/apps/finance/src/pages/CardSettingsPage.jsx
- 수정: /home/jinhak/Projects/jabis-finance/apps/finance/src/App.jsx (라우트 연결)

## 기능 요구사항

### 1. 탭 구성 (3개 탭)

#### 탭 1: 보유 법인카드 관리
- 등록된 카드 목록 (카드 뒷자리 4자리 + 법인명)
- 카드 추가 폼: 뒷자리 4자리 입력 + 법인 선택 (진학사/진학어플라이)
- 수정/삭제 버튼 (사용 중인 카드는 삭제 불가 경고)
- 카드 번호 유효성: 4자리 숫자만, 중복 불가

#### 탭 2: 본부 관리
- 본부 목록 (이름만)
- 본부 추가: 이름 입력
- 수정/삭제 (해당 본부 소속 사용자가 있으면 삭제 불가 경고)

#### 탭 3: 기안서 작성자 관리
- 사용자 목록: 이름, 이메일, 법인, 본부, 관리자 여부 표시
- 사용자 추가 폼:
  - 이름 (필수)
  - 이메일 (필수, 이메일 형식 검증)
  - 법인 선택 (진학사/진학어플라이/전체법인)
  - 본부 선택 (탭2에서 등록한 본부 목록)
  - 관리자 권한 체크박스
- 인라인 수정 모드 (수정 버튼 클릭 → 해당 행이 편집 가능)
- 삭제 확인 다이얼로그

### 2. UI 구현
각 탭의 목록은 카드 안에 리스트 아이템으로 표시.

```jsx
// 리스트 아이템 패턴
<div className="flex items-center justify-between p-4 bg-muted/50 rounded-lg border">
  <div className="flex-1">
    <div className="font-semibold">**** 1234</div>
    <div className="text-sm text-muted-foreground">법인: 진학사</div>
  </div>
  <div className="flex gap-2">
    <Button variant="outline" size="sm" onClick={...}>수정</Button>
    <Button variant="destructive" size="sm" onClick={...}>삭제</Button>
  </div>
</div>
```

## 더미 데이터
```javascript
const DUMMY_CARDS = [
  { number: '1234', company: '진학사' },
  { number: '4567', company: '진학사' },
  { number: '7890', company: '진학사' },
  { number: '1313', company: '진학어플라이' },
  { number: '1414', company: '진학어플라이' },
  { number: '1515', company: '진학어플라이' },
]

const DUMMY_DEPARTMENTS = ['서비스본부', '기획본부', 'AI본부', '경영지원본부']

const DUMMY_USERS = [
  { name: '김철수', email: 'cs.kim@jinhak.com', company: '진학사', department: '서비스본부', isAdmin: false },
  { name: '이영희', email: 'yh.lee@jinhak.com', company: '진학어플라이', department: '기획본부', isAdmin: false },
  { name: '박관리', email: 'admin@jinhak.com', company: '전체법인', department: '경영지원본부', isAdmin: true },
]
```

## UI 패턴
- PageHeader: icon={Settings}, title="카드 설정", description="법인카드, 본부, 사용자를 관리합니다"
- @jabis/ui: Card, Tabs, TabsList, TabsTrigger, TabsContent, Button, Input, Select, Dialog, Label, Checkbox
- 추가 폼은 각 탭 하단에 Card로 감싸서 배치
- 토스트 알림: 추가/수정/삭제 성공 시 표시

## 참조
- ref HTML: /home/jinhak/Projects/ref/card-system/설정.html (전체 구조 참조)
- App.jsx 라우트: ComingSoonPage → CardSettingsPage로 교체
```

---

## Task 5: jabis-common 공통 메뉴에 법인카드 신청 추가

```json
{
  "id": "req-20260222-005",
  "title": "jabis-common 공통 메뉴(업무 관리)에 법인카드 신청 메뉴 추가",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 6
}
```

### prompt

```
jabis-common의 공통 메뉴 "업무 관리" 그룹에 "법인카드 신청" 메뉴를 추가해줘.
jabis-common은 jabis-finance의 packages/ 디렉토리에 submodule로 포함되어 있음.

## 작업 대상 파일
- /home/jinhak/Projects/jabis-finance/packages/menu/src/commonGroups.js

## 요구사항

### 1. 업무 관리 그룹에 메뉴 추가
기존 "업무 관리" 그룹의 items 배열에 추가:

```javascript
{ icon: 'CreditCard', label: '법인카드 신청', path: '/card/request' }
```

"전자결재" 메뉴 아래에 배치 (전자결재와 유사한 결재 워크플로우 성격).

### 2. 확인 사항
- commonGroups.js에서 업무 관리(work) 그룹을 찾아서 items 배열 끝에 추가
- 아이콘 문자열 'CreditCard'가 lucide-react에서 사용 가능한지 확인
  (다른 아이콘들과 동일하게 문자열로 전달되는 패턴인지 확인)

## 주의사항
- jabis-common은 submodule이므로, 이 변경은 jabis-common 리포에도 반영 필요
- 단, 현재는 jabis-finance의 packages/ 내에서 수정하고, submodule 커밋은 별도 진행
- 다른 프로젝트(jabis, jabis-dev 등)에서도 이 메뉴가 나타나게 됨 (공통 메뉴이므로)

## 참조
- commonGroups.js 위치: /home/jinhak/Projects/jabis-finance/packages/menu/src/commonGroups.js
- 기존 업무 관리 메뉴: 할일, 문서작성, 목표관리, 일정, 전자결재
```

---

## Task 6: 법인카드 신청 페이지 구현 (shared-pages)

```json
{
  "id": "req-20260222-006",
  "title": "법인카드 신청 페이지 구현 (공통 shared-pages)",
  "repositoryUrl": "jabis-finance",
  "branch": "feature/card-disbursement",
  "targetBranch": "master",
  "status": "queued",
  "priority": 5
}
```

### prompt

```
jabis-common의 shared-pages에 법인카드 신청 페이지를 구현해줘.
전 직원이 사용하는 카드 불출 신청 + 본인 기안서 관리 페이지.

## 작업 대상
- 새 파일: /home/jinhak/Projects/jabis-finance/packages/shared-pages/src/pages/CardRequestPage.jsx
- 수정: /home/jinhak/Projects/jabis-finance/packages/shared-pages/src/index.js (export 추가)
- 수정: /home/jinhak/Projects/jabis-finance/apps/finance/src/App.jsx (라우트 추가)

## 기능 요구사항

### 1. 불출 기안서 작성 폼 (상단 섹션)
Card 안에 폼 배치:
- 카드 선택 (Select): 불출 가능한 카드만 표시, 불출 불가 카드는 disabled + "(불출 불가)" 표기
- 불출자 이름 (Input, readonly): 로그인 사용자 이름 자동 표시
- 사용 목적 및 내역 (Textarea): 카드 사용 목적 입력
- 사용 예정 금액 (Input): 숫자만 입력, 자동 콤마 포맷
- 결재라인 표시: 기안자 → 법인카드 담당자 (2단계)
- "불출 기안서 상신" 버튼

### 2. 비즈니스 규칙
- 1인 1카드: 미반납 카드가 있으면 추가 불출 불가 → "불출 카드 반납 후 불출 가능합니다" alert
- 불출 중인 카드는 선택 불가 (disabled)

### 3. 내 기안서 목록 (하단 섹션)
본인이 작성한 기안서만 표시 (role prop 불필요, 로그인 사용자 기준).
각 기안서:
- 기안번호 (Badge)
- 카드번호, 작성일, 예정금액
- 상태 배지: 결재 전 / 불출 중 / 반납 결재 중 / 반납 완료
- 액션 버튼:
  - "상세보기" (항상)
  - "반납 기안서 작성" (불출 중 상태일 때만)
  - "삭제" (결재 전 상태일 때만)

### 4. 반납 기안서 작성 (Dialog 모달)
"반납 기안서 작성" 클릭 시 Dialog로 표시:
- 전자결재 문서번호 (Input, 필수)
- 카드번호 (readonly, 자동 채움)
- 불출자 이름 (readonly, 자동 채움)
- 사용 내역 적요 (Textarea, 필수)
- 영수증 첨부 (file input, 이미지/PDF, 다중 파일, 드래그앤드롭)
- "반납 기안서 상신" 버튼

## 더미 데이터
```javascript
const DUMMY_AVAILABLE_CARDS = [
  { number: '1234', company: '진학사', available: true },
  { number: '4567', company: '진학사', available: false, holder: '김철수' },
  { number: '7890', company: '진학사', available: true },
]

const DUMMY_MY_PROPOSALS = [
  {
    id: 1, sequence: '00000003', cardNumber: '7890',
    content: '부서 회식 식대', expectedAmount: 300000,
    status: 'approved', createdDate: '2026.02.21 10:00',
    returnProposalId: null
  },
]
```

## UI 패턴
- PageHeader: icon={CreditCard}, title="법인카드 신청", description="법인카드 불출을 신청하고 반납할 수 있습니다"
- @jabis/ui: Card, CardContent, Button, Input, Select, Textarea, Dialog, Badge, Label
- 금액 입력: onChange에서 숫자만 필터 + toLocaleString 포맷
- 파일 업로드 영역: 점선 border + 드래그앤드롭 지원

## shared-pages/src/index.js 수정
```javascript
export { default as CardRequestPage } from './pages/CardRequestPage'
```

## App.jsx 라우트 추가
```jsx
import { CardRequestPage } from '@jabis/shared-pages'
// ...
<Route path="card/request" element={<CardRequestPage role="finance" />} />
```

## 참조
- ref HTML 불출 기안서: /home/jinhak/Projects/ref/card-system/법인카드_불출관리.html (기안서 작성 폼, 기안서 목록)
- ref HTML 반납 기안서: /home/jinhak/Projects/ref/card-system/반납기안서.html (반납 폼, 파일 업로드)
- 기존 shared-pages 패턴: /home/jinhak/Projects/jabis-finance/packages/shared-pages/src/pages/ApprovalPage.jsx
```

---

## 의존성 및 실행 순서

```
Task 1 (메뉴+라우트) ──→ Task 2 (카드현황) ──→ Task 3 (승인) ──→ Task 4 (설정)
                                                                        │
Task 5 (공통메뉴) ──→ Task 6 (신청페이지) ◄─────────────────────────────┘
```

- Task 1은 반드시 먼저 실행 (메뉴 그룹 + 라우트 뼈대)
- Task 2, 3, 4는 순차 실행 (같은 branch에서 충돌 방지)
- Task 5는 Task 1 이후 언제든 실행 가능 (별도 파일)
- Task 6는 Task 5 이후 실행 (export 의존성)

## 공통 주의사항

1. **데이터**: 현재 API 없음. 모든 데이터는 더미 상수로 구현. 추후 API 연동 시 교체 가능하도록 데이터를 컴포넌트 상단에 분리 정의
2. **스타일**: Tailwind CSS + @jabis/ui 컴포넌트만 사용. 인라인 style 사용 금지
3. **아이콘**: lucide-react에서 import. buildMenu에서는 문자열로 전달
4. **파일명**: PascalCase + Page.jsx (예: CardStatusPage.jsx)
5. **submodule**: packages/ 내 수정은 jabis-common submodule 변경을 의미. 커밋 시 주의
