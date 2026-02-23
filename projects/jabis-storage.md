# jabis-storage

MinIO 기반 파일 스토리지 서비스. 파일 업로드, 다운로드, 삭제, 목록 조회 기능을 제공한다.

## 기술 스택

- **런타임**: Node.js 20+
- **프레임워크**: Fastify 4
- **언어**: TypeScript (strict mode)
- **스토리지**: MinIO (S3 호환)
- **로깅**: Winston
- **검증**: Zod (환경변수)

## 프로젝트 구조

```
jabis-storage/
├── src/
│   ├── server.ts              # 엔트리 포인트 + graceful shutdown
│   ├── app.ts                 # Fastify 앱 빌더 (CORS, Helmet, Multipart)
│   ├── config/index.ts        # Zod 환경변수 검증
│   ├── utils/
│   │   ├── logger.ts          # Winston 로거
│   │   ├── vault.ts           # Vault 시크릿 로더
│   │   └── mimeTypes.ts       # MIME 타입 매핑
│   ├── middleware/
│   │   ├── requestId.ts       # X-Request-ID
│   │   ├── requestLogger.ts   # 요청/응답 로깅
│   │   └── errorHandler.ts    # AppError 클래스 + 전역 에러 핸들러
│   ├── services/
│   │   └── minioService.ts    # MinIO 클라이언트 래퍼 (싱글톤)
│   ├── routes/
│   │   ├── index.ts           # 라우트 등록
│   │   ├── health.ts          # /health, /health/ready
│   │   └── storage.ts         # /api/storage/* 엔드포인트
│   └── types/index.ts         # TypeScript 인터페이스
├── Dockerfile                 # Multi-stage 빌드
├── bitbucket-pipelines.yml    # CI/CD 파이프라인
└── package.json
```

## 환경 설정

| 변수 | Preview | Production | 기본값 |
|------|---------|------------|--------|
| PORT | 3400 | 3400 | 3400 |
| MINIO_ENDPOINT | minio-dev.jinhaksa.com | minio-apply.jinhaksa.com | minio-dev.jinhaksa.com |
| MINIO_PORT | 9000 | 9100 | 9000 |
| MINIO_USE_SSL | true | true | true |
| MINIO_ACCESS_KEY | (vault) | (vault) | — |
| MINIO_SECRET_KEY | (vault) | (vault) | — |
| MINIO_BUCKET | jabis-bucket | jabis-bucket | jabis-bucket |
| MAX_FILE_SIZE | 52428800 | 52428800 | 52428800 (50MB) |

## API 엔드포인트

### GET /health
헬스체크. 항상 200 반환.

### GET /health/ready
MinIO 연결 상태 포함 레디니스 프로브.

### POST /api/storage/upload
파일 업로드. `multipart/form-data` 형식.

- `file` (필수): 업로드할 파일
- `fileDir` (필수): MinIO 내 디렉토리 경로 (예: `jabis/documents`)

```json
// 응답
{
  "success": true,
  "data": {
    "fileName": "보고서.pdf",
    "objectPath": "jabis/documents/보고서.pdf",
    "size": 102400,
    "downloadUrl": "/api/storage/download?fileDir=jabis/documents&fileName=%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf"
  }
}
```

### GET /api/storage/download
파일 다운로드. 스트림으로 응답.

- `fileDir` (필수): 디렉토리 경로
- `fileName` (필수): 파일명 (URL 인코딩)

이미지 파일은 `Content-Disposition: inline`, 나머지는 `attachment`.

### GET /api/storage/list
파일 목록 조회.

- `prefix` (선택): 필터링할 경로 prefix

### POST /api/storage/manage
파일 관리 작업. `action` 필드로 구분.

```json
// 삭제
{ "action": "delete", "fileDir": "jabis/documents", "fileName": "보고서.pdf" }

// 목록
{ "action": "list", "prefix": "jabis/documents/" }
```

## 개발 명령어

```bash
npm install       # 의존성 설치
npm run dev       # 개발 서버 (tsx watch, 포트 3400)
npm run build     # TypeScript 빌드
npm start         # 프로덕션 실행
npm test          # 테스트
npm run lint      # ESLint
```

## 접근 방식 (Gateway 프록시)

jabis-storage는 별도 DNS/인그레스 없이 **jabis-api-gateway를 통해** 접근한다.

```
프론트엔드 → https://jabis-gateway.jinhakapply.com/api/storage/*
                         ↓ (프록시)
             http://jabis-storage-prod-service.jabis:3400/api/storage/*
```

- 프론트엔드에서는 `jabis-gateway.jinhakapply.com/api/storage/upload` 등으로 호출
- gateway가 내부 K3S 서비스로 투명 프록시 (multipart 업로드 포함)
- jabis-storage에 직접 접근하는 외부 DNS/인그레스는 없음

## 배포

- Bitbucket `main` 브랜치 push 시 자동 배포 (Pipeline → Harbor → Helm → K3S)
- 포트: 3400
- Helm values에 `hostAliases` 필요 (MinIO 서버 IP 매핑)
- Dockerfile ENV에 MINIO_ACCESS_KEY, MINIO_SECRET_KEY 설정

## 의존 관계

- MinIO 서버 (minio-dev / minio-apply)
- jabis-helm (Kubernetes 배포 설정)
- jabis-api-gateway (프록시 — `STORAGE_SERVICE_URL` 환경변수)

## 참고

- ara 프로젝트의 MinIO 패턴을 기반으로 구축
- `@fastify/multipart` 사용 (Express의 multer가 아님)
- 파일명 UTF-8 처리: `Buffer.from(filename, 'latin1').toString('utf8')`
- 인증 미적용 (내부 K3S 네트워크 전용). 추후 jabis-cert 연동 가능
