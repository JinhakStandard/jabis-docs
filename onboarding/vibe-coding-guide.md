# JABIS 바이브 코딩 대전제

> **이 문서는 JABIS 경영정보시스템을 확장해 나가면서 바이브 코딩을 수행할 때 반드시 준수해야 하는 대전제 문서입니다.**

---

## 1. JABIS 아키텍처 개요

### 1.1 시스템 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          K3s 클러스터                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │   인증서     │  │  API 게이트  │  │   인증서버   │                 │
│  │   관리       │  │  웨이(라우팅)│  │  (인증/인가) │                 │
│  └──────────────┘  └──────┬───────┘  └──────────────┘                 │
│                           │                                             │
│           ┌───────────────┼───────────────┬───────────────┐            │
│           │               │               │               │            │
│           ▼               ▼               ▼               ▼            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ JABIS 포털   │  │   전자결재   │  │   인사관리   │  │   재무     │ │
│  │  (AI 포털)   │  │ (/approval)  │  │    (/hr)     │  │ (/finance) │ │
│  │    (메인)    │  │ 경영지원실   │  │   인사팀     │  │  재무팀    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
│           │               │               │               │            │
│           └───────────────┴───────────────┴───────────────┘            │
│                                    │                                    │
│                                    ▼                                    │
│                          ┌──────────────────┐                          │
│                          │    공용 API      │                          │
│                          │   (REST 방식)    │                          │
│                          └────────┬─────────┘                          │
│                                   │                                    │
│                                   ▼                                    │
│                          ┌──────────────────┐                          │
│                          │   데이터베이스   │                          │
│                          └──────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 확장 철학

1. **마이크로 프론트엔드**: 각 부서가 독립적으로 리액트 앱을 개발
2. **라우팅 기반 통합**: API 게이트웨이에서 URL 경로별로 적절한 파드로 분기
3. **점진적 확장**: 기능 단위로 하나씩 붙여가며 경영정보시스템으로 성장
4. **바이브 코딩 친화**: AI 도구를 활용한 빠른 개발 지원

### 1.3 핵심 인프라

| 구성요소 | 역할 | 우선순위 |
|----------|------|----------|
| **인증서 관리** | SSL/TLS 인증서 관리 (Let's Encrypt) | 1순위 |
| **API 게이트웨이** | 라우팅, 인증 토큰 검증, 부하 분산 | 1순위 |
| **JABIS AI 포털** | 메인 포털, AI 기능, 통합 대시보드 | 2순위 |
| **각 부서별 앱** | 전자결재, 인사, 재무 등 | 3순위~ |

---

## 2. 필수 준수 사항

> **바이브 코딩 시 반드시 준수해야 하는 규칙입니다. 이를 어기면 통합 시 문제가 발생합니다.**

### 2.1 디자인 시스템 준수

#### 필수 가져오기

```jsx
// 모든 UI 컴포넌트는 @jabis/ui에서 가져오기
import { Button, Card, Dialog, Input, Badge, cn } from '@jabis/ui'

// 아이콘은 lucide-react 사용
import { Plus, Settings, Users } from 'lucide-react'
```

#### 금지 사항

```jsx
// ❌ 절대 금지 - 외부 UI 라이브러리 사용
import { Button } from '@mui/material'
import { Button } from 'antd'
import { Button } from 'react-bootstrap'

// ❌ 절대 금지 - 인라인 스타일
<div style={{ color: 'red', padding: '10px' }} />

// ❌ 절대 금지 - 색상 직접 지정
<div className="bg-[#FF5733]" />
```

#### 색상 사용 규칙

```jsx
// ✅ 올바른 색상 사용 - 의미 기반 토큰
<div className="bg-primary text-primary-foreground" />
<div className="bg-destructive text-destructive-foreground" />
<div className="bg-success text-success-foreground" />
<div className="bg-warning text-warning-foreground" />
<div className="bg-muted text-muted-foreground" />

// ✅ 상태별 색상
<Badge variant="success">승인</Badge>
<Badge variant="warning">대기</Badge>
<Badge variant="destructive">거절</Badge>
<Badge variant="pending">처리중</Badge>
```

### 2.2 컴포넌트 사용 규칙

#### 버튼 사용

```jsx
// 주요 액션
<Button variant="default">저장</Button>

// 보조 액션
<Button variant="secondary">취소</Button>
<Button variant="outline">이전</Button>

// 위험한 액션
<Button variant="destructive">삭제</Button>

// 성공/경고 액션
<Button variant="success">승인</Button>
<Button variant="warning">보류</Button>

// 크기
<Button size="sm">작은 버튼</Button>
<Button size="default">기본 버튼</Button>
<Button size="lg">큰 버튼</Button>
```

#### 카드 사용

```jsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@jabis/ui'

<Card>
  <CardHeader>
    <CardTitle>제목</CardTitle>
    <CardDescription>설명 텍스트</CardDescription>
  </CardHeader>
  <CardContent>
    {/* 내용 */}
  </CardContent>
  <CardFooter>
    <Button>액션</Button>
  </CardFooter>
</Card>
```

#### 다이얼로그(모달) 사용

```jsx
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from '@jabis/ui'

<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent className="sm:max-w-lg">
    <DialogHeader>
      <DialogTitle>다이얼로그 제목</DialogTitle>
    </DialogHeader>
    {/* 내용 */}
    <DialogFooter>
      <Button variant="outline" onClick={() => setIsOpen(false)}>취소</Button>
      <Button onClick={handleSubmit}>확인</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### 2.3 레이아웃 규칙

#### 페이지 기본 구조

```jsx
export default function SomePage() {
  return (
    <div className="h-full flex flex-col">
      {/* 페이지 헤더 */}
      <div className="border-b bg-background px-6 py-4">
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="p-2 rounded-lg bg-primary/10">
              <Icon className="h-5 w-5 text-primary" />
            </div>
            <div>
              <h1 className="text-xl font-semibold">페이지 제목</h1>
              <p className="text-sm text-muted-foreground">설명</p>
            </div>
          </div>
          <Button>액션</Button>
        </div>
      </div>

      {/* 콘텐츠 영역 */}
      <div className="flex-1 overflow-auto p-6">
        {/* 실제 콘텐츠 */}
      </div>
    </div>
  )
}
```

#### 반응형 그리드

```jsx
// 통계 카드 (4열)
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">

// 2열 레이아웃
<div className="grid grid-cols-1 lg:grid-cols-2 gap-6">

// 3열 레이아웃
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
```

---

## 3. API 통신 규칙

### 3.1 HTTP 메서드 규칙

> **JABIS는 GET과 POST만 사용합니다.**

| 메서드 | 용도 |
|--------|------|
| `GET` | 데이터 조회 (쿼리 매개변수로 필터링) |
| `POST` | 생성, 수정, 삭제 (action 필드로 구분) |

```javascript
// ❌ 사용하지 않음
PUT, PATCH, DELETE

// ✅ POST + action 패턴
// 생성
POST /api/approval/documents
본문: { action: 'create', data: { title: '...' } }

// 수정
POST /api/approval/documents
본문: { action: 'update', id: 'doc-001', data: { title: '...' } }

// 삭제
POST /api/approval/documents
본문: { action: 'delete', id: 'doc-001' }
```

### 3.2 API 응답 형식

```javascript
// 성공 응답
{
  "success": true,
  "data": { ... },
  "message": "처리가 완료되었습니다."
}

// 실패 응답
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "필수 항목이 누락되었습니다.",
    "details": [...]
  }
}

// 목록 응답 (페이지 나누기)
{
  "success": true,
  "data": {
    "items": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    }
  }
}
```

### 3.3 인증 토큰

```javascript
// 모든 API 요청에 인증 헤더 포함
const headers = {
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${accessToken}`,
}

// 토큰 만료 시 자동 갱신 로직 필수
```

---

## 4. 상태 관리 규칙

### 4.1 Zustand 스토어 패턴

```javascript
import { create } from 'zustand'

const STORAGE_KEY = 'feature_state'

export const useFeatureStore = create((set, get) => ({
  // ======== 상태 ========
  items: [],
  selectedId: null,
  isLoading: false,

  // ======== 설정자 ========
  setItems: (items) => set({ items }),
  setSelectedId: (id) => set({ selectedId: id }),
  setLoading: (loading) => set({ isLoading: loading }),

  // ======== 접근자 ========
  getItemById: (id) => get().items.find(item => item.id === id),

  // ======== 데이터 조작 ========
  addItem: (item) => set((state) => ({
    items: [...state.items, item]
  })),

  updateItem: (id, updates) => set((state) => ({
    items: state.items.map(item =>
      item.id === id ? { ...item, ...updates } : item
    )
  })),

  deleteItem: (id) => set((state) => ({
    items: state.items.filter(item => item.id !== id)
  })),

  // ======== 다이얼로그 상태 ========
  dialogs: {
    form: { open: false, mode: 'create', data: null },
    delete: { open: false, id: null },
  },

  openDialog: (name, payload = {}) => set((state) => ({
    dialogs: { ...state.dialogs, [name]: { open: true, ...payload } }
  })),

  closeDialog: (name) => set((state) => ({
    dialogs: { ...state.dialogs, [name]: { ...state.dialogs[name], open: false } }
  })),
}))
```

### 4.2 스토어 사용

```jsx
// 개별 상태 선택 (리렌더링 최적화)
const items = useFeatureStore((state) => state.items)
const addItem = useFeatureStore((state) => state.addItem)

// 여러 상태 한번에 (필요시만)
const { items, isLoading } = useFeatureStore()
```

---

## 5. 이름 짓기 규칙

### 5.1 파일/폴더 이름 짓기

| 대상 | 규칙 | 예시 |
|------|------|------|
| 컴포넌트 파일 | 파스칼케이스 | `ApprovalList.jsx`, `DocumentForm.jsx` |
| 스토어 파일 | 카멜케이스 + Store | `approvalStore.js` |
| 유틸리티 파일 | 카멜케이스 | `formatDate.js`, `validators.js` |
| 상수 파일 | 카멜케이스 | `constants.js`, `config.js` |
| 페이지 폴더 | 케밥케이스 | `pages/approval-list/` |

### 5.2 코드 이름 짓기

| 대상 | 규칙 | 예시 |
|------|------|------|
| 컴포넌트명 | 파스칼케이스 | `ApprovalCard`, `DocumentTable` |
| 함수/변수 | 카멜케이스 | `handleSubmit`, `isLoading` |
| 상수 | 대문자_스네이크케이스 | `API_BASE_URL`, `MAX_FILE_SIZE` |
| 이벤트 핸들러 | handle + 동작 | `handleClick`, `handleSubmit` |
| 불리언 변수 | is/has/can 접두사 | `isOpen`, `hasError`, `canEdit` |

### 5.3 ID 형식

```javascript
// 접두사 + 고유값 형식
{
  id: 'approval-001',
  documentId: 'doc-2024-001',
  userId: 'user-kim',
}
```

---

## 6. 폴더 구조 규칙

### 6.1 각 부서별 앱 구조

```
apps/[부서명]/
├── src/
│   ├── components/         # 앱 전용 컴포넌트
│   │   ├── common/         # 공통 컴포넌트
│   │   ├── [기능명]/       # 기능별 컴포넌트
│   │   └── layout/         # 레이아웃 컴포넌트
│   ├── pages/              # 페이지 컴포넌트
│   ├── stores/             # Zustand 스토어
│   ├── hooks/              # 커스텀 훅
│   ├── utils/              # 유틸리티 함수
│   ├── services/           # API 호출 로직
│   ├── data/               # 상수, 타입 정의
│   ├── App.jsx             # 라우팅 설정
│   ├── main.jsx            # 진입점
│   └── index.css           # 스타일 (globals.css 가져오기)
├── tailwind.config.js      # @jabis/ui 프리셋 사용
├── vite.config.js
└── package.json
```

### 6.2 Tailwind 설정

```javascript
// tailwind.config.js
const preset = require('@jabis/ui/tailwind.preset')

module.exports = {
  presets: [preset],
  content: [
    "./index.html",
    "./src/**/*.{js,jsx}",
    "../../packages/ui/src/**/*.{js,jsx}",
  ],
}
```

### 6.3 CSS 설정

```css
/* src/index.css */
@import '@jabis/ui/globals.css';

/* 앱 전용 스타일은 여기에 추가 */
```

---

## 7. 배포 및 통합 규칙

### 7.1 라우팅 경로 규칙

| 부서 | 경로 | 파드 이름 |
|------|------|----------|
| 메인 포털 | `/` | jabis-portal |
| 전자결재 | `/approval/*` | jabis-approval |
| 인사관리 | `/hr/*` | jabis-hr |
| 재무관리 | `/finance/*` | jabis-finance |
| 자산관리 | `/asset/*` | jabis-asset |

### 7.2 빌드 설정

```javascript
// vite.config.js
export default {
  base: '/approval/', // 부서별 기본 경로 설정
  build: {
    outDir: 'dist',
  },
}
```

### 7.3 환경 변수

```env
# .env.production
VITE_API_BASE_URL=https://api.jabis.company.com
VITE_AUTH_URL=https://auth.jabis.company.com
VITE_APP_NAME=전자결재
VITE_APP_VERSION=1.0.0
```

---

## 8. 코드 품질 가이드라인

### 8.1 컴포넌트 작성 원칙

1. **단일 책임 원칙**: 한 컴포넌트는 한 가지 역할만
2. **Props 명확성**: JSDoc으로 props 문서화
3. **재사용성 고려**: 범용적으로 사용 가능하게 설계

```jsx
/**
 * 결재 문서 카드 컴포넌트
 *
 * @param {string} title - 문서 제목
 * @param {string} status - 결재 상태 (pending, approved, rejected)
 * @param {string} requester - 요청자 이름
 * @param {string} date - 요청 날짜
 * @param {Function} onClick - 클릭 핸들러
 */
export default function ApprovalCard({ title, status, requester, date, onClick }) {
  // ...
}
```

### 8.2 금지 사항

```jsx
// ❌ 인라인 스타일 금지
<div style={{ marginTop: 20 }} />

// ❌ 색상 직접 지정 금지
<div className="bg-[#FF0000]" />

// ❌ 외부 UI 라이브러리 금지
import { Button } from '@mui/material'

// ❌ console.log 남기기 금지 (운영 환경)
console.log('debug:', data)

// ❌ 한글 변수명 금지
const 결재문서 = []

// ❌ 직접 DOM 조작 금지
document.getElementById('app').innerHTML = ''
```

### 8.3 권장 사항

```jsx
// ✅ cn() 유틸리티로 조건부 클래스
<div className={cn(
  "base-class",
  isActive && "active-class",
  className
)} />

// ✅ 조기 반환 패턴
if (!data) return <EmptyState />
if (isLoading) return <Loading />

// ✅ 구조 분해 할당
const { title, status } = document

// ✅ 옵셔널 체이닝
const userName = user?.profile?.name ?? '알 수 없음'
```

---

## 9. 바이브 코딩 점검표

### 9.1 시작 전 확인

- [ ] `@jabis/ui` 패키지가 의존성에 있는가?
- [ ] `tailwind.config.js`에 `@jabis/ui` 프리셋이 적용되어 있는가?
- [ ] `index.css`에 `@jabis/ui/globals.css`가 가져오기 되어 있는가?
- [ ] 라우팅 경로가 API 게이트웨이 규칙에 맞는가?

### 9.2 개발 중 확인

- [ ] 모든 UI 컴포넌트를 `@jabis/ui`에서 가져오는가?
- [ ] 색상은 의미 기반 토큰(primary, destructive 등)을 사용하는가?
- [ ] API 호출은 GET/POST만 사용하는가?
- [ ] 스토어 패턴을 따르고 있는가?
- [ ] 이름 짓기 규칙을 준수하는가?

### 9.3 완료 전 확인

- [ ] 인라인 스타일이 없는가?
- [ ] 직접 지정된 색상이 없는가?
- [ ] console.log가 모두 제거되었는가?
- [ ] 타입스크립트 오류가 없는가? (JS 사용 시 JSDoc 검증)
- [ ] 반응형 디자인이 적용되었는가?
- [ ] 다크모드에서 정상 동작하는가?

### 9.4 통합 전 확인

- [ ] 빌드가 정상적으로 완료되는가?
- [ ] 기본 경로가 올바르게 설정되었는가?
- [ ] 환경 변수가 올바르게 설정되었는가?
- [ ] API 연동 테스트가 완료되었는가?

---

## 10. 자주 하는 실수와 해결책

### 10.1 스타일 관련

| 실수 | 해결책 |
|------|--------|
| MUI/Ant Design 컴포넌트 사용 | `@jabis/ui` 컴포넌트로 교체 |
| `#FF0000` 같은 색상 직접 지정 | `bg-destructive`, `text-destructive` 사용 |
| `style={{ }}` 인라인 스타일 | Tailwind 클래스 사용 |
| 자체 버튼 스타일 정의 | `<Button variant="...">` 사용 |

### 10.2 API 관련

| 실수 | 해결책 |
|------|--------|
| PUT/DELETE 메서드 사용 | POST + action 패턴으로 변경 |
| 인증 토큰 미포함 | Authorization 헤더 추가 |
| 오류 처리 미흡 | try-catch + 사용자 피드백 |

### 10.3 상태 관리 관련

| 실수 | 해결책 |
|------|--------|
| useState 남발 | Zustand 스토어로 중앙 관리 |
| props 전달 지옥 | Zustand 스토어 직접 접근 |
| 불필요한 리렌더링 | 선택자 함수로 필요한 상태만 구독 |

---

## 11. AI 표준 적용 확인

JABIS 프로젝트에서 바이브 코딩을 시작하기 전에 **JINHAK AI 개발 표준**이 적용되어 있는지 확인하세요.

### 확인 방법
- 프로젝트 루트에 `CLAUDE.md`가 있는지 확인
- `CLAUDE.md` 하단에 `jinhak_standard_version` 메타정보가 있는지 확인
- `.claude/settings.json`에 deny 규칙이 설정되어 있는지 확인

### 미적용 시
```
/apply-standard
```
또는 [AI 표준 세팅 가이드](ai-standard-setup.md)의 빠른 적용 프롬프트를 사용하세요.

### 적용 후 사용 가능한 명령어
```
/session-start          # 세션 시작 (이전 작업 확인 + 표준 버전 체크)
/commit                 # 표준에 맞는 커밋 생성
/review-pr 123          # PR을 표준 기준으로 리뷰
/test                   # 테스트 실행 및 결과 분석
```

---

## 12. 참조 문서

| 문서 | 경로 | 설명 |
|------|------|------|
| AI 협업 정책 | `policies/ai-collaboration.md` | Claude Code 설정, hooks, skills 상세 |
| AI 표준 세팅 가이드 | `onboarding/ai-standard-setup.md` | 프로젝트에 표준 적용하는 방법 |
| 개발 규칙 | `CLAUDE.md` | 전체 코딩 컨벤션 |
| 디자인 시스템 | `claude.design.md` | UI/UX 디자인 가이드 |
| 아키텍처 | `.ai/ARCHITECTURE.md` | 시스템 구조 문서 |
| 의사결정 기록 | `.ai/DECISIONS.md` | 기술 결정 사항 |
| 세션 로그 | `.ai/SESSION_LOG.md` | 작업 기록 |
| 전사 AI 표준 저장소 | https://github.com/JinhakStandard/ai-vibecoding | JINHAK AI 개발 표준 원본 |

---

## 12. 긴급 연락처

| 담당 | 역할 | 문의 사항 |
|------|------|-----------|
| 플랫폼팀 | 인프라 관리 | K3s, API 게이트웨이, 배포 |
| UI/UX팀 | 디자인 시스템 | `@jabis/ui`, 디자인 가이드 |
| 백엔드팀 | API 개발 | API 명세, 인증/인가 |

---

> **이 문서를 준수하면 어떤 부서에서 바이브 코딩으로 개발하더라도 JABIS 경영정보시스템에 원활하게 통합될 수 있습니다.**
>
> 문서 버전: 1.1.0
> 최종 수정: 2025-05-30
