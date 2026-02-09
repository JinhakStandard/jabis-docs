# JABIS Design System — AI 프롬프트 참조 문서

> 이 문서는 AI가 JABIS 프론트엔드 코드를 생성할 때 참조하는 디자인 시스템 명세입니다.
> 라이브 문서: https://jabis-design.jinhakapply.com

---

## 1. 디자인 토큰

### 1.1 색상

| 토큰 | HEX | Tailwind 클래스 | 용도 |
|------|-----|-----------------|------|
| primary | #2B87EC | `bg-primary`, `text-primary` | 메인 액센트, 버튼, 링크 |
| primary-dark | #1A5FB4 | — | 호버 상태 |
| primary-light | #E8F2FC | — | 배경 하이라이트 |
| destructive | #EF4444 | `bg-destructive`, `text-destructive` | 삭제, 에러 |
| success | #22C55E | `bg-success`, `text-success` | 성공, 완료 |
| warning | #F59E0B | `bg-warning`, `text-warning` | 경고, 주의 |
| muted | #F3F4F6 | `bg-muted`, `text-muted-foreground` | 비활성, 보조 텍스트 |

### 1.2 시맨틱 CSS 변수 (다크모드 자동 전환)

```
--background, --foreground, --card, --card-foreground,
--popover, --popover-foreground, --border, --input,
--primary, --primary-foreground, --secondary, --secondary-foreground,
--accent, --accent-foreground, --destructive, --destructive-foreground,
--ring, --radius (0.5rem)
```

- 다크모드: `darkMode: ["class"]` → `.dark` 클래스 토글
- CSS 변수가 자동으로 다크 테마 값으로 전환되므로 `dark:` prefix 불필요

### 1.3 타이포그래피

- Font: `Pretendard, Inter, sans-serif`

| 클래스 | 용도 |
|--------|------|
| `text-3xl font-bold` | 페이지 제목 |
| `text-2xl font-bold` | 대시보드 제목 |
| `text-xl font-semibold` | 섹션 제목 |
| `text-lg font-medium` | 카드 제목 |
| `text-base` | 본문 |
| `text-sm` | 보조 텍스트 |
| `text-xs text-muted-foreground` | 캡션, 레이블 |

### 1.4 스페이싱 & Border Radius

| 스페이싱 | rem | px |
|----------|-----|----|
| 0.5 | 0.125rem | 2px |
| 1 | 0.25rem | 4px |
| 2 | 0.5rem | 8px |
| 3 | 0.75rem | 12px |
| 4 | 1rem | 16px |
| 6 | 1.5rem | 24px |
| 8 | 2rem | 32px |

| Radius | 값 |
|--------|-----|
| sm | 4px |
| md | 6px |
| lg | 8px |
| full | 9999px |

---

## 2. Import 경로

```jsx
// UI 컴포넌트
import { Button, Badge, Card, CardContent, CardHeader, CardTitle, ... } from '@jabis/ui'
import { cn } from '@jabis/ui'  // clsx + tailwind-merge 유틸

// 레이아웃
import { DashboardLayout } from '@jabis/layout'

// 코어
import { ThemeProvider, ThemeToggle, sites } from '@jabis/core'

// 인증
import { LoginPage, CallbackPage, useAuthStore, initAuthConfig, SessionTimeoutProvider } from '@jabis/auth'

// 차트
import { LineChart, Line, BarChart, Bar, PieChart, Pie, Cell, AreaChart, Area, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts'

// 아이콘
import { Users, Settings, LayoutDashboard, ... } from 'lucide-react'

// CSS (진입점에서 1회)
import '@jabis/ui/globals.css'
```

---

## 3. 기본 UI 컴포넌트

### 3.1 Button

```jsx
<Button>Default</Button>
<Button variant="destructive">삭제</Button>
<Button variant="outline">Outline</Button>
<Button variant="secondary">Secondary</Button>
<Button variant="ghost">Ghost</Button>
<Button variant="link">Link</Button>
<Button variant="success">Success</Button>
<Button variant="warning">Warning</Button>

// 사이즈: size="sm" | "default" | "lg" | "icon"
// 아이콘 조합
<Button><Download className="h-4 w-4 mr-2" />다운로드</Button>
<Button size="icon"><Plus className="h-4 w-4" /></Button>
```

### 3.2 Badge

```jsx
// 11가지 variant
<Badge>Default</Badge>
<Badge variant="secondary">Secondary</Badge>
<Badge variant="destructive">Destructive</Badge>
<Badge variant="success">Success</Badge>
<Badge variant="warning">Warning</Badge>
<Badge variant="info">Info</Badge>
<Badge variant="outline">Outline</Badge>
<Badge variant="pending">Pending</Badge>
<Badge variant="processing">Processing</Badge>
<Badge variant="completed">Completed</Badge>
<Badge variant="error">Error</Badge>
```

### 3.3 Card

```jsx
<Card>
  <CardHeader>
    <CardTitle>제목</CardTitle>
    <CardDescription>설명</CardDescription>
  </CardHeader>
  <CardContent>본문</CardContent>
  <CardFooter className="flex justify-end gap-2">
    <Button variant="outline" size="sm">취소</Button>
    <Button size="sm">확인</Button>
  </CardFooter>
</Card>
```

### 3.4 Input / Textarea / Label

```jsx
<div className="space-y-2">
  <Label htmlFor="name">이름</Label>
  <Input id="name" placeholder="입력하세요" />
</div>

// 검색 아이콘 Input
<div className="relative">
  <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-muted-foreground" />
  <Input className="pl-9" placeholder="검색..." />
</div>

<Textarea placeholder="설명" rows={3} />
```

### 3.5 Select

```jsx
<Select>
  <SelectTrigger>
    <SelectValue placeholder="선택하세요" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="opt1">옵션 1</SelectItem>
    <SelectItem value="opt2">옵션 2</SelectItem>
  </SelectContent>
</Select>
```

### 3.6 Checkbox / Switch

```jsx
<div className="flex items-center gap-2">
  <Checkbox id="terms" />
  <Label htmlFor="terms">이용약관 동의</Label>
</div>

<div className="flex items-center gap-2">
  <Switch id="notify" />
  <Label htmlFor="notify">알림 설정</Label>
</div>
```

### 3.7 Tabs

```jsx
<Tabs defaultValue="tab1">
  <TabsList>
    <TabsTrigger value="tab1">탭 1</TabsTrigger>
    <TabsTrigger value="tab2">탭 2</TabsTrigger>
  </TabsList>
  <TabsContent value="tab1">내용 1</TabsContent>
  <TabsContent value="tab2">내용 2</TabsContent>
</Tabs>
```

### 3.8 Table

```jsx
<Table>
  <TableHeader>
    <TableRow>
      <TableHead>이름</TableHead>
      <TableHead>상태</TableHead>
      <TableHead className="text-right">금액</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell className="font-medium">홍길동</TableCell>
      <TableCell><Badge variant="success">활성</Badge></TableCell>
      <TableCell className="text-right">₩1,250,000</TableCell>
    </TableRow>
  </TableBody>
</Table>
```

### 3.9 Dialog

```jsx
<Dialog>
  <DialogTrigger asChild>
    <Button variant="outline">열기</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>제목</DialogTitle>
      <DialogDescription>설명</DialogDescription>
    </DialogHeader>
    <div className="py-4">본문</div>
    <DialogFooter>
      <Button variant="outline">취소</Button>
      <Button>확인</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### 3.10 기타 컴포넌트

```jsx
// Accordion
<Accordion type="single" collapsible>
  <AccordionItem value="item-1">
    <AccordionTrigger>질문</AccordionTrigger>
    <AccordionContent>답변</AccordionContent>
  </AccordionItem>
</Accordion>

// Tooltip (반드시 TooltipProvider로 감싸야 함)
<TooltipProvider>
  <Tooltip>
    <TooltipTrigger asChild><Button>hover</Button></TooltipTrigger>
    <TooltipContent><p>설명</p></TooltipContent>
  </Tooltip>
</TooltipProvider>

// Progress
<Progress value={75} />

// Slider
<Slider value={[50]} onValueChange={setValue} max={100} step={1} />

// Avatar
<Avatar>
  <AvatarImage src="/photo.png" alt="User" />
  <AvatarFallback>홍</AvatarFallback>
</Avatar>

// Separator
<Separator />  // 수평
<Separator orientation="vertical" className="h-16" />  // 수직

// cn 유틸 (조건부 클래스 병합)
import { cn } from '@jabis/ui'
cn('px-4 py-2', isActive && 'bg-primary text-white', className)
```

---

## 4. 커스텀 컴포넌트

### 4.1 PageHeader

```jsx
import { PageHeader } from '@jabis/ui'
import { Users } from 'lucide-react'

<PageHeader
  icon={Users}           // 컴포넌트 참조 (JSX 아님!)
  title="사용자 관리"
  description="시스템 사용자를 관리합니다."
  actions={<Button size="sm">추가</Button>}
/>
```

### 4.2 StatCard

```jsx
import { StatCard } from '@jabis/ui'
import { Users } from 'lucide-react'

<StatCard
  title="총 사용자"
  icon={Users}           // 컴포넌트 참조
  value="1,250"
  subtitle="목표"
  subtitleValue="104.2%"
  trend={{ isPositive: true, label: '전월 대비', value: '+12.5%' }}
  iconBgColor="bg-success/10"   // 선택
  iconColor="text-success"       // 선택
/>
```

### 4.3 RatioCard

```jsx
import { RatioCard } from '@jabis/ui'

<RatioCard
  title="서버 가동률"
  value={99.9}
  status="good"          // good | warning | danger | normal
  description="임계값: 80%"
/>
```

### 4.4 AlertBanner

```jsx
import { AlertBanner } from '@jabis/ui'
import { AlertTriangle } from 'lucide-react'

<AlertBanner
  icon={AlertTriangle}   // 컴포넌트 참조
  title="주의 사항"
  variant="warning"       // warning | danger | info | success
  items={['서버 점검 예정', '백업을 진행해주세요.']}
/>
```

### 4.5 CollapsibleCard

```jsx
import { CollapsibleCard } from '@jabis/ui'
import { Info } from 'lucide-react'

<CollapsibleCard
  title="접을 수 있는 카드"
  icon={Info}             // 컴포넌트 참조
  defaultExpanded={true}
  storageKey="my-card"    // localStorage 키
  headerRight={<Badge>3건</Badge>}
>
  {children}
</CollapsibleCard>
```

### 4.6 Toast

```jsx
import { useToast } from '@jabis/ui'

const { showInfo, showSuccess, showWarning, showError } = useToast()

showInfo('알림', '정보 메시지입니다.')       // 우측 하단, 5초, 파란색
showSuccess('성공', '저장되었습니다.')      // 우측 하단, 5초, 초록색
showWarning('경고', '주의가 필요합니다.')   // 우측 하단, 5초, 노란색
showError('오류', '서버 연결 실패.')        // 중앙 모달, 10초, 빨간색, ESC 닫기
```

> **중요**: 커스텀 컴포넌트의 `icon` prop은 반드시 **컴포넌트 참조**로 전달해야 합니다.
> `icon={Users}` (O) / `icon={<Users />}` (X)

---

## 5. 레이아웃

### 5.1 DashboardLayout

```jsx
import { DashboardLayout } from '@jabis/layout'
import { ThemeToggle, sites } from '@jabis/core'

const menuItems = [
  { icon: 'LayoutDashboard', label: '대시보드', path: '/dashboard' },
  { icon: 'Users', label: '사용자 관리', items: [
    { icon: 'User', label: '목록', path: '/users' },
    { icon: 'UserPlus', label: '등록', path: '/users/new' },
  ]},
]

<DashboardLayout
  headerConfig={{ title: 'JABIS', shortTitle: 'J', logoSrc: '/logo.png' }}
  menuItems={menuItems}
  menuStateKey="admin"           // sessionStorage 키
  userName="홍길동"
  onLogout={logout}
  getSessionTimeInfo={getSessionTimeInfo}
  sites={sites}
  notifications={notifications}
  themeToggle={<ThemeToggle />}
  searchComponent={<SearchBar />}
>
  {children}
</DashboardLayout>
```

### 5.2 Header 주요 기능
- 로고, 앱 제목, 세션 타이머, 사이트 목록, 알림, 사용자 메뉴
- 세션 타이머 색상: 15분 이상(muted) → 5~15분(warning) → 5분 미만(destructive)

### 5.3 Sidebar
- 접기/펼치기: Desktop 64px/224px, Mobile 오버레이
- 접기 상태 sessionStorage 저장, 메뉴 그룹 접기 상태 localStorage 저장
- 아이콘: 문자열(PascalCase/kebab-case) → lucide-react 자동 변환 + 캐싱

### 5.4 앱 진입점 패턴 (main.jsx)

```jsx
import '@jabis/ui/globals.css'
import { BrowserRouter } from 'react-router-dom'
import { ThemeProvider } from '@jabis/core'
import { ToastProvider } from '@jabis/ui'
import { SessionTimeoutProvider } from '@jabis/auth'

<BrowserRouter basename="/부서명">
  <ThemeProvider defaultTheme="light" storageKey="jabis-theme">
    <ToastProvider>
      <SessionTimeoutProvider>
        <App />
      </SessionTimeoutProvider>
    </ToastProvider>
  </ThemeProvider>
</BrowserRouter>
```

---

## 6. 차트 (Recharts)

```jsx
import { LineChart, Line, BarChart, Bar, PieChart, Pie, Cell, AreaChart, Area, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts'

// JABIS 차트 컬러
const CHART_COLORS = {
  primary: '#2B87EC',
  secondary: '#1A5FB4',
  success: '#22C55E',
  warning: '#F59E0B',
  danger: '#EF4444',
}

// 기본 패턴
<ResponsiveContainer width="100%" height={300}>
  <LineChart data={data}>
    <CartesianGrid strokeDasharray="3 3" stroke="hsl(var(--border))" />
    <XAxis dataKey="name" tick={{ fontSize: 12 }} />
    <YAxis tick={{ fontSize: 12 }} />
    <Tooltip />
    <Legend />
    <Line type="monotone" dataKey="사용자" stroke="#2B87EC" strokeWidth={2} />
    <Line type="monotone" dataKey="매출" stroke="#22C55E" strokeWidth={2} />
  </LineChart>
</ResponsiveContainer>
```

지원 차트: LineChart, BarChart, PieChart, AreaChart

---

## 7. 아이콘 (lucide-react)

### 7.1 Sidebar 메뉴 아이콘 매핑

Sidebar의 `icon` 필드는 **문자열**로 지정 (자동으로 lucide-react 컴포넌트로 변환):

| 아이콘명 | 용도 |
|----------|------|
| `LayoutDashboard` | 대시보드 |
| `Users`, `User`, `UserPlus` | 사용자 관리 |
| `FileText`, `FolderOpen`, `File` | 문서/파일 |
| `Settings`, `Shield`, `Key` | 설정/보안 |
| `BarChart3`, `TrendingUp`, `PieChart` | 통계/차트 |
| `Bell`, `Mail`, `MessageSquare` | 알림/메시지 |
| `Calendar`, `Clock`, `Timer` | 일정/시간 |
| `Server`, `Database`, `HardDrive` | 인프라 |
| `GitBranch`, `Code`, `Terminal` | 개발 도구 |

### 7.2 상태 아이콘

| 아이콘 | 용도 | 색상 클래스 |
|--------|------|-------------|
| `CheckCircle` | 성공/완료 | `text-success` |
| `AlertTriangle` | 경고 | `text-warning` |
| `XCircle` | 에러/실패 | `text-destructive` |
| `Info` | 정보 | `text-primary` |
| `Loader2` | 로딩 (animate-spin) | `text-primary` |
| `AlertCircle` | 주의 | `text-destructive` |

### 7.3 사용법

```jsx
// 일반 컴포넌트에서 (직접 import)
import { Users, Plus, Trash2 } from 'lucide-react'
<Users className="h-5 w-5" />

// Sidebar 메뉴에서 (문자열)
{ icon: 'Users', label: '사용자', path: '/users' }

// 커스텀 컴포넌트에서 (컴포넌트 참조)
<StatCard icon={Users} ... />
<PageHeader icon={Users} ... />
```

---

## 8. 폼 패턴

### 8.1 검색 + 필터 바

```jsx
<div className="flex items-center gap-3">
  <div className="relative flex-1">
    <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-muted-foreground" />
    <Input className="pl-9" placeholder="검색..." value={search} onChange={e => setSearch(e.target.value)} />
  </div>
  <Select value={filter} onValueChange={setFilter}>
    <SelectTrigger className="w-[140px]">
      <SelectValue placeholder="전체" />
    </SelectTrigger>
    <SelectContent>
      <SelectItem value="all">전체</SelectItem>
      <SelectItem value="active">활성</SelectItem>
    </SelectContent>
  </Select>
  <Button variant="outline" size="icon"><Filter className="h-4 w-4" /></Button>
</div>
```

### 8.2 CRUD 다이얼로그

```jsx
<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>항목 추가</DialogTitle>
    </DialogHeader>
    <div className="space-y-4">
      <div className="space-y-2">
        <Label htmlFor="name">이름</Label>
        <Input id="name" value={name} onChange={e => setName(e.target.value)} />
      </div>
      <div className="space-y-2">
        <Label htmlFor="desc">설명</Label>
        <Textarea id="desc" rows={3} />
      </div>
    </div>
    <DialogFooter>
      <Button variant="outline" onClick={() => setOpen(false)}>취소</Button>
      <Button onClick={handleSave}>저장</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

---

## 9. 로딩 & 에러 상태

### 9.1 Skeleton

```jsx
function Skeleton({ className }) {
  return <div className={`animate-pulse rounded bg-muted ${className}`} />
}

// 카드 스켈레톤
<Card>
  <CardHeader>
    <Skeleton className="h-4 w-32" />
    <Skeleton className="h-3 w-48 mt-2" />
  </CardHeader>
  <CardContent className="space-y-2">
    <Skeleton className="h-3 w-full" />
    <Skeleton className="h-3 w-3/4" />
  </CardContent>
</Card>
```

### 9.2 Spinner

```jsx
import { Loader2 } from 'lucide-react'

// 기본
<Loader2 className="h-6 w-6 animate-spin text-primary" />

// 버튼 내부
<Button disabled>
  <Loader2 className="h-4 w-4 animate-spin mr-2" />처리 중...
</Button>
```

### 9.3 Empty State

```jsx
<div className="flex flex-col items-center justify-center py-12 text-center">
  <div className="p-3 rounded-full bg-muted mb-4">
    <Inbox className="h-8 w-8 text-muted-foreground" />
  </div>
  <h3 className="text-lg font-semibold">데이터가 없습니다</h3>
  <p className="text-sm text-muted-foreground mt-1">새 항목을 추가해보세요.</p>
  <Button className="mt-4" size="sm">항목 추가</Button>
</div>
```

### 9.4 Error Fallback

```jsx
<div className="flex flex-col items-center justify-center py-12 text-center">
  <div className="p-3 rounded-full bg-destructive/10 mb-4">
    <AlertCircle className="h-8 w-8 text-destructive" />
  </div>
  <h3 className="text-lg font-semibold">서버 오류</h3>
  <p className="text-sm text-muted-foreground mt-1">{error.message}</p>
  <Button variant="outline" className="mt-4" onClick={refetch}>
    <RefreshCw className="h-4 w-4 mr-2" />다시 시도
  </Button>
</div>
```

### 9.5 조합 패턴

```jsx
const { data, isLoading, error, refetch } = useQuery(...)

if (isLoading) return <SkeletonCard />
if (error) return <ErrorFallback title="오류" message={error.message} onRetry={refetch} />
if (!data?.length) return <EmptyState icon={Inbox} title="데이터 없음" ... />

return <DataTable data={data} />
```

---

## 10. 코딩 컨벤션

- Tailwind 클래스 사용 (인라인 스타일 지양)
- `cn()` 유틸로 조건부 클래스 병합
- 아이콘: `lucide-react` 사용
  - 일반 컴포넌트: `import { Icon } from 'lucide-react'` → `<Icon className="h-4 w-4" />`
  - Sidebar 메뉴: 문자열 아이콘명 (`'LayoutDashboard'`)
  - 커스텀 컴포넌트 icon prop: 컴포넌트 참조 (`icon={Users}`)
- 다크모드: CSS 변수 자동 전환 (시맨틱 토큰 사용 시 `dark:` prefix 불필요)
- 차트 색상: JABIS 브랜드 컬러 사용 (#2B87EC, #22C55E, #F59E0B, #EF4444)
