# UNECE R155 / R156 (Vehicle Cyber Security & Software Update)

## 개요
UNECE(United Nations Economic Commission for Europe) R155와 R156은 자동차 사이버 보안과 소프트웨어 업데이트 관리에 관한 **법적 강제 규제**입니다. EU, 일본, 한국 등 60개국 이상에서 2024년부터 신규 차종에 의무 적용됩니다. 의료 기기 분야에서도 FDA 사이버 보안 가이드라인(2023), EU MDR Cybersecurity가 유사한 방향으로 강화되고 있습니다.

```
규제 적용 현황:
  EU:    2022년 7월 신규 형식 승인, 2024년 7월 모든 신규 차량
  일본:  2022년 10월부터 적용
  한국:  2023년부터 단계적 적용
  미국:  NHTSA 가이드라인 (강제 아님, 권고)
```

---

## UNECE R155 (Cybersecurity Management System, CSMS)

### CSMS 요구사항 구조

```
R155 CSMS 3대 요소:

1. 조직 역량 (Organizational Capability)
   ├── 사이버 보안 정책 수립
   ├── 사이버 보안 담당 조직/인력
   ├── 위험 관리 프로세스
   └── 공급망 관리 (Tier 1/2/3)

2. 리스크 관리 (Risk Management)
   ├── 개발 단계: TARA (ISO/SAE 21434)
   ├── 생산 단계: 보안 검증
   └── 운용 단계: Post-Market Monitoring

3. 입증 요구 (Evidence)
   ├── TARA 문서
   ├── 보안 대응 조치 문서
   └── 인증 기관(UN Type Approval)에 제출
```

### R155 위협 카테고리 (Annex 5)

```
R155가 요구하는 위협 대응 영역:

카테고리 1: 백엔드 서버 위협
  - 클라우드 서버 침해 → 다수 차량/로봇 동시 공격
  대응: 서버 보안, 접근 제어, 침입 탐지

카테고리 2: 차량 통신채널 위협
  - 원격 공격 (Cellular, V2X, Wi-Fi, Bluetooth)
  - DoIP/UDS 포트 무단 접근
  대응: 방화벽, 인증, 암호화

카테고리 3: 업데이트 절차 위협
  - 악성 펌웨어 주입
  - 다운그레이드 공격
  대응: 코드 서명, Anti-Rollback

카테고리 4: 의도치 않은 행위
  - SW 오류, 설정 오류
  대응: 소프트웨어 개발 프로세스, 코드 리뷰

카테고리 5: 물리적 조작
  - OBD 포트 접근, 하드웨어 탈취
  대응: 물리적 보호, 디버그 포트 비활성화

카테고리 6: 운전자/사용자 행위
  - 피싱, 사회공학
  대응: 인증, 사용자 교육

카테고리 7: 연결 장치
  - USB, SD카드, 블루투스 기기를 통한 공격
  대응: 기기 인증, 화이트리스트
```

### CSMS 인증 프로세스

```
R155 CSMS 인증 흐름:

OEM → 기술 서비스 기관(TÜV, DEKRA 등) → UN 형식 승인청

단계별:
  ① CSMS 문서 제출 (TARA, 보안 정책, 조직 구조)
  ② 기술 서비스 기관 심사 (CSMS Certificate)
  ③ CSMS Certificate → 개별 차종 형식 승인에 첨부
  ④ 운용 단계: 연간 CSMS 갱신 심사

CSMS 유효 기간: 3년 (정기 갱신)
```

---

## UNECE R156 (Software Update Management System, SUMS)

### SUMS 핵심 요구사항

```
R156 SUMS 요소:

RxSWIN (Regulation x Software Identification Number):
  - 차량에 설치된 SW 버전을 법적으로 식별하는 ID
  - 형식: R155/1234 (R155로 인증된 1234번 SW)
  - 업데이트 시 RxSWIN 변경 → 형식 승인청에 통보 필요

SUMS 요구 기능:
  ① SW 버전 관리: 모든 ECU의 SW 버전 추적
  ② 사전 검사: 업데이트 적합성 검증 (차량 상태, 배터리 등)
  ③ 안전성 보장: 업데이트 중/후 기능 안전 유지
  ④ 사용자 고지: 업데이트 내용 및 동의 절차
  ⑤ 무결성 검증: 업데이트 패키지 서명 검증
  ⑥ 롤백: 업데이트 실패 시 이전 버전 복구
  ⑦ 로그: 업데이트 이력 기록 (감사 추적)
```

### OTA 업데이트 안전 절차 (R156 요구)

```
SUMS OTA 업데이트 플로우:

OTA Server                              Vehicle/Robot
     │                                       │
     │──── 업데이트 가용 알림 ────────────────►│
     │     (새 버전 RxSWIN + 설명)            │
     │                                       │ ← 사전 조건 검사:
     │                                       │   - 배터리 충분?
     │                                       │   - 주차 상태?
     │                                       │   - 수술/작업 중 아님?
     │◄─── 업데이트 동의 (사용자 확인) ─────────│
     │                                       │
     │──── 업데이트 패키지 전송 ──────────────►│
     │     (TLS 1.3 암호화)                  │
     │                                       │ ← 패키지 무결성 검증
     │                                       │   (코드 서명 확인)
     │                                       │
     │                                       │ ← 새 이미지 A/B 파티션
     │                                       │   에 설치 (기존 유지)
     │                                       │
     │                                       │ ← 설치 검증
     │                                       │   (체크섬, 기능 테스트)
     │                                       │
     │◄─── 업데이트 결과 보고 ──────────────── │
     │     (성공/실패 + RxSWIN 갱신)          │
     │                                       │
             업데이트 실패 시:
               → A/B 파티션 롤백 (이전 버전)
               → 오류 코드 보고
               → 정상 동작 유지
```

---

## UPTANE - 차량 OTA 보안 프레임워크

```
UPTANE 아키텍처 (R156 보안 요구 충족):

┌───────────────────────────────────────────────────────────┐
│                    OTA Backend                            │
│                                                           │
│  ┌──────────────┐     ┌──────────────────────────────┐   │
│  │  Image Repo  │     │      Director Repo            │   │
│  │  (SW 이미지) │     │  (차량별 업데이트 지시)        │   │
│  │  - 서명된 이미지│    │  - VIN별 타겟 이미지 지정    │   │
│  │  - 해시 목록  │     │  - 서명된 메타데이터          │   │
│  └──────────────┘     └──────────────────────────────┘   │
│          │                        │                       │
│          └──────────┬─────────────┘                       │
│                     │ TLS 1.3                             │
└─────────────────────┼─────────────────────────────────────┘
                      │
              ┌───────┴────────┐
              │    Primary ECU  │  ← 업데이트 조율
              │  (OTA Client)  │
              └───────┬────────┘
                      │ (내부 버스)
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   Secondary      Secondary      Secondary
   (ECU #1)       (ECU #2)       (ECU #3)

UPTANE 보안 특성:
  - 멀티 레벨 서명 (Director + Image Repo)
  - Threshold 서명 (m-of-n 요구)
  - 롤백 방지 (버전 번호 + 만료 시간)
  - 타겟 차량 제한 (차량별 지시)
  - Partial Compromise 내성 (서버 일부 해킹 시 내성)
```

---

## SBOM (Software Bill of Materials)

```
SBOM 관리 (R155/R156 + FDA 요구):

SBOM 포함 정보:
  - 컴포넌트 이름, 버전
  - 라이선스 정보
  - 공급자/제조사
  - CVE 취약점 현황
  - 종속성 (Dependency Tree)

SBOM 표준 형식:
  SPDX (ISO/IEC 5962:2021):
    Linux Foundation 표준
    텍스트/태그-값/JSON/XML/RDF 지원

  CycloneDX (OWASP):
    기계 판독성 우수
    취약점 정보 통합
    VEX (Vulnerability Exploitability eXchange) 지원

SBOM 예시 (SPDX JSON 형식):
{
  "SPDXID": "SPDXRef-DOCUMENT",
  "spdxVersion": "SPDX-2.3",
  "packages": [
    {
      "SPDXID": "SPDXRef-OpenSSL",
      "name": "openssl",
      "versionInfo": "3.0.8",
      "supplier": "Organization: OpenSSL Software Foundation",
      "licenseConcluded": "Apache-2.0",
      "externalRefs": [
        {
          "referenceCategory": "SECURITY",
          "referenceType": "cpe23Type",
          "referenceLocator": "cpe:2.3:a:openssl:openssl:3.0.8:*:*:*:*:*:*:*"
        }
      ]
    }
  ]
}
```

---

## 의료 로봇 규제 대응

### FDA Cybersecurity 요구 (2023 최종 가이드라인)

```
FDA "Refuse to Accept" 정책 (2023.10~):
  의료 기기 허가 신청 시 사이버 보안 문서 미제출 시 거부

필수 제출 문서:
  ① 사이버 보안 아키텍처 다이어그램
  ② 위협 모델링 (TARA 결과)
  ③ SBOM (SW 구성 목록)
  ④ 취약점 공개 정책 (Coordinated Vulnerability Disclosure)
  ⑤ 패치/업데이트 계획
  ⑥ 침투 테스트 결과

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
└────────────────────────┴──────────────┴──────────────────┘
```

### 의료 로봇 CSMS 구현 예시

```
수술 로봇 CSMS 구조:

1. 거버넌스
   - Chief Cybersecurity Officer (or 담당 이사)
   - Cybersecurity Team: 전담 인원 2명 이상
   - 보안 정책 문서 (연간 갱신)

2. 위험 관리
   - 신규 기능: TARA 수행 필수
   - 연간: TARA 재검토 + 신규 위협 평가
   - 공급망: Tier 1 공급사 보안 평가

3. Post-Market 모니터링
   - CVE 데이터베이스 주간 모니터링
   - ISAC (Information Sharing and Analysis Center) 참여
     예: H-ISAC (Healthcare ISAC)
   - 보안 이벤트 VSOC 통보 (72시간 이내)
   - 심각 취약점: 30일 이내 패치 제공

4. OTA 보안 (SUMS)
   - A/B 파티션 플래싱
   - UPTANE 프레임워크
   - 수술 중 업데이트 금지 인터록
   - 업데이트 이력 10년 보관
```

---

## Wireshark로 OTA 트래픽 분석

```bash
# OTA 업데이트 트래픽 캡처
tshark -i eth0 -f "tcp port 443 or tcp port 8883" \
  -w /tmp/ota_capture.pcap

# TLS 핸드셰이크 확인
tshark -r ota_capture.pcap \
  -Y "tls.handshake" \
  -T fields \
  -e ip.src -e ip.dst \
  -e tls.handshake.type \
  -e tls.handshake.version

# TLS 세션 키 복호화 (개발/테스트 환경에서만)
# SSLKEYLOGFILE=/tmp/tls_keys.log
tshark -r ota_capture.pcap \
  -o "tls.keylog_file:/tmp/tls_keys.log" \
  -Y "http2"

# 코드 서명 검증 확인 (DoIP + UDS)
tshark -r ota_capture.pcap \
  -Y "doip && data[0:1] == 37:00"  # RequestTransferExit
```

---

## Reference
- [UNECE R155 - Cyber security and CSMS](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-155-cyber-security-and-cyber-security)
- [UNECE R156 - Software update and SUMS](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software-update)
- [UPTANE Standard](https://uptane.github.io/docs/standard/uptane-standard)
- [FDA - Cybersecurity in Medical Devices (2023)](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/cybersecurity-medical-devices-quality-system-considerations-and-content-premarket-submissions)
- [SPDX Specification v2.3](https://spdx.github.io/spdx-spec/v2.3/)
- [CycloneDX Specification](https://cyclonedx.org/specification/overview/)
