# PRD Analysis Report - WIGTN Shield

## 분석 대상
- **문서**: `docs/prd/wigtn-shield-prd.md`
- **버전**: 1.0
- **분석일**: 2026-03-03

---

## 요약

| 카테고리 | 발견 | Critical | Major | Minor |
|----------|------|----------|-------|-------|
| 완전성 | 6 | 3 | 2 | 1 |
| 기술 실현가능성 | 4 | 2 | 2 | 0 |
| 보안 & 리스크 | 5 | 2 | 2 | 1 |
| 일관성 & 명확성 | 5 | 1 | 3 | 1 |
| **총계** | **20** | **8** | **9** | **3** |

---

## 상세 분석

---

### 🔴 Critical (즉시 수정 필요)

---

#### C-1. 관리자 인증 시스템이 완전히 없음
- **위치**: 전체 (특히 Section 6.2 보안, Section 9)
- **문제**: MFA 언급은 있지만 관리자가 어떻게 가입/로그인하는지, JWT 토큰 만료 시간이 얼마인지, 비밀번호 정책이 무엇인지 전혀 정의되지 않음. `event_actions` 테이블에서 `user_id`를 참조하지만 `users` 테이블 자체가 DB 스키마에 없음.
- **영향**: 구현 시 기반이 되는 인증 레이어가 아예 빠져 있어 모든 API를 설계할 때 기준이 없음. 보안 시스템의 접근 제어가 불완전해짐.
- **개선안**:
  ```sql
  CREATE TABLE users (
    id            VARCHAR(50) PRIMARY KEY,
    email         VARCHAR(200) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(20) NOT NULL,  -- admin / security_officer / viewer
    mfa_secret    VARCHAR(100),
    is_active     BOOLEAN DEFAULT true,
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at    TIMESTAMP WITH TIME ZONE DEFAULT NOW()
  );
  ```
  추가 API: `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`
  JWT 정책: Access Token 15분, Refresh Token 7일, 5회 실패 시 30분 잠금

---

#### C-2. 아키텍처 오류 - API 서버가 "Grafana DB"에서 조회
- **위치**: Section 8.2 시스템 아키텍처
- **문제**: 다이어그램에서 API 서버가 "Grafana DB"에서 데이터를 가져오는 것으로 표현되어 있음. Grafana는 시각화 도구이지 데이터 소스가 아님. API 서버는 **Prometheus HTTP API**와 **Loki HTTP API**를 직접 호출해야 함.
- **영향**: 이 다이어그램을 기반으로 구현하면 실제로 동작하지 않는 아키텍처가 만들어짐.
- **개선안**:
  ```
  [WIGTN Shield API Server]
       │
       ├── Prometheus API (메트릭 조회) → http://prometheus:9090/api/v1/query
       └── Loki API (로그 조회)        → http://loki:3100/loki/api/v1/query
  ```
  Grafana는 별도의 사람이 보는 시각화 레이어로 분리. API 서버와 직접 연결하지 않음.

---

#### C-3. "즉시 차단"이 불가능한 구조인데 명시됨
- **위치**: Section 4.4 (IAM Agent), Section 4.5 (DLP Agent), Section 1.4 (범위)
- **문제**: IAM Agent 설명에 "즉시 접근 차단", DLP Agent 설명에 "즉시 차단"이라고 명시되어 있음. 그러나 Section 1.4 범위에서 **"방화벽 자동 제어"는 제외**라고 선언함. 두 내용이 정면으로 충돌함.
- **영향**: 구현 팀이 "차단 기능을 만들어야 하는가?"에 대해 혼란을 겪게 되고, 잘못 구현될 경우 운영 사고로 이어질 수 있음 (자동 차단으로 정상 사용자 접근이 끊길 위험).
- **개선안**: 다음 중 하나를 선택:
  - **선택 A**: "차단 권고"로 표현 수정 → Agent는 차단을 추천하고, 관리자가 버튼으로 수동 실행
  - **선택 B**: "자동 차단"을 P2 기능으로 명시적으로 추가하고 구현 방법(Cloud Armor, iptables API 등) 정의
  - **현재 권장**: 선택 A (MVP 범위에 적합)

---

#### C-4. Agent 자체 장애 대응 방안 없음
- **위치**: 전체 (언급 없음)
- **문제**: AI Agent들이 다운되거나 오동작할 경우 누가 감시하는지, 어떻게 복구하는지 전혀 정의되지 않음. "모니터링하는 것이 다운되는" 상황은 보안 시스템에서 가장 위험한 케이스임.
- **영향**: Agent 장애 시 보안 이벤트를 아무도 감지하지 못하는 사각지대가 발생. 공격자가 이를 악용할 수 있음.
- **개선안**:
  ```
  FR-014 (P0): Agent 헬스체크 시스템
    - 각 Agent는 30초마다 heartbeat 신호 발송
    - 오케스트레이터가 2분 이상 heartbeat 없는 Agent 감지 시 즉시 알림
    - Agent 재시작 정책 (Cloud Run 자동 재시작 활용)
  ```

---

#### C-5. 서비스 연동 방법 완전 누락
- **위치**: Section 11 구현 단계 Week 1-2
- **문제**: "시범 서비스 2개 연동"이라고만 되어 있고, 실제로 사내 서비스에 어떻게 Prometheus exporter를 설치하고 Loki에 로그를 보내도록 설정하는지 가이드가 없음. 특히 다양한 기술 스택(Java/Spring, Node.js, Python 등)에 대한 연동 방법이 다름.
- **영향**: 각 팀이 자기 서비스를 직접 연동해야 하는 상황에서 어떻게 해야 하는지 몰라 지연 발생.
- **개선안**: 별도 Section으로 "서비스 연동 가이드" 추가:
  - Prometheus Node Exporter (서버 메트릭)
  - 언어별 Application Exporter 설정 (Spring Actuator, Node.js prom-client 등)
  - Fluent Bit 로그 수집 에이전트 설정 (사이드카 패턴)

---

#### C-6. 알림 중복/폭주 방지 정책 없음
- **위치**: Section 5 기능 요구사항, Section 9.4
- **문제**: 동일한 이벤트(예: DDoS 공격)가 진행 중일 때 매 2분마다 알림이 반복 발송될 수 있음. 1시간 공격이면 30번의 Slack 메시지가 쏟아짐 → 알림 피로(Alert Fatigue)로 담당자가 무시하기 시작함.
- **영향**: 보안 알림 시스템의 실효성이 떨어지고 정작 중요한 알림을 놓칠 수 있음.
- **개선안**:
  ```
  알림 정책 추가:
  - 동일 이벤트 유형/서비스 조합: 최초 1회 알림 후 30분간 suppression
  - 미해결 이벤트: 1시간마다 reminder 알림 (새 알림이 아닌 업데이트)
  - 알림 설정 API에 `suppressionMinutes` 파라미터 추가
  ```

---

### 🟡 Major (구현 전 수정 권장)

---

#### M-1. 탐지 시간 목표(2분)와 실제 파이프라인(최소 3분) 간 모순
- **위치**: Section 6.1 성능, Section 8.3 데이터 흐름
- **문제**: 데이터 흐름에 따르면 수집 1분 + 분석 2분 = 최소 3분이 소요됨. 그런데 성능 목표는 "이상 발생 후 2분 이내 탐지"임. 수학적으로 불가능한 목표.
- **개선안**: 두 가지 접근 중 선택:
  - **A**: 성능 목표를 현실적으로 수정 → "이상 발생 후 5분 이내 탐지"
  - **B**: 스트리밍 아키텍처 도입 → Kafka/Pub-Sub으로 실시간 처리 (구현 복잡도 크게 증가)
  - 권장: MVP는 A, Phase 3에서 B 검토

---

#### M-2. 사용자 역할(Role) 권한 범위 미정의
- **위치**: Section 7 사용자 스토리, Section 6.2 보안
- **문제**: "관리자", "보안 담당자", "시스템 관리자", "임원"이 등장하지만 각 역할이 어떤 API를 호출할 수 있는지, 어떤 화면을 볼 수 있는지 정의되지 않음.
- **개선안**:

  | 역할 | 이벤트 조회 | 이벤트 처리 | 설정 변경 | 리포트 |
  |------|-----------|-----------|---------|--------|
  | admin | 전체 | 가능 | 가능 | 가능 |
  | security_officer | 전체 | 가능 | 불가 | 가능 |
  | viewer | 전체 | 불가 | 불가 | 가능 |

---

#### M-3. Vertex AI 비용 추정 없음
- **위치**: Section 8.1 기술 스택, Section 11 구현 단계
- **문제**: 4개 Agent + 오케스트레이터가 2분마다 Gemini API를 호출하고, 50개 서비스의 로그를 분석하면 월 API 비용이 매우 클 수 있음. 비용 추정이나 최적화 방안이 없음.
- **예시 계산**:
  - 분석 주기: 30회/시간 × 24시간 × 30일 = 21,600 호출/월
  - 5개 Agent × 21,600 = 108,000 Gemini 호출/월
  - 50개 서비스의 로그 볼륨에 따라 비용이 수십~수백만 원 발생 가능
- **개선안**: 비용 절감 전략 추가:
  - 이상 없는 경우 ML 기반 경량 로컬 모델로 1차 필터링
  - Gemini 호출은 이상 징후 징후가 있을 때만 (이벤트 기반 트리거)
  - 예산 상한선과 Cloud Billing Alert 설정

---

#### M-4. 온프레미스 사내 서비스 ↔ Google Cloud 연결 방안 없음
- **위치**: Section 8.2 시스템 아키텍처
- **문제**: API 서버는 Cloud Run에 있고, 사내 서비스들은 온프레미스(또는 다른 클라우드)에 있을 경우, Prometheus가 사내 서버에서 데이터를 어떻게 Cloud로 전달하는지 방안이 없음. 방화벽, VPN, 네트워크 설정이 전혀 언급되지 않음.
- **개선안**: 연결 방식 결정 필요:
  - **Option A**: Prometheus Federation → 사내 Prometheus가 Cloud Prometheus로 데이터를 Push
  - **Option B**: Prometheus Remote Write → GCP Managed Prometheus 사용
  - **Option C**: VPN Tunnel → Cloud VPN으로 직접 연결
  - 권장: Google Cloud Managed Service for Prometheus (Option B)

---

### 🟡 Major (일관성 & 명확성)

---

#### M-5. 심각도 표기 혼용 (한글/영문)
- **위치**: Section 4.1 vs Section 5, API 명세
- **문제**: 오케스트레이터 설명에서 "낮음/중간/높음/긴급", API에서는 "low/medium/high/critical", 기능 요구사항에서는 혼용.
- **개선안**: `critical`은 모두 "긴급(critical)"으로 통일하거나 한 가지 표기 방식으로 통일.

---

#### M-6. DDoS Agent의 심각도 판단 기준 불합리
- **위치**: Section 4.3 DDoS 탐지 Agent
- **문제**: "트래픽 10배 급증, 응답 시간 5000ms 초과"를 "중간 심각도"로 분류했는데, 이는 서비스가 사실상 마비된 상태임. 반면 "로그인 시도 500회"는 "긴급"으로 분류함. 기준이 일관되지 않음.
- **개선안**: 명확한 심각도 기준표 추가:

  | 심각도 | DDoS 기준 | 침입 탐지 기준 |
  |--------|---------|-------------|
  | low | 트래픽 150% 증가 | 로그인 실패 10회/분 |
  | medium | 트래픽 300% 증가 | 알 수 없는 IP 접근 |
  | high | 트래픽 500% + 응답 저하 | 로그인 실패 100회/분 |
  | critical | 서비스 불능 상태 | 로그인 성공 후 권한 상승 |

---

#### M-7. GET /api/v1/events 페이지네이션 불완전
- **위치**: Section 9.1
- **문제**: 응답에 `"page": 1`이 있지만, 요청 파라미터에 `page`가 없어 다음 페이지를 조회하는 방법이 없음. `totalPages`도 없어 총 페이지 수를 알 수 없음.
- **개선안**:
  ```
  요청 파라미터 추가: page (integer, default: 1)
  응답에 추가:
    "pagination": {
      "page": 1,
      "pageSize": 20,
      "totalPages": 3,
      "total": 42
    }
  ```

---

### 🟢 Minor (개선 제안)

---

#### m-1. Slack Webhook URL 보안 취약점
- **위치**: Section 9.4 알림 발송 설정
- **문제**: 알림 설정 API에서 Slack Webhook URL을 요청 바디에 직접 포함. 이 URL이 로그에 기록되거나 응답에 포함되면 유출될 수 있음. 또한 Webhook URL을 DB에 평문으로 저장하면 DB 침해 시 알림 채널 악용 가능.
- **개선안**: Google Secret Manager에 Webhook URL 저장, API에서는 Secret 이름만 참조.

---

#### m-2. "Vortex AI" vs "Vertex AI" 확인 필요
- **위치**: 원 기획 문서
- **문제**: 원 요청에서 "Google의 Vortex AI"라고 언급했지만 PRD에서는 "Vertex AI"로 작성함. Google Cloud에서 "Vortex"라는 공식 제품명은 없으며, Google의 AI 플랫폼은 **Vertex AI**가 맞음.
- **확인 필요**: 사용자가 의도한 것이 Vertex AI인지 확인. 아니라면 다른 서비스를 가리키는 것일 수 있음.

---

#### m-3. 보안 리포트 수신자가 임원만 정의됨
- **위치**: Section 7.3
- **문제**: 주간 리포트를 임원에게만 보내는 것으로 정의되어 있지만, 보안 담당자/시스템 관리자도 리포트를 받아야 할 수 있음. 수신자 설정 방안이 없음.
- **개선안**: 알림 설정 API에 리포트 수신자 목록도 관리할 수 있도록 추가.

---

## 누락된 요구사항

| ID | 요구사항 | 권장 우선순위 | 근거 |
|----|---------|-------------|------|
| NEW-FR-001 | 관리자 인증 API (로그인/로그아웃/토큰갱신) | **P0** | 기반 기능 없음 |
| NEW-FR-002 | 사용자/역할 관리 API (CRUD) | **P0** | DB 스키마는 있는데 API 없음 |
| NEW-FR-003 | 서비스 등록/수정/삭제 API | **P0** | services 테이블은 있는데 관리 방법 없음 |
| NEW-FR-004 | Agent 헬스체크 및 장애 알림 | **P0** | Agent 자체 장애 대응 없음 |
| NEW-FR-005 | 알림 중복 방지 (Deduplication) | **P1** | 알림 폭주 방지 필수 |
| NEW-FR-006 | 서비스 연동 가이드 문서 | **P1** | 사내 팀이 직접 연동해야 함 |
| NEW-FR-007 | 이벤트 통계 API (대시보드용) | **P1** | 대시보드에 보여줄 집계 데이터 필요 |

---

## 리스크 매트릭스

| 리스크 | 발생 확률 | 영향도 | 대응 방안 |
|--------|---------|--------|---------|
| Vertex AI 비용 폭증 | 높음 | 높음 | 경량 ML 모델로 1차 필터링, 이벤트 기반 호출 |
| Agent 전체 장애 시 보안 사각지대 | 중간 | 매우높음 | Heartbeat 모니터링 + 이중화 |
| 알림 폭주로 인한 Alert Fatigue | 높음 | 중간 | Deduplication + Suppression 정책 |
| 온프레미스-Cloud 네트워크 연결 실패 | 중간 | 높음 | 연결 방식 사전 결정 및 테스트 |
| WIGTN Shield 자체 보안 침해 | 낮음 | 매우높음 | 관리자 인증 강화, API 감사 로그 |
| 개인정보 로그 수집 법적 이슈 | 중간 | 높음 | 로그 마스킹 정책, 법무팀 검토 |
| 오탐률 과다 (5% 초과) | 중간 | 중간 | 탐지 규칙 지속 튜닝, 피드백 루프 |

---

## 권장 조치

### 즉시 조치 (Critical - 구현 시작 전 필수)

1. ❗ **C-1** 관리자 인증 시스템 설계 추가 (users 테이블, 인증 API, JWT 정책)
2. ❗ **C-2** 아키텍처 다이어그램 수정 (Grafana DB → Prometheus API / Loki API)
3. ❗ **C-3** "즉시 차단" 표현 수정 → "차단 권고" 또는 구현 방법 명시
4. ❗ **C-4** Agent 헬스체크 FR 추가 (FR-014)
5. ❗ **C-5** 서비스 연동 방법 섹션 추가
6. ❗ **C-6** 알림 Deduplication 정책 추가

### 구현 전 수정 권장 (Major)

7. ⚠️ **M-1** 탐지 시간 목표 현실화 (2분 → 5분) 또는 스트리밍 아키텍처 결정
8. ⚠️ **M-2** 역할별 권한 매트릭스 추가
9. ⚠️ **M-3** Vertex AI 비용 추정 및 최적화 전략 추가
10. ⚠️ **M-4** 사내-Cloud 네트워크 연결 방식 결정
11. ⚠️ **M-5** 심각도 표기 통일
12. ⚠️ **M-6** 심각도 판단 기준표 추가
13. ⚠️ **M-7** 페이지네이션 API 수정

### 가능하면 수정 (Minor)

14. 💡 **m-1** Webhook URL Secret Manager 보관 방식 추가
15. 💡 **m-2** "Vortex AI" 명칭 확인
16. 💡 **m-3** 리포트 수신자 설정 기능 추가

---

## 다음 단계

> ✅ **Critical 이슈 6개를 해결한 후** `/implement` 명령으로 구현을 시작하세요.
>
> ⚠️ Major 이슈는 Phase 1 구현과 병행하여 수정 가능합니다.
>
> 💡 PRD 수정이 완료되면 다시 digging으로 재검토를 권장합니다.
