# UNECE R155 / R156 (Vehicle Cyber Security & Software Update)

## 개요

UNECE(United Nations Economic Commission for Europe) R155와 R156은 자동차 사이버 보안과 소프트웨어 업데이트 관리에 관한 **법적 강제 규제**입니다. EU, 일본, 한국 등 60개국 이상에서 2024년부터 신규 차종에 의무 적용됩니다. 규제의 핵심 철학은 단순한 기술 요구사항 나열이 아니라, OEM이 사이버 보안을 **지속적으로 관리하는 체계(Process)**를 갖추었음을 입증하도록 요구한다는 점입니다.

의료 기기 분야에서도 FDA 사이버 보안 가이드라인(2023), EU MDR Cybersecurity, IMDRF 가이드라인이 유사한 방향으로 강화되고 있으며, UNECE R155/R156의 접근 방식은 의료 로봇 사이버 보안 체계 설계에 직접 참고할 수 있습니다.

> **의료 로봇 관점**: UNECE R155/R156은 자동차에만 적용되는 규제이지만, 그 구조와 개념은 의료 로봇에도 그대로 적용됩니다. 수술 로봇이 네트워크에 연결되고, OTA 업데이트를 지원하게 되면서 FDA는 2023년 사이버 보안 가이드라인에서 CSMS(Cybersecurity Management System)와 SBOM을 강제화했습니다. R155/R156은 이러한 규제 흐름의 가장 체계화된 사례로, 의료 로봇 CSMS 설계의 참조 모델로 활용됩니다. 3장 `ISO_SAE_21434.md`의 TARA, `Secure_Boot.md`의 펌웨어 무결성과 함께 읽으면 전체 보안 체계를 이해할 수 있습니다.

```
규제 적용 현황:
  EU:    2022년 7월 신규 형식 승인, 2024년 7월 모든 신규 차량
  일본:  2022년 10월부터 적용
  한국:  2023년부터 단계적 적용
  미국:  NHTSA 가이드라인 (강제 아님, 권고)
  중국:  GB/T 상당 표준 자체 추진 중

의료 기기 유사 규제:
  FDA:   2023년 "Refuse to Accept" 정책 (사이버 보안 문서 미제출 시 거부)
  EU MDR: MDR/IVDR + MDCG 2019-16 Cybersecurity Guidance
  IMDRF: Principles and Practices for Medical Device Cybersecurity (2020)
```

---

## 규제 제정 배경

### 왜 법적 강제 규제가 필요했는가

2010년대 중반부터 자동차와 의료 기기의 사이버 보안 취약점이 연속해서 공개되면서 업계의 자율 규제만으로는 충분하지 않다는 인식이 형성되었습니다.

```
주요 사이버 보안 사고 연표:

2015년: Jeep Cherokee 원격 조종 (Miller & Valasek)
  - 인터넷을 통해 차량의 브레이크, 조향, 엔진 제어
  - Chrysler 차량 140만 대 리콜
  - 미국 상원 자동차 사이버 보안 법안 발의의 계기

2016년: 의료 기기 취약점 공개 (Billy Rios)
  - 인슐린 펌프의 펌웨어 무선 변조 가능성
  - FDA 안전 통신 발령

2019년: 자동차 OTA 시스템 취약점 연구
  - V2X, OBD2 포트를 통한 다수 브랜드 ECU 접근 가능

→ UNECE WP.29 실무 그룹(GRVA)에서 2020년 R155/R156 채택
→ 2021년 공식 발효
```

### R155와 R156의 관계

```
┌─────────────────────────────────────────────────────────────┐
│                    UNECE WP.29 규제 체계                     │
│                                                             │
│  ┌──────────────────┐        ┌──────────────────────────┐   │
│  │  R155            │        │  R156                    │   │
│  │  Cybersecurity   │        │  Software Update         │   │
│  │  Management      │◄──────►│  Management              │   │
│  │  System (CSMS)   │  상호   │  System (SUMS)           │   │
│  │                  │  의존  │                          │   │
│  │  "보안을 어떻게   │        │  "SW 업데이트를 어떻게    │   │
│  │  관리하는가"      │        │  안전하게 하는가"         │   │
│  └──────────────────┘        └──────────────────────────┘   │
│           │                           │                     │
│           └─────────────┬─────────────┘                     │
│                         │                                   │
│                 ISO/SAE 21434 (TARA)                        │
│                 UPTANE (OTA 보안)                           │
│                 SBOM (구성 투명성)                           │
└─────────────────────────────────────────────────────────────┘

핵심 차이:
  R155: "우리 회사는 사이버 보안을 조직적으로 관리한다" (CSMS 인증)
  R156: "우리 차량/제품의 SW를 안전하게 업데이트할 수 있다" (SUMS 인증)
```

---

## UNECE R155 (Cybersecurity Management System, CSMS)

### CSMS란 무엇인가

CSMS(Cybersecurity Management System)는 사이버 보안을 **단일 제품의 기술 요건**이 아닌 **조직 전체의 지속적 관리 체계**로 접근하는 개념입니다. ISO 9001이 품질 관리 시스템을 조직 수준에서 인증하듯이, R155는 OEM의 사이버 보안 관리 역량을 조직 수준에서 평가·인증합니다.

```
R155 CSMS 3대 요소:

1. 조직 역량 (Organizational Capability)
   ├── 사이버 보안 정책 수립 (임원 승인, 연간 갱신)
   ├── 사이버 보안 담당 조직/인력 (책임자 명확화)
   ├── 위험 관리 프로세스 (TARA 수행 절차)
   └── 공급망 관리 (Tier 1/2/3 보안 요구사항 흘림)

2. 리스크 관리 (Risk Management)
   ├── 개발 단계: TARA (ISO/SAE 21434 방법론 적용)
   ├── 생산 단계: 보안 검증 (취약점 스캔, 침투 테스트)
   └── 운용 단계: Post-Market Monitoring (CVE 모니터링)

3. 입증 요구 (Evidence)
   ├── TARA 문서 (위협 분석 결과)
   ├── 보안 대응 조치 문서 (Control 구현 증거)
   └── 인증 기관(UN Type Approval)에 제출
```

### R155 위협 카테고리 (Annex 5)

R155의 Annex 5는 차량(및 연결 기기)이 대응해야 하는 위협을 7개 카테고리로 분류합니다. 각 카테고리에 대해 TARA를 수행하고 대응 방안을 수립해야 합니다.

```
R155가 요구하는 위협 대응 영역:

카테고리 1: 백엔드 서버 위협
  위협: 클라우드 서버 침해 → 다수 차량/로봇 동시 공격
  예시: OTA 서버 해킹으로 악성 펌웨어 대량 배포
  대응: 서버 보안 강화, 접근 제어, 침입 탐지(IDS)
  의료 로봇: 병원 클라우드 서버 → 다수 수술 로봇 동시 위협

카테고리 2: 차량 통신채널 위협
  위협: 원격 공격 (Cellular, V2X, Wi-Fi, Bluetooth)
       DoIP/UDS 포트 무단 접근 (원격 진단 세션 하이재킹)
  대응: 방화벽, 메시지 인증, 암호화, 포트 화이트리스트
  의료 로봇: 5G/Wi-Fi 원격 수술 채널 → mTLS 필수

카테고리 3: 업데이트 절차 위협
  위협: 악성 펌웨어 주입, 다운그레이드 공격
  대응: 코드 서명 검증, UPTANE, Anti-Rollback Counter
  의료 로봇: OTA 업데이트 채널 → UPTANE 적용 권고

카테고리 4: 의도치 않은 행위
  위협: SW 오류, 설정 오류 → 보안 취약점
  대응: 보안 코딩 표준(CERT C), 코드 리뷰, SAST/DAST
  의료 로봇: IEC 62304 + CERT C 준수

카테고리 5: 물리적 조작
  위협: OBD 포트 접근, 하드웨어 탈취, 디버그 포트
  대응: 물리적 보호, JTAG/UART 비활성화(OTP Fuse)
  의료 로봇: 서비스 포트 물리 잠금, 유지보수 인증

카테고리 6: 운전자/사용자 행위
  위협: 피싱, 사회공학, 악성 앱 설치
  대응: 사용자 인증, 앱 화이트리스트, 사용자 교육
  의료 로봇: 의료진 PKI 인증(스마트카드), 역할 기반 접근

카테고리 7: 연결 장치
  위협: USB, SD카드, 블루투스 기기를 통한 공격
  대응: 장치 인증(IEEE 802.1AR), 미디어 화이트리스트
  의료 로봇: USB 포트 비활성화, 승인 장치만 허용
```

### CSMS 인증 프로세스

CSMS 인증은 개별 차종에 대한 형식 승인과 별개로 **OEM 조직 수준**에서 수행됩니다. CSMS 인증 없이는 차종별 형식 승인이 불가능합니다.

```
R155 CSMS 인증 흐름:

OEM → 기술 서비스 기관(TÜV, DEKRA, Bureau Veritas 등) → UN 형식 승인청

단계별 절차:
  ① CSMS 문서 제출
       - 사이버 보안 정책 및 조직도
       - 위험 관리 프로세스 (TARA 절차서)
       - 공급망 보안 관리 절차
       - Post-Market 모니터링 절차
       - 사고 대응 절차 (Incident Response)

  ② 기술 서비스 기관 심사
       - 문서 검토
       - 현장 감사 (조직, 인력, 프로세스)
       - 샘플 차종 TARA 결과 검증

  ③ CSMS Certificate 발급
       - 3년 유효 (정기 갱신)
       - 개별 차종 형식 승인 신청에 첨부

  ④ 차종별 형식 승인
       - CSMS Certificate + 차종 사이버 보안 사례(Cybersecurity Case)
       - 해당 차종의 TARA, 대응 조치 증거

  ⑤ 운용 단계: 연간 CSMS 갱신 심사
       - CVE 대응 실적
       - 보안 사고 처리 기록
       - TARA 업데이트 현황

CSMS 유효 기간: 3년 (정기 갱신 필수)
```

---

## UNECE R156 (Software Update Management System, SUMS)

### SUMS 핵심 요구사항

SUMS는 차량/기기의 SW 업데이트가 **안전하고 제어된 방식**으로 이루어짐을 보장하는 관리 체계입니다. 특히 OTA(Over-The-Air) 업데이트가 보편화되면서, 업데이트 자체가 공격 벡터가 되는 것을 방지하는 것이 핵심 목표입니다.

```
R156 SUMS 7대 핵심 요구사항:

① SW 버전 관리 (Version Tracking)
   - 모든 ECU/컴포넌트의 SW 버전 추적
   - RxSWIN(Regulation x Software Identification Number) 부여
   - 버전 변경 이력 기록 및 보관

② 사전 검사 (Pre-Condition Check)
   - 업데이트 적합성 검증
     · 차량/기기 상태: 주차 중, 배터리 충분(>20%)
     · 네트워크: 안정적 연결
   - 의료 로봇: 수술 중 업데이트 금지 인터록

③ 안전성 보장 (Safety During Update)
   - 업데이트 중 기능 안전 유지
   - A/B 파티션: 업데이트 실패 시 기존 버전으로 즉시 복귀

④ 사용자 고지 (User Notification)
   - 업데이트 내용, 필요성, 영향 범위 고지
   - 사용자 동의 절차 (선택적 업데이트의 경우)
   - 중요 보안 패치: 강제 업데이트 허용 (사전 고지)

⑤ 무결성 검증 (Integrity Verification)
   - 업데이트 패키지 코드 서명 검증 (RSA-4096 or ECDSA-P384)
   - 전송 중 무결성: TLS 1.3
   - UPTANE 멀티 레벨 서명 권고

⑥ 롤백 (Rollback)
   - 업데이트 실패 시 이전 버전 자동 복구
   - A/B 파티션 또는 복구 파티션
   - 롤백 후 정상 동작 보장

⑦ 이력 기록 (Audit Log)
   - 모든 업데이트 시도/성공/실패 기록
   - 타임스탬프 (gPTP 동기화)
   - 법규상 보관 기간: 수명주기 전체
```

### RxSWIN (Regulation x Software Identification Number)

RxSWIN은 R156이 도입한 핵심 개념으로, 차량에 탑재된 SW를 규제 차원에서 추적 가능하게 합니다.

```
RxSWIN 개념:

정의: UN 규제에 따라 형식 승인을 받은 SW 버전에 부여되는 고유 식별자

형식 예시:
  R156/OEM-A/2024-0042
  ├── R156: 해당 규제 번호
  ├── OEM-A: OEM 식별자
  └── 2024-0042: 순번 (42번째 등록 SW)

사용 예시:
  - 차량 ECU 현재 RxSWIN: R156/OEM-A/2024-0042
  - OTA 업데이트 후 신규 RxSWIN: R156/OEM-A/2024-0065
  - RxSWIN 변경 → 형식 승인청에 통보 (중요한 변경의 경우 재심사)

RxSWIN 변경 분류:
  경미한 변경: RxSWIN 서픽스 변경, 통보만 필요
    예: 버그 수정, 보안 패치
  중요한 변경: 새 RxSWIN 발급, 재형식승인 필요
    예: 기능 추가, 안전 관련 SW 변경

의료 기기에서의 유사 개념:
  - FDA: Device Software Functions (DSF) 버전 관리
  - EU MDR: UDI-DI (Unique Device Identification - Device Identifier)
```

### OTA 업데이트 안전 절차 (R156 요구)

```
SUMS OTA 업데이트 플로우:

OTA Server                              Vehicle/Robot
     │                                       │
     │──── 업데이트 가용 알림 ────────────────►│
     │     (새 버전 RxSWIN + 릴리즈 노트)      │
     │                                       │
     │                                       │ ← 사전 조건 검사:
     │                                       │   · 배터리 ≥ 20%?
     │                                       │   · 주차/정지 상태?
     │                                       │   · 수술/작업 중 아님?
     │                                       │   · 네트워크 안정?
     │                                       │   · 저장 공간 충분?
     │                                       │
     │◄─── 업데이트 동의 확인 ─────────────── │
     │     (사용자/관리자 승인 또는 자동)       │
     │                                       │
     │──── 업데이트 패키지 전송 ──────────────►│
     │     (TLS 1.3 + 증분 diff or 전체)     │
     │                                       │
     │                                       │ ← 패키지 무결성 검증
     │                                       │   · 코드 서명 확인
     │                                       │   · SHA-256 체크섬
     │                                       │   · UPTANE Director 서명
     │                                       │
     │                                       │ ← 새 이미지 → B 파티션
     │                                       │   (A 파티션: 현재 버전 유지)
     │                                       │
     │                                       │ ← 설치 검증
     │                                       │   · 체크섬 재확인
     │                                       │   · 기본 기능 테스트
     │                                       │   · Anti-Rollback 카운터 갱신
     │                                       │
     │◄─── 업데이트 결과 보고 ──────────────── │
     │     (성공: RxSWIN 갱신 + 로그 기록)    │
     │     (실패: A 파티션으로 롤백 + 오류코드) │

업데이트 실패 시나리오:
  시나리오 1 - 다운로드 실패:
    → 전송 중단, 부분 파일 삭제, 이전 버전 유지
  시나리오 2 - 서명 검증 실패:
    → 즉시 폐기, 보안 이벤트 로그, VSOC 통보
  시나리오 3 - 설치 후 검증 실패:
    → A 파티션으로 자동 롤백, 오류 보고
  시나리오 4 - 업데이트 중 전원 차단:
    → 재부팅 시 B 파티션 부팅 실패 감지 → A 파티션으로 복귀
```

---

## UPTANE - 차량 OTA 보안 프레임워크

UPTANE는 R156의 OTA 보안 요구사항을 충족하기 위해 설계된 오픈소스 보안 프레임워크입니다. 기존 TUF(The Update Framework)를 차량 환경에 맞게 확장한 것으로, 서버 일부가 해킹되더라도 악성 업데이트가 차량에 설치되지 않도록 설계된 것이 핵심 특징입니다.

```
UPTANE 아키텍처:

┌─────────────────────────────────────────────────────────────┐
│                    OTA Backend                              │
│                                                             │
│  ┌──────────────────┐     ┌────────────────────────────┐   │
│  │   Image Repo     │     │      Director Repo          │   │
│  │                  │     │                            │   │
│  │  ・서명된 이미지  │     │  ・차량별 업데이트 지시      │   │
│  │  ・전체 SW 목록  │     │  ・VIN별 타겟 이미지 지정   │   │
│  │  ・해시 + 크기   │     │  ・서명된 메타데이터        │   │
│  │  ・공개 배포 가능│     │  ・차량 특정 정보 포함      │   │
│  │                  │     │                            │   │
│  │  Compromise 내성: │     │  Compromise 내성:          │   │
│  │  이미지가 유효하다는│   │  어떤 차량에 어떤 이미지가  │   │
│  │  증거만 제공      │     │  가야 하는지만 제공         │   │
│  └──────────────────┘     └────────────────────────────┘   │
│          │                         │                       │
│          └─────────────┬───────────┘                       │
│                        │ TLS 1.3                           │
└────────────────────────┼───────────────────────────────────┘
                         │
                 ┌───────┴────────┐
                 │  Primary ECU   │  ← OTA 클라이언트 (메인 ECU)
                 │  (OTA Manager) │    Director + Image Repo 양쪽 검증
                 └───────┬────────┘
                         │ 내부 버스 (Ethernet/CAN)
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
      Secondary      Secondary      Secondary
      (ECU #1)       (ECU #2)       (ECU #3)
      직접 검증       Primary 중계   직접 검증

UPTANE 보안 특성:
  ✓ 멀티 레벨 서명: Director + Image Repo 양쪽 서명 필수
  ✓ Threshold 서명: m-of-n 키 요구 (키 단독 탈취 방지)
  ✓ 롤백 방지: 버전 번호 + 메타데이터 만료 시간
  ✓ 타겟 차량 제한: Director가 특정 VIN에만 지시
  ✓ Partial Compromise 내성: 서버 일부 해킹에도 악성 업데이트 차단
  ✓ Freeze/Replay 방지: 메타데이터 만료 타임스탬프
```

### UPTANE 검증 플로우 (상세)

```
UPTANE 이미지 검증 순서 (Primary ECU):

1. Director Repo에서 메타데이터 다운로드
   ├── Root: 신뢰 앵커 (키 교체 지원)
   ├── Targets: 이 차량에 설치할 이미지 목록 + 해시
   ├── Snapshot: Targets 파일 목록의 일관성
   └── Timestamp: 메타데이터 최신성 (롤백 방지)

2. Image Repo에서 메타데이터 다운로드
   └── Director와 동일한 구조, 독립 서명

3. 교차 검증 (Cross-Verification)
   ├── Director Targets ∩ Image Repo Targets = 설치 가능 이미지
   ├── 해시 일치 확인 (Director와 Image Repo의 SHA-256 비교)
   └── 불일치 시: 업데이트 중단 + 보안 이벤트 로그

4. 이미지 다운로드 후 최종 검증
   ├── 실제 파일 SHA-256 재계산
   └── Director/Image Repo 메타데이터와 비교

5. Secondary ECU에 이미지 전달 (검증 완료 후)
   └── Secondary도 자체 검증 수행 (Full Verification 모드)
```

---

## SBOM (Software Bill of Materials)

SBOM은 소프트웨어를 구성하는 모든 컴포넌트의 목록으로, 취약점이 발견되었을 때 영향을 받는 제품을 신속하게 파악하고 대응하기 위해 필수적입니다. R155는 SBOM을 CSMS의 핵심 입력 문서로 요구하며, FDA 2023 가이드라인은 의료 기기 허가 신청 시 SBOM 제출을 의무화했습니다.

```
SBOM 포함 정보 (NTIA 최소 요소 + R155 추가 요구):

필수 요소 (NTIA 2021):
  ① 공급자 이름 (Supplier Name)
  ② 컴포넌트 이름 (Component Name)
  ③ 컴포넌트 버전 (Version of the Component)
  ④ 다른 식별자 (Other Unique Identifiers): CPE, PURL
  ⑤ 종속성 관계 (Dependency Relationship)
  ⑥ SBOM 작성자 (Author of SBOM Data)
  ⑦ 타임스탬프 (Timestamp)

R155/FDA 추가 요구:
  ⑧ 라이선스 정보 (License): GPL, Apache, MIT 등
  ⑨ CVE 현황 (Known Vulnerabilities)
  ⑩ VEX (Vulnerability Exploitability eXchange): 취약점 영향 여부
  ⑪ 지원 수명주기 종료(EOL) 날짜

SBOM 표준 형식:

  SPDX (ISO/IEC 5962:2021):
    주관: Linux Foundation
    형식: 텍스트/태그-값/JSON/XML/RDF
    특징: 라이선스 분석에 강점
    채택: Linux 배포판, Yocto Project

  CycloneDX (OWASP):
    주관: OWASP
    형식: JSON/XML
    특징: 취약점 정보 통합, VEX 지원
    채택: 의료 기기(FDA 권고), 자동차 OEM
```

```json
// SBOM 예시 (CycloneDX JSON 형식)
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "version": 1,
  "metadata": {
    "timestamp": "2024-01-15T09:00:00Z",
    "component": {
      "type": "firmware",
      "name": "robot-control-system",
      "version": "3.2.1"
    }
  },
  "components": [
    {
      "type": "library",
      "name": "openssl",
      "version": "3.0.8",
      "purl": "pkg:generic/openssl@3.0.8",
      "licenses": [{"license": {"id": "Apache-2.0"}}],
      "externalReferences": [
        {
          "type": "advisories",
          "url": "https://www.openssl.org/news/vulnerabilities.html"
        }
      ]
    },
    {
      "type": "library",
      "name": "ros2-humble",
      "version": "0.10.0",
      "purl": "pkg:generic/ros2@0.10.0-humble",
      "licenses": [{"license": {"id": "Apache-2.0"}}]
    }
  ],
  "vulnerabilities": [
    {
      "id": "CVE-2023-0215",
      "affects": [{"ref": "pkg:generic/openssl@3.0.7"}],
      "ratings": [{"severity": "high", "score": 7.5}],
      "analysis": {
        "state": "in_triage",
        "detail": "Evaluating impact on robot firmware"
      }
    }
  ]
}
```

### SBOM 기반 CVE 모니터링 자동화

```bash
#!/bin/bash
# SBOM 기반 취약점 자동 모니터링 스크립트
# R155 Post-Market Monitoring 요구 충족

SBOM_FILE=/etc/robot/sbom.cyclonedx.json
REPORT_DIR=/var/log/security/cve
DATE=$(date +%Y%m%d)

mkdir -p $REPORT_DIR

# grype로 SBOM 기반 취약점 스캔
grype sbom:$SBOM_FILE \
  --output json \
  --file $REPORT_DIR/cve_report_${DATE}.json

# 심각도별 카운트
CRITICAL=$(jq '.matches | map(select(.vulnerability.severity == "Critical")) | length' \
  $REPORT_DIR/cve_report_${DATE}.json)
HIGH=$(jq '.matches | map(select(.vulnerability.severity == "High")) | length' \
  $REPORT_DIR/cve_report_${DATE}.json)

echo "CVE 스캔 결과 ($DATE): Critical=$CRITICAL, High=$HIGH"

# Critical/High 발견 시 VSOC(Vehicle/사내 SOC) 자동 알림
if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
    # 의무 대응 기한 계산 (R155: Critical 30일, High 90일)
    DEADLINE_CRITICAL=$(date -d "+30 days" +%Y-%m-%d)
    DEADLINE_HIGH=$(date -d "+90 days" +%Y-%m-%d)

    curl -s -X POST https://vsoc.company.com/api/v1/alert \
        -H "Authorization: Bearer ${VSOC_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "{
            \"system\": \"robot-r001\",
            \"scan_date\": \"${DATE}\",
            \"critical_count\": ${CRITICAL},
            \"high_count\": ${HIGH},
            \"deadline_critical\": \"${DEADLINE_CRITICAL}\",
            \"deadline_high\": \"${DEADLINE_HIGH}\",
            \"report_path\": \"${REPORT_DIR}/cve_report_${DATE}.json\"
        }"
fi

# 주간 트렌드 분석 (R155 보고 자료)
if [ "$(date +%u)" = "1" ]; then  # 월요일마다
    echo "=== 주간 CVE 트렌드 보고서 ===" > $REPORT_DIR/weekly_trend.txt
    ls $REPORT_DIR/cve_report_*.json | tail -7 | while read f; do
        DATE_f=$(basename $f .json | cut -d_ -f3)
        COUNT=$(jq '.matches | length' $f)
        echo "$DATE_f: $COUNT CVEs"
    done >> $REPORT_DIR/weekly_trend.txt
fi
```

---

## 의료 로봇 규제 대응

### FDA Cybersecurity 요구 (2023 최종 가이드라인)

FDA의 2023년 사이버 보안 가이드라인은 510(k), PMA 등 모든 의료 기기 허가 신청에 적용되며, 사이버 보안 문서 미제출 시 검토 자체를 거부(Refuse to Accept)하는 강력한 정책을 시행합니다.

```
FDA "Refuse to Accept" 정책 (2023.10~):
  의료 기기 허가 신청 시 사이버 보안 문서 미제출 시 접수 거부

필수 제출 문서:
  ① 사이버 보안 아키텍처 다이어그램
       - 네트워크 연결 경계, 데이터 흐름 표시
  ② 위협 모델링 (TARA 결과)
       - STRIDE 또는 PASTA 방법론 결과
       - 각 위협에 대한 대응 조치
  ③ SBOM (SW 구성 목록)
       - CycloneDX 또는 SPDX 형식
       - 모든 SOUP(3rd party SW) 포함
  ④ 취약점 공개 정책 (Coordinated Vulnerability Disclosure)
       - CVE 보고 접수 채널
       - 대응 시간 약속 (Critical: 30일 이내)
  ⑤ 패치/업데이트 계획
       - OTA 업데이트 메커니즘
       - 중단 없는 패치 절차
  ⑥ 침투 테스트 결과
       - 독립된 제3자 기관 수행 권고

R155/R156과 FDA 요구 비교:
┌────────────────────────┬──────────────┬──────────────────┐
│ 요구사항               │ R155/R156    │ FDA 2023         │
├────────────────────────┼──────────────┼──────────────────┤
│ TARA/위협 모델링       │ 필수 (R155)  │ 필수             │
│ SBOM                   │ 권고 (R155)  │ 필수             │
│ OTA 보안               │ 필수 (R156)  │ 권고             │
│ 취약점 공개 정책        │ 권고         │ 필수             │
│ Post-Market 모니터링    │ 필수 (R155)  │ 필수             │
│ CSMS/QMS 인증          │ 필수 (R155)  │ QMS (ISO 13485) │
│ 침투 테스트            │ CAL 3/4 요구 │ 권고             │
│ 규제 기간              │ 형식 승인 갱신│ PMA 심사 포함    │
└────────────────────────┴──────────────┴──────────────────┘
```

### 의료 로봇 CSMS 구현 예시

```
수술 로봇 CSMS 4계층 구조:

1. 거버넌스 계층 (Governance)
   담당: Chief Medical Device Security Officer (CMSDO)
   ├── 사이버 보안 정책 (연간 이사회 승인)
   ├── Cybersecurity Team: 전담 인원 2명 이상
   │     · Security Engineer × 1 (기술 대응)
   │     · Risk Manager × 1 (TARA, 규제 대응)
   ├── 보안 예산 (연간 매출의 2~5% 권고)
   └── 공급망 보안 평가 (Tier 1 공급사 연간 감사)

2. 위험 관리 계층 (Risk Management)
   ├── 신규 기능: TARA 수행 필수 (ISO/SAE 21434)
   ├── 연간: TARA 재검토 + 신규 위협 평가
   │     · NVD CVE 신규 발행 주간 모니터링
   │     · H-ISAC (Healthcare ISAC) 정보 공유 참여
   ├── 공급망: Tier 1 공급사 SBOM + 보안 평가
   └── 침투 테스트: 연간 1회 (주요 변경 시 추가)

3. 기술 보안 계층 (Technical Controls)
   ├── 통신 보안: mTLS 1.3, MACsec, SecOC
   ├── 장치 인증: IEEE 802.1AR DevID + PKI
   ├── Secure Boot: Chain of Trust + Anti-Rollback
   ├── OTA 보안: UPTANE + A/B 파티션
   └── IDS: Ethernet 트래픽 이상 탐지

4. Post-Market 모니터링 계층
   ├── CVE 모니터링: NVD API 주간 자동 스캔
   ├── 보안 이벤트 보고: VSOC → 72시간 이내 규제 기관 통보
   ├── Critical 취약점: 30일 이내 패치 배포
   ├── High 취약점: 90일 이내 패치 배포
   └── 업데이트 이력: 10년 보관 (ISO 13485 요구)
```

---

## Wireshark로 OTA 트래픽 분석

OTA 업데이트 과정에서 실제 패킷을 분석하여 R156의 보안 요구사항이 올바르게 구현되었는지 검증할 수 있습니다.

```bash
# OTA 업데이트 트래픽 캡처
# (테스트/개발 환경에서만 수행)
tshark -i eth0 \
  -f "tcp port 443 or tcp port 8883 or tcp port 30509" \
  -w /tmp/ota_capture.pcap

# TLS 핸드셰이크 확인 (TLS 1.3 사용 여부)
tshark -r ota_capture.pcap \
  -Y "tls.handshake" \
  -T fields \
  -e ip.src -e ip.dst \
  -e tls.handshake.type \
  -e tls.handshake.version

# TLS 1.3 사용 확인:
# tls.handshake.version = 0x0304 (TLS 1.3)
# handshake.type = 1 (Client Hello), 2 (Server Hello)

# 인증서 체인 확인 (mTLS 검증)
tshark -r ota_capture.pcap \
  -Y "tls.handshake.type == 11" \
  -T fields \
  -e tls.handshake.certificate

# TLS 세션 키 복호화 (개발/테스트 환경에서만)
# 실행 전: SSLKEYLOGFILE=/tmp/tls_keys.log 환경변수 설정
tshark -r ota_capture.pcap \
  -o "tls.keylog_file:/tmp/tls_keys.log" \
  -Y "http2" \
  -T fields \
  -e http2.headers.method \
  -e http2.headers.path

# UPTANE 메타데이터 요청 확인 (HTTP/2)
# Director: /director/targets.json
# Image Repo: /repo/targets.json
tshark -r ota_capture.pcap \
  -o "tls.keylog_file:/tmp/tls_keys.log" \
  -Y 'http2.headers.path contains "targets.json"'

# 코드 서명 검증 확인 (DoIP + UDS)
# UDS: RequestTransferExit (0x37) - 업데이트 완료 시퀀스
tshark -r ota_capture.pcap \
  -Y "doip" \
  -T fields \
  -e ip.src -e doip.payload_type -e data
```

---

## 규정 준수 체크리스트

### R155 CSMS 준수 체크리스트

```
조직 요구사항:
  ☐ 사이버 보안 담당자 지정 (임원급 또는 전담팀)
  ☐ 사이버 보안 정책 문서화 (연간 갱신)
  ☐ 공급망 보안 요구사항 정의 및 공급사 계약 반영
  ☐ 사고 대응 절차(IRP) 수립

기술 요구사항:
  ☐ TARA 수행 (신규 시스템 또는 연간 재검토)
  ☐ STRIDE 위협 모델링 결과 문서화
  ☐ CAL 별 보안 통제 구현 증거
  ☐ SBOM 작성 및 갱신

운용 요구사항:
  ☐ CVE 모니터링 프로세스 운용
  ☐ 취약점 공개 정책(VDP) 수립
  ☐ 패치 배포 이력 기록
  ☐ ISAC 참여 (H-ISAC, Auto-ISAC 등)
```

### R156 SUMS 준수 체크리스트

```
업데이트 관리:
  ☐ 모든 SW 컴포넌트 버전 추적 체계
  ☐ RxSWIN 발급 및 관리 절차
  ☐ 업데이트 이력 기록 (감사 추적)
  ☐ 형식 승인청 변경 통보 절차

보안 요구사항:
  ☐ 업데이트 패키지 코드 서명 (RSA-4096 또는 ECDSA-P384)
  ☐ TLS 1.3 전송 암호화
  ☐ UPTANE 또는 동등 수준 보안 프레임워크
  ☐ Anti-Rollback 메커니즘

안전성 요구사항:
  ☐ 사전 조건 검사 구현 (배터리, 상태)
  ☐ A/B 파티션 또는 롤백 메커니즘
  ☐ 업데이트 중 기능 안전 유지 (수술 중 인터록)
  ☐ 사용자 고지 및 동의 절차
```

---

## 트러블슈팅

### 문제 1: UPTANE 메타데이터 검증 실패

```
증상: OTA 클라이언트가 "Metadata verification failed" 오류 출력
      업데이트 중단, 이전 버전 유지

원인 분석:
  1. 서버 시간 동기화 문제
     → 메타데이터 만료 타임스탬프 불일치
     → 확인: timedatectl status, ntpstat

  2. Director와 Image Repo 서명 불일치
     → 빌드 파이프라인 키 동기화 오류
     → 확인: uptane-client check-metadata --verbose

  3. 인증서 만료
     → Root 키 또는 Targets 서명 키 만료
     → 확인: openssl x509 -in director_root.pem -noout -dates

해결:
  # 시간 동기화 강제
  chronyc makestep

  # UPTANE 메타데이터 재동기화
  uptane-client refresh-metadata --force

  # 키 만료 시 키 교체 절차 실행 (UPTANE Root Key Rotation)
  uptane-admin rotate-root-key --new-key /secure/new_root.key
```

### 문제 2: A/B 파티션 롤백 실패

```
증상: 업데이트 후 B 파티션 부팅 실패, A 파티션 복구 안 됨

원인 분석:
  1. Bootloader 롤백 카운터 오류
     → 확인: fw_printenv upgrade_available bootcount

  2. A 파티션 마운트 오류
     → 확인: blkid /dev/mmcblk0p1 /dev/mmcblk0p2

  3. dm-verity Root Hash 불일치
     → A 파티션이 오염된 경우
     → 확인: veritysetup verify /dev/mmcblk0p2 /dev/mmcblk0p3 [root_hash]

해결:
  # U-Boot 환경 변수 확인 및 복구
  fw_setenv upgrade_available 0
  fw_setenv bootcount 0

  # Recovery 파티션에서 부팅 (3rd 파티션)
  # Bootloader에서: run bootcmd_recovery

  # 전체 재플래싱 (공장 모드)
  fastboot flash boot_a factory_boot.img
  fastboot flash system_a factory_system.img
```

### 문제 3: SBOM CVE 과탐지 (False Positive)

```
증상: grype가 수백 개의 CVE를 보고, 실제 영향 없는 것이 대부분

원인: SBOM 버전 정보 부정확, 또는 해당 컴포넌트가 실제 사용하는
      코드 경로에 취약 함수가 없음

해결:
  # VEX (Vulnerability Exploitability eXchange) 생성
  # 각 CVE에 대해 영향 여부 명시

  cat > vex.json << 'EOF'
  {
    "vulnerabilities": [
      {
        "id": "CVE-2023-0215",
        "analysis": {
          "state": "not_affected",
          "justification": "code_not_reachable",
          "detail": "The vulnerable function is not called in our build configuration"
        }
      }
    ]
  }
  EOF

  # VEX 파일을 SBOM에 링크
  grype sbom:robot_sbom.json --add-cpes-if-none \
    --vex vex.json

  # PURL(Package URL) 정확화로 오탐 줄이기
  # 예: pkg:rpm/openssl@3.0.8-1.el9 (버전 + 배포판 명확화)
```

---

## Reference
- [UNECE R155 - Cyber security and CSMS](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-155-cyber-security-and-cyber-security)
- [UNECE R156 - Software update and SUMS](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software-update)
- [UPTANE Standard v2.1.0](https://uptane.github.io/docs/standard/uptane-standard)
- [UPTANE IEEE/ISTO 6100.1.0.0](https://uptane.github.io/)
- [FDA - Cybersecurity in Medical Devices (2023)](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/cybersecurity-medical-devices-quality-system-considerations-and-content-premarket-submissions)
- [SPDX Specification v2.3](https://spdx.github.io/spdx-spec/v2.3/)
- [CycloneDX Specification v1.5](https://cyclonedx.org/specification/overview/)
- [NTIA SBOM Minimum Elements](https://www.ntia.gov/report/2021/minimum-elements-software-bill-materials-sbom)
- [ISO/SAE 21434:2021](https://www.iso.org/standard/70918.html)
- [Auto-ISAC Best Practices](https://automotiveisac.com/best-practices/)
