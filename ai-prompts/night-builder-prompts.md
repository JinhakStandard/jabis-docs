# JABIS Night Builder - 야간 자동 구현 시스템

> 비개발자가 낮에 요청하면, AI가 밤에 자동으로 구현하고 배포하는 시스템

---

## 시스템 개요

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              업무 시간 (Day)                                     │
│                                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                    │
│  │   비개발자    │     │  AI 요청     │     │   Request    │                    │
│  │   담당자      │────▶│  챗봇 UI    │────▶│   Queue DB   │                    │
│  └──────────────┘     └──────────────┘     └──────────────┘                    │
│                                                                                 │
│  요청 예시:                                                                     │
│  - "회원가입에 휴대폰 인증 추가해주세요"                                          │
│  - "검색 결과를 엑셀로 다운로드 가능하게 해주세요"                                 │
│  - "관리자 대시보드에 일별 통계 차트 추가해주세요"                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 저녁 6시 이후
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              야간 시간 (Night)                                   │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                        Night Builder Engine                               │  │
│  │                                                                          │  │
│  │   while (queue.hasNext() && isNightTime()) {                             │  │
│  │                                                                          │  │
│  │     1. 요청 분석 (Planner Agent)                                         │  │
│  │        └─▶ DDL 필요? → API Gateway 스키마 요청                           │  │
│  │        └─▶ 외부 API 필요? → 연동 정보 확인                               │  │
│  │                                                                          │  │
│  │     2. 코드 구현 (AI Orchestra)                                          │  │
│  │        └─▶ git checkout -b feature/{request-id}                          │  │
│  │        └─▶ Coder + Debugger + Reviewer                                   │  │
│  │                                                                          │  │
│  │     3. 테스트 & 검증                                                      │  │
│  │        └─▶ npm test, lint, type-check                                    │  │
│  │        └─▶ E2E 테스트 (선택)                                             │  │
│  │                                                                          │  │
│  │     4. 배포                                                               │  │
│  │        └─▶ git push → ArgoCD 자동 배포                                   │  │
│  │        └─▶ 요청 상태 = "완료"                                            │  │
│  │                                                                          │  │
│  │   }                                                                      │  │
│  │                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                            ☀️ 아침 출근 → 구현 완료!
```

---

## Part 1: AI 요청 챗봇 UI

### 프롬프트 1-1: 요청 입력 챗봇 인터페이스

```
React + TypeScript로 JABIS 사이트에 들어갈 "AI 요청" 챗봇 UI를 만들어줘.

## 기능 요구사항

1. **챗봇 스타일 인터페이스**
   - 우측 하단 플로팅 버튼으로 열기/닫기
   - 대화형 UI (사용자 입력 + AI 응답 형태)
   - 모바일 반응형

2. **요청 입력 플로우**
   ```
   Step 1: 요청 유형 선택
   - 새 기능 추가
   - 기존 기능 수정
   - UI/UX 개선
   - 버그 수정 요청
   - 기타
   
   Step 2: 상세 내용 입력
   - 자유 텍스트 입력
   - 파일 첨부 (스크린샷, 기획서 등)
   - 참고 URL 입력
   
   Step 3: 우선순위 선택
   - 긴급 (당일 처리 시도)
   - 보통 (순서대로)
   - 낮음 (여유 있을 때)
   
   Step 4: 확인 및 제출
   - 요청 요약 보여주기
   - AI가 이해한 내용 확인 (Planner 요약)
   - 수정 또는 제출
   ```

3. **요청 목록 조회**
   - 내가 한 요청 목록
   - 상태별 필터: 대기중, 진행중, 완료, 실패
   - 상세 보기 (구현 결과, 변경된 파일 등)

4. **실시간 알림**
   - 요청 상태 변경 시 알림
   - 구현 완료 시 "확인해보세요" 링크

## 타입 정의

```typescript
interface FeatureRequest {
  id: string;
  projectId: string;
  
  // 요청 정보
  type: 'new_feature' | 'modification' | 'ui_improvement' | 'bug_fix' | 'other';
  title: string;
  description: string;
  attachments: Attachment[];
  referenceUrls: string[];
  priority: 'urgent' | 'normal' | 'low';
  
  // AI 분석 결과
  analysis?: {
    summary: string;           // AI가 이해한 요약
    estimatedComplexity: 'simple' | 'medium' | 'complex';
    requiredChanges: string[]; // 예상 변경 사항
    requiresDDL: boolean;      // DB 스키마 변경 필요 여부
    ddlDescription?: string;   // DDL 변경 설명
    estimatedTime: string;     // 예상 소요 시간
  };
  
  // 상태
  status: RequestStatus;
  statusHistory: StatusChange[];
  
  // 결과
  result?: {
    success: boolean;
    commitHash?: string;
    prUrl?: string;
    changedFiles?: string[];
    deployedAt?: string;
    errorMessage?: string;
  };
  
  // 메타
  requesterId: string;
  requesterName: string;
  createdAt: string;
  updatedAt: string;
  scheduledFor?: string;  // 처리 예정 시간
  completedAt?: string;
}

type RequestStatus = 
  | 'draft'           // 작성 중
  | 'submitted'       // 제출됨
  | 'analyzing'       // AI 분석 중
  | 'waiting_approval'// DDL 승인 대기
  | 'queued'          // 큐 대기 중
  | 'in_progress'     // 구현 중
  | 'testing'         // 테스트 중
  | 'deploying'       // 배포 중
  | 'completed'       // 완료
  | 'failed'          // 실패
  | 'cancelled';      // 취소됨

interface Attachment {
  id: string;
  filename: string;
  url: string;
  type: 'image' | 'document' | 'other';
  size: number;
}
```

## 프로젝트 구조

```
jabis-request-chatbot/
├── src/
│   ├── components/
│   │   ├── ChatBot/
│   │   │   ├── ChatBot.tsx           # 메인 컴포넌트
│   │   │   ├── ChatWindow.tsx        # 채팅 창
│   │   │   ├── MessageBubble.tsx     # 메시지 버블
│   │   │   ├── RequestForm.tsx       # 요청 폼 (Step별)
│   │   │   ├── RequestTypeSelector.tsx
│   │   │   ├── PrioritySelector.tsx
│   │   │   ├── FileUploader.tsx
│   │   │   └── ConfirmationStep.tsx
│   │   ├── RequestList/
│   │   │   ├── RequestList.tsx       # 요청 목록
│   │   │   ├── RequestCard.tsx       # 요청 카드
│   │   │   ├── RequestDetail.tsx     # 상세 모달
│   │   │   └── StatusBadge.tsx
│   │   └── common/
│   │       └── FloatingButton.tsx
│   ├── hooks/
│   │   ├── useRequests.ts
│   │   └── useSocket.ts
│   ├── api/
│   │   └── requestApi.ts
│   └── types.ts
├── package.json
└── tsconfig.json
```

## UI 디자인

```
┌─────────────────────────────────────┐
│  🤖 AI 기능 요청                 ✕  │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 안녕하세요! 어떤 기능이      │   │
│  │ 필요하신가요?               │   │
│  └─────────────────────────────┘   │
│                                     │
│       ┌─────────────────────────┐  │
│       │ 검색 결과를 엑셀로      │  │
│       │ 다운로드 할 수 있게     │  │
│       │ 해주세요.              │  │
│       └─────────────────────────┘  │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ 네, 이해했습니다! 📋        │   │
│  │                             │   │
│  │ 요약: 검색 결과 엑셀 내보내기│   │
│  │ 예상 복잡도: 보통           │   │
│  │ DB 변경: 불필요             │   │
│  │ 예상 시간: 2-3시간          │   │
│  │                             │   │
│  │ [수정하기] [제출하기 ✓]     │   │
│  └─────────────────────────────┘   │
│                                     │
├─────────────────────────────────────┤
│  💬 메시지를 입력하세요...      📎  │
└─────────────────────────────────────┘
```

진학어플라이 디자인 시스템(파란색 #2563eb)에 맞게 
shadcn/ui 컴포넌트 기반으로 만들어줘.
```

---

### 프롬프트 1-2: 요청 분석 API (Planner 연동)

```
Node.js + TypeScript로 사용자 요청을 AI가 분석하는 API를 만들어줘.

## 기능 요구사항

1. **요청 분석 엔드포인트**
   - POST /api/requests/analyze
   - 사용자 입력을 AI Planner가 분석
   - 분석 결과 반환

2. **분석 내용**
   - 요청 요약 (한 줄)
   - 복잡도 판단
   - 필요한 변경 사항 목록
   - DDL 필요 여부 및 내용
   - 예상 소요 시간
   - 의존성 (다른 기능과 연관)

3. **AI 프롬프트**

```typescript
const analyzePrompt = `
# 기능 요청 분석

## 프로젝트 정보
- 프로젝트: ${projectName}
- 기술 스택: React, Node.js, TypeScript, PostgreSQL
- 기존 구조: ${projectStructure}

## 사용자 요청
${userRequest}

## 첨부 파일
${attachments.map(a => `- ${a.filename}: ${a.description}`).join('\n')}

## 분석 지시사항

다음 형식으로 분석 결과를 JSON으로 출력하세요:

{
  "summary": "요청을 한 줄로 요약",
  "estimatedComplexity": "simple | medium | complex",
  "requiredChanges": [
    "변경 사항 1",
    "변경 사항 2"
  ],
  "affectedFiles": [
    "src/pages/Search.tsx",
    "src/api/export.ts"
  ],
  "requiresDDL": true | false,
  "ddlChanges": [
    {
      "type": "ADD_COLUMN | CREATE_TABLE | ALTER_TABLE | ...",
      "table": "테이블명",
      "description": "변경 설명",
      "sql": "ALTER TABLE ... ADD COLUMN ..."
    }
  ],
  "externalDependencies": [
    {
      "name": "xlsx",
      "version": "^0.18.5",
      "reason": "엑셀 파일 생성을 위해 필요"
    }
  ],
  "estimatedTime": "2-3시간",
  "risks": [
    "주의해야 할 사항"
  ],
  "clarificationNeeded": [
    "추가로 확인이 필요한 사항 (없으면 빈 배열)"
  ]
}
`;
```

4. **DDL 분석 시 추가 검증**
   - 기존 테이블과 충돌 여부
   - 마이그레이션 필요 여부
   - 데이터 손실 위험 여부

## API 구조

```typescript
// POST /api/requests
// 요청 생성
interface CreateRequestBody {
  type: RequestType;
  title: string;
  description: string;
  attachmentIds: string[];
  referenceUrls: string[];
  priority: Priority;
}

// POST /api/requests/:id/analyze
// 요청 분석 (AI)
interface AnalyzeResponse {
  analysis: RequestAnalysis;
  needsClarification: boolean;
  clarificationQuestions: string[];
}

// POST /api/requests/:id/submit
// 요청 제출 (큐에 추가)
interface SubmitResponse {
  queuePosition: number;
  estimatedStartTime: string;
}

// GET /api/requests
// 내 요청 목록
interface ListRequestsQuery {
  status?: RequestStatus;
  page?: number;
  limit?: number;
}
```

Express + Prisma로 구현하고,
AI 분석은 Claude API 직접 호출해줘.
```

---

## Part 2: Request Queue 관리

### 프롬프트 2-1: 요청 큐 및 스케줄러

```
Node.js + TypeScript로 요청 큐와 야간 스케줄러를 만들어줘.

## 기능 요구사항

1. **요청 큐 관리**
   - 우선순위 기반 정렬 (urgent > normal > low)
   - 같은 우선순위 내에서는 FIFO
   - 프로젝트별 격리 (한 프로젝트 실패가 다른 프로젝트에 영향 X)

2. **스케줄러**
   - 설정된 야간 시간에만 처리 (기본: 22:00 ~ 06:00)
   - 업무 시간에는 큐만 쌓고 처리 안 함
   - 처리 중 업무 시간 되면 현재 작업 완료 후 중지

3. **처리 로직**
   ```
   1. 큐에서 다음 요청 가져오기
   2. 상태 → "in_progress"
   3. DDL 필요시 → API Gateway에 요청
   4. AI Orchestra 실행
   5. 테스트 실행
   6. 성공 → 배포, 상태 → "completed"
      실패 → 롤백, 상태 → "failed", 재시도 큐에 추가 (최대 2회)
   7. 다음 요청 처리
   ```

4. **동시 처리**
   - 프로젝트별로 1개씩만 동시 처리
   - 전체 최대 동시 처리: 3개 (리소스 제한)

5. **모니터링**
   - 현재 처리 중인 요청
   - 대기 중인 요청 수
   - 예상 완료 시간

## 타입 정의

```typescript
interface QueueConfig {
  // 운영 시간
  operatingHours: {
    start: string;  // "22:00"
    end: string;    // "06:00"
    timezone: string; // "Asia/Seoul"
  };
  
  // 동시 처리
  concurrency: {
    maxTotal: number;      // 전체 최대 (기본: 3)
    maxPerProject: number; // 프로젝트당 최대 (기본: 1)
  };
  
  // 재시도
  retry: {
    maxAttempts: number;  // 최대 재시도 (기본: 2)
    backoffMinutes: number; // 재시도 간격 (기본: 30)
  };
  
  // 타임아웃
  timeout: {
    analysis: number;    // 분석 타임아웃 (분)
    implementation: number; // 구현 타임아웃 (분)
    testing: number;     // 테스트 타임아웃 (분)
  };
}

interface QueueItem {
  requestId: string;
  projectId: string;
  priority: number;  // 1 (urgent) ~ 3 (low)
  attempts: number;
  scheduledFor: Date;
  createdAt: Date;
}

interface ProcessingStatus {
  requestId: string;
  projectId: string;
  stage: 'analyzing' | 'ddl_requesting' | 'implementing' | 'testing' | 'deploying';
  progress: number;  // 0-100
  startedAt: Date;
  estimatedCompletion: Date;
  logs: string[];
}
```

## 프로젝트 구조

```
jabis-night-builder/
├── src/
│   ├── index.ts
│   ├── queue/
│   │   ├── RequestQueue.ts       # 우선순위 큐
│   │   ├── QueueScheduler.ts     # 야간 스케줄러
│   │   └── QueueProcessor.ts     # 큐 처리기
│   ├── processor/
│   │   ├── RequestProcessor.ts   # 요청 처리 메인
│   │   ├── DDLHandler.ts         # DDL 요청 처리
│   │   ├── ImplementationRunner.ts # AI Orchestra 실행
│   │   └── DeploymentHandler.ts  # 배포 처리
│   ├── scheduler/
│   │   ├── TimeChecker.ts        # 운영 시간 체크
│   │   └── CronManager.ts        # 크론 관리
│   ├── monitoring/
│   │   ├── StatusTracker.ts
│   │   └── MetricsCollector.ts
│   └── types.ts
├── prisma/
│   └── schema.prisma
└── package.json
```

node-cron으로 스케줄링하고,
Bull 큐 라이브러리로 안정적인 큐 관리해줘.
```

---

### 프롬프트 2-2: API Gateway DDL 연동

```
Night Builder에서 API Gateway로 DDL 요청을 보내는 모듈을 만들어줘.

## 기능 요구사항

1. **DDL 요청 생성**
   - AI 분석 결과의 DDL 변경사항을 API Gateway 형식으로 변환
   - 요청 ID와 연결하여 추적 가능하게

2. **요청 상태 관리**
   - pending: 요청 생성됨
   - approved: 승인됨 (수동 또는 자동)
   - executed: 실행 완료
   - failed: 실행 실패

3. **자동 승인 규칙**
   - ADD_COLUMN (nullable): 자동 승인
   - CREATE_TABLE: 자동 승인
   - ALTER_COLUMN, DROP: 수동 승인 필요

4. **실행 대기**
   - DDL이 실행될 때까지 코드 구현 대기
   - 타임아웃 설정 (기본: 30분)
   - 타임아웃 시 요청 실패 처리

## API Gateway 연동

```typescript
// API Gateway DDL 요청 형식
interface DDLRequest {
  id: string;
  requestId: string;       // Night Builder 요청 ID
  projectId: string;
  
  changes: DDLChange[];
  
  autoApprove: boolean;
  requester: string;
  reason: string;
  
  status: 'pending' | 'approved' | 'executing' | 'executed' | 'failed';
  
  createdAt: string;
  executedAt?: string;
}

interface DDLChange {
  type: 'CREATE_TABLE' | 'ALTER_TABLE' | 'ADD_COLUMN' | 'ALTER_COLUMN' | 'DROP_COLUMN' | 'ADD_INDEX' | 'DROP_TABLE';
  database: string;
  table: string;
  description: string;
  sql: string;
  rollbackSql: string;  // 롤백 SQL
  riskLevel: 'low' | 'medium' | 'high';
}

// DDL Handler
class DDLHandler {
  private apiGatewayUrl: string;
  
  async requestDDL(request: FeatureRequest): Promise<DDLRequest> {
    const ddlRequest: DDLRequest = {
      id: generateId(),
      requestId: request.id,
      projectId: request.projectId,
      changes: request.analysis.ddlChanges.map(change => ({
        ...change,
        rollbackSql: this.generateRollbackSQL(change),
        riskLevel: this.assessRisk(change),
      })),
      autoApprove: this.canAutoApprove(request.analysis.ddlChanges),
      requester: 'night-builder',
      reason: `Feature Request: ${request.title}`,
      status: 'pending',
      createdAt: new Date().toISOString(),
    };
    
    // API Gateway로 전송
    const response = await fetch(`${this.apiGatewayUrl}/api/ddl/requests`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(ddlRequest),
    });
    
    return response.json();
  }
  
  async waitForExecution(ddlRequestId: string, timeoutMs: number): Promise<boolean> {
    // 폴링으로 상태 확인
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeoutMs) {
      const status = await this.checkStatus(ddlRequestId);
      
      if (status === 'executed') return true;
      if (status === 'failed') throw new Error('DDL execution failed');
      
      await sleep(10000); // 10초 간격 체크
    }
    
    throw new Error('DDL execution timeout');
  }
  
  private canAutoApprove(changes: DDLChange[]): boolean {
    return changes.every(c => 
      c.type === 'CREATE_TABLE' ||
      (c.type === 'ADD_COLUMN' && c.sql.includes('NULL')) ||
      c.type === 'ADD_INDEX'
    );
  }
}
```

API Gateway에서 DDL 상태가 변경되면 
WebSocket 또는 Webhook으로 알림받을 수 있게 해줘.
```

---

## Part 3: AI Orchestra 연동

### 프롬프트 3-1: Implementation Runner

```
Night Builder에서 AI Orchestra를 호출하여 실제 구현을 수행하는 
ImplementationRunner를 만들어줘.

## 기능 요구사항

1. **구현 전 준비**
   - Git 저장소 최신화 (pull)
   - feature/{request-id} 브랜치 생성
   - 의존성 설치 (npm install)

2. **AI Orchestra 실행**
   - 복잡도에 따른 모드 선택
     - simple: 단순 수정
     - medium: 중간 규모 기능
     - full: 복잡한 기능 (Planner 포함)
   - 요청 분석 결과를 컨텍스트로 전달
   - 실시간 진행 상황 업데이트

3. **구현 프롬프트 생성**

```typescript
const buildImplementationPrompt = (request: FeatureRequest): string => `
# 기능 구현 요청

## 요청 정보
- 제목: ${request.title}
- 유형: ${request.type}
- 우선순위: ${request.priority}

## 요청 상세
${request.description}

## AI 분석 결과
${JSON.stringify(request.analysis, null, 2)}

## 프로젝트 컨텍스트
- 기술 스택: React 18, Node.js 20, TypeScript 5, PostgreSQL 15
- 스타일: Tailwind CSS, shadcn/ui
- API: REST, /api/** 경로

## 지시사항

1. 분석 결과의 requiredChanges를 순서대로 구현하세요
2. affectedFiles에 명시된 파일들을 수정하세요
3. 기존 코드 스타일과 패턴을 따르세요
4. TypeScript 타입을 정확히 정의하세요
5. 에러 핸들링을 포함하세요
6. 필요한 경우 테스트 코드도 작성하세요

## DDL 정보 (이미 적용됨)
${request.analysis.ddlChanges?.map(d => d.sql).join('\n') || '없음'}

## 주의사항
${request.analysis.risks?.join('\n') || '없음'}
`;
```

4. **진행 상황 추적**
   - AI Orchestra 출력 파싱
   - 단계별 진행률 계산
   - DB 및 WebSocket으로 상태 업데이트

5. **결과 수집**
   - 변경된 파일 목록
   - 커밋 메시지 생성
   - 구현 요약

## 구현

```typescript
// src/processor/ImplementationRunner.ts

import { spawn } from 'child_process';
import { FeatureRequest } from '../types';

interface ImplementationResult {
  success: boolean;
  changedFiles: string[];
  commitHash?: string;
  summary: string;
  logs: string[];
  duration: number;
}

export class ImplementationRunner {
  constructor(
    private orchestraPath: string,
    private workspacePath: string,
    private config: RunnerConfig
  ) {}

  async run(request: FeatureRequest): Promise<ImplementationResult> {
    const projectPath = path.join(this.workspacePath, request.projectId);
    const startTime = Date.now();
    const logs: string[] = [];
    
    try {
      // 1. Git 준비
      await this.updateStatus(request.id, 'implementing', 10, 'Git 저장소 준비 중...');
      await this.prepareGit(projectPath, request);
      
      // 2. 의존성 설치 (새 패키지가 있는 경우)
      if (request.analysis.externalDependencies?.length) {
        await this.updateStatus(request.id, 'implementing', 20, '의존성 설치 중...');
        await this.installDependencies(projectPath, request.analysis.externalDependencies);
      }
      
      // 3. AI Orchestra 실행
      await this.updateStatus(request.id, 'implementing', 30, 'AI 구현 시작...');
      const prompt = this.buildImplementationPrompt(request);
      const mode = this.selectMode(request.analysis.estimatedComplexity);
      
      const orchestraResult = await this.runOrchestra(projectPath, prompt, mode, (progress, message) => {
        this.updateStatus(request.id, 'implementing', 30 + (progress * 0.5), message);
        logs.push(message);
      });
      
      // 4. 결과 확인
      const changedFiles = await this.getChangedFiles(projectPath);
      
      // 5. 커밋
      await this.updateStatus(request.id, 'implementing', 90, '변경사항 커밋 중...');
      const commitHash = await this.commitChanges(projectPath, request);
      
      return {
        success: true,
        changedFiles,
        commitHash,
        summary: orchestraResult.summary,
        logs,
        duration: Date.now() - startTime,
      };
      
    } catch (error) {
      logs.push(`Error: ${error.message}`);
      
      // 롤백
      await this.rollback(projectPath, request);
      
      return {
        success: false,
        changedFiles: [],
        summary: `구현 실패: ${error.message}`,
        logs,
        duration: Date.now() - startTime,
      };
    }
  }

  private selectMode(complexity: string): 'simple' | 'medium' | 'full' {
    switch (complexity) {
      case 'simple': return 'simple';
      case 'medium': return 'medium';
      case 'complex': return 'full';
      default: return 'medium';
    }
  }

  private async runOrchestra(
    projectPath: string,
    prompt: string,
    mode: string,
    onProgress: (progress: number, message: string) => void
  ): Promise<{ summary: string }> {
    return new Promise((resolve, reject) => {
      const process = spawn(this.orchestraPath, [
        'run', prompt,
        '--path', projectPath,
        '--mode', mode,
        '--auto-fix',
        '--no-dashboard',
      ]);
      
      let output = '';
      
      process.stdout.on('data', (data) => {
        const text = data.toString();
        output += text;
        
        // 진행 상황 파싱
        const progress = this.parseProgress(text);
        if (progress) {
          onProgress(progress.percent, progress.message);
        }
      });
      
      process.on('close', (code) => {
        if (code === 0) {
          resolve({ summary: this.parseSummary(output) });
        } else {
          reject(new Error(`Orchestra failed with code ${code}`));
        }
      });
    });
  }
}
```

테스트 실행과 배포까지 이어지도록 
전체 파이프라인을 완성해줘.
```

---

### 프롬프트 3-2: 테스트 및 배포 핸들러

```
구현 완료 후 테스트 실행과 자동 배포를 처리하는 모듈을 만들어줘.

## 기능 요구사항

1. **테스트 실행**
   - npm test
   - npm run lint
   - npm run type-check
   - E2E 테스트 (설정된 경우)

2. **테스트 결과 판단**
   - 모든 테스트 통과 → 배포 진행
   - 테스트 실패 → AI에게 수정 요청 (1회 재시도)
   - 재시도도 실패 → 요청 실패 처리

3. **배포**
   - Git push (feature 브랜치)
   - PR 생성 또는 main에 직접 머지 (설정에 따라)
   - ArgoCD Sync 트리거 (선택)

4. **롤백 지원**
   - 배포 후 헬스체크 실패 시 자동 롤백
   - 이전 커밋으로 되돌리기

## 구현

```typescript
// src/processor/TestRunner.ts

interface TestResult {
  passed: boolean;
  results: {
    unit: { passed: boolean; output: string };
    lint: { passed: boolean; output: string };
    typeCheck: { passed: boolean; output: string };
    e2e?: { passed: boolean; output: string };
  };
  summary: string;
}

export class TestRunner {
  async runTests(projectPath: string, config: TestConfig): Promise<TestResult> {
    const results: TestResult['results'] = {
      unit: { passed: false, output: '' },
      lint: { passed: false, output: '' },
      typeCheck: { passed: false, output: '' },
    };
    
    // Unit tests
    const unitResult = await this.exec('npm', ['test', '--', '--passWithNoTests'], projectPath);
    results.unit = { passed: unitResult.code === 0, output: unitResult.output };
    
    // Lint
    const lintResult = await this.exec('npm', ['run', 'lint'], projectPath);
    results.lint = { passed: lintResult.code === 0, output: lintResult.output };
    
    // Type check
    const typeResult = await this.exec('npx', ['tsc', '--noEmit'], projectPath);
    results.typeCheck = { passed: typeResult.code === 0, output: typeResult.output };
    
    // E2E (optional)
    if (config.runE2E) {
      const e2eResult = await this.exec('npm', ['run', 'test:e2e'], projectPath);
      results.e2e = { passed: e2eResult.code === 0, output: e2eResult.output };
    }
    
    const passed = results.unit.passed && 
                   results.lint.passed && 
                   results.typeCheck.passed &&
                   (!results.e2e || results.e2e.passed);
    
    return {
      passed,
      results,
      summary: this.generateSummary(results),
    };
  }
}

// src/processor/DeploymentHandler.ts

interface DeploymentResult {
  success: boolean;
  prUrl?: string;
  deployedCommit?: string;
  deployedAt?: string;
  healthCheckPassed?: boolean;
}

export class DeploymentHandler {
  constructor(
    private bitbucketApi: BitbucketAPI,
    private argocdApi: ArgoCDAPI,
    private config: DeploymentConfig
  ) {}

  async deploy(
    projectPath: string,
    request: FeatureRequest,
    commitHash: string
  ): Promise<DeploymentResult> {
    
    // 1. Push feature branch
    await this.gitPush(projectPath, `feature/${request.id}`);
    
    if (this.config.requirePR) {
      // 2a. PR 생성
      const pr = await this.bitbucketApi.createPR({
        title: `[Auto] ${request.title}`,
        description: this.generatePRDescription(request),
        sourceBranch: `feature/${request.id}`,
        targetBranch: request.analysis.targetBranch || 'main',
        reviewers: this.config.defaultReviewers,
      });
      
      return {
        success: true,
        prUrl: pr.url,
        deployedCommit: commitHash,
      };
      
    } else {
      // 2b. 직접 머지 & 배포
      await this.gitMergeToMain(projectPath, `feature/${request.id}`);
      await this.gitPush(projectPath, 'main');
      
      // ArgoCD Sync
      if (this.config.autoSync) {
        await this.argocdApi.sync(request.projectId);
        
        // Health check
        const healthy = await this.waitForHealthy(request.projectId, 300000); // 5분
        
        if (!healthy) {
          // 롤백
          await this.argocdApi.rollback(request.projectId);
          return {
            success: false,
            healthCheckPassed: false,
          };
        }
      }
      
      return {
        success: true,
        deployedCommit: commitHash,
        deployedAt: new Date().toISOString(),
        healthCheckPassed: true,
      };
    }
  }

  private generatePRDescription(request: FeatureRequest): string {
    return `
## 🤖 AI 자동 구현

### 요청 정보
- **요청 ID**: ${request.id}
- **요청자**: ${request.requesterName}
- **유형**: ${request.type}

### 요청 내용
${request.description}

### 변경 사항
${request.result?.changedFiles?.map(f => `- \`${f}\``).join('\n')}

### AI 분석
- 복잡도: ${request.analysis.estimatedComplexity}
- 예상 시간: ${request.analysis.estimatedTime}

---
*이 PR은 JABIS Night Builder에 의해 자동 생성되었습니다.*
    `;
  }
}
```

테스트 실패 시 AI에게 수정 요청하는 로직도 포함해줘.
```

---

## Part 4: 관리자 대시보드

### 프롬프트 4-1: Night Builder 모니터링 대시보드

```
React + TypeScript로 Night Builder 관리자 대시보드를 만들어줘.

## 기능 요구사항

1. **실시간 현황**
   - 현재 처리 중인 요청
   - 대기 큐 상태
   - Night Builder 운영 상태 (활성/대기/오류)

2. **요청 관리**
   - 전체 요청 목록 (필터: 상태, 프로젝트, 날짜)
   - 요청 상세 보기
   - 수동 재시도
   - 요청 취소
   - 우선순위 변경

3. **스케줄 설정**
   - 운영 시간 설정
   - 동시 처리 수 설정
   - 프로젝트별 설정

4. **통계**
   - 일별/주별/월별 처리 현황
   - 성공률
   - 평균 처리 시간
   - 프로젝트별 통계

5. **DDL 승인 관리**
   - 승인 대기 DDL 목록
   - 승인/거부 처리
   - DDL 실행 히스토리

6. **로그 뷰어**
   - 실시간 로그 스트림
   - 요청별 로그 조회

## 페이지 구조

```
/dashboard
├── /                     # 메인 대시보드 (현황)
├── /requests             # 요청 목록
├── /requests/:id         # 요청 상세
├── /ddl                  # DDL 승인 관리
├── /settings             # 설정
├── /logs                 # 로그 뷰어
└── /stats                # 통계
```

## UI 디자인 (메인 대시보드)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  🌙 JABIS Night Builder                                    [설정] [로그아웃] │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  🔄 처리 중      │  │  ⏳ 대기 중      │  │  ✅ 오늘 완료    │             │
│  │       2        │  │       12       │  │       8        │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                             │
│  운영 상태: 🟢 야간 모드 활성 (22:00 ~ 06:00)                                │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  현재 처리 중                                                          │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │ #REQ-001 │ 엑셀 다운로드 기능 │ project-a │ ████████░░ 80% │ 23분 │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │ #REQ-002 │ 통계 차트 추가    │ project-b │ ███░░░░░░░ 30% │ 5분  │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  대기 큐 (다음 5개)                                           [전체보기] │ │
│  │                                                                       │ │
│  │  1. #REQ-003 │ 🔴 긴급 │ 회원가입 필드 추가 │ project-a │ DDL 필요   │ │
│  │  2. #REQ-004 │ 🟡 보통 │ 검색 필터 개선     │ project-c │           │ │
│  │  3. #REQ-005 │ 🟡 보통 │ 페이지네이션 추가  │ project-a │           │ │
│  │  ...                                                                  │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 프로젝트 구조

```
jabis-night-builder-dashboard/
├── src/
│   ├── App.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Requests/
│   │   │   ├── RequestList.tsx
│   │   │   └── RequestDetail.tsx
│   │   ├── DDLApproval.tsx
│   │   ├── Settings.tsx
│   │   ├── Logs.tsx
│   │   └── Stats.tsx
│   ├── components/
│   │   ├── StatusCard.tsx
│   │   ├── ProcessingItem.tsx
│   │   ├── QueueList.tsx
│   │   ├── ProgressBar.tsx
│   │   ├── LogViewer.tsx
│   │   └── StatsChart.tsx
│   ├── hooks/
│   │   ├── useNightBuilder.ts
│   │   ├── useRequests.ts
│   │   └── useSocket.ts
│   └── api/
│       └── nightBuilderApi.ts
└── package.json
```

Vite + React Query + Socket.IO로 만들고,
Recharts로 통계 차트 구현해줘.
```

---

## Part 5: 통합 및 설정

### 프롬프트 5-1: 프로젝트 설정 관리

```
Night Builder에서 프로젝트별 설정을 관리하는 시스템을 만들어줘.

## 기능 요구사항

1. **프로젝트 설정**
   ```typescript
   interface ProjectConfig {
     id: string;
     name: string;
     repoUrl: string;
     defaultBranch: string;
     
     // Night Builder 설정
     nightBuilder: {
       enabled: boolean;
       
       // 배포 설정
       deployment: {
         autoDeploy: boolean;     // 자동 배포 여부
         requirePR: boolean;      // PR 필수 여부
         defaultReviewers: string[];
         autoMergeOnPass: boolean;
       };
       
       // 테스트 설정
       testing: {
         runUnit: boolean;
         runLint: boolean;
         runTypeCheck: boolean;
         runE2E: boolean;
         e2eCommand?: string;
       };
       
       // DDL 설정
       ddl: {
         autoApproveAddColumn: boolean;
         autoApproveCreateTable: boolean;
         requireManualApproval: string[];  // 항상 수동 승인 필요한 테이블
       };
       
       // 제한 설정
       limits: {
         maxRequestsPerDay: number;
         maxConcurrent: number;
         timeoutMinutes: number;
       };
       
       // 알림 설정
       notifications: {
         slack?: string;
         email?: string[];
         onSuccess: boolean;
         onFailure: boolean;
       };
     };
     
     // AI 설정
     ai: {
       model: 'haiku' | 'sonnet' | 'opus';
       maxTokens: number;
       temperature: number;
     };
   }
   ```

2. **글로벌 설정**
   ```typescript
   interface GlobalConfig {
     // 운영 시간
     operatingHours: {
       enabled: boolean;
       start: string;  // "22:00"
       end: string;    // "06:00"
       timezone: string;
       excludeDates: string[];  // 제외 날짜 (휴일 등)
     };
     
     // 리소스 제한
     resources: {
       maxTotalConcurrent: number;
       cpuLimit: string;
       memoryLimit: string;
     };
     
     // 기본값
     defaults: {
       model: string;
       timeout: number;
       maxRetries: number;
     };
   }
   ```

3. **설정 UI**
   - 프로젝트 목록 및 설정
   - 글로벌 설정
   - 설정 가져오기/내보내기

프로젝트 설정 CRUD API와 설정 UI를 만들어줘.
```

---

### 프롬프트 5-2: Helm Chart 및 배포

```
JABIS Night Builder 전체 시스템을 K3S에 배포하는 Helm Chart를 만들어줘.

## 구성 요소

1. **jabis-night-builder** (메인 서비스)
   - Queue Processor
   - Scheduler
   - API Server

2. **jabis-request-api** (요청 API)
   - 챗봇에서 호출하는 API

3. **jabis-night-builder-dashboard** (관리자 대시보드)
   - 모니터링 UI

4. **의존성**
   - Redis (Bull Queue용)
   - PostgreSQL (기존 jabis-db 사용)

## values.yaml

```yaml
global:
  imageRegistry: harbor.jinhak.com/jabis

nightBuilder:
  replicaCount: 1  # 스케줄러는 1개만
  
  config:
    operatingHours:
      enabled: true
      start: "22:00"
      end: "06:00"
      timezone: "Asia/Seoul"
    
    concurrency:
      maxTotal: 3
      maxPerProject: 1
    
    orchestraPath: "/usr/local/bin/ai-orchestra"
  
  workspace:
    storageClass: "local-path"
    size: "50Gi"
  
  resources:
    limits:
      cpu: "2"
      memory: "4Gi"

requestApi:
  replicaCount: 2
  ingress:
    enabled: true
    host: jabis-request.jinhak.com

dashboard:
  replicaCount: 1
  ingress:
    enabled: true
    host: jabis-night.jinhak.com

redis:
  enabled: true
  # 또는 external:
  # external:
  #   host: redis.jabis.svc.cluster.local
  #   port: 6379

database:
  external:
    host: jabis-db.jabis.svc.cluster.local
    port: 5432
    database: jabis_night_builder

apiGateway:
  url: http://jabis-gateway.jabis.svc.cluster.local
```

Helm Chart 전체 구조와 설치 가이드를 만들어줘.
```

---

## 전체 시스템 연동 흐름

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                전체 시스템 흐름                                   │
│                                                                                 │
│  [09:00] 비개발자 A: "검색 결과에 정렬 기능 추가해주세요"                          │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   AI 요청 챗봇      │  → POST /api/requests                                  │
│  │   (JABIS 사이트)    │  → 요청 저장 (status: submitted)                        │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   Planner AI 분석   │  → 복잡도: medium                                      │
│  │                     │  → DDL: 불필요                                         │
│  │                     │  → 예상 시간: 2시간                                    │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   Request Queue     │  → status: queued                                     │
│  │   (Bull + Redis)    │  → queue position: 5                                  │
│  └─────────────────────┘                                                        │
│                                                                                 │
│  ════════════════════════════════════════════════════════════════════════════  │
│                                                                                 │
│  [22:00] Night Builder 활성화                                                   │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   Queue Processor   │  → 큐에서 요청 가져오기                                 │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐     ┌─────────────────────┐                           │
│  │  DDL 필요 여부 확인  │────▶│  API Gateway 요청   │ (DDL 필요시)               │
│  └─────────────────────┘     │  DDL 실행 대기      │                           │
│                │             └─────────────────────┘                           │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   Git 준비          │  → git pull                                           │
│  │                     │  → git checkout -b feature/REQ-001                    │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   AI Orchestra      │  → mode: medium                                       │
│  │   실행              │  → Coder + Debugger                                   │
│  │                     │  → 코드 구현                                          │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   테스트 실행       │  → npm test ✓                                         │
│  │                     │  → npm run lint ✓                                     │
│  │                     │  → tsc --noEmit ✓                                     │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   배포              │  → git push                                           │
│  │                     │  → PR 생성 또는 자동 머지                              │
│  │                     │  → ArgoCD Sync                                        │
│  └─────────────────────┘                                                        │
│                │                                                                │
│                ▼                                                                │
│  ┌─────────────────────┐                                                        │
│  │   완료 처리         │  → status: completed                                  │
│  │                     │  → 알림 전송                                          │
│  └─────────────────────┘                                                        │
│                                                                                 │
│  ════════════════════════════════════════════════════════════════════════════  │
│                                                                                 │
│  [09:00 다음날] 비개발자 A 출근                                                  │
│                                                                                 │
│  📱 알림: "요청하신 '검색 결과 정렬 기능'이 구현되었습니다!"                       │
│  🔗 링크: 배포된 페이지에서 확인                                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 구현 로드맵

```
Week 1                    Week 2                    Week 3
├────────────────────────┼────────────────────────┼────────────────────────┤
│                        │                        │                        │
│  AI 요청 챗봇 UI       │  Request Queue         │  Implementation        │
│  (프롬프트 1-1)        │  (프롬프트 2-1)        │  Runner                │
│                        │                        │  (프롬프트 3-1)        │
│  요청 분석 API         │  DDL Handler           │                        │
│  (프롬프트 1-2)        │  (프롬프트 2-2)        │  Test & Deploy         │
│                        │                        │  (프롬프트 3-2)        │
│                        │                        │                        │
├────────────────────────┴────────────────────────┴────────────────────────┤
                                    │
Week 4                    Week 5                    Week 6
├────────────────────────┼────────────────────────┼────────────────────────┤
│                        │                        │                        │
│  관리자 대시보드       │  프로젝트 설정         │  통합 테스트           │
│  (프롬프트 4-1)        │  (프롬프트 5-1)        │                        │
│                        │                        │  Helm Chart            │
│                        │  Helm Chart            │  배포                  │
│                        │  (프롬프트 5-2)        │                        │
│                        │                        │  문서화                │
└────────────────────────┴────────────────────────┴────────────────────────┘
```

---

## 보안 고려사항

1. **요청 검증**
   - 악의적 코드 실행 방지
   - 허용된 프로젝트만 처리
   - 요청 내용 필터링

2. **Git 권한**
   - 전용 서비스 계정 사용
   - 최소 권한 원칙 (특정 저장소만)
   - 감사 로그 기록

3. **DDL 제한**
   - DROP TABLE 등 위험 명령 제한
   - 프로덕션 DB 직접 변경 불가
   - 마이그레이션 통한 변경만

4. **리소스 제한**
   - 일일 최대 요청 수
   - 실행 시간 제한
   - 메모리/CPU 제한

---

## 장애 대응

| 상황 | 대응 |
|------|------|
| AI Orchestra 실행 실패 | 롤백 후 재시도 (최대 2회) |
| 테스트 실패 | AI에게 수정 요청 후 재시도 |
| DDL 실행 실패 | 요청 실패 처리, 수동 개입 필요 |
| 배포 후 헬스체크 실패 | ArgoCD 자동 롤백 |
| Night Builder 크래시 | 마지막 처리 요청부터 재시작 |
