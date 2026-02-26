# API 디버깅 정책

> AI가 API 관련 코드를 수정할 때, 반드시 실제 응답 데이터를 먼저 확인하는 정책입니다.
> 필드명 추측으로 인한 버그를 방지합니다.

---

## 1. 원칙: 데이터 확인 후 코딩

**API 관련 버그 수정 또는 UI 데이터 바인딩 작업 시, 코드를 수정하기 전에 반드시 실제 API 응답을 curl로 확인한다.**

### 1.1 왜 필요한가

| 문제 | 원인 | 실제 사례 |
|------|------|----------|
| 필드명 불일치 | API는 `userName`인데 `approverName`으로 접근 | 결재선 이름 null |
| ID 타입 혼동 | OAuth sub ID vs employee ID | JOIN 실패 → null |
| 응답 구조 추측 | 플랫 객체로 가정했으나 래퍼 객체 반환 | React error #31 |
| 데이터 포맷 | JSON 블록 배열을 문자열로 가정 | content 파싱 실패 |

### 1.2 필수 확인 순서

```
1. curl로 실제 API 응답 조회
2. 응답 데이터의 필드명, 타입, 구조 확인
3. 프론트엔드 코드가 기대하는 필드명과 대조
4. 불일치 발견 → 어느 쪽을 맞출지 결정 후 수정
```

---

## 2. API Gateway 접근 방법

### 2.1 운영 환경

```bash
# 기본 URL
https://jabis-gateway.jinhakapply.com

# 인증: x-user-email 헤더
curl -s -H "x-user-email: {사용자이메일}" "https://jabis-gateway.jinhakapply.com/api/..."
```

### 2.2 인증 방식 (우선순위)

| 방식 | 헤더 | 용도 |
|------|------|------|
| Bearer JWT | `Authorization: Bearer {token}` | 운영 (jabis-cert introspect) |
| 이메일 헤더 | `x-user-email: user@domain.com` | 디버깅/개발 |
| Dev 토큰 | `x-dev-token: {token}` | 미리보기/개발 환경 전용 |

### 2.3 사용자 ID 추출 규칙

API Gateway는 이메일에서 `@` 앞부분만 userId로 사용:
- `navskh@jinhakapply.com` → `navskh`
- JWT의 `sub` 또는 `email` 클레임에서 동일하게 추출

---

## 3. 주요 API 엔드포인트 (디버깅용)

### 3.1 전자결재

```bash
# 문서 목록 (boxType: all, pending, progress, completed, rejected, draft, reference, scheduled, department)
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/approval/documents?boxType=all&pageSize=5" | python3 -m json.tool

# 문서 상세 (lines, references, relatedDocs, processingHistory, attachments 포함)
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/approval/documents/{문서ID}" | python3 -m json.tool

# 통계 (8개 박스별 건수)
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/approval/stats" | python3 -m json.tool

# 양식 목록
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/approval/forms" | python3 -m json.tool
```

### 3.2 조직도

```bash
# 직원 목록
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/organization/employees" | python3 -m json.tool

# 부서 목록
curl -s -H "x-user-email: navskh@jinhakapply.com" \
  "https://jabis-gateway.jinhakapply.com/api/organization/departments" | python3 -m json.tool
```

> 새 API 엔드포인트가 추가되면 이 문서에 디버깅용 curl 예제를 추가할 것.

---

## 4. 디버깅 체크리스트

API 관련 버그 수정 시 다음을 순서대로 확인:

- [ ] **실제 API 응답 조회**: curl로 데이터를 가져와서 필드명/구조 확인
- [ ] **프론트엔드 기대값 대조**: 컴포넌트가 어떤 필드명으로 접근하는지 확인
- [ ] **ID 타입 확인**: OAuth sub ID, employee ID, user email 중 어떤 것이 사용되는지 확인
- [ ] **DB JOIN 경로 확인**: `organization.users.id` (OAuth sub) vs `organization.employees.id` (직원 ID) 구분
- [ ] **데이터 포맷 확인**: JSON 구조(BlockNote 블록 등), 날짜 형식, null 가능 여부

---

## 5. ID 체계 (혼동 주의)

JABIS에서 "사용자 ID"는 문맥에 따라 다른 값을 가리킵니다:

| ID 종류 | 테이블 | 예시 값 | 용도 |
|---------|--------|---------|------|
| OAuth sub | `organization.users.id` | `bb6f469a-d266-...` (UUID) | OAuth 인증 |
| 이메일 prefix | (추출값) | `navskh` | API Gateway userId |
| Employee ID | `organization.employees.id` | `emp-628` | 조직도, 결재선 |
| Employee ID (old) | `organization.employees.employee_id` | `navskh` | 레거시 호환 |

### 5.1 테이블 관계

```
organization.users.id (OAuth sub, UUID)
    └── users.employee_id → organization.employees.id (emp-XXX)
                                └── employees.department_id → organization.departments.id
```

### 5.2 결재선 데이터의 ID

- `approval.lines.user_id` / `employee_id`: 프론트엔드가 `approver.id` (= employees.id, `emp-XXX`)를 저장
- JOIN 시 `COALESCE(employee_id, user_id)`로 employees 테이블 직접 매칭

---

## 6. 관련 문서

| 문서 | 위치 |
|------|------|
| API 명세 | `jabis-docs/api-specs/` |
| DB 스키마 | `jabis-docs/db-schemas/` |
| 배포 정책 | `jabis-docs/policies/deployment.md` |
| 코딩 스타일 | `jabis-docs/policies/coding-style.md` |
