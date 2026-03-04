# 근태 관리 기능 구현 내역

> 작업일: 2026-03-03
> 실행 방식: Night Mode (NightExecutor 배치 실행)

---

## 1. 개요

기존 pljec hr 시스템의 근태 관리 기능을 JABIS 에코시스템으로 이관하는 작업.
GPS 기반 출퇴근 기록, 근태 등록(외근/출장 등), HR 관리자 전용 근태 조회 기능을 구현했다.

### 요구사항

1. **공통 메뉴 "근태 관리"** — 모든 역할에서 접근 가능한 GPS 기반 출퇴근 기록 + 근태 등록
2. **jabis-hr "인사 근태 관리"** — HR 관리자 전용 전 직원 근태 조회/통계
3. **한국시간(KST) 기준** — 모든 시간 표시 및 비교

---

## 2. DB 스키마

### 2.1 사전 마이그레이션 (수동 실행 완료)

Night mode 실행 전에 운영 DB에 직접 실행한 마이그레이션:

- **organization.attendance_types** — 근태 유형 마스터 (10개 기본 데이터)
- **organization.attendance_records** — 일별 근태 기록 (직원당 하루 1건, UNIQUE 제약)

마이그레이션 SQL은 채팅에서 직접 전달 → 사용자가 실행.
상세 스키마: `jabis-docs/db-schemas/attendance-schema.md`

### 2.2 테이블 구조

#### attendance_types

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | TEXT PK | 유형 ID (normal, field_work 등) |
| name | TEXT | 표시명 (정상출근, 외근 등) |
| category | TEXT | work / leave / special |
| description | TEXT | 설명 |
| requires_approval | BOOLEAN | 승인 필요 여부 |
| is_active | BOOLEAN | 활성 여부 |
| sort_order | INTEGER | 정렬 순서 |

#### attendance_records

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | TEXT PK | UUID |
| employee_id | TEXT FK | employees 참조 |
| date | DATE | 근무일 (KST 기준) |
| attendance_type_id | TEXT FK | 근태 유형 |
| clock_in_at | TIMESTAMPTZ | 출근 시각 |
| clock_in_latitude/longitude/accuracy | NUMERIC | GPS 좌표 (출근) |
| clock_out_at | TIMESTAMPTZ | 퇴근 시각 |
| clock_out_latitude/longitude/accuracy | NUMERIC | GPS 좌표 (퇴근) |
| status | TEXT | pending/normal/late/early_leave/absent/registered |
| note | TEXT | 비고 |
| registered_by | TEXT FK | 등록자 (본인 or HR) |
| UNIQUE | | (employee_id, date) |

### 2.3 기본 근태 유형 (10개)

| ID | 이름 | 카테고리 |
|----|------|---------|
| normal | 정상출근 | work |
| field_work | 외근 | work |
| business_trip | 출장 | work |
| work_from_home | 재택근무 | work |
| annual_leave | 연차 | leave |
| half_day_am | 오전 반차 | leave |
| half_day_pm | 오후 반차 | leave |
| sick_leave | 병가 | leave |
| early_leave | 조퇴 | special |
| compensatory | 대체휴무 | leave |

---

## 3. Night Mode 태스크 실행 결과

### Task 0: jabis-api-gateway 근태 API

| 항목 | 내용 |
|------|------|
| 커밋 | `71a6000` — 12:32 |
| 메시지 | feat: 근태 관리 API 구현 (출퇴근, 근태 등록, 관리자 조회) |

**생성된 파일:**
- `src/types/attendance.ts` — AttendanceType, AttendanceRecord, AdminAttendanceRecord 인터페이스
- `src/repositories/attendanceRepository.ts` — DB 접근 레이어
- `src/services/attendanceService.ts` — 비즈니스 로직
- `src/routes/attendance.ts` — Fastify 라우트 (8개 엔드포인트)

**API 엔드포인트:**

| 메서드 | 경로 | 설명 | 접근 |
|--------|------|------|------|
| GET | /api/attendance/types | 근태 유형 목록 | 전체 |
| GET | /api/attendance/today | 오늘 출퇴근 상태 | 전체 |
| POST | /api/attendance | 출근/퇴근/근태등록 (action으로 구분) | 전체 |
| GET | /api/attendance | 내 근태 기간 조회 | 전체 |
| GET | /api/attendance/admin | 전 직원 근태 조회 | HR only |
| GET | /api/attendance/stats | 근태 통계 | HR only |

### Task 1: jabis-common 공통 메뉴 + AttendancePage

| 항목 | 내용 |
|------|------|
| 커밋 | `0938f54` — 12:41 |
| 메시지 | feat: 공통 메뉴 근태 관리 추가 + AttendancePage 구현 |

**변경/생성된 파일:**
- `menu/src/commonGroups.js` — work 그룹에 "근태 관리" 메뉴 추가
- `shared-pages/src/stores/attendanceApi.js` — 근태 API 클라이언트
- `shared-pages/src/pages/AttendancePage.jsx` — 공통 근태 관리 페이지
- `shared-pages/src/index.js` — AttendancePage export 추가

**AttendancePage 기능 (3탭):**
1. **출퇴근** — GPS 위치 기반 출근/퇴근 버튼, 실시간 시계, 오늘 상태 표시
2. **근태 등록** — 날짜/유형/비고 입력, 과거 날짜만 선택 가능
3. **내 근태 현황** — 기간별 조회, 상태별 Badge, 월별 요약

**GPS 처리:**
- `navigator.geolocation.getCurrentPosition({ enableHighAccuracy: true })`
- 권한 거부 시에도 좌표 null로 출퇴근 가능

**submodule 동기화 완료:**
- jabis, jabis-hr, jabis-dev, jabis-producer, jabis-finance, jabis-design-system

### Task 2: jabis-hr + 소비 프로젝트 라우트 등록

| 항목 | 내용 |
|------|------|
| 커밋 (jabis-hr) | `4482d0a` — 12:57 |
| 메시지 | feat: 근태 관리 기능 통합 (공통 출퇴근 + 인사 근태 관리) |

**jabis-hr 변경사항:**
- `App.jsx` — AttendancePage import + `/attendance` 라우트 추가
- `App.jsx` — 메뉴 라벨 "근태 관리" → "인사 근태 관리" 변경
- `App.jsx` — `/employee/attendance` 라우트: ComingSoonPage → HrAttendancePage
- `pages/HrAttendancePage.jsx` — HR 관리자 전용 근태 관리 페이지 신규

**HrAttendancePage 기능:**
- 상단 통계 카드 4개 (총 근무일, 정상출근율, 지각, 결근)
- 부서/기간/상태 필터
- 전 직원 근태 테이블 (페이지네이션)
- 상태별 Badge 컬러

**소비 프로젝트 라우트 등록:**

| 프로젝트 | 커밋 | 시간 |
|---------|------|------|
| jabis (메인, 전 역할) | `31ed4db` | 13:00 |
| jabis-dev | `088dd16` | 13:00 |
| jabis-producer | `0a7d8ff` | 13:01 |
| jabis-finance | `ee01d88` | 13:01 |

### Task 3: jabis-docs 문서 업데이트

| 항목 | 내용 |
|------|------|
| 커밋 | `2b46a31` — 13:21 |
| 메시지 | docs: 근태 관리 스키마, API 스펙, 프로젝트 문서 추가 |

**생성/수정된 문서:**
- `db-schemas/attendance-schema.md` — DDL, 컬럼 설명, 인덱스
- `api-specs/attendance-api.md` — 8개 엔드포인트 스펙
- `projects/jabis-api-gateway.md` — attendance 파일/API 섹션 추가
- `projects/jabis-hr.md` — 근태 관리 기능 섹션 추가
- `projects/jabis-common.md` — AttendancePage, 메뉴 항목 반영

---

## 4. 영향받은 프로젝트 & 파일 요약

| 프로젝트 | 변경 유형 | 주요 파일 |
|---------|----------|----------|
| jabis-api-gateway | 신규 API | types/attendance.ts, repositories/attendanceRepository.ts, services/attendanceService.ts, routes/attendance.ts |
| jabis-common | 메뉴 + 페이지 | menu/src/commonGroups.js, shared-pages/src/pages/AttendancePage.jsx, shared-pages/src/stores/attendanceApi.js |
| jabis-hr | 라우트 + 페이지 | App.jsx, pages/HrAttendancePage.jsx |
| jabis | 라우트 추가 | App.jsx (전 역할) |
| jabis-dev | 라우트 추가 | App.jsx |
| jabis-producer | 라우트 추가 | App.jsx |
| jabis-finance | 라우트 추가 | App.jsx |
| jabis-docs | 문서 | attendance-schema.md, attendance-api.md, 프로젝트 문서 3개 |

---

## 5. 메뉴 구조

### 공통 메뉴 (모든 역할)
```
업무 관리
├── 할일
├── 문서작성
├── 목표관리
├── 일정
├── 전자결재
├── 근태 관리        ← 신규 (/attendance)
└── 법인카드 신청
```

### jabis-hr 전용 메뉴
```
인사 관리
├── 직원 정보
├── 인사이동
├── 직급/직책 관리
├── 인사 근태 관리    ← 라벨 변경, 페이지 구현 (/employee/attendance)
└── 휴가 관리
```

---

## 6. KST 처리 방식

| 위치 | 방식 |
|------|------|
| DB 저장 | TIMESTAMPTZ (UTC 저장) |
| 백엔드 날짜 계산 | `(NOW() AT TIME ZONE 'Asia/Seoul')::DATE` |
| 백엔드 지각 판정 | `EXTRACT(HOUR FROM NOW() AT TIME ZONE 'Asia/Seoul') >= 9` |
| 프론트엔드 표시 | `toLocaleString('ko-KR', { timeZone: 'Asia/Seoul' })` |

---

---

## 7. GPS 반경 기반 출근 제한 (2026-03-03 추가)

### 7.1 개요

HR 관리자가 출근 허용 영역(사무실 좌표 + 반경)을 등록하고, 출근 시 Haversine 공식으로 GPS 반경을 검증하는 기능.

### 7.2 DB 테이블

- `organization.attendance_zones` — 근태 영역 (이름, 위도, 경도, 반경, 활성여부)
- 상세: `jabis-docs/db-schemas/attendance-schema.md`

### 7.3 백엔드 변경 (jabis-api-gateway)

- `src/types/attendance.ts` — `AttendanceZone` 인터페이스 추가
- `src/repositories/attendanceRepository.ts` — Zone CRUD 메서드 6개 추가
- `src/services/attendanceService.ts` — `haversineDistance()` 함수 추가, `clockIn`에 GPS 검증
- `src/routes/attendance.ts` — `GET/POST /api/attendance/zones` 엔드포인트 추가

### 7.4 프론트엔드 변경 (jabis-hr)

- `App.jsx` — 메뉴 재구조: "인사 관리"에서 근태 메뉴 분리 → "근태 관리" 그룹 신설
  - "직원 근태 현황" (/attendance/admin) — 기존 HrAttendancePage
  - "근태 영역 지정" (/attendance/zones) — 신규 AttendanceZonesPage
- `pages/AttendanceZonesPage.jsx` — HR 관리자용 영역 CRUD 페이지

### 7.5 검증 정책

- 활성 영역 0개 → GPS 검증 없음 (자유 출근)
- 활성 영역 1개 이상 → GPS 필수, 영역 중 하나라도 반경 내이면 출근 허용
- 퇴근(clock-out)에는 GPS 검증 미적용

---

## 8. Kakao Maps 출근 지도 다이얼로그 (2026-03-04 추가)

### 8.1 개요

출근 버튼 클릭 시 Kakao Maps 지도 다이얼로그를 표시하여, 현재 위치와 등록된 근태 영역을 시각적으로 보여주는 기능. 출근 가능 여부를 지도 위에서 직관적으로 확인할 수 있다.

### 8.2 동작 흐름

```
출근하기 클릭
  → GPS 위치 + 활성 영역 동시 요청
  → 영역 0개: 바로 출근 처리 (기존 동작)
  → 영역 1개 이상: 지도 다이얼로그 표시
    → 반경 내: 초록색 "출근 가능 지역입니다" + 출근 확인 버튼 활성
    → 반경 밖: 빨간색 "출근 가능 지역이 아닙니다" + 가장 가까운 영역까지 거리 표시 + 버튼 비활성
```

### 8.3 프론트엔드 컴포넌트

#### ClockInMapDialog.jsx (jabis-common/shared-pages)

- **Kakao Maps SDK 동적 로딩**: `<script>` 태그로 `//dapi.kakao.com/v2/maps/sdk.js` 로드
- **환경변수**: `VITE_KAKAO_MAP_KEY` — Kakao Developers JavaScript 키
- **Haversine 거리 계산**: 프론트엔드에서도 거리를 계산하여 UI 표시
- **지도 요소**:
  - 현재 위치: 파란색 마커 + "내 위치" 라벨
  - 근태 영역: Circle (반경 시각화) + CustomOverlay (영역 이름)
  - 영역 내: 초록색 원, 영역 밖: 회색 원
- **Props**: `open`, `onOpenChange`, `latitude`, `longitude`, `zones`, `onConfirmClockIn`, `loading`

#### AttendancePage.jsx ClockTab 변경사항

- `handleClockInClick`: GPS + `getActiveZones()` 동시 호출 → 영역 유무에 따라 분기
- `handleConfirmClockIn`: 지도 다이얼로그에서 확인 클릭 → 실제 출근 API 호출
- **근무시간 타이머**: 출근 후 실시간 `X시간 Y분` 표시 (1초마다 갱신)
- **퇴근 버튼 활성화**: 출근 상태(clockInAt 존재 + clockOutAt 없음)에서만 퇴근 가능

### 8.4 Kakao Maps 설정

- **API 키 종류**: JavaScript 키 (REST API 키 아님)
- **등록된 도메인**: `localhost`, `jabis-maker.ngrok-free.dev`, `jabis.jinhakapply.com`
- **환경변수 이름**: `VITE_KAKAO_MAP_KEY`
- **적용 프로젝트**: jabis, jabis-hr, jabis-dev, jabis-producer, jabis-finance, jabis-design-system
- **SDK URL**: `//dapi.kakao.com/v2/maps/sdk.js?appkey={KEY}&autoload=false`

### 8.5 API 엔드포인트

- `GET /api/attendance/zones/active` — 활성 영역 목록 (인증 필수, HR 권한 불필요)
- 상세: `jabis-docs/api-specs/attendance-api.md` 7절

---

## 9. 후속 작업 (미완료)

- [ ] 운영 배포 후 E2E 테스트 실행 (ai-e2e-tester)
- [ ] GPS 출퇴근 모바일 환경 테스트
- [ ] 근태 승인 워크플로우 (requires_approval 유형에 대한 전자결재 연동)
- [ ] 휴가 관리 메뉴와의 연계 (연차/반차 사용 시 자동 근태 등록)
- [ ] 근태 데이터 대시보드 위젯 (jabis-hr DashboardPage에 요약 카드)
