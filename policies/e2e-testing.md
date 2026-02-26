# JABIS E2E 테스트 정책

> ai-e2e-tester를 활용한 E2E 테스트 가이드입니다.
> 환경별 정의는 `policies/environments.md`를 참조하세요.

---

## 1. 개요

**ai-e2e-tester**는 자연어 프롬프트로 브라우저 E2E 테스트를 실행하는 AI 기반 테스트 서버입니다.
Playwright를 사용하여 AI가 브라우저를 직접 조작하고, 결과를 자동으로 판정합니다.

- **서버 URL**: `https://jabis-tester.ngrok.app`
- **API 문서**: `https://jabis-tester.ngrok.app/docs`
- **포트**: 4820 (로컬)

### 테스트 종류

| 종류 | 대상 환경 | 시점 | 로그인 |
|------|----------|------|--------|
| **미리보기 테스트** | 미리보기 (preview) | 커밋 전 | 불필요 |
| **운영 테스트** | 운영 (production) | 배포 후 | 필요 (OAuth) |

---

## 2. 미리보기 테스트

### 2.1 개요

미리보기 빌드(`pnpm build:preview`) 결과물을 대상으로 하는 E2E 테스트.
**커밋/배포 전에 빌드 결과물이 정상 동작하는지 검증**하는 것이 목적이다.

### 2.2 표준 작업 플로우 (TDD 기반)

작업 요청 시 **테스트 기준을 먼저 정의**한 후 구현에 들어간다.
전체 플로우는 `policies/work-flow.md`를 참조.

```
1. 테스트 정의  — 이번 작업에서 검증할 항목을 먼저 정한다
2. 계획 수립    — 구현 방향과 영향 범위를 파악한다
3. 코드 구현    — 계획에 따라 코드를 작성한다
4. 빌드         — pnpm build:preview
5. 테스트 실행  — ai-e2e-tester로 E2E 테스트
6. 커밋 & push  — 테스트 통과 시
```

### 2.3 검증 범위

미리보기 테스트에서는 **모든 것을 테스트할 필요는 없다.** 다음 항목을 중심으로 검증한다:

| 항목 | 필수 여부 | 설명 |
|------|----------|------|
| 페이지 렌더링 | **필수** | 흰 화면이 아닌 실제 콘텐츠가 표시되는지 |
| 메뉴/네비게이션 | **필수** | 메뉴 표시, 클릭 시 페이지 이동 |
| API 데이터 로드 | 가능한 경우 | X-Dev-Token으로 API 호출이 가능한 경우 데이터 로드 확인 |
| 폼/인터랙션 | 해당 시 | 작업 내용이 폼이나 인터랙션에 관련된 경우 |

### 2.4 테스트 시점

| 작업 유형 | 미리보기 테스트 | 설명 |
|----------|--------------|------|
| UI 컴포넌트 추가/수정 | **필수** | 페이지 렌더링, 인터랙션 확인 |
| 라우팅 변경 | **필수** | 페이지 이동, URL 확인 |
| API 연동 변경 | **필수** | 데이터 로드 확인 |
| 메뉴/네비게이션 변경 | **필수** | 메뉴 표시, 링크 동작 확인 |
| 스타일만 변경 | 선택 | 레이아웃 깨짐 확인 |
| 백엔드 전용 변경 | 불필요 | — |
| 문서/설정 변경 | 불필요 | — |

### 2.5 API 통신 방식

```
ai-e2e-tester (Playwright)
  → https://jabis-maker.ngrok-free.dev/preview/{프로젝트}/ 접속
    → 프론트엔드가 API 호출
      → https://jabis-gateway.jinhakapply.com/api/...
      → 인증: X-Dev-Token (빌드 시 .env.preview에서 주입)
```

- OAuth 없이 X-Dev-Token으로 인증 우회
- 운영 DB의 실제 데이터를 사용
- 로그인 절차 없이 바로 페이지 접근 가능

### 2.6 실패 시 처리

1. 테스트 실패 → **원인 분석** (어디에 문제가 있는지 확인)
2. 문제 수정 → 재빌드(`pnpm build:preview`) → 재테스트
3. **최대 2회 재시도** — 2회 재시도 후에도 실패하면 사용자에게 실패 사실과 원인을 보고

### 2.7 프롬프트 작성 가이드

고정 템플릿은 없으나, 다음 원칙에 따라 프롬프트를 작성한다:

1. **대상 URL을 명시** — `https://jabis-maker.ngrok-free.dev/preview/{프로젝트명}/`
2. **검증 기준을 구체적으로** — "잘 되는지"보다 "버튼이 보이는지", "텍스트가 표시되는지" 등
3. **단계별로 기술** — 복잡한 시나리오는 번호를 매겨 순서대로
4. **실패 기준도 명시** — "흰 화면이 아닌", "에러 메시지가 없는" 등

**프롬프트 예시:**

```
https://jabis-maker.ngrok-free.dev/preview/jabis-hr/ 에 접속해서:
1. 페이지가 정상적으로 로드되는지 확인 (흰 화면이 아닌 실제 콘텐츠)
2. 좌측 메뉴가 표시되는지 확인
3. 메뉴 항목을 클릭했을 때 페이지가 이동하는지 확인
```

---

## 3. 운영 테스트

### 3.1 개요

배포 완료 후 **운영 환경에서 실제 서비스가 정상 동작하는지 검증**하는 E2E 테스트.
OAuth 로그인을 자동 수행한 후 전체 기능을 검증한다.

### 3.2 시점

- **사용자가 직접 지시했을 때만 실행** (자동 실행 아님)
- 배포(push) 후 Pipeline + ArgoCD 배포가 완료된 시점에 사용자가 "운영 테스트 해봐"라고 지시하면 실행
- 미리보기 테스트와 달리 자동 플로우에 포함되지 않음

### 3.3 로그인 처리

운영 환경은 OAuth 로그인이 필수이므로, 테스트 프롬프트에 **로그인 절차를 포함**해야 한다.

- 테스트 계정 정보는 Claude Code의 private 메모리에만 저장 (jabis-docs에 기록 금지)
- 프롬프트에 로그인 절차를 단계별로 기술한다

### 3.4 검증 범위

운영 테스트에서는 **요청된 기능을 전부 검증**한다. 미리보기 테스트와 달리 범위를 제한하지 않는다.

- 로그인 플로우
- 페이지 렌더링
- 메뉴/네비게이션
- API 데이터 로드
- 폼/인터랙션
- 권한별 접근
- 작업에서 요청된 모든 기능

### 3.5 API 통신 방식

```
ai-e2e-tester (Playwright)
  → https://jabis.jinhakapply.com/ (또는 해당 운영 URL) 접속
    → OAuth 로그인 수행 (테스트 계정)
      → jabis-cert에서 JWT 쿠키 발급
    → 프론트엔드가 API 호출
      → https://jabis-gateway.jinhakapply.com/api/...
      → 인증: JWT 쿠키 (자동 전송)
```

### 3.6 결과 보고

- **passed / failed 만 보고** — 성공이면 성공, 실패면 실패와 원인만 간단히 전달
- 미리보기 테스트처럼 자동 재시도하지 않음 (사용자 판단에 맡김)

### 3.7 운영 URL 목록

| 프로젝트 | URL |
|---------|-----|
| jabis | `https://jabis.jinhakapply.com` |
| jabis-hr | `https://jabis.jinhakapply.com/hr` |
| jabis-dev | `https://jabis.jinhakapply.com/dev` |
| jabis-producer | `https://jabis.jinhakapply.com/producer` |
| jabis-finance | `https://jabis.jinhakapply.com/finance` |
| jabis-design-system | `https://jabis-design.jinhakapply.com` |

---

## 4. API 사용법

### 4.1 테스트 실행

```bash
curl -s -X POST https://jabis-tester.ngrok.app/api/tests \
  -H "Content-Type: application/json" \
  -d '{"prompt":"테스트 시나리오"}' | jq
```

**요청 파라미터:**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `prompt` | string | O | 자연어 테스트 시나리오 |
| `browserType` | string | X | chromium(기본) / firefox / webkit |
| `headless` | boolean | X | true(기본) / false |

**응답:**
```json
{
  "success": true,
  "data": {
    "id": "테스트ID",
    "status": "pending",
    "prompt": "...",
    "startedAt": "ISO 8601"
  }
}
```

### 4.2 테스트 결과 조회 (폴링)

```bash
curl -s https://jabis-tester.ngrok.app/api/tests/{testId} | jq
```

**status 값:**
- `pending` — 대기 중
- `running` — 실행 중
- `passed` — 성공
- `failed` — 실패
- `cancelled` — 취소

**완료 시 반환 정보:**
- `summary` — AI가 생성한 결과 요약
- `steps[]` — 실행된 브라우저 액션 목록 (tool, input, result, screenshotPath)
- `durationMs` — 소요 시간

### 4.3 테스트 목록 조회

```bash
curl -s "https://jabis-tester.ngrok.app/api/tests?status=failed&pageSize=10" | jq
```

### 4.4 스크린샷 조회

```bash
curl -s https://jabis-tester.ngrok.app/api/screenshots/{filename} --output screenshot.png
```

### 4.5 테스트 취소/삭제

```bash
curl -s -X DELETE https://jabis-tester.ngrok.app/api/tests/{testId} | jq
```

---

## 5. Claude Code 실행 패턴

```bash
# 1. 테스트 시작
TEST_ID=$(curl -s -X POST https://jabis-tester.ngrok.app/api/tests \
  -H "Content-Type: application/json" \
  -d '{"prompt":"테스트 내용"}' | jq -r '.data.id')

# 2. 결과 폴링 (5초 간격, 최대 60회 = 5분)
for i in $(seq 1 60); do
  RESULT=$(curl -s https://jabis-tester.ngrok.app/api/tests/$TEST_ID)
  STATUS=$(echo $RESULT | jq -r '.data.status')
  if [ "$STATUS" = "passed" ] || [ "$STATUS" = "failed" ]; then
    echo $RESULT | jq '.data | {status, summary, durationMs}'
    break
  fi
  sleep 5
done
```

---

## 6. 브라우저 액션 목록

AI가 사용할 수 있는 브라우저 액션:

| 카테고리 | 액션 | 설명 |
|---------|------|------|
| 네비게이션 | `navigate`, `go_back`, `reload` | 페이지 이동 |
| 상호작용 | `click`, `type_text`, `select_option`, `hover`, `press_key` | 요소 조작 |
| 검증 | `assert_visible`, `assert_text`, `assert_url`, `wait_for_element` | 상태 확인 |
| 데이터 | `get_text`, `get_attribute`, `snapshot` | 정보 추출 |
| 유틸리티 | `screenshot`, `wait`, `ask_user` | 보조 기능 |

---

## 7. 제약사항

| 항목 | 제한 |
|------|------|
| 동시 실행 | 최대 3개 |
| AI 턴 | 최대 50회 |
| 네비게이션 타임아웃 | 30초 |
| 요소 조작 타임아웃 | 10초 |
| wait 액션 | 최대 10초 |
| 페이지 크기 | 최대 100 |
| 미리보기 테스트 재시도 | 최대 2회 |

---

## 8. 참고

- API 문서: https://jabis-tester.ngrok.app/docs
- 응답 형식: JABIS 표준 `{ success, data/error }` 패턴
- 인증: `AUTH_TOKEN` 환경변수 설정 시 `Authorization: Bearer {token}` 필요
- 환경 정의: `policies/environments.md`
