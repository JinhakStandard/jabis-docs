# JabisLab 프론트엔드 연동 가이드

## Swagger UI 접근 방법

### 개발 환경

API Gateway 개발 서버가 실행 중일 때:

```
http://localhost:3100/api-docs
```

JabisLab 프론트엔드 개발 서버를 통해서도 접근 가능합니다 (Vite 프록시 설정됨):

```
http://localhost:5173/api-docs
```

### 운영 환경

운영 환경에서는 `OPENAPI_DOCS_ENABLED=true` 환경변수가 설정된 경우에만 접근 가능합니다:

```
https://jabis-gateway.jinhakapply.com/api-docs
```

## 명세 파일 직접 접근

| URL | 형식 | 설명 |
|-----|------|------|
| `/api-docs/openapi.yaml` | YAML | 원본 OpenAPI 명세 |
| `/api-docs/openapi.json` | JSON | JSON 변환된 명세 |

## API 그룹 안내

| 태그 | 설명 | 주요 용도 |
|------|------|----------|
| Health | 헬스체크, Prometheus 메트릭 | 모니터링 |
| Authentication | 인증 테스트 | 토큰 검증 |
| AI Orchestra | 세션 이벤트, 동기화 | Orchestra 클라이언트 연동 |
| Night Builder | 기능 요청, 디버그 작업 | Night Builder 에이전트 연동 |
| Dashboard - Orchestra | 세션/인스턴스/개발자 대시보드 | JabisLab 대시보드 페이지 |
| Dashboard - Night Builder | 요청/작업 대시보드 | JabisLab 대시보드 페이지 |
| Performance | 성능 통계/병목/알림 | 서버 모니터링 |

## Vite 프록시 설정

JabisLab의 `apps/prototype/vite.config.js`에 다음 프록시가 설정되어 있습니다:

```javascript
proxy: {
  '/api/dashboard': {
    target: dashboardApiUrl,
    changeOrigin: true,
    secure: false,
  },
  '/api-docs': {
    target: dashboardApiUrl,
    changeOrigin: true,
    secure: false,
  },
}
```

`dashboardApiUrl`은 `VITE_DASHBOARD_API_TARGET` 환경변수로 설정되며, 기본값은 `https://jabis-gateway.jinhakapply.com`입니다.

## 로컬 개발 시 명세 린트

```bash
# API Gateway 프로젝트에서
npm run openapi:lint        # 명세 문법 검증
npm run openapi:bundle      # 단일 YAML 번들링 (dist/openapi.yaml)
npm run openapi:bundle:json # JSON 변환 포함 번들링
npm run openapi:preview     # Redocly 프리뷰 서버
```
