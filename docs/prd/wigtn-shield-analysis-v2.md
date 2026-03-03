# PRD Analysis Report v2 - WIGTN Shield

## 분석 대상
- **문서**: `docs/prd/wigtn-shield-prd.md`
- **버전**: 2.1 (보안 위협 추가 반영 후)
- **분석일**: 2026-03-03

---

## 요약

| 카테고리 | 발견 | Critical | Major | Minor |
|----------|------|----------|-------|-------|
| 완전성 | 5 | 2 | 2 | 1 |
| 기술 실현가능성 | 4 | 1 | 2 | 1 |
| 보안 (역설) | 4 | 3 | 1 | 0 |
| 일관성 & 명확성 | 4 | 0 | 3 | 1 |
| **총계** | **17** | **6** | **8** | **3** |

> v1 분석 대비 Critical 8→6으로 감소 (기존 이슈들 해결됨).
> 신규 이슈 17개 발생 (기능 확장에 따른 새로운 복잡성).

---

## 상세 분석

---

### 🔴 Critical (즉시 수정 필요)

---

#### C-1. 보안 도구가 스스로 시크릿을 평문 저장하는 역설
- **위치**: Section 8.3 config.yaml 구조
- **문제**: `alerts.email.smtp.password`, `alerts.slack.webhook` 등 민감한 자격증명이 `config.yaml`에 평문으로 저장됨. WIGTN Shield는 시크릿 노출을 **탐지**하는 도구인데, 정작 자기 자신의 설정 파일에서 시크릿을 평문으로 보관하는 아이러니한 구조.
- **영향**: `config.yaml`이 실수로 git에 커밋되거나 파일 시스템 접근이 허용되면 Slack Webhook, 이메일 계정 전체가 노출. 오픈소스 패키지라서 사용자가 이 패턴을 그대로 따라할 위험도 높음.
- **개선안**:
  ```yaml
  # 방법 A: 환경변수 참조
  alerts:
    slack:
      webhook: ${WIGTN_SLACK_WEBHOOK}
    email:
      smtp:
        password: ${WIGTN_SMTP_PASSWORD}

  # 방법 B: 별도 .secrets 파일 (config.yaml과 분리, .gitignore 필수)
  # wigtn-shield init 시 .secrets.yaml 생성 + .gitignore 자동 추가
  ```
  `wigtn-shield init` 실행 시 `.gitignore`에 `config.yaml` 또는 `.secrets.yaml` 자동 추가.

---

#### C-2. Thought Signature의 서명 키(SERVER_SECRET_KEY) 관리 방법 미정의
- **위치**: Section 4.7 Thought Signature, Section 10 DB 스키마
- **문제**: `HMAC-SHA256(thought_payload, SERVER_SECRET_KEY)`라고 정의했지만, `SERVER_SECRET_KEY`를 어디서 생성하고 어디에 저장하며 어떻게 로테이션하는지 전혀 없음. 이 키가 노출되면 누구든 thought_payload를 조작한 뒤 올바른 서명을 만들 수 있어 **Thought Signature 전체가 무의미해짐.**
- **영향**: WIGTN Shield의 핵심 차별점인 "변조 불가능한 보안 감사 기록"이 기술적으로 보장되지 않는 상태.
- **개선안**:
  ```
  생성: wigtn-shield init 시 자동으로 SERVER_SECRET_KEY (256-bit) 생성
  저장: OS 키체인 (keytar 라이브러리) 또는 .secrets.yaml (600 권한)
  로테이션: wigtn-shield rotate-key 명령 제공
           → 기존 이벤트: 이전 키 버전으로 검증 (signature_version 활용)
           → 신규 이벤트: 새 키로 서명
  키 백업: 최초 init 시 복구 코드 출력 (관리자가 안전하게 보관)
  ```

---

#### C-3. 구현 단계(Phase 1-3)에 신규 FR이 전혀 반영되지 않음
- **위치**: Section 12 구현 단계
- **문제**: FR-022(Thought Signature)~FR-029(랜섬웨어) 8개가 추가됐지만 Phase 1-3 계획에는 전혀 언급이 없음. 특히 FR-022(Thought Signature)는 P0인데도 어떤 Phase에도 없음.
- **영향**: 개발팀이 계획 없이 구현을 시작하면 Phase 1 MVP가 예상보다 훨씬 커져 일정 초과.
- **개선안**: Phase별 재배치 (아래 권장 일정 참고):
  ```
  Phase 1 추가:
    ✅ FR-022 Thought Signature (P0, 모든 Agent에 필수)
    ✅ FR-024 Injection Attack Agent (P0)

  Phase 2 추가:
    ✅ FR-025 Supply Chain Agent
    ✅ FR-026 Lateral Movement Agent
    ✅ FR-027 Secret Leakage Agent

  Phase 3 추가:
    ✅ FR-023 Live Interception Agent
    ✅ FR-028 Misconfiguration Agent
    ✅ FR-029 Ransomware Detection
  ```

---

#### C-4. Agent 수 10개 이상으로 증가 → 비용/성능 재산정 없음
- **위치**: Section 8.4 실시간 스트리밍 아키텍처
- **문제**: 원래 4개 Agent 기준 "월 50만~150만원" 비용 추정을 했는데, 현재 Agent가 IDS/DDoS/IAM/DLP/Live/Injection/Supply Chain/Lateral Movement/Secret Leakage = 9개로 늘었음. 단순 계산으로도 비용이 2배 이상.
- **영향**: 사용자가 실제 운영 후 예상 초과 비용 발생 → 오픈소스 채택 포기.
- **개선안**:
  ```
  Agent 실행 전략 재설계:

  1. Tier 1 (항상 실행, 경량 모델): IDS, DDoS, Injection, Secret Leakage
     → 규칙 기반 1차 필터, Gemini 호출 최소화

  2. Tier 2 (이상 감지 시 실행, 전체 모델): IAM, DLP, Lateral Movement
     → Tier 1이 "의심" 판정 시에만 활성화

  3. Tier 3 (주기 실행): Supply Chain (1일 1회), Misconfiguration (1시간 1회)
     → 스트리밍 아닌 스케줄링으로 비용 절감

  4. 선택 활성화: Live Interception, Ransomware
     → config에서 enabled: false 기본값

  수정 비용 추정:
    기본 구성 (Tier 1+2):  월 30만~80만원
    전체 구성 (모두 활성화): 월 150만~400만원
  ```

---

#### C-5. FR-028 (설정 오류), FR-029 (랜섬웨어) Agent 구현 명세 없음
- **위치**: Section 5 FR 테이블, Section 4 Agent 팀 구성
- **문제**: FR 테이블에 이름만 있고, 다른 Agent들처럼 심각도 기준/감시 대상/기술 방법을 설명하는 섹션(4.x)이 없음. 특히 랜섬웨어 탐지는 파일 시스템 접근 권한이 필요한데 아키텍처와의 통합 방법이 없음.
- **개선안**: 4.14 설정 오류 탐지 Agent, 4.15 랜섬웨어 지표 탐지 Agent 섹션 추가 필요.

---

#### C-6. WIGTN Shield npm 패키지 자체의 공급망 보안 없음
- **위치**: 전체 (언급 없음)
- **문제**: FR-025로 "다른 서비스의 supply chain 공격을 탐지"하는 기능을 만들면서, WIGTN Shield 자체 npm 패키지의 보안은 아무것도 없음. 악의적 기여자가 wigtn-shield 코드를 오염시키면 설치한 모든 사용자에게 영향.
- **영향**: 특히 사내 보안 모니터링 도구이므로 오염 시 사내 모든 서비스의 로그/메트릭에 접근 가능 → 최악의 공급망 공격 표적.
- **개선안**:
  ```
  npm publish 보안:
    - npm 2FA 강제 활성화
    - GitHub Actions에서만 publish (로컬 publish 금지)
    - Sigstore/npm provenance로 패키지 서명
    - package-lock.json 엄격 관리
    - SECURITY.md + 취약점 신고 채널 (GitHub Security Advisories)
    - 주요 의존성은 최소화 (supply chain surface 축소)
  ```

---

### 🟡 Major (구현 전 수정 권장)

---

#### M-1. Lateral Movement Agent의 Baseline 학습 기간 미정의
- **위치**: Section 4.11 횡적 이동 탐지 Agent
- **문제**: "정상적인 통신 패턴 Baseline 학습 후 이탈 탐지"라고 했는데, 최초 설치 후 Baseline이 없는 상태에서 어떻게 동작하는지, 학습 기간이 얼마인지 없음. 초기 2~4주간 모든 트래픽이 "비정상"으로 판단될 수 있음 → 대량 오탐.
- **개선안**:
  ```
  Baseline 구축 정책:
    초기 모드: 처음 14일간 "학습 모드" (탐지 없이 패턴만 수집)
    전환: wigtn-shield status 에서 "학습 진행률 78%" 표시
    완료: 학습 완료 후 자동으로 탐지 모드 전환 + 관리자 알림
    수동: wigtn-shield agent lateral-movement --mode=active 로 강제 전환

  config.yaml 추가:
    agents:
      lateralMovement:
        baselineDays: 14  # 학습 기간
        sensitivity: medium  # low | medium | high
  ```

---

#### M-2. Secret Leakage Agent와 DLP Agent 역할 중복
- **위치**: Section 4.5 (DLP), Section 4.12 (Secret Leakage)
- **문제**: DLP는 "고객 데이터 외부 유출"을, Secret Leakage는 "자격증명 로그 노출"을 담당한다고 하지만 실제로는:
  - "로그에 DB 비밀번호 노출" → Secret Leakage? DLP?
  - "API 키가 외부 IP로 전송" → Secret Leakage? DLP?
  두 Agent가 동시에 동일 이벤트를 탐지해 중복 알림 발생 가능.
- **개선안**: 명확한 경계 정의:
  ```
  DLP Agent 담당:
    → 외부 네트워크로의 데이터 전송 패턴 (목적지 기반)
    → 고객 PII, 비즈니스 기밀 데이터 대량 이동

  Secret Leakage Agent 담당:
    → 로컬 로그/파일/응답에서 자격증명 패턴 감지 (내용 기반)
    → 코드, 환경변수, 에러 메시지의 시크릿 노출

  겹치는 경우 처리: 오케스트레이터가 동일 타임스탬프 ±5초 내
  같은 서비스에서 발생한 두 이벤트를 하나로 병합
  ```

---

#### M-3. Supply Chain Agent CVE 스캔 주기 및 외부 API 의존성 미정의
- **위치**: Section 4.10 공급망 공격 탐지 Agent
- **문제**:
  1. NIST NVD API는 API Key 없이 사용 시 rate limit 5 req/30초로 엄격히 제한됨. 50개 서비스의 수백 개 패키지를 스캔하면 즉시 차단.
  2. OSV API 호출 실패 시 fallback 없음 → 공급망 탐지 자체가 중단.
  3. 오프라인 환경(인터넷 차단 사내망)에서는 동작 불가.
- **개선안**:
  ```
  1. NIST NVD API Key config에 추가:
     supplyChain:
       nvdApiKey: ${WIGTN_NVD_API_KEY}  # 옵션, 없으면 public rate limit
       scanIntervalHours: 24            # 일 1회 기본
       offlineMode: false               # true시 로컬 CVE DB 사용

  2. CVE 데이터 로컬 캐싱: 마지막 스캔 결과를 SQLite에 저장
     → API 장애 시 캐시 데이터로 계속 운영
  ```

---

#### M-4. Live Interception Agent에 Thought Signature 적용 여부 불명확
- **위치**: Section 4.8 (Live Interception), Section 4.7 (Thought Signature)
- **문제**: Thought Signature가 "모든 AI 판단에 적용"인지 "기존 4개 Agent에만 적용"인지 명시되지 않음. Live Interception은 실시간 WebSocket 스트림에서 즉각 판단하는데, 이 경우에도 thought_payload를 생성하고 서명하는지 불명확.
- **영향**: Live Interception 이벤트만 감사 증적이 없으면 법적 증거력이 불균일.
- **개선안**: "모든 Agent가 이벤트를 생성할 때 반드시 Thought Signature를 포함"으로 명시. Live Interception은 비동기로 서명 생성 (응답 지연 최소화).

---

#### M-5. Injection Agent와 WAF 역할 혼용 우려
- **위치**: Section 4.9 인젝션 공격 탐지 Agent
- **문제**: "SQL Injection 차단"은 WAF(Web Application Firewall)의 역할이고, WIGTN Shield는 "탐지 후 알림"만 해야 하는데 이 구분이 명확하지 않음. Agent가 차단까지 시도하면 기술적으로 복잡해지고 성능 영향도 큼.
- **개선안**: Injection Agent 설명에 명시:
  ```
  역할 경계:
    WIGTN Shield (탐지+알림): 로그에서 인젝션 패턴 감지 → 관리자 알림
    WAF (차단 담당):           실제 요청 차단은 WAF/API Gateway에 위임

  설정 오류 탐지(FR-028)와 연계:
    WAF가 비활성화된 경우 → Misconfiguration 이벤트로 별도 경고
  ```

---

#### M-6. 성공 지표에 새로운 Agent들의 측정 방법 없음
- **위치**: Section 13 성공 지표
- **문제**: 성공 지표가 초기 4개 Agent 기준으로만 작성됨. Injection 탐지율, Supply Chain 취약점 발견율, Secret Leakage 오탐률 등이 없음.
- **개선안**: 신규 Agent별 성공 지표 추가:
  ```
  | Injection 탐지율 | N/A | OWASP Top 10 인젝션 패턴 95% 탐지 | 침투 테스트 |
  | CVE 탐지 SLA    | N/A | CVE 공개 후 24시간 내 알림          | NVD 공개일 비교 |
  | Secret 오탐률   | N/A | 10% 이하 (로그 패턴 특성상 높음)    | 월별 검토 |
  ```

---

#### M-7. config.yaml 버전 업그레이드 마이그레이션 정책 없음
- **위치**: Section 8.3 config.yaml (version: "1")
- **문제**: `version: "1"`이 있지만 WIGTN Shield가 버전업되면서 config 구조가 바뀔 때 이전 버전의 config를 어떻게 처리하는지 없음. 오픈소스 사용자는 업그레이드 후 툴이 갑자기 동작 안 하는 상황 발생 가능.
- **개선안**: `wigtn-shield migrate-config` 명령 추가 또는 시작 시 자동 마이그레이션 + 백업.

---

#### M-8. 개인정보보호법 준수 방안 여전히 없음
- **위치**: 전체
- **문제**: 로그에 개인정보(IP 주소, 사용자 이름, 이메일, 접속 기록)가 포함되고 Loki에 90일 보존하는데, GDPR/개인정보보호법 준수 방안이 없음. 특히 Thought Signature의 `thought_payload`에도 개인정보가 포함될 수 있음. v1 분석에서도 지적됐으나 여전히 미해결.
- **개선안**:
  ```
  log_anonymization 설정 추가:
    privacy:
      anonymizeIPs: true      # IP 마지막 옥텟 마스킹: 192.168.1.xxx
      anonymizeUsernames: false  # 사용자 식별 필요 시 false
      retentionDays: 90          # 보존 기간 법적 요건 충족
      gdprMode: false            # EU 서비스 사용 시 true

  thought_payload 저장 시 PII 자동 마스킹 적용
  ```

---

### 🟢 Minor (개선 제안)

---

#### m-1. Injection Agent의 탐지 대상이 로그 기반으로만 한정됨
- **위치**: Section 4.9
- **문제**: "DB 쿼리 로그, 시스템 로그에서 패턴 감지"라고 했는데, 많은 환경에서 DB 쿼리 로그가 비활성화되어 있음. 로그가 없으면 탐지 자체가 불가능한 전제 조건.
- **개선안**: 설치 가이드에 "Injection 탐지를 위한 DB 쿼리 로그 활성화 방법" 포함.

---

#### m-2. FR-029 랜섬웨어 탐지의 파일 시스템 접근 권한 문제
- **위치**: Section 5, FR-029
- **문제**: 랜섬웨어 탐지는 파일 시스템을 직접 감시해야 하는데, WIGTN Shield는 npm CLI로 사용자 권한으로 실행됨. root/system 권한 없이는 다른 프로세스의 파일 시스템 변경을 감시하기 어려움.
- **개선안**: FR-029 구현 시 "파일 시스템 Agent는 systemd 서비스로 root 권한 실행" 또는 "OS 레벨 inotify/FSEvents 활용" 방안 명시.

---

#### m-3. Web UI의 CSRF 방어 미정의
- **위치**: Section 6.2 보안
- **문제**: JWT 인증을 사용하지만, Web UI가 있으므로 CSRF 공격 방어가 필요. JWT를 쿠키에 저장하면 CSRF 취약, localStorage에 저장하면 XSS 취약. 저장 방식이 명시되지 않음.
- **개선안**: JWT는 httpOnly + SameSite=Strict 쿠키로 저장 (CSRF + XSS 동시 방어). 또는 Authorization 헤더만 사용 (localStorage + CSP 강화).

---

## 누락된 요구사항 (신규)

| ID | 요구사항 | 권장 우선순위 |
|----|---------|-------------|
| NEW-001 | 시크릿 환경변수 주입 지원 (config.yaml 평문 제거) | **P0** |
| NEW-002 | SERVER_SECRET_KEY 생성/저장/로테이션 메커니즘 | **P0** |
| NEW-003 | Baseline 학습 모드 (Lateral Movement Agent) | **P1** |
| NEW-004 | Agent 실행 Tier 전략 (비용 최적화) | **P1** |
| NEW-005 | WIGTN Shield 패키지 자체 서명/무결성 정책 | **P1** |
| NEW-006 | 개인정보 마스킹 (privacy.anonymizeIPs) | **P1** |
| NEW-007 | FR-028/FR-029 Agent 상세 명세 섹션 | **P2** |
| NEW-008 | config 마이그레이션 명령 (wigtn-shield migrate-config) | **P2** |

---

## 보안 위협 커버리지 최종 현황

| 위협 카테고리 | OWASP / MITRE | 커버 여부 | 담당 |
|-------------|-------------|:---:|------|
| Broken Access Control | OWASP A01 | ✅ | IAM Agent |
| Cryptographic Failures | OWASP A02 | ⚠️ | 부분 (Thought Sig) |
| Injection | OWASP A03 | ✅ | Injection Agent (FR-024) |
| Insecure Design | OWASP A04 | ❌ | 미구현 |
| Security Misconfiguration | OWASP A05 | ✅ | FR-028 (P2) |
| Vulnerable Components | OWASP A06 | ✅ | Supply Chain Agent (FR-025) |
| Auth Failures | OWASP A07 | ✅ | IDS Agent + JWT 정책 |
| Software Integrity Failures | OWASP A08 | ✅ | Supply Chain + Thought Sig |
| Security Logging Failures | OWASP A09 | ✅ | 전체 이벤트 기록 |
| SSRF | OWASP A10 | ❌ | 미구현 |
| Brute Force / Credential Stuffing | MITRE T1110 | ✅ | IDS Agent |
| Lateral Movement | MITRE T1021 | ✅ | Lateral Movement Agent (FR-026) |
| Data Exfiltration | MITRE T1041 | ✅ | DLP Agent |
| Command & Control | MITRE T1071 | ✅ | Lateral Movement Agent |
| DDoS | MITRE T1498 | ✅ | DDoS Agent |
| Supply Chain | MITRE T1195 | ✅ | Supply Chain Agent (FR-025) |
| Ransomware | MITRE T1486 | ✅ | FR-029 (P2) |

**미커버 항목:**
- OWASP A04 (Insecure Design): 설계 단계 이슈라 런타임 탐지 불가 — 범위 외로 명시 권장
- OWASP A10 (SSRF): 서버가 외부 URL을 요청하도록 유도하는 공격 — P2로 추가 검토

---

## 리스크 매트릭스 (업데이트)

| 리스크 | 발생 확률 | 영향도 | 대응 방안 |
|--------|---------|--------|---------|
| config.yaml 시크릿 노출 | **높음** | **매우높음** | 환경변수 주입 방식으로 전환 |
| SERVER_SECRET_KEY 탈취 | 낮음 | 매우높음 | OS 키체인 저장 + 로테이션 |
| Agent 비용 폭증 (10개+) | **높음** | 높음 | Tier 전략으로 비용 최적화 |
| Lateral Movement 초기 오탐 | **높음** | 중간 | Baseline 학습 모드 필수 |
| WIGTN Shield 자체 공급망 공격 | 낮음 | **매우높음** | 패키지 서명 + 2FA publish |
| Supply Chain API 장애 | 중간 | 중간 | 로컬 CVE 캐싱 |

---

## 권장 조치

### 즉시 조치 (Critical — 구현 시작 전 필수)

1. ❗ **C-1** config.yaml 시크릿 → 환경변수 참조 방식으로 전환
2. ❗ **C-2** SERVER_SECRET_KEY 생성/저장/로테이션 설계
3. ❗ **C-3** Phase 1-3 일정에 FR-022~FR-029 배치
4. ❗ **C-4** Agent Tier 전략으로 비용 재산정
5. ❗ **C-5** FR-028, FR-029 Agent 상세 명세 추가
6. ❗ **C-6** npm 패키지 보안 정책 추가

### 구현 전 수정 권장 (Major)

7. ⚠️ **M-1** Lateral Movement Baseline 학습 기간 설계
8. ⚠️ **M-2** DLP vs Secret Leakage 역할 경계 명확화
9. ⚠️ **M-3** Supply Chain NVD API Key 및 오프라인 모드
10. ⚠️ **M-4** Live Interception에도 Thought Signature 적용 명시
11. ⚠️ **M-5** Injection Agent 역할 경계 (탐지만, 차단 제외)
12. ⚠️ **M-6** 신규 Agent 성공 지표 추가
13. ⚠️ **M-7** config 마이그레이션 명령 설계
14. ⚠️ **M-8** 개인정보 마스킹 설정 추가

### 가능하면 수정 (Minor)

15. 💡 **m-1** 로그 활성화 가이드 (Injection 전제 조건)
16. 💡 **m-2** 랜섬웨어 Agent 권한 모델 명시
17. 💡 **m-3** JWT 저장 방식 (httpOnly 쿠키 권장)

---

## 전체 PRD 진화 히스토리

| 버전 | 변경 | Critical 수 |
|------|------|------------|
| v1.0 | 최초 작성 | 8 |
| v2.0 | digging 반영 (CLI, 인증, 아키텍처 수정) | 6 (해결 후 신규) |
| v2.1 | Thought Sig, Live API, 보안 위협 6개 추가 | 6 (신규) |
| **v2.2 목표** | C-1~C-6 해결 | **0** |

---

## 다음 단계

> ✅ **C-1~C-6 해결 후** `/implement` 명령으로 구현 시작 권장.
> Critical이 0이 되면 WIGTN Shield는 구현 준비 완료 상태입니다.
