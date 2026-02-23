# jabis-storage API 명세

> jabis-api-gateway를 통해 프록시되는 파일 스토리지 API.
> 프론트엔드에서 직접 jabis-storage에 접근하지 않고, gateway를 경유한다.

---

## 공통 사항

### Base URL
```
https://jabis-gateway.jinhakapply.com/api/storage
```

### 공통 응답 형식
성공 시:
```json
{
  "success": true,
  "data": { ... }
}
```
에러 시:
```json
{
  "success": false,
  "error": "에러 메시지"
}
```

### 파일 크기 제한
- 최대 50MB (`52428800` bytes)

---

## 엔드포인트

### POST /api/storage/upload

파일 업로드. `multipart/form-data` 형식.

#### 요청
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `file` | File | O | 업로드할 파일 (최대 50MB) |
| `fileDir` | string | O | 저장 디렉토리 경로 (예: `finance/receipts`) |

#### curl 예시
```bash
curl -X POST \
  -F "file=@보고서.pdf" \
  -F "fileDir=finance/receipts" \
  https://jabis-gateway.jinhakapply.com/api/storage/upload
```

#### 응답 (200)
```json
{
  "success": true,
  "data": {
    "fileName": "보고서.pdf",
    "objectPath": "finance/receipts/보고서.pdf",
    "size": 102400,
    "downloadUrl": "/api/storage/download?fileDir=finance/receipts&fileName=%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf"
  }
}
```

---

### GET /api/storage/download

파일 다운로드. 바이너리 스트림으로 응답.

#### 쿼리 파라미터
| 파라미터 | 필수 | 설명 |
|---------|------|------|
| `fileDir` | O | 디렉토리 경로 |
| `fileName` | O | 파일명 (URL 인코딩) |

#### curl 예시
```bash
curl -O "https://jabis-gateway.jinhakapply.com/api/storage/download?fileDir=finance/receipts&fileName=%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf"
```

#### 응답 헤더
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="보고서.pdf"
Content-Length: 102400
```

> 이미지 파일(jpg, png, gif, webp 등)은 `Content-Disposition: inline`으로 반환.

---

### GET /api/storage/list

파일 목록 조회.

#### 쿼리 파라미터
| 파라미터 | 필수 | 설명 |
|---------|------|------|
| `prefix` | X | 필터링할 경로 prefix (예: `finance/`) |

#### curl 예시
```bash
curl "https://jabis-gateway.jinhakapply.com/api/storage/list?prefix=finance/receipts/"
```

#### 응답 (200)
```json
{
  "success": true,
  "data": {
    "files": [
      {
        "name": "finance/receipts/보고서.pdf",
        "size": 102400,
        "lastModified": "2026-02-23T08:50:24.332Z",
        "etag": "3af201ef8d0c0ab2fbc56ac0dbdfdbf1"
      }
    ],
    "prefix": "finance/receipts/",
    "count": 1
  }
}
```

---

### POST /api/storage/manage

파일 관리 (삭제, 목록). `action` 필드로 구분.

#### 삭제 요청
```json
{
  "action": "delete",
  "fileDir": "finance/receipts",
  "fileName": "보고서.pdf"
}
```

#### 삭제 응답 (200)
```json
{
  "success": true,
  "data": {
    "message": "파일이 삭제되었습니다.",
    "objectPath": "finance/receipts/보고서.pdf"
  }
}
```

#### 목록 요청
```json
{
  "action": "list",
  "prefix": "finance/receipts/"
}
```

---

## 프론트엔드 사용 예시

### 파일 업로드 (React)
```javascript
const GATEWAY_URL = import.meta.env.VITE_API_URL; // https://jabis-gateway.jinhakapply.com

async function uploadFile(file, fileDir) {
  const formData = new FormData();
  formData.append('file', file);
  formData.append('fileDir', fileDir);

  const res = await fetch(`${GATEWAY_URL}/api/storage/upload`, {
    method: 'POST',
    body: formData,
    // Content-Type은 설정하지 않음 (브라우저가 boundary 자동 생성)
  });

  return res.json();
}
```

### 파일 다운로드 (브라우저)
```javascript
function downloadFile(fileDir, fileName) {
  const params = new URLSearchParams({ fileDir, fileName });
  window.open(`${GATEWAY_URL}/api/storage/download?${params}`, '_blank');
}
```

### 파일 삭제
```javascript
async function deleteFile(fileDir, fileName) {
  const res = await fetch(`${GATEWAY_URL}/api/storage/manage`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action: 'delete', fileDir, fileName }),
  });

  return res.json();
}
```

---

## 아키텍처

```
프론트엔드 (React SPA)
    ↓ HTTPS
jabis-api-gateway (프록시)
    ↓ HTTP (K3S 내부)
jabis-storage (Fastify + MinIO)
    ↓
MinIO (minio-apply.jinhaksa.com:9100)
    ↓
jabis-bucket
```
