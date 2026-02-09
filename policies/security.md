# JABIS 보안 정책

> 이 문서는 JABIS 전체 프로젝트에 적용되는 보안 정책입니다.

---

## 1. 시크릿 관리

### 원칙
- 하드코딩된 비밀번호, API 키, 토큰 **절대 금지**
- 환경변수(`.env.*`) 또는 **Vault Agent**를 통해 주입
- `.env` 파일은 반드시 `.gitignore`에 포함

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
