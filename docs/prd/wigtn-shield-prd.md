# WIGTN Shield - AI 기반 사내 보안 모니터링 플랫폼 PRD

> **Version**: 3.0
> **Created**: 2026-03-03
> **Updated**: 2026-03-03 (digging v2 전체 반영)
> **Status**: Draft
> **배포 형태**: npm 오픈소스 CLI 패키지

---

## 쉽게 이해하는 한 줄 요약

> "사내 서버들을 24시간 감시하는 AI 경비원 팀을 고용하고,
> 이상한 일이 생기면 즉시 담당자에게 보고하는 시스템 — npm으로 5분 안에 설치"

---

## 목차

1. [문제 정의](#1-문제-정의)
2. [전체 구조 한눈에 보기](#2-전체-구조-한눈에-보기)
3. [핵심 구성 요소](#3-핵심-구성-요소)
4. [AI Agent 팀 구성](#4-ai-agent-팀-구성)
5. [기능 요구사항](#5-기능-요구사항)
6. [비기능 요구사항](#6-비기능-요구사항)
7. [사용자 스토리](#7-사용자-스토리)
8. [기술 설계](#8-기술-설계)
9. [API 명세](#9-api-명세)
10. [데이터베이스 스키마](#10-데이터베이스-스키마)
11. [오픈소스 설치 가이드 구조](#11-오픈소스-설치-가이드-구조)
12. [Canvas 대시보드 UI/UX 설계](#12-canvas-대시보드-uiux-설계)
13. [구현 단계](#13-구현-단계)
14. [성공 지표](#14-성공-지표)

---

## 1. 문제 정의

### 1.1 현재 상황 (문제)

현재 사내 서비스들은 각자 따로 운영되고 있어서:

- 서비스 A의 서버 로그는 서비스 A 팀만 봄
- 서비스 B에 이상 징후가 생겨도 다른 팀은 모름
- 보안 위협을 사람이 24시간 수동으로 확인해야 함
- "언제, 어디서, 무엇이 이상했는지" 추적이 어려움

```
현재 상태:
  [서비스 A 서버] → 각자 로그 확인 (사람이 직접)
  [서비스 B 서버] → 각자 로그 확인 (사람이 직접)
  [서비스 C 서버] → 각자 로그 확인 (사람이 직접)

  문제점: 전체 그림을 보는 사람이 없다!
```

### 1.2 목표 (해결책)

```
목표 상태:
  [서비스 A 서버] ─┐
  [서비스 B 서버] ─┼→ [중앙 수집소] → [AI 분석팀] → [관리자 알림]
  [서비스 C 서버] ─┘

  핵심: AI가 모든 서버를 동시에 감시하고 이상 시 즉시 보고!
  설치: npm install -g wigtn-shield && wigtn-shield init && wigtn-shield start
```

### 1.3 목표

- 모든 사내 서비스의 상태를 한 화면에서 확인
- AI Agent가 24시간 자동으로 보안 이상 징후 탐지
- 이상 발생 시 담당자에게 **2분 이내** 알림 (스트리밍 아키텍처)
- 보안 사고 원인 추적 시간을 80% 단축
- npm으로 누구나 5분 안에 설치 가능한 오픈소스

### 1.4 범위 (이번에 만들 것 vs 만들지 않을 것)

| 포함 (만들 것) | 제외 (만들지 않을 것) |
|--------------|-------------------|
| 로그/메트릭 중앙 수집 | 멀티 AI 제공자 지원 (Vertex AI 전용) |
| AI 기반 이상 탐지 | 서비스 코드 자동 수정 |
| **Canvas 대시보드 (에이전트 캐릭터 기반 실시간 시각화)** | 외부 서비스 모니터링 |
| Slack/이메일 알림 | 사용자 행동 분석 (별도 기획) |
| 보안 리포트 자동 생성 | - |
| npm CLI 패키지 배포 | - |
| Agent 자가 복구 시스템 | - |
| 자동 차단 (P2 — config autoBlock 옵션) | - |

---

## 2. 전체 구조 한눈에 보기

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WIGTN SHIELD (npm CLI)                            │
│         wigtn-shield init / start / status / stop                    │
└─────────────────────────────────────────────────────────────────────┘

[ 1단계: 데이터 수집 — 로컬 사내망 ]
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 서비스 A │  │ 서비스 B │  │ 서비스 C │  ← 사내 모든 서비스
│  서버    │  │  서버    │  │  서버    │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │ Prometheus   │ Fluent Bit  │ Exporter
     ▼              ▼             ▼
┌─────────────────────────────────────────┐
│  Prometheus (메트릭)  │  Loki (로그)     │
│  :9090               │  :3100           │
└─────────────────────────────────────────┘
                    │ HTTP API (실시간 스트림)
                    ▼
[ 2단계: WIGTN Shield 엔진 — 사내 서버에서 실행 ]
┌─────────────────────────────────────────────────────┐
│  wigtn-shield 프로세스                               │
│  (systemd/pm2 보호 + Heartbeat 자가 감시)            │
│                                                     │
│  ┌──────────┐    ┌──────────────────────────────────┐   │
│  │ Stream   │    │     AI Agent 오케스트레이터        │   │
│  │ Processor│───▶│  (LangGraph + Vertex AI)          │   │
│  │ (실시간)  │    │  ┌──────┬──────┬──────┬────┐     │   │
│  └──────────┘    │  │ IDS  │DDoS  │ IAM  │DLP │     │   │
│                  │  └──────┴──────┴──────┴────┘     │   │
│  ┌──────────┐    │  ┌──────────────────────────┐    │   │
│  │Live API  │───▶│  │  Live Interception Agent  │    │   │
│  │(화면/음성)│    │  │  (Gemini Live API)        │    │   │
│  │WebSocket │    │  └──────────────────────────┘    │   │
│  └──────────┘    │       ↓ 판단 시 Thought Signature  │   │
│                  │       (HMAC-SHA256 서명 → DB 저장)  │   │
│  ┌──────────┐    └──────────────────────────────────┘   │
│  │ REST API │    ← Web UI / 외부 연동용                  │
│  │ :3000    │                                           │
│  └──────────┘                                           │
└────────────────────────────────────────────────────────┘
                    │ Vertex AI API 호출 (외부 인터넷)
                    ▼ (gcloud auth 인증)
        ┌───────────────────────┐
        │  Google Vertex AI     │
        │  (Gemini 모델)         │
        └───────────────────────┘
                    │
[ 3단계: 알림 발송 ]
┌─────────────────────────────────────────┐
│  알림 발송 시스템 (억제 + 리마인더 포함)  │
│  ┌──────┐ ┌──────┐ ┌────────────────┐  │
│  │Slack │ │Email │ │Web UI 대시보드  │  │
│  └──────┘ └──────┘ └────────────────┘  │
└─────────────────────────────────────────┘
```

---

## 3. 핵심 구성 요소

### 구성 요소를 "경비 시스템"에 비유해서 이해하기

| 구성 요소 | 실제 역할 | 비유 |
|---------|---------|------|
| **Prometheus** | 각 서버에서 숫자 데이터 수집 (CPU, 메모리, 요청 수 등) | 온도계, 전력계 |
| **Loki** | 각 서버에서 텍스트 로그 수집 (에러 메시지, 접속 기록 등) | CCTV 녹화본 |
| **Stream Processor** | Prometheus/Loki의 데이터를 실시간으로 읽어 AI에 전달 | CCTV 실시간 모니터 |
| **Grafana** | 수집된 데이터를 그래프로 시각화 (선택 구성 요소) | 중앙 모니터실 화면 |
| **AI Agent** | 데이터를 분석해서 이상 징후 판단 | 분석 전문 경비원 |
| **Web UI** | 이벤트 확인, 알림 설정, 보고서 조회 | 경비 통제 센터 |
| **알림 시스템** | 이상 발견 시 담당자에게 보고 (중복 억제 포함) | 비상 연락망 |

---

## 4. AI Agent 팀 구성

### 4.1 오케스트레이터 Agent (팀장)

**역할**: 전체 상황을 파악하고 각 전문 Agent에게 업무를 배분

```
오케스트레이터 Agent
├── 역할: 총괄 지휘관
├── 하는 일:
│   ├── 각 Agent의 분석 결과를 종합
│   ├── 심각도 최종 판단 (low / medium / high / critical)
│   ├── 알림 발송 결정 (중복 억제 정책 적용)
│   └── 일일/주간 보안 요약 리포트 생성
└── 기술: Google Vertex AI (Gemini 기반) + LangGraph
```

### 4.2 침입 탐지 Agent (IDS Agent)

```
침입 탐지 Agent
├── 역할: 경비원 (출입자 검문)
├── 심각도 기준:
│   ├── low:      로그인 실패 10회/분
│   ├── medium:   알 수 없는 IP에서 접근
│   ├── high:     로그인 실패 100회/분 (Brute Force)
│   └── critical: 로그인 성공 후 즉시 권한 상승 시도
├── 탐지 예시:
│   "외부 IP에서 1분에 500번 로그인 시도 감지"
│   → critical 알림 발송
└── 기술: 이상 탐지 ML 모델 + 규칙 기반 탐지
```

### 4.3 DDoS 탐지 Agent

```
DDoS 탐지 Agent
├── 역할: 교통 경찰 (비정상 트래픽 통제)
├── 심각도 기준:
│   ├── low:      트래픽 150% 증가
│   ├── medium:   트래픽 300% 증가
│   ├── high:     트래픽 500% 증가 + 응답 지연
│   └── critical: 서비스 사실상 불능 상태
├── 탐지 예시:
│   "API 서버 트래픽 10배 급증, 응답 시간 5000ms 초과"
│   → critical 알림 (서비스 불능 상태)
└── 기술: 통계적 이상치 탐지 (Z-Score, Isolation Forest)
```

### 4.4 접근 권한 Agent (IAM Agent)

```
접근 권한 Agent
├── 역할: 출입 관리자 (배지 확인)
├── 심각도 기준:
│   ├── low:      업무 외 시간대 접근
│   ├── medium:   비정상적인 리소스 접근 패턴
│   ├── high:     권한 없는 리소스 접근 시도
│   └── critical: 갑작스러운 권한 상승 + 민감 데이터 접근
├── 탐지 예시:
│   "마케팅팀 계정이 DB 관리자 권한으로 접속 시도"
│   → critical 알림
│   [P2] autoBlock: true 설정 시 → 자동 차단 API 호출
└── 기술: RBAC 규칙 엔진 + 행동 기반 이상 탐지
```

### 4.5 데이터 유출 탐지 Agent (DLP Agent)

```
데이터 유출 탐지 Agent
├── 역할: 자산 보호 요원 (반출 검사)
├── ⚠️ Secret Leakage Agent와의 역할 구분:
│   DLP Agent:             외부 네트워크 전송 패턴 감시 (아웃바운드 트래픽)
│   Secret Leakage Agent:  로그/파일 내 자격증명 문자열 탐지 (로컬 파일/로그)
│   이벤트 병합 정책:
│     동일 서비스에서 ±5초 내 두 Agent 동시 탐지 시
│     오케스트레이터가 "내부 노출 + 외부 유출 복합 위협"으로 심각도 상향 통합
├── 심각도 기준:
│   ├── low:      비업무 시간 데이터 조회
│   ├── medium:   대용량 데이터 조회 패턴
│   ├── high:     개인정보/기밀 파일 접근 급증
│   └── critical: 고객 데이터 외부 IP로 전송 시도
├── 탐지 예시:
│   "고객 DB 10만 건 데이터 외부 IP로 전송 시도"
│   → critical 알림
│   [P2] autoBlock: true 설정 시 → 자동 차단 API 호출
└── 기술: 데이터 분류 모델 + 트래픽 패턴 분석
```

### 4.6 Agent 자가 복구 시스템 (3단계 방어)

```
3단계 Agent 장애 방어:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1단계 | OS 레벨 방어 (systemd / pm2)
      | 목적: 프로세스 완전 종료 방지
      | 동작: 프로세스 사망 감지 즉시 자동 재시작
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2단계 | Heartbeat 좀비 감지
      | 목적: 프로세스는 살아있지만 일 안 하는 상태 감지
      | 동작: 각 Agent는 30초마다 heartbeat 신호 발송
      |       오케스트레이터가 2분 이상 신호 없으면 감지
      |       → 해당 Agent 강제 재시작 실행
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
3단계 | 재시작 보고서 발송
      | 목적: 관리자에게 장애 사실 즉시 통보
      | 동작: 재시작 발생 시 알림 발송
      |       내용: "⚠️ [Agent명] 비정상 종료 감지.
      |             자동 재시작 완료. 원인을 확인하세요."
      |       포함 정보: 재시작 시각, 직전 처리 이벤트,
      |                  재시작 횟수 (비정상 반복 시 escalation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 4.7 Thought Signature (추론 증적) 시스템

**역할**: AI가 판단한 순간의 추론 상태를 변조 불가능하게 봉인

```
Thought Signature란?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
일반 AI 보안 도구:
  [이벤트 발생] → AI 분석 → "비정상 접근입니다" (결과만 기록)
  ❌ 문제: "왜 그렇게 판단했는지" 사후에 증명 불가, 조작 여지 있음

WIGTN Shield (Thought Signature):
  [이벤트 발생] → AI 분석 → 추론 상태 캡처 → HMAC-SHA256 서명
                           └─ 판단 근거     └─ 서버 시크릿 키로 서명
                              (사용 데이터,      DB에 저장 (변경 불가)
                               판단 과정,
                               신뢰도 점수)
  ✅ 효과: "언제, 무엇을 보고, 어떻게 추론했는지" 법적으로 증명 가능
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Thought Signature 구성 요소:
┌─────────────────────────────────────────────────────────┐
│  thought_payload (서명 전 원문, 암호화 저장)              │
│  {                                                      │
│    "agentId": "agent_ids",                              │
│    "timestamp": "2026-03-03T14:30:00.123Z",             │
│    "inputData": {                                       │
│      "logLines": [...],        ← AI가 실제 본 데이터     │
│      "metrics": {...}                                   │
│    },                                                   │
│    "reasoningSteps": [         ← 추론 단계               │
│      "로그인 실패 횟수: 500회/분 감지",                   │
│      "정상 임계값 10회/분 초과 (50배)",                   │
│      "발신 IP 203.x.x.x: 알려진 공격 IP 목록에 없음",     │
│      "패턴이 Dictionary Attack과 일치 (신뢰도 0.97)"     │
│    ],                                                   │
│    "confidence": 0.97,         ← 판단 신뢰도             │
│    "severityJustification": "critical: 서비스 침해 임박" │
│  }                                                      │
├─────────────────────────────────────────────────────────┤
│  thought_signature (서명값, 검증용)                       │
│  = HMAC-SHA256(thought_payload, SERVER_SECRET_KEY)      │
│  = "a3f8c2d1e9b4..."  (64자 hex)                       │
└─────────────────────────────────────────────────────────┘

검증 방법:
  wigtn-shield verify --event-id evt_abc123
  → thought_payload를 SERVER_SECRET_KEY로 재서명 후 비교
  → 일치: "✅ 추론 증적 무결성 확인"
  → 불일치: "❌ 기록이 변조되었습니다 — 보안 사고 조사 필요"

적용 범위:
  → IDS / DDoS / IAM / DLP / Injection / Supply Chain /
     Lateral Movement / Secret Leakage / Misconfiguration Agent
  → Live Interception Agent: 실시간 특성상 비동기로 서명 생성
    (이벤트 저장 즉시 백그라운드에서 서명, 지연 최대 200ms)
  → Ransomware Agent: 파일 시스템 이벤트도 동일하게 서명 적용
  ※ 모든 Agent가 이벤트 생성 시 반드시 Thought Signature 포함
```

**SERVER_SECRET_KEY 관리 정책**

```
생성:
  wigtn-shield init 실행 시 256-bit 난수 자동 생성
  저장 우선순위:
    1순위: OS 키체인 (keytar 라이브러리 — macOS Keychain, Windows Credential, Linux Secret Service)
    2순위: 키체인 불가 환경 → .wigtn-secret (600 권한, 루트 디렉토리)
           .gitignore 자동 추가
  백업:
    init 완료 시 복구 코드 24단어 출력 → 관리자가 안전하게 보관
    "이 코드를 잃으면 기존 이벤트 서명 검증 불가"

로테이션:
  wigtn-shield rotate-key
  → 새 SERVER_SECRET_KEY 생성
  → 기존 이벤트: signature_version으로 이전 키 추적 (다중 키 지원)
  → 신규 이벤트: 새 키로 서명

유출 시 대응:
  wigtn-shield rotate-key --emergency
  → 즉시 새 키로 교체 + 관리자 긴급 알림 발송
  → 기존 이벤트는 "키 교체 전 서명" 으로 상태 표시
```

---

### 4.8 Live Interception Agent (실시간 개입 Agent)

**역할**: 로그가 쌓이기 전에 실시간 화면/음성 스트림을 감시하고 즉시 개입

```
기존 방식 (사후 대응):
  공격 발생 → 로그 생성 → 수집(30초) → AI 분석 → 알림 → 대응
  총 소요: 수 분

Live Interception (사전 개입):
  공격 발생 → Gemini Live API가 화면/음성 실시간 감시
             → 이상 패턴 감지 즉시(초 단위) 개입
  총 소요: 수 초

감시 대상:
├── 화면(Vision): 서비스 관리 패널, 터미널 세션, DB 콘솔
│   예시: "관리자 화면에서 대량 DELETE 쿼리 입력 중"
│         → 즉시 세션 일시 중단 + 관리자 인증 재요청
│
├── 음성(Audio): 보안 회의, 콜센터, 음성 인증 채널
│   예시: "비밀번호를 음성으로 전달하는 패턴 감지"
│         → 보안 경고 + 해당 세션 플래그
│
└── 텍스트 스트림: 실시간 채팅, 입력 중인 명령어
    예시: "rm -rf / 입력 중 감지"
          → 명령 실행 전 차단 + critical 알림

기술: Gemini Live API (WebSocket 기반 실시간 스트리밍)

설정 (config.yaml):
  liveInterception:
    enabled: false          # 기본값 비활성화 (사생활 보호)
    targets:
      - type: screen
        serviceId: admin-panel
      - type: terminal
        serviceId: db-server
    interventionMode: alert  # alert(알림만) | pause(일시중단) | block(차단)
```

---

### 4.9 인젝션 공격 탐지 Agent (Injection Agent)

**역할**: SQL·Command·LDAP 인젝션 공격을 애플리케이션 로그에서 탐지 (OWASP Top 10 A03)

```
인젝션 탐지 Agent
├── 역할: 문서 위조 감지원 (입력값 검열)
├── ⚠️ 역할 경계 (WAF와 구분):
│   WIGTN Shield: 로그 기반 탐지 + 관리자 알림 (사후 감지)
│   WAF / API Gateway: 실제 요청 차단 (사전 방어) ← WIGTN Shield 범위 아님
│   → WAF 미설치 환경 감지 시: Misconfiguration Agent(FR-028)로 경고 연계
│   → 전제 조건: DB 쿼리 로그 활성화 필요 (연동 가이드 제공)
├── 감시 대상:
│   ├── SQL Injection: DB 쿼리 로그에서 ' OR '1'='1', UNION SELECT, DROP TABLE 패턴
│   ├── Command Injection: 시스템 로그에서 ; rm -rf, | cat /etc/passwd 패턴
│   └── LDAP Injection: 인증 로그에서 *)(uid=*))(|(uid=* 패턴
├── 심각도 기준:
│   ├── medium:   인젝션 시도 감지 (실패)
│   ├── high:     반복적 인젝션 시도 (공격 탐색 중)
│   └── critical: 인젝션 성공 징후 (비정상 쿼리 결과 반환)
└── 기술: 정규식 패턴 매칭 + Loki 로그 스트리밍
```

---

### 4.10 공급망 공격 탐지 Agent (Supply Chain Agent)

**역할**: 의존성 패키지 변경 및 알려진 CVE 취약점 모니터링

```
공급망 탐지 Agent
├── 역할: 납품 검수원 (들어오는 재료 검사)
├── 실행 방식: Tier 3 (스케줄) — 기본 24시간 1회
├── 감시 대상:
│   ├── 의존성 변경 탐지: package.json, requirements.txt, pom.xml 변경 이벤트
│   ├── 악성 패키지 대조: OSV(Open Source Vulnerabilities) DB 조회
│   ├── CVE 취약점: NIST NVD API로 사용 중인 패키지 취약점 스캔
│   └── Typosquatting: lodash → 1odash 같은 오타 위장 패키지 감지
├── 심각도 기준:
│   ├── low:      CVSS 4.0 미만 취약점
│   ├── medium:   CVSS 4.0~7.0 취약점
│   ├── high:     CVSS 7.0~9.0 취약점
│   └── critical: CVSS 9.0 이상 / 악성 패키지 직접 설치
├── API 안정성:
│   ├── NIST NVD API: nvdApiKey 없으면 5 req/30초 제한 → API Key 권장
│   ├── 스캔 결과 SQLite에 24시간 캐싱 → API 장애 시 캐시로 운영
│   └── offlineMode: true 설정 시 외부 API 호출 없이 로컬 캐시만 사용
├── 특이점:
│   WIGTN Shield 자체도 npm 패키지이므로 자기 자신의 의존성도 감시 대상
└── 기술: OSV API, NIST NVD API, chokidar (파일 감시), SQLite 캐시
```

---

### 4.11 횡적 이동 탐지 Agent (Lateral Movement Agent)

**역할**: 침입자가 내부 네트워크에서 다른 서비스로 이동하는 패턴 탐지 (MITRE ATT&CK T1021)

```
횡적 이동 탐지 Agent
├── 역할: 내부 순찰 경비원 (건물 내 이상 이동 감시)
├── 핵심 아이디어:
│   각 서비스 간의 "정상적인 통신 패턴 Baseline" 학습 후
│   이탈하는 패턴을 탐지
├── ⚠️ Baseline 학습 정책 (초기 오탐 방지):
│   초기 모드 (baselineDays 기간): 탐지 없이 패턴만 수집
│   → Web UI에 학습 진행률 표시: "Baseline 학습 중 78% (11/14일)"
│   → wigtn-shield status 명령으로 진행률 확인 가능
│   전환: 학습 완료 후 자동으로 탐지 모드 전환 + 관리자 알림
│   수동 전환: wigtn-shield agent lateral-movement --mode=active
│   baselineDays: 0 설정 시 즉시 탐지 모드 (오탐 감수)
├── 감시 대상:
│   ├── 서비스 A가 평소 호출하지 않던 서비스 B에 갑자기 대량 요청
│   ├── 내부 서비스에서 다른 내부 서비스로 비정상적 포트 스캔
│   ├── 단일 계정이 짧은 시간에 다수 서비스 순차 접근
│   └── 내부 서버가 외부로 C2(Command & Control) 서버 연결 시도
├── 심각도 기준:
│   ├── medium:   새로운 내부 서비스 간 통신 패턴
│   ├── high:     빠른 속도로 다수 서비스 연결 시도
│   └── critical: 외부 C2 서버 연결 + 내부 확산 동시 발생
└── 기술: 서비스 간 트래픽 그래프 분석 + Baseline 이상치 탐지
```

---

### 4.12 시크릿 노출 탐지 Agent (Secret Leakage Agent)

**역할**: 로그·환경변수·응답값에 민감한 자격증명이 평문으로 노출되는 것을 탐지

```
시크릿 노출 탐지 Agent
├── 역할: 기밀 문서 유출 방지 요원
├── 감시 대상:
│   ├── 로그 파일: API Key, JWT Token, DB 비밀번호 패턴 스캔
│   │   예: "key=AKIA...", "password=abc123", "Bearer eyJ..."
│   ├── HTTP 응답 로그: 사용자에게 내부 정보가 노출되는 에러 응답
│   │   예: DB connection string이 500 에러 메시지에 포함
│   ├── 환경변수 출력: env, printenv 명령 실행 로그
│   └── Git 커밋 로그: 코드에 시크릿 하드코딩 후 커밋 이벤트
├── 심각도 기준:
│   ├── medium:   에러 응답에 내부 경로/서버명 노출
│   ├── high:     로그에 JWT 토큰 또는 API Key 패턴 감지
│   └── critical: DB 비밀번호, 암호화 키 평문 노출
├── 대표 패턴 (정규식):
│   ├── AWS Key:      AKIA[0-9A-Z]{16}
│   ├── JWT:          eyJ[A-Za-z0-9+/=]+\.[A-Za-z0-9+/=]+\.[A-Za-z0-9+/=]+
│   ├── Generic Key:  (api[_-]?key|secret|password)\s*[=:]\s*\S{8,}
│   └── DB URL:       (mysql|postgres|mongodb):\/\/[^:]+:[^@]+@
└── 기술: Loki 로그 스트리밍 + 정규식 패턴 라이브러리 (detect-secrets 기반)
```

---

### 4.13 설정 오류 탐지 Agent (Misconfiguration Agent)

**역할**: 불필요하게 열린 포트, 기본값 설정, 취약한 TLS 등 잘못된 보안 설정을 주기적으로 스캔 (OWASP Top 10 A05)

```
설정 오류 탐지 Agent
├── 역할: 시설 안전 점검관 (정기 검사)
├── 실행 방식: Tier 3 (스케줄) — 기본 60분 1회
├── 감시 대상:
│   ├── 포트 노출:   외부 접근 가능한 불필요 포트 (예: DB 포트 3306이 공개망에 노출)
│   ├── 기본 비밀번호: 변경하지 않은 기본 인증 정보 감지
│   │   예: Grafana admin/admin, Redis no-password
│   ├── 취약한 TLS:  TLS 1.0/1.1 사용 중인 엔드포인트
│   ├── HTTP 평문:   HTTPS 미적용 서비스 감지
│   ├── WAF 미설치: API Gateway / WAF 없이 직접 노출된 서비스
│   │   → Injection Agent(FR-024)와 연계하여 이중 경고
│   └── 불필요한 디버그 엔드포인트: /actuator, /_debug, /phpinfo 노출
├── 심각도 기준:
│   ├── low:      비권장 TLS 버전 사용
│   ├── medium:   HTTP 평문 통신, 불필요한 포트 노출
│   ├── high:     WAF 미설치 + 인젝션 취약점 의심
│   └── critical: 기본 비밀번호 사용 중 또는 DB가 공개망에 직접 노출
├── 탐지 예시:
│   "DB 서버 포트 5432가 0.0.0.0:5432로 바인딩 — 공개망 접근 가능"
│   → critical 알림 + 즉각 방화벽 규칙 수정 권고
└── 기술: nmap 포트 스캔 대안(직접 소켓), config 파일 분석, TLS 핸드셰이크 검사
```

---

### 4.14 랜섬웨어 지표 탐지 Agent (Ransomware Agent)

**역할**: 파일 시스템에서 랜섬웨어 특유의 행동 패턴(대규모 암호화, 확장자 변경, 백업 삭제)을 탐지

```
랜섬웨어 지표 탐지 Agent
├── 역할: 화재 감지 스프링클러 (초기 진압)
├── 실행 방식: Optional (기본 비활성화) — 활성화 시 실시간 파일 시스템 감시
├── 감시 대상 (MITRE ATT&CK T1486):
│   ├── 대규모 파일 암호화: 짧은 시간 내 다수 파일 수정 (1분 내 100개 이상)
│   ├── 확장자 대량 변경:  .jpg → .encrypted, .docx → .locked 등
│   ├── Shadow Copy 삭제: vssadmin delete shadows, wbadmin delete 명령 감지
│   └── 백업 디렉토리 삭제: /backup, /var/backup 경로 대규모 삭제
├── 심각도 기준:
│   ├── medium:   비정상적으로 많은 파일 수정 패턴
│   ├── high:     파일 확장자 대량 변경 감지
│   └── critical: Shadow Copy 삭제 + 파일 암호화 동시 감지
├── 탐지 예시:
│   "1분 내 /data 디렉토리에서 파일 247개 확장자 변경 감지"
│   → critical 알림 + 즉각 서버 격리 권고
├── 설정 (config.yaml):
│   ransomware:
│     enabled: false          # 기본 비활성화 (파일 I/O 성능 영향)
│     watchPaths: ["/data", "/home"]
│     encryptionRateThreshold: 100  # 1분 내 파일 변경 임계값
└── 기술: chokidar (파일 감시), 패턴 매칭, 시스템 명령 로그 분석
```

---

### 4.15 Agent 협업 예시

```
실제 시나리오:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
① IDS Agent:  "외부 IP에서 로그인 성공 감지" (high)
        ↓
② 오케스트레이터: IAM Agent + DLP Agent에 추가 조사 지시
        ↓
③ IAM Agent:  "해당 계정이 DB에 접근 시도 확인" (high)
④ DLP Agent:  "고객 데이터 대량 조회 패턴 감지" (critical)
        ↓
⑤ 오케스트레이터: "종합 판단 - critical"
        ↓
⑥ 알림 발송: Slack 긴급 채널 + 이메일 동시 발송
        ↓  (억제 정책 활성화: 이후 30분간 동일 이벤트 억제)
⑦ 30분 후: "⏰ 리마인더: 고객 데이터 유출 위협 미해결"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 5. 기능 요구사항

| ID | 기능 | 설명 | 우선순위 |
|----|------|------|---------|
| FR-001 | 로그 중앙 수집 | Loki HTTP API로 사내 서비스 로그 실시간 수집 | P0 |
| FR-002 | 메트릭 수집 | Prometheus HTTP API로 CPU, 메모리, 네트워크 수집 | P0 |
| FR-003 | 실시간 스트림 처리 | Prometheus/Loki 데이터 실시간 스트리밍으로 2분 내 탐지 | P0 |
| FR-004 | 관리자 Web UI | 이벤트 목록, 서비스 상태, 설정 관리용 웹 대시보드 | P0 |
| FR-005 | Web UI 인증 | 이메일+비밀번호 로그인, JWT 기반 세션 관리 | P0 |
| FR-006 | 침입 탐지 Agent | 비정상 접근 시도 자동 감지 (4단계 심각도 기준) | P0 |
| FR-007 | DDoS 탐지 Agent | 트래픽 급증 및 공격 패턴 감지 (4단계 심각도 기준) | P0 |
| FR-008 | 접근 권한 Agent | 권한 위반 및 이상 행동 감지 (4단계 심각도 기준) | P0 |
| FR-009 | 데이터 유출 Agent | 중요 데이터 외부 유출 감지 (4단계 심각도 기준) | P0 |
| FR-010 | Agent 헬스체크 | 3단계 방어 (systemd/pm2 + Heartbeat + 재시작 보고서) | P0 |
| FR-011 | Slack 알림 | 이상 탐지 시 Slack 채널 자동 알림 | P0 |
| FR-012 | 알림 중복 억제 | 동일 이벤트 30분 억제 + 주기적 리마인더 발송 | P0 |
| FR-013 | 서비스 등록/관리 | Web UI 및 API로 모니터링 대상 서비스 CRUD | P0 |
| FR-014 | 이메일 알림 | 심각도 high/critical 사고 시 이메일 발송 | P1 |
| FR-015 | 보안 리포트 | 일일/주간 보안 현황 리포트 자동 생성 및 이메일 발송 | P1 |
| FR-016 | 알림 히스토리 | 발송된 알림 내역 조회 및 관리 | P1 |
| FR-017 | 이벤트 통계 API | 대시보드용 집계 데이터 (심각도별, 서비스별, 기간별) | P1 |
| FR-018 | 서비스 연동 가이드 | 각 언어/프레임워크별 Prometheus/Loki 연동 방법 문서 | P1 |
| FR-019 | 자동 차단 (P2) | config autoBlock: true 설정 시 critical 이벤트 자동 차단 API 호출 | P2 |
| FR-020 | 사고 대응 가이드 | 이상 탐지 시 AI가 대응 방법 제안 | P2 |
| FR-021 | SMS 알림 | 긴급 상황 SMS 발송 | P2 |
| **FR-022** | **Thought Signature (추론 증적)** | AI가 판단한 시점의 추론 상태를 HMAC-SHA256으로 서명 → security_events에 저장. 변조 불가능한 보안 감사 기록. WIGTN-Shield의 핵심 차별점. | **P0** |
| **FR-023** | **Live Interception Agent** | Gemini Live API를 통해 실시간 화면/음성 스트림을 감시하고 이상 감지 즉시 개입. 사후 대응(Reactive)에서 사전 개입(Proactive)으로 전환. | **P1** |
| **FR-024** | **인젝션 공격 탐지 Agent** | 애플리케이션/DB 로그에서 SQL Injection, Command Injection, LDAP Injection 패턴 탐지. OWASP Top 10 A03. | **P0** |
| **FR-025** | **공급망 공격 탐지 Agent** | 서비스의 의존성 패키지(npm/pip/maven) 변경 감지, 알려진 악성 패키지 목록 대조, CVE 취약점 탐지. | **P1** |
| **FR-026** | **횡적 이동 탐지 Agent** | 침입 성공 후 내부 서비스 간 비정상적 API 호출/접근 패턴 탐지. 서비스 A → 서비스 B로의 이상 이동 감지. | **P1** |
| **FR-027** | **시크릿 노출 탐지 Agent** | 로그에 API 키, 비밀번호, DB 연결 문자열, 토큰이 평문으로 기록되는 것을 실시간 탐지. | **P1** |
| **FR-028** | **설정 오류 탐지 Agent** | 불필요하게 열린 포트, 기본 비밀번호 사용, 불필요한 서비스 노출, 취약한 TLS 버전 감지. OWASP Top 10 A05. | **P2** |
| **FR-029** | **랜섬웨어 지표 탐지** | 파일 시스템에서 대규모 암호화 패턴, 파일 확장자 대량 변경, Shadow Copy 삭제 시도 탐지. | **P2** |
| **FR-030** | **Canvas 대시보드 UI** | 각 Agent의 페르소나 캐릭터를 Canvas에 시각화, 실시간 애니메이션으로 활동 상태 표현. 비전문가도 한눈에 전체 보안 상태 파악 가능. | **P1** |

---

## 6. 비기능 요구사항

### 6.1 성능

| 항목 | 목표 |
|------|------|
| 로그 스트리밍 지연 | 실시간 대비 최대 10초 이내 |
| 이상 탐지 반응 시간 | 이상 발생 후 **2분 이내** 탐지 (스트리밍) |
| 알림 발송 시간 | 탐지 후 30초 이내 발송 |
| Web UI 로딩 | 3초 이내 |
| 동시 수집 가능 서비스 수 | 최소 50개 서비스 |

### 6.2 보안

| 항목 | 내용 |
|------|------|
| 접근 제어 | 이메일+비밀번호 로그인 + JWT (Access 15분, Refresh 7일) |
| 세션 쿠키 | httpOnly=true (XSS 방어), SameSite=strict (CSRF 방어), HTTPS 환경에서 Secure=true |
| 비밀번호 정책 | 최소 8자, 대소문자+숫자+특수문자 1개 이상 |
| 로그인 잠금 | 5회 실패 시 30분 계정 잠금 |
| 역할 기반 권한 | admin / security_officer / viewer (아래 매트릭스 참고) |
| 데이터 암호화 | 전송 중 TLS 1.3, 저장 시 AES-256 |
| API 보안 | JWT 기반 인증, Rate Limiting (100 req/분) |
| 감사 로그 | 모든 관리자 행동 기록 (변경 불가) |
| Vertex AI 인증 | gcloud auth로 Service Account 인증 |
| Thought Signature | 모든 AI 판단 이벤트에 HMAC-SHA256 서명 → 감사 증거 |

### 6.3 역할별 권한 매트릭스

| 역할 | 이벤트 조회 | 이벤트 처리 | 서비스 관리 | 알림 설정 | 사용자 관리 | 리포트 |
|------|-----------|-----------|-----------|---------|-----------|--------|
| **admin** | ✅ 전체 | ✅ 가능 | ✅ 가능 | ✅ 가능 | ✅ 가능 | ✅ 가능 |
| **security_officer** | ✅ 전체 | ✅ 가능 | ❌ 불가 | ❌ 불가 | ❌ 불가 | ✅ 가능 |
| **viewer** | ✅ 전체 | ❌ 불가 | ❌ 불가 | ❌ 불가 | ❌ 불가 | ✅ 가능 |

### 6.4 가용성

| 항목 | 목표 |
|------|------|
| 서비스 가동률 | 99.9% (Agent 3단계 방어로 보장) |
| 데이터 보존 기간 | 로그 90일, 이벤트 365일 |
| 백업 | 일 1회 자동 백업 |

### 6.5 개인정보 보호 (Privacy / GDPR)

| 항목 | 내용 |
|------|------|
| IP 익명화 | anonymizeIPs: true 시 마지막 옥텟 마스킹 (192.168.1.xxx) |
| 사용자명 익명화 | anonymizeUsernames: false 기본값 (보안 분석 필요), 설정으로 활성화 가능 |
| GDPR 모드 | gdprMode: true 시 EU 개인정보보호법 준수 — 데이터 최소화, 처리 목적 기록 |
| 데이터 보유 한계 | 이벤트 최대 365일, 만료 후 자동 삭제 |
| 수집 목적 | 보안 모니터링 목적으로만 수집, 제3자 전달 없음 |
| 로그 내 개인정보 | Secret Leakage Agent 탐지 후 관리자가 마스킹 처리 가능 |

> **주의**: 개인정보 수집이 발생할 수 있으므로 조직 내 개인정보 담당자와 협의 후 gdprMode 설정을 결정하세요.

---

## 7. 사용자 스토리

### 7.1 시스템 설치자 (오픈소스 사용자)

```
As a 개발자/관리자,
I want to npm 명령어 몇 개로 wigtn-shield를 설치하고 싶다
So that 복잡한 인프라 설정 없이 빠르게 보안 모니터링을 시작할 수 있다

수용 기준:
  Scenario: 최초 설치
    Given Node.js와 gcloud CLI가 설치되어 있고
    When npm install -g wigtn-shield && wigtn-shield init 을 실행하면
    Then config.yaml 파일이 생성되고 필수 설정 항목 안내가 나오며
    And wigtn-shield start 실행 시 Web UI가 http://localhost:3000 에서 열린다
```

### 7.2 보안 담당자

```
As a 보안 담당자,
I want to 모든 사내 서비스의 보안 상태를 한 화면에서 보고 싶다
So that 빠르게 전체 상황을 파악하고 위협에 대응할 수 있다

수용 기준:
  Scenario: 대시보드 접근
    Given 보안 담당자가 Web UI에 로그인되어 있고
    When 메인 대시보드에 접근하면
    Then 모든 서비스의 상태(정상/경고/위험)가 색상으로 표시되고
    And 현재 활성화된 보안 이벤트 수가 심각도별로 보인다
```

```
As a 보안 담당자,
I want to 이상 징후 발생 시 즉시 알림을 받고 싶다 (중복 없이)
So that 알림 피로 없이 진짜 위협에 집중할 수 있다

수용 기준:
  Scenario: 중복 억제 알림
    Given DDoS 공격이 1시간 동안 지속되고 있을 때
    When critical 이벤트가 최초 탐지되면
    Then 즉시 Slack 긴급 채널에 1회 알림이 오고
    And 이후 30분 동안 같은 이벤트로 추가 알림은 발송되지 않으며
    And 30분 후 미해결 시 "⏰ 리마인더" 알림이 1회 발송된다
```

### 7.3 시스템 관리자

```
As a 시스템 관리자,
I want to 특정 서비스의 로그를 빠르게 검색하고 싶다
So that 문제 원인을 신속하게 파악할 수 있다

수용 기준:
  Scenario: 로그 검색
    Given 관리자가 로그 검색 화면에 있고
    When 서비스명과 시간 범위를 선택하고 키워드를 입력하면
    Then 3초 이내에 관련 로그가 시간순으로 나열된다
```

### 7.4 임원진 (비기술직)

```
As a 임원,
I want to 주간 보안 현황을 쉽게 이해할 수 있는 리포트를 받고 싶다
So that 보안 투자 결정에 데이터 기반 판단을 할 수 있다

수용 기준:
  Scenario: 주간 리포트 수신
    Given 매주 월요일 오전 9시가 되면
    When 설정된 수신자 목록(admin 설정)으로 이메일이 발송되고
    Then 지난 주 보안 이벤트 요약, 심각도별 통계, 주요 위협이 포함되고
    And 그래프와 함께 비기술인도 이해할 수 있는 문장으로 설명된다
```

---

## 8. 기술 설계

### 8.1 기술 스택

```
┌─────────────────────────────────────────────────────────────┐
│                    기술 스택 구성                             │
├─────────────┬───────────────────────────────────────────────┤
│ CLI/패키지  │ Node.js, Commander.js, npm                     │
│ 데이터 수집  │ Prometheus HTTP API, Loki HTTP API             │
│ 스트리밍    │ Prometheus remote_write, Loki tail API          │
│ 데이터 저장  │ SQLite (기본, 로컬) / PostgreSQL (선택)         │
│ Web UI      │ React, Vite, TailwindCSS                       │
│ 백엔드      │ FastAPI (Python) 또는 Express (Node.js)         │
│ AI/ML      │ Google Vertex AI (Gemini 2.0)                  │
│ AI 프레임워크│ LangGraph (Agent 오케스트레이션)                │
│ GCP 인증    │ gcloud auth / Application Default Credentials  │
│ 알림        │ Slack Webhook, Nodemailer (이메일)              │
│ 프로세스    │ pm2 또는 systemd 서비스 등록                    │
│ 패키지 배포  │ npm publish                                    │
└─────────────┴───────────────────────────────────────────────┘
```

**SQLite 선택 이유**: 오픈소스 사용자가 별도 DB 설치 없이 바로 실행 가능하게 하기 위함.
PostgreSQL은 대규모 환경을 위한 선택 옵션으로 config에서 전환 가능.

### 8.2 수정된 시스템 아키텍처

```
[사내 서비스들] — 로컬 네트워크 —
    │
    │ 메트릭 Prometheus 수집         로그 Loki 수집
    ▼                                ▼
[Prometheus :9090]              [Loki :3100]
    │                                │
    │  HTTP API (직접 호출)           │  HTTP API (직접 호출)
    └────────────┬────────────────────┘
                 │
    [Stream Processor]  ← Prometheus/Loki tail API로 실시간 수신
                 │
    [WIGTN Shield Core] ← 사내 서버에서 실행 (npm CLI)
         │            │
    [REST API]   [AI Agent 오케스트레이터]
    [:3000]           │
         │       [IDS][DDoS][IAM][DLP]
    [Web UI]          │
                      │ HTTPS (외부 인터넷)
                      ▼
              [Google Vertex AI]  ← gcloud auth 인증

    ▼
[알림 발송] → Slack Webhook, 이메일 (SMTP)

[Grafana :4000] ← 선택 구성요소, 시각화 전용
                  (wigtn-shield와 직접 연결 없음)
```

### 8.3 config.yaml 구조

```yaml
# wigtn-shield 설정 파일 (config.yaml)
# wigtn-shield init 으로 자동 생성
#
# ⚠️  보안 주의사항:
#   - 이 파일은 git에 커밋하지 마세요 (.gitignore 자동 추가됨)
#   - 시크릿 값은 환경변수로 주입하세요 (${ENV_VAR} 문법)
#   - wigtn-shield init 실행 시 SERVER_SECRET_KEY 자동 생성

version: "1"

# 모니터링 대상 서비스
services:
  - name: my-api-server
    prometheus: http://localhost:9090
    loki: http://localhost:3100
    labels:
      env: production
      team: backend

# Google Vertex AI 설정
ai:
  provider: vertex-ai
  projectId: my-gcp-project-id      # 환경변수: ${WIGTN_GCP_PROJECT_ID}
  location: us-central1
  model: gemini-2.0-flash            # Tier 1/2용 기본 모델 (비용 절감)
  # 인증: gcloud auth application-default login

# 알림 설정 — 시크릿은 환경변수로 주입 (평문 입력 금지)
alerts:
  slack:
    enabled: true
    webhook: ${WIGTN_SLACK_WEBHOOK}   # ← 환경변수 참조
    channels:
      critical: "#security-emergency"
      high: "#security-alerts"
      medium: "#security-monitor"
  email:
    enabled: false
    smtp:
      host: smtp.gmail.com
      port: 587
      user: ${WIGTN_SMTP_USER}        # ← 환경변수 참조
      password: ${WIGTN_SMTP_PASSWORD} # ← 환경변수 참조
    recipients:
      critical: ["ciso@company.com"]
      high: ["admin@company.com"]
      weekly_report: ["cto@company.com", "admin@company.com"]
  deduplication:
    suppressionMinutes: 30
    reminderIntervalMinutes: 30

# Agent 실행 Tier 전략 (비용 최적화)
# Tier 1: 항상 실행, 경량 규칙 기반 → Gemini 호출 최소화
# Tier 2: Tier 1 의심 판정 시에만 활성화
# Tier 3: 스케줄 실행 (스트리밍 아님)
agents:
  tier1:  # 항상 실행 (경량)
    ids:           { enabled: true }
    ddos:          { enabled: true }
    injection:     { enabled: true }
    secretLeakage: { enabled: true }
  tier2:  # Tier 1 트리거 시 실행
    iam:            { enabled: true }
    dlp:            { enabled: true }
    lateralMovement:
      enabled: true
      baselineDays: 14       # 학습 기간 (0이면 즉시 탐지 모드)
      sensitivity: medium    # low | medium | high
  tier3:  # 스케줄 실행
    supplyChain:
      enabled: true
      scanIntervalHours: 24
      nvdApiKey: ${WIGTN_NVD_API_KEY}  # NIST NVD API Key (없으면 public rate limit)
      offlineMode: false                # true면 로컬 캐시만 사용
    misconfiguration:
      enabled: true
      scanIntervalMinutes: 60
  optional:  # 기본값 비활성화
    liveInterception:
      enabled: false
      targets: []
      interventionMode: alert  # alert | pause | block
    ransomware:
      enabled: false
      watchPaths: ["/data", "/home"]  # 감시할 경로

# 자동 차단 (P2, 기본 비활성화)
autoBlock:
  enabled: false
  triggerSeverity: critical
  # blockApiUrl: ${WIGTN_BLOCK_API_URL}

# 데이터베이스
database:
  type: sqlite
  path: ./data/wigtn.db
  # postgres 사용 시:
  # type: postgres
  # url: ${WIGTN_DATABASE_URL}

# 웹 서버
server:
  port: 3000
  host: localhost
  adminEmail: admin@company.com
  # ⚠️ 최초 실행 후 Web UI에서 즉시 비밀번호 변경 필요
  adminPassword: ${WIGTN_ADMIN_PASSWORD}
  cookie:
    httpOnly: true       # XSS 방어
    sameSite: "strict"   # CSRF 방어
    secure: false        # HTTPS 환경에서는 true로 변경

# 개인정보 마스킹 (GDPR / 개인정보보호법 준수)
privacy:
  anonymizeIPs: true        # IP 마지막 옥텟 마스킹: 192.168.1.xxx
  anonymizeUsernames: false  # 보안 분석 필요 시 false 유지
  gdprMode: false            # EU 서비스 사용 시 true

# 로그 보존 정책
retention:
  eventsInDays: 365
  notificationLogsInDays: 90
```

> **환경변수 설정 예시** (`.env` 파일 또는 OS 환경변수):
> ```bash
> WIGTN_SLACK_WEBHOOK=https://hooks.slack.com/services/xxx
> WIGTN_SMTP_USER=security@company.com
> WIGTN_SMTP_PASSWORD=your-app-password
> WIGTN_NVD_API_KEY=your-nvd-api-key
> WIGTN_ADMIN_PASSWORD=your-strong-password
> WIGTN_GCP_PROJECT_ID=my-gcp-project
> ```
> `wigtn-shield init` 실행 시 `.env.example` 자동 생성 및 `.gitignore`에 `.env`, `config.yaml` 자동 추가.

### 8.4 실시간 스트리밍 아키텍처

```
2분 이내 탐지를 위한 데이터 흐름:

[Prometheus]  →  remote_write  →  [Stream Processor]
                 (push 방식)        │
[Loki]       →  /loki/api/v1/tail → │
                 (WebSocket)        │
                                    ▼
                             [이상 탐지 큐]
                                    │
                             [AI Agent 호출]  ← 이상 패턴 감지 시만
                                    │          (비용 최적화)
                             [결과 저장 + 알림]

전통적 Polling 방식:  수집 1분 + 분석 2분 = 탐지까지 3분
스트리밍 방식:        이벤트 발생 즉시 분석 대기열 진입 = 탐지까지 ~30초

* Vertex AI 비용 최적화:
  - 정상 패턴: 경량 통계 모델(로컬)로 1차 필터링
  - 이상 패턴 의심 시에만 Vertex AI Gemini 호출
  - 예상 비용: 50개 서비스 기준 월 50만~150만원 (트래픽에 따라 변동)
```

### 8.5 Agent Tier 비용 전략

Agent가 10개 이상으로 확장되면 모든 Agent가 동시에 Vertex AI를 호출할 경우 비용이 선형적으로 증가합니다. 이를 방지하기 위해 **3계층 실행 전략**을 적용합니다.

```
┌──────────────────────────────────────────────────────────────┐
│                  Agent Tier 전략 (비용 최적화)                  │
├──────────────┬────────────────────────────────────────────────┤
│ Tier 1       │ 항상 실행 — 경량 규칙 기반 (Vertex AI 최소화)    │
│ (Always-on)  │ 해당: IDS, DDoS, Injection, SecretLeakage      │
│              │ 원리: 정규식 + 통계 모델로 1차 필터링            │
│              │       의심 판정 시에만 Vertex AI 호출            │
├──────────────┼────────────────────────────────────────────────┤
│ Tier 2       │ Tier 1 의심 판정 시에만 활성화                   │
│ (Triggered)  │ 해당: IAM, DLP, LateralMovement                │
│              │ 원리: Tier 1이 트리거 → 오케스트레이터가 위임     │
│              │       평소에는 대기 상태 (비용 0)                │
├──────────────┼────────────────────────────────────────────────┤
│ Tier 3       │ 스케줄 실행 (스트리밍 없음)                      │
│ (Scheduled)  │ 해당: SupplyChain (24h), Misconfiguration (1h)  │
│              │ 원리: 정해진 시간에만 실행 → 예측 가능한 비용     │
├──────────────┼────────────────────────────────────────────────┤
│ Optional     │ 기본 비활성화 (관리자가 명시적 활성화 필요)        │
│              │ 해당: LiveInterception, Ransomware              │
│              │ 원리: 높은 비용/성능 영향 → 필요한 환경만 사용    │
└──────────────┴────────────────────────────────────────────────┘

비용 비교 예시 (50개 서비스, 일 기준):
  전략 없음 (10개 Agent 상시):  약 5만~15만원/일
  Tier 전략 적용:              약 5천~2만원/일 (70~90% 절감)

이유: 실제 이상 패턴은 전체 이벤트의 1~5%에 불과
      99%의 정상 트래픽은 로컬 모델로 처리 → Vertex AI 호출 없음
```

---

## 9. API 명세

### 9.1 인증 API

#### `POST /api/v1/auth/login`

**Authentication**: None

**Request Body**:
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "accessToken": "string (JWT, 15분)",
    "refreshToken": "string (7일)",
    "user": {
      "id": "string",
      "email": "string",
      "role": "admin | security_officer | viewer"
    }
  }
}
```

**Error Responses**:
| Status | Code | 설명 |
|--------|------|------|
| 401 | INVALID_CREDENTIALS | 이메일/비밀번호 불일치 |
| 403 | ACCOUNT_LOCKED | 5회 실패로 30분 잠금 |

---

#### `POST /api/v1/auth/refresh`

**Request Body**:
```json
{ "refreshToken": "string (required)" }
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": { "accessToken": "string (새 JWT)" }
}
```

---

### 9.2 이상 이벤트 API

#### `GET /api/v1/events`

**Authentication**: Required (JWT)

**Query Parameters**:
| 파라미터 | 필수 | 설명 |
|---------|------|------|
| severity | No | low / medium / high / critical |
| service | No | 서비스명 |
| status | No | open / investigating / resolved / false_positive |
| from | No | 시작 시각 (ISO 8601) |
| to | No | 종료 시각 (ISO 8601) |
| page | No | 페이지 번호 (default: 1) |
| limit | No | 페이지 크기 (default: 20, max: 100) |

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "events": [
      {
        "id": "evt_abc123",
        "severity": "critical",
        "type": "intrusion_detected",
        "service": "api-server",
        "title": "비정상 로그인 시도 감지",
        "description": "외부 IP 203.0.113.5에서 5분간 300회 로그인 시도",
        "detectedAt": "2026-03-03T14:30:00Z",
        "agentId": "agent_ids",
        "status": "open",
        "recommendation": "해당 IP 차단 및 계정 임시 잠금 권고",
        "isSuppressed": false,
        "reminderSentAt": null,
        "isLiveIntercepted": false,
        "thoughtSignature": {
          "signature": "a3f8c2d1e9b47f3c...",
          "signatureVersion": "v1",
          "confidence": 0.97,
          "reasoningSteps": [
            "로그인 실패 횟수: 500회/분 감지",
            "정상 임계값 10회/분 초과 (50배)",
            "패턴이 Dictionary Attack과 일치"
          ],
          "verificationUrl": "/api/v1/events/evt_abc123/verify-signature"
        }
      }
    ],
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalPages": 3,
      "total": 42
    }
  }
}
```

---

#### `POST /api/v1/events/{eventId}/acknowledge`

**Authentication**: Required (JWT, security_officer 이상)

**Request Body**:
```json
{
  "action": "acknowledged | investigating | resolved | false_positive",
  "memo": "string (optional)"
}
```

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "eventId": "evt_abc123",
    "status": "investigating",
    "acknowledgedBy": "user_001",
    "acknowledgedAt": "2026-03-03T14:35:00Z"
  }
}
```

---

### 9.3 서비스 관리 API

#### `GET /api/v1/services`

**Authentication**: Required (JWT)

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "services": [
      {
        "id": "svc_001",
        "name": "api-server",
        "status": "warning",
        "metrics": {
          "cpuUsage": 78.5,
          "memoryUsage": 65.2,
          "requestRate": 1200,
          "errorRate": 2.3
        },
        "activeEvents": 2,
        "lastCheckedAt": "2026-03-03T14:29:00Z"
      }
    ],
    "summary": {
      "total": 15,
      "healthy": 12,
      "warning": 2,
      "critical": 1
    }
  }
}
```

---

#### `POST /api/v1/services`

**Authentication**: Required (JWT, admin만)

**Request Body**:
```json
{
  "name": "string (required)",
  "prometheusUrl": "string (required) - http://localhost:9090",
  "lokiUrl": "string (required) - http://localhost:3100",
  "description": "string (optional)",
  "labels": { "env": "production", "team": "backend" }
}
```

---

### 9.4 이벤트 통계 API

#### `GET /api/v1/stats/events`

**Authentication**: Required (JWT)

**Query Parameters**:
| 파라미터 | 설명 |
|---------|------|
| period | 7d / 30d / 90d (default: 7d) |

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "period": "7d",
    "totalEvents": 142,
    "bySeverity": {
      "critical": 3,
      "high": 12,
      "medium": 45,
      "low": 82
    },
    "byType": {
      "intrusion_detected": 23,
      "ddos": 5,
      "iam_violation": 67,
      "dlp": 47
    },
    "falsePositiveRate": 0.03,
    "avgDetectionMinutes": 1.8,
    "trend": [
      { "date": "2026-02-25", "count": 18 },
      { "date": "2026-02-26", "count": 22 }
    ]
  }
}
```

---

### 9.5 Thought Signature 검증 API

#### `GET /api/v1/events/{eventId}/verify-signature`

**Description**: 보안 이벤트의 추론 증적(Thought Signature) 무결성 검증. 감사 목적 또는 법적 증거 제출 시 활용.

**Authentication**: Required (JWT, security_officer 이상)

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "eventId": "evt_abc123",
    "isValid": true,
    "verifiedAt": "2026-03-03T15:00:00Z",
    "signatureVersion": "v1",
    "thoughtPayload": {
      "agentId": "agent_ids",
      "timestamp": "2026-03-03T14:30:00.123Z",
      "confidence": 0.97,
      "reasoningSteps": [
        "로그인 실패 횟수: 500회/분 감지",
        "정상 임계값 10회/분 초과 (50배)",
        "발신 IP: 알려진 공격 IP 목록에 없음",
        "패턴이 Dictionary Attack과 일치 (신뢰도 0.97)"
      ],
      "severityJustification": "critical: 서비스 침해 임박"
    }
  }
}
```

**조작 감지 시 Response 200 OK** (isValid: false):
```json
{
  "success": true,
  "data": {
    "eventId": "evt_abc123",
    "isValid": false,
    "tamperedAt": "검증 불가 (서명 불일치)",
    "alert": "⚠️ 이 이벤트 기록이 변조되었습니다. 즉시 보안 감사를 시작하세요."
  }
}
```

---

### 9.6 Agent 상태 API

#### `GET /api/v1/agents/health`

**Authentication**: Required (JWT)

**Response 200 OK**:
```json
{
  "success": true,
  "data": {
    "agents": [
      {
        "id": "agent_ids",
        "type": "intrusion_detection",
        "status": "healthy",
        "lastHeartbeatAt": "2026-03-03T14:29:30Z",
        "restartCount": 0,
        "lastRestartAt": null
      },
      {
        "id": "agent_ddos",
        "type": "ddos",
        "status": "restarted",
        "lastHeartbeatAt": "2026-03-03T14:25:00Z",
        "restartCount": 1,
        "lastRestartAt": "2026-03-03T14:25:10Z",
        "restartReason": "heartbeat_timeout"
      }
    ]
  }
}
```

---

### 9.7 알림 설정 API

#### `PUT /api/v1/notifications/settings`

**Authentication**: Required (JWT, admin만)

**Request Body**:
```json
{
  "slack": {
    "enabled": true,
    "webhookSecretName": "slack-webhook-prod",
    "channels": {
      "critical": "#security-emergency",
      "high": "#security-alerts",
      "medium": "#security-monitor"
    }
  },
  "email": {
    "enabled": true,
    "recipients": {
      "critical": ["ciso@company.com"],
      "high": ["admin@company.com"],
      "weekly_report": ["cto@company.com", "admin@company.com"]
    }
  },
  "deduplication": {
    "suppressionMinutes": 30,
    "reminderIntervalMinutes": 30
  }
}
```

---

## 10. 데이터베이스 스키마

```sql
-- 관리자 계정
CREATE TABLE users (
  id            VARCHAR(50) PRIMARY KEY,
  email         VARCHAR(200) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(20) NOT NULL DEFAULT 'viewer',  -- admin / security_officer / viewer
  failed_login_count INTEGER DEFAULT 0,
  locked_until  TIMESTAMP WITH TIME ZONE,
  last_login_at TIMESTAMP WITH TIME ZONE,
  is_active     BOOLEAN DEFAULT true,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 등록된 서비스 목록
CREATE TABLE services (
  id              VARCHAR(50) PRIMARY KEY,
  name            VARCHAR(100) NOT NULL,
  description     TEXT,
  prometheus_url  VARCHAR(500) NOT NULL,
  loki_url        VARCHAR(500) NOT NULL,
  labels          JSONB DEFAULT '{}',
  is_active       BOOLEAN DEFAULT true,
  created_by      VARCHAR(50) REFERENCES users(id),
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 보안 이벤트 기록
CREATE TABLE security_events (
  id                  VARCHAR(50) PRIMARY KEY,
  service_id          VARCHAR(50) REFERENCES services(id),
  severity            VARCHAR(20) NOT NULL,   -- low / medium / high / critical
  event_type          VARCHAR(50) NOT NULL,   -- intrusion / ddos / iam_violation / dlp / live_interception
  title               VARCHAR(200) NOT NULL,
  description         TEXT NOT NULL,
  recommendation      TEXT,
  agent_id            VARCHAR(50),
  raw_data            JSONB,
  status              VARCHAR(20) DEFAULT 'open',

  -- ✅ Thought Signature (추론 증적) — WIGTN Shield 핵심 차별점
  -- AI가 판단한 시점의 추론 상태를 암호화 서명으로 봉인
  -- 변조 시 thought_signature 검증 실패 → 감사 증거로 활용 가능
  thought_payload     TEXT,           -- JSON 직렬화 후 AES-256 암호화 저장
                                      -- (inputData, reasoningSteps, confidence, severityJustification)
  thought_signature   VARCHAR(64),    -- HMAC-SHA256(thought_payload, SERVER_SECRET_KEY) hex
  signature_version   VARCHAR(10) DEFAULT 'v1', -- 서명 알고리즘 버전 (추후 업그레이드 대비)

  -- 알림 중복 억제 관련
  is_suppressed       BOOLEAN DEFAULT false,
  suppressed_until    TIMESTAMP WITH TIME ZONE,
  last_reminder_at    TIMESTAMP WITH TIME ZONE,

  -- 자동 차단 관련 (P2)
  auto_blocked        BOOLEAN DEFAULT false,
  auto_blocked_at     TIMESTAMP WITH TIME ZONE,

  -- Live Interception 관련
  is_live_intercepted BOOLEAN DEFAULT false,   -- Live API로 감지된 이벤트 여부
  live_stream_type    VARCHAR(20),             -- screen / audio / terminal / text

  detected_at         TIMESTAMP WITH TIME ZONE NOT NULL,
  resolved_at         TIMESTAMP WITH TIME ZONE
);

-- 이벤트 처리 이력
CREATE TABLE event_actions (
  id          VARCHAR(50) PRIMARY KEY,
  event_id    VARCHAR(50) REFERENCES security_events(id),
  user_id     VARCHAR(50) REFERENCES users(id),
  action      VARCHAR(50) NOT NULL,
  memo        TEXT,
  created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 발송된 알림 이력
CREATE TABLE notification_logs (
  id              VARCHAR(50) PRIMARY KEY,
  event_id        VARCHAR(50) REFERENCES security_events(id),
  channel         VARCHAR(20) NOT NULL,   -- slack / email / sms
  notification_type VARCHAR(20) DEFAULT 'alert',  -- alert / reminder / agent_restart
  recipient       VARCHAR(200) NOT NULL,
  is_sent         BOOLEAN DEFAULT false,
  sent_at         TIMESTAMP WITH TIME ZONE,
  error_message   TEXT
);

-- Agent 상태 이력
CREATE TABLE agent_health_logs (
  id              VARCHAR(50) PRIMARY KEY,
  agent_id        VARCHAR(50) NOT NULL,
  event_type      VARCHAR(30) NOT NULL,   -- heartbeat / restart / zombie_detected
  detail          TEXT,
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 11. 오픈소스 설치 가이드 구조

### 11.1 사전 요구사항

```
필수:
  ✅ Node.js 18+
  ✅ Google Cloud 계정 + Vertex AI API 활성화
  ✅ gcloud CLI 설치 및 인증 (gcloud auth application-default login)

사내 인프라 (하나 이상):
  ✅ Prometheus (메트릭 수집)
  ✅ Loki (로그 수집)

선택:
  ⬜ Grafana (시각화 대시보드, 별도 구성)
  ⬜ pm2 또는 systemd (운영 환경 프로세스 관리)
```

### 11.2 빠른 시작 (Quick Start)

```bash
# 1. 설치
npm install -g wigtn-shield

# 2. 설정 파일 생성 (SERVER_SECRET_KEY 자동 생성 + .env.example 생성)
wigtn-shield init
# → config.yaml 생성 + 설정 안내
# → .gitignore에 config.yaml, .env 자동 추가

# 3. GCP 인증
gcloud auth application-default login

# 4. 환경변수 설정 (.env 파일 편집)
# WIGTN_SLACK_WEBHOOK, WIGTN_ADMIN_PASSWORD 등 입력

# 5. 실행
wigtn-shield start
# → http://localhost:3000 에서 Web UI 접속

# 6. 첫 로그인
# Email: config.yaml의 adminEmail
# Password: .env의 WIGTN_ADMIN_PASSWORD (변경 권장)
```

**CLI 전체 명령어 목록**:

| 명령어 | 설명 |
|--------|------|
| `wigtn-shield init` | 초기화: config.yaml 생성, SERVER_SECRET_KEY 생성 |
| `wigtn-shield start` | 모든 서비스 시작 (Web UI + Agent + Stream Processor) |
| `wigtn-shield stop` | 서비스 중지 |
| `wigtn-shield status` | 각 Agent 상태 및 Baseline 진행률 확인 |
| `wigtn-shield verify --event-id <id>` | Thought Signature 무결성 검증 |
| `wigtn-shield rotate-key` | SERVER_SECRET_KEY 정기 로테이션 |
| `wigtn-shield rotate-key --emergency` | 키 유출 시 즉시 교체 + 긴급 알림 |
| `wigtn-shield migrate-config` | 이전 버전 config.yaml을 현재 스키마로 자동 마이그레이션 |
| `wigtn-shield agent lateral-movement --mode=active` | Lateral Movement Agent 수동 전환 |

### 11.3 운영 환경 설정 (pm2)

```bash
# pm2로 1차 방어 (프로세스 자동 재시작)
npm install -g pm2
pm2 start wigtn-shield --name "wigtn-shield"
pm2 startup   # 서버 재부팅 시 자동 시작
pm2 save
```

### 11.4 서비스 연동 가이드

각 언어/프레임워크별 Prometheus + Loki 연동 방법 (docs/ 디렉토리에 제공):

| 언어/프레임워크 | Prometheus | Loki |
|--------------|-----------|------|
| Spring Boot | Actuator + Micrometer | Logback + Loki4j |
| Node.js/Express | prom-client | winston + loki-winston |
| Python/FastAPI | prometheus-fastapi-instrumentator | python-logging-loki |
| Go | prometheus/client_golang | grafana/loki |
| 범용 (사이드카) | Node Exporter | Fluent Bit |

### 11.5 npm 패키지 보안 정책 (C-6 반영)

wigtn-shield 자체가 npm 공개 패키지이므로, 패키지 공급망 공격에 대한 방어가 필요합니다.

```
오픈소스 패키지 보안 대책:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Sigstore 패키지 서명 (npm provenance)
   → GitHub Actions에서만 npm publish 실행
   → npm publish --provenance
   → 사용자가 설치 시 서명 검증 가능

2. 2FA npm 배포 계정
   → npm 계정에 2FA 활성화 필수
   → publish 토큰은 GitHub Secrets에만 저장

3. 의존성 최소화 원칙
   → production 의존성은 반드시 필요한 것만 포함
   → devDependencies는 배포 패키지에서 제외

4. 주기적 취약점 점검
   → GitHub Dependabot으로 자동 PR
   → 릴리스 전 npm audit 통과 필수

5. WIGTN Shield 자체 공급망 감시
   → Supply Chain Agent(FR-025)가 자기 자신의
     node_modules도 감시 대상에 포함
   → "의사가 자신을 치료하는" 자기 방어 구조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 12. Canvas 대시보드 UI/UX 설계

> **핵심 원칙**: 정적인 숫자 화면이 아닌, 각 Agent의 페르소나를 가진 살아있는 캐릭터들이 실시간으로 자신의 활동을 보여주는 인터랙티브 캔버스 대시보드

### 12.1 설계 철학

```
기존 보안 대시보드:
  [숫자 그래프] [로그 표] [경고 횟수] → 전문가만 이해 가능

WIGTN Shield Canvas:
  [캐릭터들이 각자 역할 수행 중] → 누구나 직관적으로 파악
  "경비원이 뛰어가고 있다" = 이상 탐지 진행 중
  "모든 캐릭터가 한가롭다" = 정상 상태
```

### 12.2 Agent 캐릭터 설계

각 Agent는 자신의 역할을 상징하는 시각적 캐릭터를 가집니다.

| Agent | 캐릭터 | 정상 상태 | 탐지 중 | Critical |
|-------|--------|----------|--------|---------|
| **오케스트레이터** | 팀장 지휘관 (망원경+지도) | 조용히 지도 관찰 | 무전기 들고 지시 | 비상벨 울리며 뛰어다님 |
| **IDS (침입탐지)** | 경비원 (배지+손전등) | 순찰 중 (걷기) | 손전등 비추며 검문 | 비상버튼 누르며 달려감 |
| **DDoS** | 교통경찰 (호루라기+봉) | 교통정리 중 | 트래픽 급증 표지 들고 정리 | 도로 봉쇄 바리케이드 설치 |
| **IAM (접근권한)** | 출입관리자 (배지판독기) | 배지 확인 중 | 비정상 배지 거부 중 | 경보 + 출입문 잠금 |
| **DLP (데이터유출)** | 세관 검사관 (검사대+스캐너) | 반출물 검사 중 | X-ray 이상 감지 | 긴급 압류 조치 |
| **Injection** | 문서 위조 감지원 (돋보기) | 서류 검토 중 | 위조 패턴 분석 | 위조 서류 압수 |
| **Supply Chain** | 납품 검수원 (체크리스트) | 재고 확인 중 | 불량품 의심 검사 | 위험 납품 반송 조치 |
| **Lateral Movement** | 내부 순찰 경비원 (지도) | 내부 순찰 중 | 비정상 이동 추적 | 침입자 추격 |
| **Secret Leakage** | 기밀문서 관리관 (금고) | 금고 점검 중 | 문서 유출 감지 | 긴급 봉인 조치 |
| **Misconfiguration** | 안전 점검관 (클립보드) | 시설 점검 중 | 위반 항목 표시 | 위험 구역 폐쇄 조치 |
| **Ransomware** | 소방관 (소화기) | 대기 중 | 이상 연기 감지 | 비상 진화 출동 |
| **Live Interception** | CCTV 운영요원 (모니터) | 화면 모니터링 중 | 이상 화면 확대 | 세션 즉시 중단 |

### 12.3 Canvas 레이아웃

```
┌─────────────────────────────────────────────────────────────┐
│  WIGTN Shield Dashboard                    🔴 Critical: 2   │
│  [전체 상태: ⚠️ 경고]   서비스: 15/15   [날짜: 2026-03-03]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Canvas 메인 뷰                       │   │
│  │                                                      │   │
│  │  🏢 건물 배경 (사내 서버팜 이미지)                     │   │
│  │                                                      │   │
│  │  [🔭팀장]    [🔦경비원]    [🚦교통경찰]                │   │
│  │   정상        이상탐지!     정상                       │   │
│  │                                                      │   │
│  │  [🎫출입관리] [📦세관]    [🔍문서감지원]               │   │
│  │   정상         정상         조사중...                  │   │
│  │                                                      │   │
│  │  [📋납품검수] [🗺️순찰]   [🔒기밀관리]                 │   │
│  │   정상         정상         정상                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  [사이드 패널]                    [이벤트 스트림]             │
│  ┌─────────────────────┐         ┌────────────────────────┐ │
│  │ Agent: IDS          │         │ 14:30 🔴 침입 탐지!    │ │
│  │ 상태: 이상 탐지 중  │         │ 14:28 🟡 IAM 경고      │ │
│  │ 이벤트: 500회/분    │         │ 14:25 🟢 정상 복귀     │ │
│  │ 신뢰도: 0.97        │         └────────────────────────┘ │
│  └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
```

### 12.4 실시간 애니메이션 상태

캐릭터들은 현재 상태에 따라 4가지 애니메이션 상태를 가집니다:

```
상태           애니메이션              색상 테마
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
idle           천천히 좌우 흔들기       초록 (🟢 정상)
working        분주하게 움직임          파랑 (🔵 분석 중)
alert          빠르게 달리거나 손짓     노랑 (🟡 경고)
critical       격렬한 움직임 + 깜빡임  빨강 (🔴 위험)
offline        회색 + 수면 아이콘       회색 (⚪ 비활성)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 12.5 인터랙션 명세

| 인터랙션 | 동작 |
|---------|------|
| 캐릭터 클릭 | 해당 Agent 상세 패널 슬라이드인 (추론 단계, 최근 이벤트, 설정) |
| 캐릭터 호버 | 툴팁: "현재 무엇을 하고 있는가" 한 줄 설명 |
| Critical 이벤트 발생 | 해당 캐릭터로 카메라 포커스 이동 + 화면 테두리 빨강 플래시 |
| Baseline 학습 중 | Lateral Movement 캐릭터 위에 📚 학습 진행률 표시 |
| Agent 재시작 | 캐릭터가 쓰러졌다가 일어나는 애니메이션 |
| 이벤트 해결(resolve) | 해당 캐릭터가 OK 사인 + 초록 이펙트 |

### 12.6 비전문가 접근성 원칙

```
✅ 모든 상태는 텍스트 없이도 캐릭터 모션만으로 이해 가능
✅ 심각도는 색상 + 크기 + 애니메이션 속도로 직관적 표현
✅ 중요 이벤트는 캐릭터가 화면 중앙으로 이동하여 강조
✅ 툴팁은 기술 용어 없이 "지금 일어나는 일"을 평어로 설명
   예: "Brute Force Attack" → "누군가 비밀번호를 계속 시도하고 있어요"
✅ 대시보드 상단에 전체 상태 요약 (정상/주의/위험 3단계만)
✅ 임원 모드: 기술 패널 숨기고 요약만 표시 (role: viewer 기본)
```

### 12.7 기술 구현 스펙

```
Canvas 라이브러리: Konva.js (React 호환, 2D Canvas)
캐릭터 에셋:      SVG 기반 (해상도 독립, 경량)
애니메이션:       Konva Tween + Framer Motion (React 레이어)
실시간 업데이트:  WebSocket (/ws/agents) — Agent 상태 변경 시 push
렌더링 최적화:    캐릭터별 dirty flag — 변경된 캐릭터만 재렌더링
반응형:           모바일에서는 그리드 레이아웃으로 자동 전환
다크 모드:        보안 환경 특성상 다크 모드 기본값
```

### 12.8 WebSocket API (Canvas 실시간 연동)

#### `WS /ws/agents`

**Authentication**: JWT (Upgrade 요청 시 쿼리 파라미터: `?token=<jwt>`)

**서버 → 클라이언트 메시지 형식**:
```json
{
  "type": "agent_state_update",
  "data": {
    "agentId": "agent_ids",
    "state": "alert",
    "message": "500회/분 로그인 시도 감지 중",
    "severity": "high",
    "confidence": 0.92,
    "timestamp": "2026-03-03T14:30:00Z"
  }
}
```

**이벤트 타입**:
| type | 설명 |
|------|------|
| `agent_state_update` | Agent 상태 변경 (idle/working/alert/critical/offline) |
| `new_event` | 새 보안 이벤트 발생 |
| `event_resolved` | 이벤트 해결 |
| `agent_restarted` | Agent 재시작 발생 |
| `baseline_progress` | Lateral Movement Baseline 학습 진행률 |

---

## 13. 구현 단계

### Phase 1: 기반 구축 (MVP) — 4주

```
목표: npm install 후 핵심 보안 모니터링 동작

Week 1-2: 핵심 인프라
  ✅ npm CLI 패키지 구조 설계
     → wigtn-shield init / start / stop / status / verify / rotate-key / migrate-config
  ✅ config.yaml 파서 + 환경변수 ${VAR} 치환 구현 (C-1)
  ✅ SERVER_SECRET_KEY 자동 생성 + OS 키체인 저장 (C-2)
  ✅ Prometheus HTTP API 직접 연동
  ✅ Loki HTTP API 직접 연동
  ✅ Stream Processor 구현 (실시간 수신)
  ✅ SQLite 기반 DB 초기화

Week 3-4: Tier 1 Agent + 인증 + 알림
  ✅ 오케스트레이터 Agent (LangGraph + Vertex AI gcloud auth)
  ✅ FR-006 침입 탐지 Agent (Tier 1) + FR-022 Thought Signature
  ✅ FR-007 DDoS 탐지 Agent (Tier 1)
  ✅ FR-024 인젝션 공격 탐지 Agent (Tier 1, P0)
  ✅ FR-027 시크릿 노출 탐지 Agent (Tier 1)
  ✅ Web UI + JWT 로그인 (httpOnly 쿠키, m-3) + Canvas 대시보드 기본 (FR-030)
  ✅ Slack 알림 + 중복 억제 정책

산출물: npm install 후 Tier 1 Agent 4개 + Thought Signature 동작 확인
```

### Phase 2: Agent 확장 + 안정화 — 4주

```
목표: Tier 2 Agent + Canvas UI 완성 + 3단계 장애 방어

Week 5-6:
  ✅ FR-008 IAM 접근 권한 Agent (Tier 2) 구현
  ✅ FR-009 DLP 데이터 유출 Agent (Tier 2) + DLP/Secret Leakage 이벤트 병합 (M-2)
  ✅ FR-026 횡적 이동 탐지 Agent (Tier 2) + Baseline 학습 모드 (14일)
  ✅ Agent Heartbeat + 자동 재시작 구현 (FR-010)
  ✅ 재시작 보고서 알림

Week 7-8:
  ✅ FR-025 공급망 공격 탐지 Agent (Tier 3) + NVD API 캐싱 (M-3)
  ✅ Canvas 대시보드 캐릭터 애니메이션 완성 (FR-030)
  ✅ 이메일 알림 연동
  ✅ 이벤트 통계 API + 대시보드 차트
  ✅ 서비스 등록/관리 UI

산출물: Tier 1+2 Agent 동작, Canvas UI 비전문가 친화적 대시보드, Agent 자가 복구 확인
```

### Phase 3: 고도화 + npm 배포 — 4주

```
목표: P2 Agent 완성, 리포트 자동화, npm 공개

Week 9-10:
  ✅ FR-028 설정 오류 탐지 Agent (Tier 3, P2) 구현
  ✅ FR-029 랜섬웨어 지표 탐지 Agent (Optional, P2) 구현
  ✅ FR-023 Live Interception Agent (Optional, P1) 구현
  ✅ 일일/주간 보안 리포트 자동 생성 + 이메일
  ✅ 서비스 연동 가이드 문서 작성
  ✅ PostgreSQL 지원 (선택 옵션)
  ✅ migrate-config 명령어 구현 (M-7)

Week 11-12:
  ✅ 성능 최적화 및 부하 테스트
  ✅ 보안 취약점 점검 (npm audit, Sigstore provenance 설정 — C-6)
  ✅ npm publish --provenance (wigtn-shield)
  ✅ README, CONTRIBUTING, LICENSE 작성
  ✅ [P2] 자동 차단 기능 구현 (config autoBlock)

산출물: npm에 공개된 패키지 (Sigstore 서명 포함), FR-001~FR-030 모두 완성
```

---

## 14. 성공 지표

| 지표 | 현재 | 목표 | 측정 방법 |
|------|------|------|---------|
| 위협 탐지율 | 수동 (불완전) | 알려진 공격 패턴 90% 탐지 | 침투 테스트로 검증 |
| 평균 탐지 시간 | 수 시간 (수동) | **2분 이내** (스트리밍) | 이벤트 로그 분석 |
| 평균 알림 시간 | 없음 | 탐지 후 30초 이내 | 알림 발송 시각 기록 |
| 오탐률 | N/A | 5% 이하 | 월별 이벤트 검토 |
| 관리자 대응 시간 | 평균 2시간 | 30분 이내 | 이벤트 처리 이력 |
| 모니터링 커버리지 | 일부 수동 | 사내 서비스 100% | 연동 서비스 수 |
| npm 주간 다운로드 | 0 | 출시 후 3개월 내 100+ | npm stats |
| Agent 가동률 | N/A | 99.9% (3단계 방어) | Heartbeat 로그 |
| Thought Signature 커버리지 | N/A | 이벤트 100% 서명 | DB 서명 완결성 검증 |
| Supply Chain 스캔 커버리지 | N/A | 등록 서비스 100% | 스캔 이력 확인 |
| Tier 1 Vertex AI 호출율 | N/A | 전체 이벤트의 5% 이하 | 비용 모니터링 |
| Canvas UI 비전문가 이해도 | N/A | 사용자 테스트 80% 만족 | 사용자 인터뷰 |

---

## 부록 A: 용어 설명

| 용어 | 설명 |
|------|------|
| **Prometheus** | 서버의 숫자 데이터(CPU, 메모리 등)를 수집하는 오픈소스 도구 |
| **Loki** | 서버 로그(텍스트 기록)를 모아서 검색 가능하게 만드는 도구 |
| **Grafana** | 수집된 데이터를 그래프로 시각화하는 도구 (wigtn-shield와 별도) |
| **Stream Processor** | 로그/메트릭을 실시간으로 받아 AI에 전달하는 파이프라인 |
| **Agent** | 특정 역할을 자율적으로 수행하는 AI 프로그램 |
| **Vertex AI** | Google Cloud의 AI/ML 서비스 플랫폼 |
| **LangGraph** | 여러 AI Agent를 조율하는 프레임워크 |
| **gcloud auth** | Google Cloud SDK 인증 도구 |
| **Heartbeat** | Agent가 주기적으로 "나 살아있어요" 신호를 보내는 것 |
| **Alert Fatigue** | 알림이 너무 많아 담당자가 무감각해지는 현상 |
| **Suppression** | 동일 이벤트 알림을 일정 시간 억제하는 것 |
| **DDoS** | 서버에 엄청난 양의 가짜 요청을 보내 서비스를 마비시키는 공격 |
| **IAM** | Identity and Access Management, 접근 권한 관리 |
| **DLP** | Data Loss Prevention, 데이터 유출 방지 |
| **False Positive** | 정상인데 비정상으로 잘못 감지하는 경우 |

---

## 부록 B: digging 반영 내역

| 이슈 | 선택 | 반영 내용 |
|------|------|---------|
| C-1 관리자 인증 없음 | Web UI + JWT | users 테이블, 인증 API, 역할 매트릭스 추가 |
| C-2 아키텍처 오류 | Prometheus + Loki API 직접 | 아키텍처 다이어그램 수정 |
| C-3 차단 정책 충돌 | P2 자동 차단 (config on/off) | Phase 3에 P2 기능으로 명시 |
| C-4 Agent 장애 대응 없음 | systemd/pm2 + Heartbeat 혼합 | 3단계 방어 구조 섹션 추가 |
| C-5 연동 가이드 없음 | 언어별 가이드 문서 | 섹션 11.4 추가 |
| C-6 알림 중복 없음 | 자동 억제 + 30분 리마인더 | FR-012, 알림 설정 API 수정 |
| M-1 탐지 시간 모순 | 스트리밍 도입 (2분 목표 유지) | Stream Processor 추가, 비용 최적화 전략 |
| M-2 역할 권한 없음 | - | 6.3 역할별 권한 매트릭스 추가 |
| M-3 Vertex AI 비용 | Vertex AI 단일 (비용 최적화) | 경량 모델 1차 필터링 전략 추가 |
| M-4 네트워크 연결 | gcloud auth | config.yaml에 gcloud auth 방식 명시 |
| M-7 페이지네이션 | - | pagination 객체 추가 (page, pageSize, totalPages, total) |
| 오픈소스 npm | CLI 도구 | 전체 아키텍처 CLI 기반으로 재설계 |
| **Thought Signature 누락** | FR-022 (P0) | 4.7절, DB 스키마, API 명세, 검증 API 추가 |
| **Live Interception 누락** | FR-023 (P1) | 4.8절, 아키텍처 다이어그램, config 구조 추가 |
| **인젝션 공격 탐지 누락** | FR-024 (P0) | 4.9절, Injection Agent 추가 |
| **공급망 공격 탐지 누락** | FR-025 (P1) | 4.10절, Supply Chain Agent 추가 |
| **횡적 이동 탐지 누락** | FR-026 (P1) | 4.11절, Lateral Movement Agent 추가 |
| **시크릿 노출 탐지 누락** | FR-027 (P1) | 4.12절, Secret Leakage Agent 추가 |
| **설정 오류 탐지 누락** | FR-028 (P2) | 4.13절, Misconfiguration Agent 섹션 추가 |
| **랜섬웨어 지표 탐지 누락** | FR-029 (P2) | 4.14절, Ransomware Agent 섹션 추가 |

**digging v2 반영 내역 (v3.0)**:

| 이슈 | 반영 내용 |
|------|---------|
| C-1 config.yaml 평문 시크릿 | config.yaml 전면 개정, 모든 시크릿 ${ENV_VAR}로 교체 |
| C-2 SERVER_SECRET_KEY 미정의 | 4.7절에 키 생성/저장/로테이션/긴급교체 전체 정책 추가 |
| C-3 Phase 1-3 누락 FR | Phase 1에 FR-022/024/027, Phase 2에 FR-025/026, Phase 3에 FR-023/028/029 배치 |
| C-4 Agent 비용 폭증 | 8.5절 Agent Tier 비용 전략 섹션 추가 + config.yaml Tier 구조화 |
| C-5 FR-028/029 에이전트 스펙 없음 | 4.13절(Misconfiguration), 4.14절(Ransomware) 상세 스펙 추가 |
| C-6 npm 패키지 보안 정책 없음 | 11.5절 npm 패키지 보안 정책 섹션 추가 (Sigstore, 2FA, Dependabot) |
| M-1 Lateral Movement 초기 오탐 | baselineDays 학습 모드 + Web UI 진행률 표시 추가 |
| M-2 DLP vs Secret Leakage 경계 | 4.5절에 역할 경계 명확화 + ±5초 이벤트 병합 정책 추가 |
| M-3 Supply Chain NVD 안정성 | NVD API Key 설정, SQLite 24h 캐싱, offlineMode 추가 |
| M-4 Injection Agent WAF 혼동 | 4.9절에 WAF 역할 경계 명확화 주석 추가 |
| M-5 Thought Signature Live 범위 | 4.7절에 Live Interception 비동기 서명(200ms) 명시 |
| M-6 성공 지표 부족 | 14절에 Thought Signature 커버리지, Supply Chain 스캔율, Tier 비용율 KPI 추가 |
| M-7 config migrate 없음 | 11.2절에 migrate-config CLI 명령어 추가 |
| M-8 개인정보/GDPR 비기능 누락 | 6.5절 개인정보 보호 NFR 섹션 추가 |
| m-1 Live Interception 범위 모호 | config.yaml interventionMode 3단계 옵션 명시 |
| m-2 사용자 스토리 부족 | (Phase 2에서 사용자 테스트 항목으로 반영) |
| m-3 JWT 쿠키 보안 속성 | 6.2절 NFR에 httpOnly + SameSite=strict + Secure 조건 추가 |
| **UI Canvas 대시보드 요구사항** | FR-030 추가, 12절 Canvas 대시보드 UI/UX 설계 섹션 신규 추가 |
