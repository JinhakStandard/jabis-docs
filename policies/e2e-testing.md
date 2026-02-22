# JABIS E2E 테스트 정책

> ai-e2e-tester를 활용한 E2E 테스트 가이드입니다.

---

## 1. 개요

**ai-e2e-tester**는 자연어 프롬프트로 브라우저 E2E 테스트를 실행하는 AI 기반 테스트 서버입니다.
Playwright를 사용하여 AI가 브라우저를 직접 조작하고, 결과를 자동으로 판정합니다.

- **서버 URL**: `https://jabis-tester.ngrok.app`
- **API 문서**: `https://jabis-tester.ngrok.app/docs`
- **포트**: 4820 (로컬)

---

## 2. 테스트 시점

Claude Code가 다음 작업을 완료한 후 E2E 테스트를 실행합니다:

| 작업 유형 | 테스트 필요 여부 | 설명 |
|----------|---------------|------|
| UI 컴포넌트 추가/수정 | **필수** | 페이지 렌더링, 인터랙션 확인 |
| 라우팅 변경 | **필수** | 페이지 이동, URL 확인 |
| API 연동 변경 | **필수** | 데이터 로드, 에러 처리 확인 |
| 메뉴/네비게이션 변경 | **필수** | 메뉴 표시, 링크 동작 확인 |
| 스타일만 변경 | 선택 | 레이아웃 깨짐 확인 |
| 백엔드 전용 변경 | 선택 | 프론트에 영향 있으면 테스트 |
| 문서/설정 변경 | 불필요 | — |

---

## 3. API 사용법

### 3.1 테스트 실행

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

### 3.2 테스트 결과 조회 (폴링)

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

### 3.3 테스트 목록 조회

```bash
curl -s "https://jabis-tester.ngrok.app/api/tests?status=failed&pageSize=10" | jq
```

### 3.4 스크린샷 조회

```bash
curl -s https://jabis-tester.ngrok.app/api/screenshots/{filename} --output screenshot.png
```

### 3.5 테스트 취소/삭제

```bash
curl -s -X DELETE https://jabis-tester.ngrok.app/api/tests/{testId} | jq
```

---

## 4. 테스트 프롬프트 작성 가이드

### 4.1 미리보기 환경 테스트

빌드 후 미리보기 URL을 대상으로 테스트합니다.

```
https://jabis-maker.ngrok-free.dev/preview/{프로젝트명}/ 페이지에 접속해서:
1. 페이지가 정상적으로 로드되는지 확인
2. 메뉴가 표시되는지 확인
3. "대시보드" 메뉴를 클릭해서 이동되는지 확인
```

### 4.2 좋은 프롬프트 예시

```
# 페이지 로드 확인
https://jabis-maker.ngrok-free.dev/preview/jabis-hr/ 에 접속해서 페이지가 정상 렌더링되는지 확인해줘.
흰 화면이 아닌 실제 콘텐츠가 보여야 함.

# 네비게이션 테스트
https://jabis-maker.ngrok-free.dev/preview/jabis/ 에 접속해서 좌측 메뉴의 각 항목을 클릭했을 때
페이지가 정상 이동하는지 확인해줘.

# 폼 인터랙션 테스트
https://jabis-maker.ngrok-free.dev/preview/jabis-finance/ 에 접속해서:
1. 좌측 메뉴에서 "결재" 항목을 클릭
2. 결재 목록이 표시되는지 확인
3. 첫 번째 항목을 클릭해서 상세 페이지로 이동하는지 확인
```

### 4.3 프롬프트 작성 원칙

1. **대상 URL을 명시** — 어떤 환경의 어떤 페이지인지 정확히 지정
2. **검증 기준을 구체적으로** — "잘 되는지"보다 "버튼이 보이는지", "텍스트가 표시되는지" 등
3. **단계별로 기술** — 복잡한 시나리오는 번호를 매겨 순서대로
4. **실패 기준도 명시** — "흰 화면이 아닌", "에러 메시지가 없는" 등

---

## 5. Claude Code 워크플로우

### 5.1 개발 → 테스트 플로우

```
1. 코드 수정
2. pnpm build:preview (미리보기 빌드)
3. ai-e2e-tester로 테스트 실행
4. 결과 확인 (passed → 다음 단계, failed → 수정 후 재테스트)
5. 커밋 & push
```

### 5.2 테스트 실행 패턴 (Claude Code용)

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

---

## 8. 참고

- API 문서: https://jabis-tester.ngrok.app/docs
- 응답 형식: JABIS 표준 `{ success, data/error }` 패턴
- 인증: `AUTH_TOKEN` 환경변수 설정 시 `Authorization: Bearer {token}` 필요
