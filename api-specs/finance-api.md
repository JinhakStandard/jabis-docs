# Finance API 명세 (법인카드 관리)

> **Base Path**: `/api/finance`
> **인증**: JWT Bearer 토큰 필수
> **인가**: CUD 작업은 JWT roles에 `finance` 포함 필요 (관리자)

---

## 부서 관리 (organization 연동)

### GET /api/finance/departments

organization.departments를 기준으로 부서 목록을 반환. card_departments 카드 설정이 있으면 병합.

**응답**:
```json
{
  "success": true,
  "data": [
    {
      "orgDepartmentId": "division-support",
      "orgDepartmentCode": "SUP",
      "name": "경영지원본부",
      "orgDepartmentType": "division",
      "parentId": "company-jinhak",
      "status": "active",
      "cardDepartmentId": "uuid-or-null",
      "sortOrder": 1,
      "hasCardConfig": true
    }
  ]
}
```

### POST /api/finance/departments

관리자 전용. card_departments의 카드 설정만 관리 (organization.departments 수정 불가).

**create** — 조직 부서에 카드 설정 추가:
```json
{
  "action": "create",
  "data": {
    "orgDepartmentId": "division-support",
    "sortOrder": 1
  }
}
```

**update** — 카드 설정 수정 (부서명 수정 불가):
```json
{
  "action": "update",
  "id": "card-department-uuid",
  "data": {
    "sortOrder": 2
  }
}
```

**delete** — 카드 설정 삭제:
```json
{
  "action": "delete",
  "id": "card-department-uuid"
}
```

---

## 기타 엔드포인트

상세 명세는 OpenAPI 문서 참조: `jabis-api-gateway/docs/openapi/paths/finance.yaml`

| 엔드포인트 | 메서드 | 설명 |
|-----------|--------|------|
| `/api/finance/me` | GET | 현재 사용자 JWT 정보 |
| `/api/finance/stats` | GET | 대시보드 통계 |
| `/api/finance/cards` | GET/POST | 카드 목록/관리 |
| `/api/finance/users` | GET/POST | 사용자 목록/관리 |
| `/api/finance/checkout-requests` | GET/POST | 불출 기안서 |
| `/api/finance/checkout-requests/my` | GET | 내 불출 기안서 |
| `/api/finance/checkout-requests/:id` | GET | 불출 기안서 상세 |
| `/api/finance/return-requests` | GET/POST | 반납 기안서 |
| `/api/finance/return-requests/my` | GET | 내 반납 기안서 |
| `/api/finance/receipts` | GET/POST | 영수증 관리 |
| `/api/finance/cards/:id/history` | GET | 카드 이력 |
