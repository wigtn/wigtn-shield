# WIGTN Shield — 5인 바이브코딩 협업 가이드

> **핵심 원칙**: 충돌 최소화 = "각자 완전히 분리된 디렉토리를 소유하고,
> 공유 경계(types, API contract)는 한 명만 수정한다"

---

## 역할 배정표

| 역할 | 코드명 | 담당 영역 | 소유 디렉토리 |
|------|--------|---------|-------------|
| **인프라 리드** | `@infra` | CLI, 스트림, DB, 공유 타입 | `packages/cli/` `packages/stream/` `packages/shared/` |
| **AI Agent 개발자** | `@agents` | Agent 구현, Thought Signature | `packages/agents/` |
| **백엔드 API 개발자** | `@api` | REST API, 인증, 알림 | `packages/api/` |
| **프론트엔드 개발자** | `@frontend` | Canvas UI, WebSocket 클라이언트 | `packages/web/` |
| **DevOps / 통합** | `@devops` | CI/CD, 테스트, 문서화 | `scripts/` `docs/` `.github/` |

---

## 역할별 작업 범위

### @infra — 인프라 리드

**담당 기능 (PRD 참조)**:
- FR-001 로그 중앙 수집 (Loki HTTP API)
- FR-002 메트릭 수집 (Prometheus HTTP API)
- FR-003 실시간 스트림 처리 (Stream Processor)
- FR-010 Agent 헬스체크 구조 (Heartbeat 인프라)
- FR-022 Thought Signature 암호화 기반 (HMAC, keytar)
- `wigtn-shield init/start/stop/status/verify/rotate-key/migrate-config` CLI 명령어
- DB 스키마 전체 관리 (migration 파일)
- 공유 타입 정의 (`types.ts`, `events.ts`, `config-schema.ts`)

**소유 파일**:
```
packages/
  shared/          ← 전체 소유 (타입/이벤트/스키마)
  cli/             ← 전체 소유
  stream/          ← 전체 소유
```

**절대 건드리지 않는 영역**: `packages/agents/`, `packages/api/`, `packages/web/`

---

### @agents — AI Agent 개발자

**담당 기능 (PRD 참조)**:
- FR-006 IDS Agent
- FR-007 DDoS Agent
- FR-008 IAM Agent
- FR-009 DLP Agent
- FR-024 Injection Agent
- FR-025 Supply Chain Agent
- FR-026 Lateral Movement Agent
- FR-027 Secret Leakage Agent
- FR-028 Misconfiguration Agent
- FR-029 Ransomware Agent
- FR-023 Live Interception Agent
- LangGraph 오케스트레이터 구현
- Thought Signature 실제 서명 로직 (`@infra`가 만든 HMAC 기반 활용)

**소유 파일**:
```
packages/
  agents/          ← 전체 소유
    orchestrator/
    ids/
    ddos/
    iam/
    dlp/
    injection/
    supply-chain/
    lateral-movement/
    secret-leakage/
    misconfiguration/
    ransomware/
    live-interception/
```

**절대 건드리지 않는 영역**: `packages/cli/`, `packages/api/`, `packages/web/`
**읽기 전용**: `packages/shared/types.ts`, `packages/shared/events.ts`

---

### @api — 백엔드 API 개발자

**담당 기능 (PRD 참조)**:
- FR-005 Web UI 인증 (JWT, httpOnly 쿠키)
- FR-011 Slack 알림
- FR-012 알림 중복 억제 (30분 억제 + 리마인더)
- FR-013 서비스 등록/관리 CRUD
- FR-014 이메일 알림
- FR-015 보안 리포트
- FR-016 알림 히스토리
- FR-017 이벤트 통계 API
- FR-019 자동 차단 (P2)
- REST API 전체 (9장 API 명세 구현)

**소유 파일**:
```
packages/
  api/             ← 전체 소유
    routes/
    middleware/
    services/
      auth/
      alerts/
      notifications/
      reports/
    openapi.yaml   ← @frontend가 참조하는 계약 파일
```

**절대 건드리지 않는 영역**: `packages/cli/`, `packages/agents/`, `packages/web/`
**읽기 전용**: `packages/shared/types.ts`, `packages/shared/db/`
**@agents와 인터페이스**: `packages/shared/agent-interface.ts`를 통해 Agent 호출 (직접 import 금지)

---

### @frontend — 프론트엔드 개발자

**담당 기능 (PRD 참조)**:
- FR-004 관리자 Web UI 전체
- FR-030 Canvas 대시보드 (12장 전체 — 캐릭터, 애니메이션, 실시간)
- WebSocket 클라이언트 (`/ws/agents`)
- 이벤트 목록, 서비스 상태, 통계 차트

**소유 파일**:
```
packages/
  web/             ← 전체 소유
    src/
      canvas/      ← 캐릭터 + 애니메이션
      pages/
      components/
      hooks/
      api-client/  ← openapi.yaml 기반 자동 생성 클라이언트
```

**절대 건드리지 않는 영역**: `packages/cli/`, `packages/agents/`, `packages/api/`
**참조 방식**: `packages/api/openapi.yaml`만 참조 (API 클라이언트 자동 생성)

---

### @devops — DevOps / 통합

**담당 기능**:
- GitHub Actions CI/CD 파이프라인
- npm publish --provenance (Sigstore 서명)
- 통합 테스트 (E2E)
- README.md, CONTRIBUTING.md, LICENSE
- docs/ 문서 관리
- 각 Phase 완료 시 통합 검증

**소유 파일**:
```
scripts/           ← 전체 소유
docs/              ← 전체 소유 (PRD 포함)
.github/           ← 전체 소유
package.json       ← 루트만 소유 (workspace 설정)
.gitignore         ← 전체 소유
```

**절대 건드리지 않는 영역**: `packages/` 내부 소스 코드
**예외**: 통합 이슈 발견 시 해당 담당자에게 PR 요청으로만 해결

---

## 충돌 최소화 핵심 규칙

### 규칙 1: 공유 영역 변경 프로토콜

```
공유 파일 (packages/shared/*) 변경 시:
  1. @infra가 PR 생성
  2. 영향받는 역할(@agents, @api, @frontend)에 메시지 알림
  3. 24시간 내 리뷰 없으면 self-merge 가능
  4. 변경 후 반드시 CHANGELOG 업데이트
```

### 규칙 2: API Contract 불변 원칙

```
packages/api/openapi.yaml 변경 시:
  - @api만 수정 가능
  - 기존 엔드포인트 BREAKING CHANGE 금지 (버전 업 또는 deprecated 필드 유지)
  - @frontend는 항상 이 파일에서 타입/클라이언트 자동 생성
  - 새 엔드포인트 추가는 자유, 기존 삭제/변경은 팀 합의 필수
```

### 규칙 3: DB Schema 단일 소유

```
DB 스키마 파일 (packages/shared/db/migrations/*):
  - @infra만 수정 가능
  - 다른 역할이 DB 변경 필요 시 → @infra에 Issue 등록
  - Migration 파일은 절대 직접 수정 금지 (새 migration 파일만 추가)
```

### 규칙 4: 브랜치 전략

```
main                    ← 안정 버전 (merge는 @devops만)
  └── feat/infra        ← @infra 작업 브랜치
  └── feat/agents       ← @agents 작업 브랜치
  └── feat/api          ← @api 작업 브랜치
  └── feat/frontend     ← @frontend 작업 브랜치
  └── feat/devops       ← @devops 작업 브랜치

PR 규칙:
  - 자신의 소유 영역만 수정한 PR: self-merge 가능
  - 공유 영역(shared/) 포함 PR: 최소 1명 리뷰 필수
  - 다른 역할 영역 포함 PR: 해당 역할 담당자 리뷰 필수
```

### 규칙 5: Mock-first 개발

```
@agents, @api, @frontend는 다른 팀 코드가 없어도 진행 가능:

@agents 개발 시:
  → packages/shared/agent-interface.ts의 IAgentInput/IAgentOutput으로만 개발
  → Vertex AI mock 사용 가능 (MOCK_AI=true 환경변수)

@api 개발 시:
  → Agent 결과를 IAgentOutput mock으로 대체
  → DB는 @infra가 정의한 스키마 기반 SQLite in-memory 사용

@frontend 개발 시:
  → openapi.yaml 기반 MSW(Mock Service Worker) 사용
  → WebSocket도 MSW로 mock
```

---

## 역할 선언 방식 (바이브코딩 프롬프트)

Claude Code에서 작업 시 첫 메시지에 역할을 선언하면 해당 영역만 작업합니다:

```
나는 @agents 역할입니다.
IDS Agent의 Brute Force 탐지 로직을 구현해줘.
```

```
나는 @frontend 역할입니다.
Canvas에서 IDS 경비원 캐릭터가 alert 상태일 때
빠르게 달리는 애니메이션을 만들어줘.
```

```
나는 @api 역할입니다.
POST /api/v1/events/{eventId}/acknowledge API를 구현해줘.
```

**역할 선언 없이 요청 시**: "어떤 역할로 작업하시나요?" 확인 후 진행

---

## 개발 시작 체크리스트

### Week 0 (Phase 시작 전): @infra + @devops 선행 작업

```
@infra (먼저 완료해야 함):
  □ monorepo 구조 초기화 (pnpm workspace)
  □ packages/shared/types.ts — 공유 타입 전체 정의
  □ packages/shared/events.ts — 이벤트 타입 정의
  □ packages/shared/agent-interface.ts — IAgentInput/Output 인터페이스
  □ packages/shared/db/ — DB 스키마 + migration 초기화
  □ packages/shared/config-schema.ts — config.yaml 스키마

@devops (동시에 진행):
  □ GitHub Actions 기본 파이프라인 설정
  □ PR 템플릿 생성 (역할 체크박스 포함)
  □ .gitignore, LICENSE, README 뼈대
```

### @infra 완료 후: 다른 역할들 병렬 시작 가능

```
@agents: packages/agents/ 디렉토리 생성 + IDS Agent 시작
@api: packages/api/ 디렉토리 생성 + 인증 API 시작
@frontend: packages/web/ 디렉토리 생성 + Canvas 뼈대 시작
```

---

## 공유 타입 계약 (packages/shared/types.ts 예시)

```typescript
// @infra가 정의하고 관리 — 모든 역할이 읽기 전용으로 사용

export type Severity = 'low' | 'medium' | 'high' | 'critical';
export type AgentStatus = 'idle' | 'working' | 'alert' | 'critical' | 'offline';
export type EventStatus = 'open' | 'investigating' | 'resolved' | 'false_positive';
export type AgentTier = 'tier1' | 'tier2' | 'tier3' | 'optional';

export interface SecurityEvent {
  id: string;
  serviceId: string;
  severity: Severity;
  eventType: string;
  title: string;
  description: string;
  recommendation?: string;
  agentId: string;
  status: EventStatus;
  thoughtSignature?: ThoughtSignature;
  isLiveIntercepted: boolean;
  detectedAt: string;
}

export interface ThoughtSignature {
  signature: string;
  signatureVersion: string;
  confidence: number;
  reasoningSteps: string[];
}

export interface AgentState {
  agentId: string;
  status: AgentStatus;
  tier: AgentTier;
  lastHeartbeatAt: string;
  restartCount: number;
  message?: string;
}

// @agents가 구현하는 인터페이스 (agent-interface.ts)
export interface IAgentInput {
  serviceId: string;
  logLines?: string[];
  metrics?: Record<string, number>;
  triggerEvent?: SecurityEvent;
}

export interface IAgentOutput {
  shouldAlert: boolean;
  severity?: Severity;
  event?: Omit<SecurityEvent, 'id' | 'detectedAt'>;
}
```

---

## 충돌 발생 시 해결 프로토콜

```
1. 같은 파일에 두 명이 작업한 경우:
   → 각자의 브랜치에서 먼저 rebase (git rebase main)
   → 충돌 파일의 소유 역할 확인
   → 소유 역할이 다르면: 소유 역할이 맞게 해결
   → 공유 영역이면: @infra + 관련 역할이 함께 해결

2. API 계약 불일치 발견 시:
   → @api에 Issue 등록 → @api가 openapi.yaml 수정 → @frontend 자동 재생성

3. DB 스키마 변경 필요 시:
   → @infra에 Issue 등록 → @infra가 새 migration 파일 추가 → PR 머지 후 다른 역할 반영
```

---

## 역할별 Claude Code 사용 팁

```bash
# 자신의 역할 선언 + 작업 범위 명시 예시

# @infra
"나는 @infra 역할입니다.
packages/shared/types.ts에 SecurityEvent 타입을 추가해줘.
packages/agents, packages/api, packages/web은 건드리지 마."

# @agents
"나는 @agents 역할입니다.
packages/agents/ids/index.ts에서 Brute Force 탐지 로직을 구현해줘.
IAgentInput/Output은 packages/shared/agent-interface.ts를 import해서 사용해."

# @api
"나는 @api 역할입니다.
packages/api/routes/events.ts에 acknowledge 엔드포인트를 구현해줘.
openapi.yaml도 동시에 업데이트해줘."

# @frontend
"나는 @frontend 역할입니다.
packages/web/src/canvas/agents/IDSCharacter.tsx를 만들어줘.
API 호출은 packages/web/src/api-client/ (openapi 자동생성)만 사용해."

# @devops
"나는 @devops 역할입니다.
.github/workflows/ci.yml에 PR 시 npm test 자동 실행 파이프라인을 추가해줘."
```

---

## 참고: PRD 기반 Phase별 담당 분배

| Phase | 주 담당 | 병렬 작업 가능 |
|-------|--------|-------------|
| **Week 0** 인터페이스 설계 | @infra + @devops | — |
| **Phase 1** MVP | @infra (CLI+Stream), @agents (IDS+Thought Sig), @api (Auth+Slack) | @frontend (뼈대) |
| **Phase 2** Agent 확장 | @agents (IAM+DLP+LateralMov+SupplyChain), @frontend (Canvas 완성) | @api (통계+리포트) |
| **Phase 3** npm 배포 | @agents (Live+Misc+Ransomware), @devops (npm publish+CI) | @api (P2 기능), @frontend (최적화) |
