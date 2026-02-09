# 진학어플라이 AI 프롬프트 템플릿

## 목차
1. [기본 시스템 프롬프트](#1-기본-시스템-프롬프트)
2. [페이지별 프롬프트 샘플](#2-페이지별-프롬프트-샘플)
3. [컴포넌트별 프롬프트 샘플](#3-컴포넌트별-프롬프트-샘플)
4. [사용 팁](#4-사용-팁)

---

## 1. 기본 시스템 프롬프트

> 모든 프로젝트에 공통으로 사용하는 디자인 시스템 정의 프롬프트입니다.
> 새로운 페이지나 컴포넌트를 요청할 때 이 프롬프트를 먼저 제공하세요.

```
당신은 진학어플라이의 프론트엔드 개발자입니다.
아래 디자인 시스템을 준수하여 React + Shadcn/ui + Tailwind CSS 코드를 작성하세요.

## 디자인 시스템

### 브랜드 정보
- 회사명: 진학어플라이
- 서비스: 원서접수 플랫폼
- 디자인 컨셉: "Trust & Clarity" (신뢰와 명확함)

### 컬러 팔레트
Primary Colors:
- primary: #2B87EC (Blue 400) - 주요 버튼, 링크, 강조
- primary-dark: #1A5FB4 (Blue 600) - 호버, 액티브 상태
- primary-light: #E8F2FC (Blue 100) - 배경 하이라이트

Neutral Colors:
- foreground: #1F2937 (Gray 800) - 제목
- muted-foreground: #6B7280 (Gray 500) - 보조 텍스트
- background: #FFFFFF - 기본 배경
- muted: #F3F4F6 (Gray 100) - 섹션 배경
- border: #E5E7EB (Gray 200) - 테두리

Semantic Colors:
- success: #22C55E (Green 500)
- warning: #F59E0B (Amber 500)
- destructive: #EF4444 (Red 500)

### 타이포그래피
- 한글 폰트: Pretendard
- 영문 폰트: Inter
- 제목: font-semibold 또는 font-bold
- 본문: font-normal

### 스타일 규칙
- 모서리: rounded-lg (8px) 카드, rounded-md (6px) 버튼, rounded (4px) 인풋
- 그림자: shadow-sm 기본, shadow-md 모달/드롭다운
- 여백: 8의 배수 사용 (p-2, p-4, p-6, p-8 = 8, 16, 24, 32px)
- 간격: space-y-4, gap-4 등 일관된 간격 유지

### 컴포넌트 스타일
- 버튼: Shadcn Button 사용, Primary는 bg-[#2B87EC] hover:bg-[#1A5FB4]
- 카드: Shadcn Card 사용, border border-gray-200 rounded-lg
- 입력: Shadcn Input 사용, focus:ring-[#2B87EC]
- 테이블: Shadcn Table 사용, 줄무늬 배경 적용

### UX 원칙
- 명확한 시각적 계층 구조
- 충분한 여백으로 가독성 확보
- 진행 상태는 항상 명확하게 표시
- 에러/성공 메시지는 즉시 피드백
- 모바일 반응형 필수 고려
```

---

## 2. 페이지별 프롬프트 샘플

### 2.1 대시보드 페이지

```
[기본 시스템 프롬프트 포함]

## 요청사항
내부 관리자용 대시보드 페이지를 만들어주세요.

### 페이지 구성
1. 상단 헤더
   - 좌측: 페이지 제목 "대시보드"
   - 우측: 날짜 범위 선택 (DateRangePicker)

2. 통계 카드 섹션 (4열 그리드)
   - 오늘 접수 건수
   - 이번 주 접수 건수
   - 처리 대기 건수
   - 완료 건수
   - 각 카드에 전일/전주 대비 증감률 표시

3. 차트 섹션 (2열 그리드)
   - 좌측: 일별 접수 추이 (Line Chart)
   - 우측: 학교별 접수 현황 (Bar Chart)

4. 최근 접수 목록 테이블
   - 컬럼: 접수번호, 학교명, 지원자명, 접수일시, 상태
   - 상태 배지: 대기(노랑), 처리중(파랑), 완료(초록), 반려(빨강)
   - 페이지당 10건, 페이지네이션

### 사용 컴포넌트
- Shadcn: Card, Table, Badge, Button, DateRangePicker
- 차트: Recharts (LineChart, BarChart)
- 아이콘: Lucide React

### 추가 요구사항
- 반응형: 모바일에서 카드 1열, 차트 1열로 변경
- 로딩 상태: Skeleton 컴포넌트 사용
- 데이터는 목업으로 작성
```

---

### 2.2 목록/테이블 페이지

```
[기본 시스템 프롬프트 포함]

## 요청사항
원서 접수 목록 페이지를 만들어주세요.

### 페이지 구성
1. 상단 영역
   - 페이지 제목: "원서 접수 관리"
   - 우측: "새 원서 등록" 버튼 (Primary)

2. 필터/검색 영역
   - 검색: 지원자명, 접수번호로 검색
   - 필터: 상태 (전체/대기/처리중/완료/반려), 기간, 학교
   - 필터 초기화 버튼

3. 테이블
   - 컬럼: 체크박스, 접수번호, 학교명, 모집단위, 지원자명, 연락처, 접수일시, 상태, 액션
   - 정렬: 컬럼 헤더 클릭으로 정렬
   - 액션: 상세보기, 수정, 삭제 (드롭다운 메뉴)

4. 하단 영역
   - 좌측: 선택된 항목 일괄 처리 버튼
   - 우측: 페이지네이션 (이전/다음, 페이지 번호)

### 상태 배지 색상
- 대기: bg-amber-100 text-amber-800
- 처리중: bg-blue-100 text-blue-800
- 완료: bg-green-100 text-green-800
- 반려: bg-red-100 text-red-800

### 사용 컴포넌트
- Shadcn: Table, Input, Select, Button, Badge, DropdownMenu, Checkbox, Pagination

### 추가 요구사항
- 빈 상태: 데이터 없을 때 안내 메시지
- 로딩: 테이블 Skeleton
- 행 호버 효과: hover:bg-gray-50
```

---

### 2.3 폼/입력 페이지

```
[기본 시스템 프롬프트 포함]

## 요청사항
원서 접수 등록 폼 페이지를 만들어주세요.

### 페이지 구성
1. 상단
   - 뒤로가기 버튼 + 페이지 제목 "원서 등록"
   - 우측: 임시저장, 등록 버튼

2. 진행 스텝 표시
   - Step 1: 기본정보 → Step 2: 지원정보 → Step 3: 서류첨부 → Step 4: 확인

3. 폼 영역 (카드 내부)
   
   [Step 1: 기본정보]
   - 지원자명 (필수)
   - 생년월일 (DatePicker, 필수)
   - 성별 (Radio: 남/여)
   - 연락처 (필수, 휴대폰 형식 검증)
   - 이메일 (필수, 이메일 형식 검증)
   - 주소 (우편번호 검색 + 상세주소)

   [Step 2: 지원정보]
   - 지원 학교 (Select, 필수)
   - 모집단위 (Select, 학교 선택 후 활성화)
   - 전형 유형 (Select)
   - 지원 동기 (Textarea, 최대 500자)

   [Step 3: 서류첨부]
   - 증명사진 (이미지 업로드, 미리보기)
   - 성적증명서 (PDF 업로드)
   - 기타 서류 (다중 파일 업로드)

   [Step 4: 확인]
   - 입력 정보 요약 표시
   - 개인정보 동의 체크박스

4. 하단 버튼
   - 이전 단계 / 다음 단계 (또는 제출)

### 폼 검증
- 필수 필드 표시: 라벨 옆 빨간 * 표시
- 실시간 검증: 입력 시 즉시 에러 메시지
- 에러 스타일: border-red-500, 하단에 빨간 텍스트

### 사용 컴포넌트
- Shadcn: Card, Input, Select, RadioGroup, Checkbox, Button, DatePicker, Textarea
- Form: React Hook Form + Zod 검증
- 파일업로드: 드래그앤드롭 지원

### 추가 요구사항
- 자동저장: 30초마다 임시저장
- 이탈 방지: 작성 중 페이지 이탈 시 확인 모달
- 반응형: 모바일에서 1열 레이아웃
```

---

### 2.4 로그인 페이지

```
[기본 시스템 프롬프트 포함]

## 요청사항
관리자 로그인 페이지를 만들어주세요.

### 페이지 구성
1. 전체 레이아웃
   - 화면 중앙 정렬
   - 배경: 연한 그라데이션 또는 패턴 (primary-light 활용)

2. 로그인 카드 (최대 너비 400px)
   - 상단: 진학어플라이 로고 + "관리자 로그인" 텍스트
   - 아이디 입력 (아이콘 포함)
   - 비밀번호 입력 (아이콘 + 표시/숨김 토글)
   - 로그인 상태 유지 체크박스
   - 로그인 버튼 (Primary, 전체 너비)
   - 구분선
   - 아이디 찾기 | 비밀번호 찾기 링크

3. 하단 푸터
   - 저작권 표시
   - 이용약관, 개인정보처리방침 링크

### 상호작용
- 로그인 버튼: 로딩 스피너 표시
- 에러: 카드 상단에 Alert 컴포넌트로 표시
- Enter 키로 로그인 가능

### 사용 컴포넌트
- Shadcn: Card, Input, Button, Checkbox, Alert
- 아이콘: Lucide (User, Lock, Eye, EyeOff)

### 추가 요구사항
- 다크모드 지원 (선택사항)
- 모바일 최적화
```

---

### 2.5 상세 보기 페이지

```
[기본 시스템 프롬프트 포함]

## 요청사항
원서 접수 상세 페이지를 만들어주세요.

### 페이지 구성
1. 상단 헤더
   - 뒤로가기 + 접수번호 표시
   - 상태 배지
   - 액션 버튼: 수정, 인쇄, 삭제

2. 정보 카드들 (2열 그리드)
   
   [지원자 정보 카드]
   - 증명사진 (좌측)
   - 이름, 생년월일, 연락처, 이메일, 주소

   [지원 정보 카드]
   - 지원학교, 모집단위, 전형유형
   - 지원동기 (접기/펼치기)

   [첨부서류 카드]
   - 파일 목록 (아이콘 + 파일명 + 다운로드)
   - 미리보기 버튼

   [처리 이력 카드]
   - 타임라인 형태
   - 일시, 처리자, 처리내용, 비고

3. 하단 처리 영역 (관리자용)
   - 처리 상태 변경 Select
   - 처리 메모 Textarea
   - 저장 버튼

### 사용 컴포넌트
- Shadcn: Card, Badge, Button, Select, Textarea, Separator
- 타임라인: 커스텀 또는 수직 스텝퍼 활용

### 추가 요구사항
- 인쇄 스타일시트 포함
- 반응형: 모바일에서 1열
```

---

## 3. 컴포넌트별 프롬프트 샘플

### 3.1 공통 헤더 컴포넌트

```
[기본 시스템 프롬프트 포함]

## 요청사항
내부 시스템용 공통 헤더 컴포넌트를 만들어주세요.

### 구성요소
1. 좌측
   - 로고 (이미지 또는 텍스트 "진학어플라이")
   - 메인 네비게이션 (대시보드, 원서관리, 학교관리, 설정)

2. 우측
   - 알림 아이콘 (뱃지로 개수 표시)
   - 사용자 드롭다운 (프로필, 설정, 로그아웃)

### Props
- currentPath: string (현재 활성 메뉴 표시용)
- user: { name: string, role: string, avatar?: string }
- notifications: number

### 반응형
- 모바일: 햄버거 메뉴로 변경, Sheet 컴포넌트 사용

### 사용 컴포넌트
- Shadcn: NavigationMenu, DropdownMenu, Sheet, Avatar, Badge
- 아이콘: Lucide (Bell, User, Menu, LogOut, Settings)
```

---

### 3.2 데이터 테이블 컴포넌트

```
[기본 시스템 프롬프트 포함]

## 요청사항
재사용 가능한 데이터 테이블 컴포넌트를 만들어주세요.

### 기능
- 컬럼 정의 (columns prop)
- 데이터 바인딩 (data prop)
- 정렬 (sortable columns)
- 선택 (체크박스, 단일/다중)
- 페이지네이션
- 로딩 상태 (Skeleton)
- 빈 상태 메시지

### Props 인터페이스
interface DataTableProps<T> {
  columns: ColumnDef<T>[]
  data: T[]
  loading?: boolean
  selectable?: boolean
  onSelectionChange?: (selected: T[]) => void
  pagination?: {
    pageSize: number
    pageIndex: number
    totalCount: number
    onPageChange: (page: number) => void
  }
  emptyMessage?: string
}

### 스타일
- 헤더: bg-gray-50 font-medium
- 행: hover:bg-gray-50, 줄무늬 (even:bg-gray-50/50)
- 정렬 아이콘: 오름차순/내림차순 표시

### 사용 라이브러리
- @tanstack/react-table
- Shadcn: Table, Checkbox, Button, Skeleton
```

---

### 3.3 통계 카드 컴포넌트

```
[기본 시스템 프롬프트 포함]

## 요청사항
대시보드용 통계 카드 컴포넌트를 만들어주세요.

### Props
interface StatCardProps {
  title: string
  value: string | number
  change?: {
    value: number
    type: 'increase' | 'decrease'
  }
  icon?: LucideIcon
  description?: string
  loading?: boolean
}

### 디자인
- 카드 배경: 흰색, border, rounded-lg
- 아이콘: 좌측 상단, primary 배경의 원형 안에
- 제목: text-sm text-muted-foreground
- 값: text-2xl font-bold
- 변화량: 증가(초록, ↑), 감소(빨강, ↓)
- 설명: text-xs text-muted-foreground

### 호버 효과
- shadow-md로 살짝 떠오르는 효과
- transition-all duration-200

### 사용 컴포넌트
- Shadcn: Card, Skeleton
- 아이콘: Lucide (TrendingUp, TrendingDown)
```

---

### 3.4 파일 업로드 컴포넌트

```
[기본 시스템 프롬프트 포함]

## 요청사항
드래그앤드롭 파일 업로드 컴포넌트를 만들어주세요.

### Props
interface FileUploadProps {
  accept?: string  // e.g., "image/*,.pdf"
  multiple?: boolean
  maxSize?: number  // bytes
  maxFiles?: number
  value?: File[]
  onChange: (files: File[]) => void
  onError?: (error: string) => void
}

### UI 상태
1. 기본 상태
   - 점선 테두리 박스
   - 업로드 아이콘 + "파일을 드래그하거나 클릭하여 업로드"
   - 허용 형식, 최대 크기 안내

2. 드래그 오버 상태
   - 테두리 색상 primary로 변경
   - 배경 primary-light

3. 업로드된 파일 목록
   - 파일 아이콘 + 파일명 + 크기
   - 삭제 버튼 (X)
   - 이미지인 경우 썸네일 미리보기

4. 에러 상태
   - 에러 메시지 표시 (빨간색)

### 검증
- 파일 크기 초과 시 에러
- 허용되지 않은 형식 에러
- 최대 파일 개수 초과 에러

### 사용 컴포넌트
- react-dropzone
- Shadcn: Button
- 아이콘: Lucide (Upload, File, Image, X)
```

---

## 4. 사용 팁

### 4.1 프롬프트 사용 순서

```
1. 기본 시스템 프롬프트를 먼저 제공
2. 원하는 페이지/컴포넌트 프롬프트 추가
3. 프로젝트 특성에 맞게 요구사항 수정
4. 생성된 코드 검토 후 피드백으로 수정 요청
```

### 4.2 효과적인 수정 요청 예시

```
// 컬러 수정
"버튼 호버 색상을 #1A5FB4에서 #2563EB로 변경해주세요"

// 레이아웃 수정  
"카드 섹션을 3열 그리드에서 4열 그리드로 변경해주세요"

// 기능 추가
"테이블에 CSV 내보내기 버튼을 추가해주세요"

// 스타일 수정
"전체적으로 여백을 더 넓게 조정해주세요 (p-4 → p-6)"
```

### 4.3 Tailwind 설정 (tailwind.config.js)

```javascript
// 프로젝트 초기 설정 시 이 설정을 적용하세요
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: '#2B87EC',
          dark: '#1A5FB4',
          light: '#E8F2FC',
        },
      },
      fontFamily: {
        sans: ['Pretendard', 'Inter', 'sans-serif'],
      },
    },
  },
}
```

### 4.4 자주 사용하는 Shadcn 설치 명령어

```bash
# 필수 컴포넌트
npx shadcn@latest add button card input table select checkbox
npx shadcn@latest add dropdown-menu dialog alert badge

# 폼 관련
npx shadcn@latest add form label textarea radio-group

# 레이아웃/네비게이션
npx shadcn@latest add navigation-menu sheet tabs

# 유틸리티
npx shadcn@latest add skeleton toast sonner
```

---

## 부록: 빠른 참조 치트시트

### 컬러 클래스
```
Primary: bg-[#2B87EC] text-white hover:bg-[#1A5FB4]
Primary Light BG: bg-[#E8F2FC]
Success: bg-green-500 / text-green-600 / bg-green-100
Warning: bg-amber-500 / text-amber-600 / bg-amber-100
Error: bg-red-500 / text-red-600 / bg-red-100
```

### 간격 규칙
```
컴포넌트 내부: p-4 (16px)
카드 간격: gap-4 또는 gap-6
섹션 간격: space-y-6 또는 space-y-8
페이지 패딩: p-6 (데스크톱), p-4 (모바일)
```

### 타이포그래피
```
페이지 제목: text-2xl font-bold
섹션 제목: text-lg font-semibold
카드 제목: text-base font-medium
본문: text-sm
보조 텍스트: text-sm text-muted-foreground
```
