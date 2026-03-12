# jabis-notify

> 통합 알림 서비스 — 이메일(SMTP) / Teams Webhook 발송

---

## 기술 스택

- Express 4 + TypeScript
- Node.js 20
- nodemailer (Office365 SMTP)
- Zod (환경변수 검증)
- Winston (로깅)

## 프로젝트 구조

```
src/
├── server.ts              # 진입점 + graceful shutdown
├── app.ts                 # Express 앱 설정
├── config/
│   └── index.ts           # Zod 환경변수 검증
├── routes/
│   ├── health.ts          # GET /health
│   └── notify.ts          # POST /api/notify/send
├── services/
│   ├── email.service.ts   # nodemailer SMTP 발송
│   ├── teams.service.ts   # Teams Incoming Webhook
│   └── template.service.ts # HTML 템플릿 렌더링
├── templates/
│   ├── base.html              # 공통 레이아웃
│   ├── approval-request.html  # 결재 요청
│   ├── approval-complete.html # 결재 완료
│   ├── corporate-card-request.html   # 법인카드 배정 요청
│   ├── corporate-card-approved.html  # 법인카드 승인/반려
│   └── attendance-approved.html      # 근태 승인/반려
├── middleware/
│   ├── auth.ts            # Bearer SERVICE_TOKEN 검증
│   └── errorHandler.ts    # 커스텀 에러 클래스 + 전역 핸들러
├── utils/
│   ├── logger.ts          # Winston 로거
│   └── vault.ts           # Vault 시크릿 읽기
└── types/
    └── index.ts           # 타입 정의
```

## 주요 명령어

```bash
npm install     # 의존성 설치
npm run dev     # 개발 서버 (tsx watch)
npm run build   # TypeScript 빌드
npm start       # 프로덕션 실행
npm run lint    # ESLint
npm run typecheck # 타입 체크
```

## 포트

- 3200

## 환경변수

| 변수 | 기본값 | 설명 |
|------|--------|------|
| NODE_ENV | development | 실행 환경 |
| PORT | 3200 | 서버 포트 |
| SMTP_HOST | smtp.office365.com | SMTP 호스트 |
| SMTP_PORT | 587 | SMTP 포트 |
| SMTP_USER | jabis@jinhakapply.com | SMTP 사용자 |
| SMTP_PASS | (vault) | SMTP 비밀번호 |
| SMTP_FROM | jabis@jinhakapply.com | 발신자 주소 |
| SERVICE_TOKEN | (vault) | 내부 서비스 인증 토큰 |

## API 스펙

### POST /api/notify/send

**인증**: `Authorization: Bearer {SERVICE_TOKEN}`

**요청**:
```json
{
  "channels": ["email", "teams"],
  "template": "approval-request",
  "recipients": {
    "email": ["approver@jinhakapply.com"],
    "teams": "https://jinhaksa.webhook.office.com/..."
  },
  "data": {
    "documentName": "2024년 상반기 예산 검토",
    "requestorName": "홍길동",
    "requestorDept": "개발팀",
    "requestedAt": "2024-03-12 09:30",
    "approvalUrl": "https://jabis.jinhakapply.com/approval/123"
  }
}
```

**응답**:
```json
{
  "success": true,
  "results": {
    "email": { "success": true, "count": 1 },
    "teams": { "success": true }
  },
  "requestId": "uuid"
}
```

### 지원 템플릿

| 템플릿 | 용도 |
|--------|------|
| `approval-request` | 결재 요청 알림 |
| `approval-complete` | 결재 완료 (승인/반려) |
| `corporate-card-request` | 법인카드 배정 요청 |
| `corporate-card-approved` | 법인카드 배정 결과 |
| `attendance-approved` | 근태 처리 결과 |

## K3S 내부 URL

```
http://jabis-notify-prod-service.jabis-prod:3200
```

## 호출 서비스

- jabis-api-gateway → 결재/근태/법인카드 이벤트 발생 시 호출

## Dockerfile

- node:20-alpine 멀티스테이지 빌드
- non-root user (nodejs:1001)
- Vault Agent `/jabis-notify/keys` 마운트
- HEALTHCHECK: `GET /health`

## Bitbucket Pipeline

- Pattern A (gateway 동일)
- main/master → prod 배포
- alpha → alpha 배포
