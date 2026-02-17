# JABIS 보안 정책

> 이 문서는 JABIS 전체 프로젝트에 적용되는 보안 정책입니다.

---

## 1. 시크릿 관리

### 원칙
- 하드코딩된 비밀번호, API 키, 토큰 **절대 금지**
- 환경변수(`.env.*`) 또는 **Vault Agent**를 통해 주입
- `.env` 파일은 반드시 `.gitignore`에 포함

### 환경변수 3단계 구조

| 단계 | 위치 | 대상 | 예시 |
|------|------|------|------|
| 1단계 | Dockerfile ENV | 비민감 설정값 | NODE_ENV, PORT, DB_HOST, DB_PORT |
| 2단계 | Vault Agent → keys/vault-key.json | 민감 시크릿 | PASSWORD, SECRET, TOKEN, API_KEY |
| 3단계 | .env 파일 (로컬 전용) | 개발 환경 설정 | .gitignore로 제외, 배포 안 됨 |

```
[Docker Build]                    [K8S Pod 시작]
Dockerfile ENV (비민감) ──────►   ENV 변수로 설정
                                  +
                                  Vault Agent init container
                                  → keys/vault-key.json 주입 (민감)
                                  +
                                  앱이 vault-key.json 읽어서 사용
```

> **예외**: jabis-emergency-console은 내부 전용 툴로 .env.production이 커밋됨

### Vault 연동 규칙
- Vault 시크릿 작업 시 반드시 `vault-key.json`과 Vault 서버 키 **비교 후** 진행
- Docker 이미지에 시크릿 포함 금지 — Vault Agent init container 사용
- Helm values에 민감 정보 금지

### Vault 작업 체크리스트
1. 로컬 `keys/vault-key.json` 전체 키 목록 확인
2. Vault 서버에 등록된 키와 비교
3. 비교 결과 테이블로 명확하게 보여주기
4. **누락된 키가 있으면 작업 진행 전에 먼저 해결**

```
| 키 이름 | 로컬 | Vault | 상태 |
|---------|------|-------|------|
| KEY_A   | O    | O     | OK   |
| KEY_B   | O    | X     | 누락! |
```

---

## 2. 인증

- **jabis-cert**가 통합인증 담당 (Microsoft OAuth + PKCE)
- Client Secret은 프론트엔드에서 사용 금지 (쿠키 기반 인증)
- CORS: 허용 Origin 명시적 등록 필수
- 내부 API(`/api/internal/*`)는 K3S 내부 네트워크에서만 접근 가능

### 인증 플로우
```
클라이언트 앱 → jabis-cert/api/oauth/authorize → Microsoft OAuth (PKCE)
→ jabis-cert callback → 클라이언트/auth/callback → userinfo (쿠키 인증)
```

### 미리보기용 Dev Token 인증 bypass

미리보기(ngrok) 환경에서는 OAuth 로그인이 불가하므로(cross-domain 쿠키 제한), API 게이트웨이에 **Dev Token 기반 인증 bypass**를 사용한다.

**동작 원리:**
1. API 게이트웨이에 `DEV_ACCESS_TOKEN` 환경변수 설정 (Dockerfile `ENV`)
2. 프론트엔드 미리보기 빌드 시 `VITE_DEV_TOKEN` 환경변수로 동일한 토큰 설정 (`.env.preview`)
3. 프론트엔드가 API 요청 시 `X-Dev-Token` 헤더에 토큰을 포함
4. 게이트웨이의 `getUserId()`에서 토큰 일치 시 `dev-user`로 인증 처리

**적용 범위:**
- `approval` 라우트 (전자결재)
- `organization` 라우트 (조직/부서)

**보안 고려사항:**
- 운영 환경에서도 토큰이 설정되어 있으므로, 토큰이 유출되면 인증 없이 API 접근 가능
- 토큰은 `.env.preview`에만 설정하고 `.env.production`에는 설정하지 않음 (운영 빌드에 토큰 미포함)
- JABIS는 사내 시스템이며 게이트웨이 도메인은 외부 비공개이므로 리스크는 제한적

```
# 미리보기 API 호출 흐름
미리보기 브라우저 (ngrok)
  → fetch('https://jabis-gateway.jinhakapply.com/api/approval/forms',
      { headers: { 'X-Dev-Token': '...' } })
  → 게이트웨이: 토큰 검증 → dev-user로 처리 → 정상 응답
```

---

## 3. DB 보안

- 파라미터 바인딩 필수 (`$1`, `$2` 또는 `@param`) — SQL Injection 방지
- DB 에러 메시지 클라이언트 노출 금지 — 일반 메시지로 치환
- SELECT 외 쿼리(INSERT/UPDATE/DELETE)는 parameters 검증 필수

---

## 4. 배포 보안

- Docker 이미지에 시크릿 포함 금지
- Vault Agent init container로 시크릿 주입
- Helm values에 민감 정보 직접 기재 금지
- Bitbucket Pipeline에서 환경변수로 Harbor 인증 처리

---

## 5. 코드 보안

| 항목 | 규칙 |
|------|------|
| XSS | 사용자 입력 sanitize 필수 |
| CSRF | 쿠키 기반 인증 시 SameSite 설정 |
| 의존성 | 정기적 `npm audit` 실행 |
| 로그 | 민감 정보(비밀번호, 토큰) 로그 출력 금지 |

---

## 6. AI 개발 보안

### 6.1 AI에게 전달 금지 항목

AI(Claude Code, Claude.ai) 프롬프트에 **절대 포함하지 말아야 할 정보:**

| 분류 | 금지 항목 |
|------|----------|
| 인증 정보 | 비밀번호, API Key, 토큰, 인증서 Private Key |
| 접속 정보 | 프로덕션 DB 접속 정보, 내부 서버 IP/포트 |
| 개인정보 | 실명, 주민번호, 전화번호, 주소, 계좌번호 |
| 사업 기밀 | 미공개 사업 계획, 계약 조건, 재무 데이터 |
| 보안 설정 | 방화벽 규칙, 보안 그룹 설정, Vault 경로 상세 |

### 6.2 AI 생성 코드 보안 검증 체크리스트

- [ ] SQL Injection 방지: ORM 또는 파라미터 바인딩 사용 여부
- [ ] XSS 방지: 사용자 입력의 이스케이프 처리 여부
- [ ] 인증/인가: API 엔드포인트에 적절한 권한 검증 존재 여부
- [ ] 입력 검증: 시스템 경계에서 사용자 입력 검증 여부
- [ ] 에러 처리: 민감 정보가 에러 응답에 노출되지 않는지
- [ ] 하드코딩: URL, 포트, 비밀키가 하드코딩되지 않았는지
- [ ] 로깅: 민감 정보가 로그에 출력되지 않는지

### 6.3 Claude Code deny 규칙 (물리적 차단)

`.claude/settings.json`의 deny 규칙으로 위험 명령을 물리적으로 차단합니다:

```json
{
  "deny": [
    "Bash(rm -rf *)",
    "Bash(git push --force*)",
    "Bash(git reset --hard*)",
    "Bash(git clean -f*)",
    "Bash(git config *)",
    "Bash(*--no-verify*)"
  ]
}
```

> deny 규칙은 프로젝트 전체에 **강제 적용**되며, 개인 설정으로 우회할 수 없습니다.

### 6.4 Agent Teams (멀티 에이전트) 보안

| 항목 | 규칙 |
|------|------|
| deny 규칙 상속 | 서브에이전트는 메인 에이전트의 deny 규칙을 자동 상속 |
| SubagentStart 알림 | 서브에이전트 시작 시 보안 규칙 전파 확인 hook 설정 |
| 데이터 격리 | 서브에이전트 간 민감 데이터 직접 전달 금지 |
| 병렬 작업 범위 | 서브에이전트에게 프로덕션 환경 접근 명령 위임 금지 |
| 감사 추적 | 서브에이전트가 수행한 파일 변경도 동일한 감사 기준 적용 |

### 6.5 디버그 로그 보안

```typescript
// 금지: 토큰/키를 로그에 출력
console.log('API response:', response)         // response에 토큰 포함 가능
console.log('Auth header:', headers.authorization) // 토큰 직접 노출

// 권장: 민감 필드 마스킹 후 출력
console.log('API response status:', response.status)
console.log('Auth header:', headers.authorization ? '[MASKED]' : 'none')
```

주의 항목:
- OAuth 토큰, Refresh Token이 에러 응답에 포함되지 않는지 확인
- API Key가 요청/응답 로깅에 노출되지 않는지 확인
- Stack trace에 환경 변수 값이 포함되지 않는지 확인

> 상세 ISMS 보안 가이드는 전사 AI 표준의 `SECURITY_ISMS.md` 참조
> AI 협업 정책 전체는 [AI 협업 정책](ai-collaboration.md) 참조
