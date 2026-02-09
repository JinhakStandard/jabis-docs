# JABIS Auto-Debugger 바이브코딩 프롬프트

> 비개발자가 만든 JABIS 기능의 오류를 자동으로 감지하고 수정하는 시스템

---

## 시스템 개요

```
[비개발자 바이브코딩]
        │
        ▼
[Git Push → ArgoCD 배포]
        │
        ▼
[K3S Pod 실행 + 로그 수집]
        │
        ▼
[Error Detection Service]
        │
        ▼
[Debug Agent PC로 알림]
        │
        ▼
[AI Orchestra 자동 디버깅]
        │
        ▼
[자동 Commit & 재배포]
```

---

## Part 1: 로그 수집 Sidecar 컨테이너

### 프롬프트 1-1: JABIS Log Collector Sidecar

```
Node.js + TypeScript로 K3S sidecar 컨테이너용 로그 수집기를 만들어줘.

## 기능 요구사항

1. **로그 수집**
   - 메인 컨테이너의 stdout/stderr를 실시간 캡처
   - 공유 볼륨 `/var/log/app`에서 로그 파일 tail
   - 로그 포맷 자동 감지 (JSON, plain text)

2. **에러 감지**
   - JavaScript/Node.js 에러 패턴 감지
     - `Error:`, `TypeError:`, `ReferenceError:`
     - `Uncaught Exception`, `Unhandled Rejection`
     - Stack trace 패턴 (`at ... (file:line:col)`)
   - React 에러 패턴
     - `React error boundary`, `Minified React error`
   - HTTP 에러 (4xx, 5xx 응답)

3. **구조화 및 전송**
   - 감지된 에러를 구조화된 JSON으로 변환
   - Redis Stream으로 전송 (jabis:errors:stream)
   - 전송 실패 시 로컬 버퍼링

4. **환경 변수**
   - JABIS_PROJECT_ID: 프로젝트 식별자
   - JABIS_REPO_URL: Bitbucket 저장소 URL
   - JABIS_BRANCH: 배포된 브랜치명
   - JABIS_COMMIT: 배포된 커밋 해시
   - REDIS_URL: Redis 연결 주소

## 타입 정의

```typescript
interface JabisLogEntry {
  timestamp: string;
  level: 'info' | 'warn' | 'error' | 'fatal';
  projectId: string;
  repoUrl: string;
  branch: string;
  commitHash: string;
  podName: string;
  namespace: string;
  message: string;
  stack?: string;
  context?: {
    requestId?: string;
    userId?: string;
    endpoint?: string;
  };
}

interface ErrorEvent {
  id: string;
  projectId: string;
  severity: 'critical' | 'error' | 'warning';
  error: {
    type: string;
    message: string;
    stack: string;
    file?: string;
    line?: number;
    column?: number;
  };
  metadata: {
    repoUrl: string;
    branch: string;
    commitHash: string;
    podName: string;
    namespace: string;
  };
  relatedLogs: JabisLogEntry[];
  timestamp: string;
}
```

## 프로젝트 구조

```
jabis-log-collector/
├── src/
│   ├── index.ts              # 진입점
│   ├── collectors/
│   │   ├── StdoutCollector.ts
│   │   └── FileCollector.ts
│   ├── parsers/
│   │   ├── ErrorParser.ts    # 에러 패턴 파싱
│   │   └── StackTraceParser.ts
│   ├── transport/
│   │   ├── RedisTransport.ts
│   │   └── BufferManager.ts  # 오프라인 버퍼
│   └── types.ts
├── Dockerfile
├── package.json
└── tsconfig.json
```

Dockerfile은 node:20-alpine 기반으로 만들고, 
메모리 제한 128MB 내에서 동작하도록 최적화해줘.
```

---

### 프롬프트 1-2: K3S Deployment 설정

```
JABIS Node.js 앱과 Log Collector Sidecar를 함께 배포하는 K3S Deployment YAML을 만들어줘.

## 요구사항

1. **메인 컨테이너**
   - 이미지: harbor.jinhak.com/jabis/${PROJECT_ID}:${TAG}
   - 포트: 3000
   - 리소스: CPU 500m, Memory 512Mi

2. **Sidecar 컨테이너**
   - 이미지: harbor.jinhak.com/jabis/log-collector:latest
   - 공유 볼륨으로 로그 디렉토리 마운트
   - 환경 변수로 프로젝트 정보 주입

3. **레이블**
   - jabis.jinhak.com/project-id: ${PROJECT_ID}
   - jabis.jinhak.com/repo-url: ${REPO_URL}
   - jabis.jinhak.com/branch: ${BRANCH}

4. **ConfigMap/Secret 연동**
   - Redis 연결 정보
   - 프로젝트별 설정

Helm Chart 형태로 만들면 더 좋아.
```

---

## Part 2: Error Detection Service

### 프롬프트 2-1: Error Detection & Deduplication Service

```
Node.js + TypeScript로 Redis Stream에서 에러를 소비하고 중복 제거 후 
디버그 태스크를 생성하는 서비스를 만들어줘.

## 기능 요구사항

1. **Redis Stream Consumer**
   - Consumer Group으로 에러 이벤트 소비
   - 여러 인스턴스에서 분산 처리 가능

2. **에러 중복 제거**
   - 동일 에러 판단 기준:
     - 같은 프로젝트
     - 같은 파일/라인
     - 같은 에러 메시지 (해시 비교)
   - 5분 내 동일 에러는 카운트만 증가
   - 새로운 에러만 디버그 태스크 생성

3. **심각도 분류**
   - critical: Uncaught Exception, 서버 크래시
   - error: 일반 에러, 4xx/5xx 반복
   - warning: 경고 레벨 로그

4. **디버그 태스크 생성**
   - PostgreSQL에 태스크 저장
   - WebSocket으로 Debug Agent에 알림
   - 프로젝트별 우선순위 큐 관리

## 타입 정의

```typescript
interface DebugTask {
  id: string;
  projectId: string;
  projectName: string;
  repoUrl: string;
  branch: string;
  commitHash: string;
  
  error: {
    type: string;
    message: string;
    stack: string;
    file?: string;
    line?: number;
  };
  
  severity: 'critical' | 'error' | 'warning';
  status: 'pending' | 'assigned' | 'analyzing' | 'fixing' | 'testing' | 'committed' | 'failed';
  
  assignedAgent?: string;  // Debug Agent PC ID
  attempts: number;
  maxAttempts: number;     // 기본 3
  
  relatedLogs: any[];
  
  createdAt: string;
  updatedAt: string;
  completedAt?: string;
  
  result?: {
    success: boolean;
    fixCommit?: string;
    analysis?: string;
    changes?: string[];
  };
}
```

## 프로젝트 구조

```
jabis-error-detector/
├── src/
│   ├── index.ts
│   ├── consumers/
│   │   └── ErrorStreamConsumer.ts
│   ├── services/
│   │   ├── DeduplicationService.ts
│   │   ├── TaskCreator.ts
│   │   └── NotificationService.ts
│   ├── repositories/
│   │   └── DebugTaskRepository.ts
│   ├── websocket/
│   │   └── AgentNotifier.ts
│   └── types.ts
├── prisma/
│   └── schema.prisma
├── Dockerfile
└── package.json
```

Prisma로 PostgreSQL 스키마 정의하고,
실시간 WebSocket 알림은 Socket.IO 사용해줘.
```

---

## Part 3: Debug Agent (PC 상주 프로그램)

### 프롬프트 3-1: Debug Agent Core

```
Windows/Mac에서 상주 실행되며 JABIS 에러를 자동으로 수정하는 
Debug Agent를 Node.js + TypeScript로 만들어줘.

## 기능 요구사항

1. **서버 연결**
   - WebSocket으로 Error Detection Service 연결
   - 자동 재연결 (exponential backoff)
   - 연결 상태 시스템 트레이 표시

2. **태스크 수신 및 처리**
   - 디버그 태스크 수신
   - 우선순위(critical > error > warning) 순서대로 처리
   - 동시 처리 제한 (기본 1개)

3. **Git 작업**
   - 소스 없으면: git clone
   - 소스 있으면: git fetch && git checkout {branch} && git pull
   - 작업용 브랜치 생성: fix/{task-id}

4. **AI Orchestra 연동**
   - DebuggerAgent 호출
   - 에러 컨텍스트 전달
   - 수정 결과 수집

5. **수정 완료 처리**
   - npm test / npm run lint 실행
   - 테스트 통과 시 commit & push
   - 원본 브랜치에 PR 생성 또는 직접 머지

6. **결과 보고**
   - 성공/실패 결과를 서버에 전송
   - 로컬 히스토리 저장

## 설정 파일

```yaml
# ~/.jabis-debug-agent/config.yaml
server:
  url: wss://jabis-monitor.jinhak.com
  reconnectInterval: 5000

git:
  username: jabis-bot
  email: jabis-bot@jinhak.com
  credentialHelper: manager  # Windows credential manager

workspace:
  basePath: ~/jabis-debug-workspace
  maxProjects: 10

orchestra:
  path: ~/.local/bin/ai-orchestra
  model: opus
  autoFix: true
  autoFixConfidence: 0.8

processing:
  concurrent: 1
  priority:
    critical: 1
    error: 2
    warning: 3
```

## 프로젝트 구조

```
jabis-debug-agent/
├── src/
│   ├── index.ts              # 진입점
│   ├── agent/
│   │   ├── DebugAgent.ts     # 메인 에이전트 클래스
│   │   └── TaskProcessor.ts  # 태스크 처리 로직
│   ├── git/
│   │   ├── GitManager.ts     # Git 작업 관리
│   │   └── PRCreator.ts      # PR 생성 (Bitbucket API)
│   ├── orchestra/
│   │   └── OrchestraRunner.ts  # AI Orchestra 실행
│   ├── websocket/
│   │   └── ServerConnection.ts
│   ├── ui/
│   │   └── TrayIcon.ts       # 시스템 트레이
│   ├── config/
│   │   └── ConfigLoader.ts
│   └── types.ts
├── package.json
└── tsconfig.json
```

시스템 트레이 아이콘은 electron 없이 
'systray2' 패키지로 가볍게 만들어줘.
```

---

### 프롬프트 3-2: AI Orchestra 디버깅 통합

```
Debug Agent에서 AI Orchestra DebuggerAgent를 호출하는 
OrchestraRunner 클래스를 만들어줘.

## 기능 요구사항

1. **프롬프트 구성**
   - 에러 정보 (메시지, 스택 트레이스)
   - 관련 로그 컨텍스트
   - 프로젝트 구조
   - 최근 커밋 히스토리 (git log -5)

2. **실행 모드 선택**
   - critical 에러: full 모드 (Planner → Coder → Debugger → Reviewer)
   - error: medium 모드 (Coder + Debugger)
   - warning: simple 모드 (단일 수정)

3. **결과 파싱**
   - 수정된 파일 목록
   - 변경 내용 요약
   - 테스트 결과

4. **롤백 지원**
   - 수정 전 git stash
   - 테스트 실패 시 롤백

## 구현

```typescript
// src/orchestra/OrchestraRunner.ts

import { spawn } from 'child_process';
import * as path from 'path';
import { DebugTask } from '../types';

export interface OrchestraResult {
  success: boolean;
  analysis: string;
  changes: FileChange[];
  testsPassed: boolean;
  duration: number;
}

export interface FileChange {
  file: string;
  type: 'modified' | 'created' | 'deleted';
  summary: string;
}

export class OrchestraRunner {
  constructor(
    private orchestraPath: string,
    private config: OrchestraConfig
  ) {}

  async runDebug(task: DebugTask, projectPath: string): Promise<OrchestraResult> {
    // 프롬프트 생성
    const prompt = this.buildDebugPrompt(task);
    
    // 모드 결정
    const mode = this.selectMode(task.severity);
    
    // 실행
    // ...
  }

  private buildDebugPrompt(task: DebugTask): string {
    return `
# 에러 수정 요청

## 에러 정보
- 유형: ${task.error.type}
- 메시지: ${task.error.message}
- 파일: ${task.error.file || '알 수 없음'}
- 라인: ${task.error.line || '알 수 없음'}

## 스택 트레이스
\`\`\`
${task.error.stack}
\`\`\`

## 관련 로그
${task.relatedLogs.map(log => `[${log.timestamp}] ${log.level}: ${log.message}`).join('\n')}

## 지시사항
1. 에러 원인을 분석하세요
2. 최소한의 변경으로 에러를 수정하세요
3. 기존 기능에 영향을 주지 않도록 주의하세요
4. 수정 후 관련 테스트가 통과하는지 확인하세요
`;
  }

  private selectMode(severity: 'critical' | 'error' | 'warning'): string {
    switch (severity) {
      case 'critical': return 'full';
      case 'error': return 'medium';
      case 'warning': return 'simple';
    }
  }
}
```

실제 ai-orchestra CLI 호출 및 결과 파싱 로직을 완성해줘.
```

---

### 프롬프트 3-3: Bitbucket PR 생성

```
Bitbucket Server REST API를 사용해서 PR을 자동 생성하는 
PRCreator 클래스를 만들어줘.

## 기능 요구사항

1. **PR 생성**
   - fix/{task-id} 브랜치 → 원본 브랜치
   - AI가 생성한 분석 결과를 PR 설명에 포함
   - 자동 라벨링: "auto-fix", "jabis-debugger"

2. **리뷰어 자동 지정**
   - 프로젝트 설정에서 기본 리뷰어 지정
   - 또는 원본 커밋 작성자 지정

3. **자동 머지 옵션**
   - 테스트 통과 + 신뢰도 높은 수정은 자동 머지
   - 그 외는 PR로 남김

## Bitbucket API

```typescript
// Bitbucket Server REST API
const BITBUCKET_BASE = 'https://bitbucket.jinhak.com/rest/api/1.0';

// PR 생성
POST /projects/{projectKey}/repos/{repositorySlug}/pull-requests
{
  "title": "[Auto-Fix] {에러 타입} 수정",
  "description": "{분석 결과}",
  "fromRef": {
    "id": "refs/heads/fix/{task-id}"
  },
  "toRef": {
    "id": "refs/heads/{original-branch}"
  },
  "reviewers": [
    { "user": { "name": "{reviewer}" } }
  ]
}
```

에러 핸들링과 재시도 로직도 포함해줘.
```

---

## Part 4: 모니터링 대시보드

### 프롬프트 4-1: Debug Status Dashboard

```
React + TypeScript로 JABIS Auto-Debugger 상태를 모니터링하는 
대시보드를 만들어줘.

## 기능 요구사항

1. **실시간 에러 피드**
   - WebSocket으로 실시간 에러 표시
   - 프로젝트별 필터링
   - 심각도별 색상 구분

2. **디버그 태스크 목록**
   - 상태별 분류: pending, fixing, completed, failed
   - 각 태스크 상세 정보 조회
   - 수동 재시도 버튼

3. **Debug Agent 상태**
   - 연결된 Agent PC 목록
   - 각 Agent의 현재 작업 상태
   - 처리 통계

4. **프로젝트별 통계**
   - 에러 발생 추이 (차트)
   - 자동 수정 성공률
   - 평균 수정 시간

5. **알림 설정**
   - 특정 프로젝트 에러 알림
   - 심각도별 알림 설정

## 기술 스택

- React 18 + TypeScript
- TanStack Query (데이터 fetching)
- Socket.IO Client (실시간)
- Recharts (차트)
- Tailwind CSS
- shadcn/ui 컴포넌트

## 프로젝트 구조

```
jabis-debug-dashboard/
├── src/
│   ├── App.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── TaskDetail.tsx
│   │   ├── Projects.tsx
│   │   └── Agents.tsx
│   ├── components/
│   │   ├── ErrorFeed.tsx
│   │   ├── TaskList.tsx
│   │   ├── AgentStatus.tsx
│   │   └── StatsChart.tsx
│   ├── hooks/
│   │   ├── useSocket.ts
│   │   └── useDebugTasks.ts
│   ├── api/
│   │   └── debugApi.ts
│   └── types.ts
├── package.json
└── vite.config.ts
```

Vite로 빌드하고, 진학어플라이 디자인 시스템에 맞게 
파란색(#2563eb) 기반으로 디자인해줘.
```

---

## Part 5: 통합 설정

### 프롬프트 5-1: 프로젝트 등록 및 설정

```
JABIS 프로젝트를 Auto-Debugger에 등록하는 설정 시스템을 만들어줘.

## 기능 요구사항

1. **프로젝트 등록 API**
   ```typescript
   interface JabisProject {
     id: string;
     name: string;
     repoUrl: string;
     defaultBranch: string;
     
     // Debug 설정
     debugConfig: {
       enabled: boolean;
       autoFix: boolean;           // 자동 수정 활성화
       autoMerge: boolean;         // 테스트 통과 시 자동 머지
       confidenceThreshold: number; // 자동 수정 신뢰도 임계값 (0.0 ~ 1.0)
       maxAttempts: number;        // 최대 재시도 횟수
     };
     
     // 알림 설정
     notification: {
       slack?: string;             // Slack 채널
       email?: string[];           // 이메일 목록
     };
     
     // 담당자
     owners: string[];             // 기본 리뷰어/알림 대상
     
     createdAt: string;
     updatedAt: string;
   }
   ```

2. **설정 UI**
   - 프로젝트 목록 관리
   - 프로젝트별 상세 설정
   - 일괄 설정 변경

3. **K3S 연동**
   - Pod 레이블과 프로젝트 매핑
   - 자동 등록 (새 Pod 감지 시)

프로젝트 등록 화면과 API를 만들어줘.
```

---

## Part 6: 전체 시스템 배포

### 프롬프트 6-1: Helm Chart 및 배포

```
JABIS Auto-Debugger 전체 시스템을 K3S에 배포하는 Helm Chart를 만들어줘.

## 구성 요소

1. **jabis-error-detector**
   - Deployment (2 replicas)
   - Service
   - HPA

2. **jabis-debug-dashboard**
   - Deployment
   - Service
   - Ingress (jabis-debug.jinhak.com)

3. **Redis** (로그 스트림용)
   - StatefulSet
   - 기존 Redis 사용 시 외부 연결

4. **PostgreSQL** (태스크 저장용)
   - 기존 jabis-db 사용

## values.yaml 예시

```yaml
global:
  imageRegistry: harbor.jinhak.com/jabis

errorDetector:
  replicaCount: 2
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
  redis:
    external: true
    host: jabis-redis.jabis.svc.cluster.local
    port: 6379

dashboard:
  replicaCount: 1
  ingress:
    enabled: true
    host: jabis-debug.jinhak.com

agent:
  # Debug Agent는 PC에 직접 설치
  websocketUrl: wss://jabis-debug.jinhak.com/agent
```

Helm Chart 전체와 설치 가이드를 만들어줘.
```

---

## 구현 순서 가이드

### Phase 1: 기본 인프라 (1주)
1. 프롬프트 1-1: Log Collector Sidecar
2. 프롬프트 1-2: K3S Deployment 설정

### Phase 2: 에러 감지 (1주)
3. 프롬프트 2-1: Error Detection Service

### Phase 3: Debug Agent (2주)
4. 프롬프트 3-1: Debug Agent Core
5. 프롬프트 3-2: AI Orchestra 통합
6. 프롬프트 3-3: Bitbucket PR 생성

### Phase 4: 대시보드 (1주)
7. 프롬프트 4-1: Debug Status Dashboard

### Phase 5: 통합 및 배포 (1주)
8. 프롬프트 5-1: 프로젝트 설정
9. 프롬프트 6-1: Helm Chart

---

## 보안 고려사항

1. **Git Credentials**
   - Debug Agent PC에 Bitbucket 토큰 안전하게 저장
   - Windows Credential Manager / macOS Keychain 활용

2. **WebSocket 인증**
   - JWT 토큰 기반 인증
   - Agent 등록 시 고유 키 발급

3. **자동 수정 제한**
   - 특정 파일/디렉토리 수정 불가 설정 (예: 인증 관련)
   - 일일 자동 수정 횟수 제한

4. **감사 로그**
   - 모든 자동 수정 기록
   - 롤백 히스토리 보관

---

## 운영 시나리오

### 시나리오 1: 일반적인 에러 수정

```
1. 비개발자가 JABIS에 새 기능 추가 (바이브코딩)
2. Git push → ArgoCD 배포
3. 사용자 접속 시 TypeError 발생
4. Log Collector가 에러 감지 → Redis Stream 전송
5. Error Detector가 에러 수신 → 중복 아님 확인
6. Debug Task 생성 → Debug Agent PC로 알림
7. Debug Agent가 소스 pull
8. AI Orchestra DebuggerAgent 실행
9. 수정 완료 → npm test 통과
10. 자동 commit & push
11. ArgoCD 자동 배포
12. 에러 해결!
```

### 시나리오 2: 복잡한 에러 (AI 수정 실패)

```
1-6. 위와 동일
7. Debug Agent가 소스 pull
8. AI Orchestra 실행 → 수정 시도
9. npm test 실패 → 롤백
10. 재시도 (최대 3회)
11. 3회 모두 실패
12. PR 생성 + Slack 알림
13. 담당자가 수동 수정
```

### 시나리오 3: Critical 에러 (즉시 대응)

```
1. Uncaught Exception으로 서버 다운
2. 모든 replica에서 에러 감지
3. 중복 제거 후 단일 critical 태스크 생성
4. 최우선순위로 Debug Agent 할당
5. Full 모드로 AI Orchestra 실행
6. 빠른 수정 → 테스트 → 자동 머지
7. 긴급 배포 완료
```
