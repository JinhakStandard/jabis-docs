# jabis-finance

재무담당자(회계팀) 전용 프론트엔드 프로젝트.

## 기본 정보

| 항목 | 값 |
|------|-----|
| 역할 | finance (재무) |
| 기술 스택 | React 18, Vite 5, Zustand, Tailwind CSS |
| 패키지 매니저 | pnpm |
| base path | `/finance/` (운영), `/preview/jabis-finance/` (미리보기) |
| 공유 패키지 | @jabis/ui, @jabis/layout, @jabis/auth, @jabis/core, @jabis/menu, @jabis/shared-pages |
| submodule | packages/ → jabis-common |

## 빌드 명령

```bash
pnpm install          # 의존성 설치
pnpm dev              # 개발 서버
pnpm build            # 운영 빌드
pnpm build:preview    # 미리보기 빌드
```

## 메뉴 구조

### 재무 전용 메뉴 (6그룹, 18개 서브메뉴)

| 그룹 | 메뉴 | 경로 | 상태 |
|------|------|------|------|
| 결산/재무보고 | 결산 현황 | /closing/status | 준비중 |
| | 재무제표 | /closing/statements | 준비중 |
| | 자금현황 보고 | /closing/funds | 준비중 |
| 예산 관리 | 예산 편성 | /budget/planning | 준비중 |
| | 예산 집행 | /budget/execution | 준비중 |
| | 예산 대비 실적 | /budget/performance | 준비중 |
| 전표/회계 | 전표 입력 | /accounting/entry | 준비중 |
| | 전표 조회 | /accounting/list | 준비중 |
| | 계정과목 관리 | /accounting/accounts | 준비중 |
| 자금 관리 | 자금 현황 | /treasury/status | 준비중 |
| | 입출금 내역 | /treasury/transactions | 준비중 |
| | 자금 계획 | /treasury/planning | 준비중 |
| 세무 | 부가세 관리 | /tax/vat | 준비중 |
| | 법인세 | /tax/corporate | 준비중 |
| | 세금계산서 | /tax/invoice | 준비중 |
| 자산 관리 | 고정자산 대장 | /asset/fixed | 준비중 |
| | 감가상각 | /asset/depreciation | 준비중 |
| | 자산 실사 | /asset/inventory | 준비중 |

### 공통 메뉴 (@jabis/menu)
- 업무 관리: 할일, 문서작성, 전자결재, 일정
- 소통: 메신저, 메일, 회의실 예약
- 사내 생활: 트래져 블로그, 조식, 뉴스, 맛집, 추억 공유
- 조직/관리: 조직도
- 기타: 회계 전표, 사내 사이트

## DashboardLayout 설정

```javascript
headerConfig: { title: 'JABIS', subtitle: '재무' }
menuStateKey: 'finance'
userRole: 'finance'
roleBadgeLabel: '재무담당자'
```

## 환경 설정

| 환경 | OAuth | Gateway URL |
|------|-------|-------------|
| .env.local | 비활성 | localhost:3000 |
| .env.preview | 비활성 (DEV_TOKEN) | jabis-gateway.jinhakapply.com |
| .env.production | 활성 (client_id 미발급) | jabis-gateway.jinhakapply.com |

## 배포 미완료 사항
- [ ] Bitbucket 리포 생성
- [ ] jabis-cert OAuth client_id 발급
- [ ] jabis-helm ingress/values 설정
- [ ] .env.production에 client_id 반영
