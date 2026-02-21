# ISO/SAE 21434 (Automotive Cybersecurity Engineering)

## 개요
ISO/SAE 21434:2021은 자동차 사이버 보안 엔지니어링 표준으로, 차량의 전기/전자 시스템에 대한 사이버 위협을 체계적으로 관리하기 위해 ISO와 SAE가 공동으로 제정했습니다. ISO 26262가 기능 안전(우발적 오류)을 다루는 반면, ISO/SAE 21434는 **의도적 공격(Intentional Attack)**에 의한 위협을 다룹니다.

의료 로봇에 연결성(Connectivity)이 증가함에 따라 이 표준의 개념이 의료 기기 사이버 보안(FDA 가이드라인, MDCG 2019-16)에도 직접 적용됩니다.

---

## 기능 안전 vs 사이버 보안

```
┌───────────────────┬─────────────────────┬─────────────────────┐
│                   │   ISO 26262         │   ISO/SAE 21434     │
│                   │  (Functional Safety) │  (Cybersecurity)    │
├───────────────────┼─────────────────────┼─────────────────────┤
│ 위협 유형         │ 우발적 오류          │ 의도적 공격          │
│ 평가 방법         │ HARA                │ TARA                │
│ 등급 체계         │ ASIL A~D            │ CAL 1~4             │
│ 목표              │ Safety Goal         │ Cybersecurity Goal  │
│ 수명주기          │ Safety Lifecycle    │ Cybersecurity LC    │
│ 사후 관리         │ Field Monitoring    │ VSOC (운용 모니터링) │
└───────────────────┴─────────────────────┴─────────────────────┘
```

---

## TARA (Threat Analysis and Risk Assessment)

```
TARA 프로세스:

1단계: Asset Identification (자산 식별)
  ┌───────────────────────────────────────────────────┐
  │ 자산 유형:                                        │
  │  - 데이터 자산: ECU 펌웨어, 개인정보, 키/인증서    │
  │  - 기능 자산: 원격 진단, OTA 업데이트, 제어 명령   │
  │  - 운영 자산: 정상 동작, 가용성                   │
  │                                                   │
  │ 보안 속성 (CSMS - Cybersecurity Management System)│
  │  C: Confidentiality (기밀성)                      │
  │  I: Integrity (무결성)                            │
  │  A: Availability (가용성)                         │
  └───────────────────────────────────────────────────┘

2단계: Threat Modeling (위협 모델링)
  STRIDE 방법론 적용:
    S: Spoofing     - ID 위장 (예: ECU 주소 스푸핑)
    T: Tampering    - 데이터 변조 (예: CAN 메시지 조작)
    R: Repudiation  - 부인 (예: 로그 삭제)
    I: Information  - 정보 유출 (예: DTC 데이터 추출)
    D: Denial of Service - 서비스 거부 (예: 네트워크 플러딩)
    E: Elevation    - 권한 상승 (예: 진단 세션 무단 접근)

3단계: Impact Assessment (영향 평가)
  영향 범주:
    Safety  (S): 사망/부상 가능성
    Financial (F): 금전적 피해
    Operational (O): 서비스 중단
    Privacy (P): 개인정보 침해

  영향 레벨:
    Severe (3): 생명 위협 / 심각한 재정 손실
    Major  (2): 중상 / 서비스 전면 중단
    Moderate(1): 경상 / 부분 중단
    Negligible(0): 무시 가능

4단계: Attack Feasibility (공격 실현 가능성)
  평가 기준 (CVSS 기반):
    Elapsed Time : 수분/수시간/수일/수주/수개월
    Expertise    : 초보/전문가/고급전문가
    Knowledge    : 공개/내부 정보 필요
    Window of Opportunity: 항상/제한적/어려움
    Equipment    : 표준 장비/특수 장비/맞춤형

5단계: Risk Level 결정
  Risk = Impact × Feasibility

  Risk Treatment:
    Avoid    : 기능 제거 (위험이 너무 큼)
    Reduce   : 대응 방안 구현 (Cybersecurity Control)
    Share    : 위험 전가 (보험, 3rd-party)
    Accept   : 잔존 위험 수용 (Residual Risk)
```

---

## CAL (Cybersecurity Assurance Level)

```
CAL 1~4 분류:

CAL 4 ← 가장 엄격 (안전 위협 + 높은 공격 가능성)
  예: 원격 제어 채널, OTA 업데이트 서버
  요구: 코드 리뷰 100%, 침투 테스트, 형식 검증

CAL 3 ← 안전 위협 또는 높은 공격 가능성
  예: Ethernet 진단 포트, Bluetooth 인터페이스
  요구: 코드 리뷰, 취약점 분석

CAL 2 ← 중간 위험
  예: CAN/LIN 버스 인터페이스
  요구: 기본 코드 리뷰, 취약점 스캔

CAL 1 ← 낮은 위험
  예: 내부 버스, 물리 접근 필요 포트
  요구: 기본 사이버 보안 검토

QM   ← 사이버 보안 요구 없음
```

---

## 사이버 보안 수명 주기

```
ISO/SAE 21434 수명 주기:

┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  [콘셉트]─[개발]─[생산]─[운용/유지]─[폐기]                    │
│     │        │       │         │         │                   │
│    TARA    구현    품질 검증  VSOC 모니터링  보안 삭제          │
│  Goal 정의  보안 코딩 침투 테스트  패치 관리   데이터 삭제       │
│                                                              │
│  Post-Market Surveillance (운용 중 지속 관리):               │
│    CVE 모니터링 → 취약점 평가 → 패치 → OTA 업데이트           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## TARA 실습: 의료 로봇 시나리오

```
자산: OTA 업데이트 채널 (DoIP + TLS)

위협 시나리오:
  T1: 악성 펌웨어 주입 (Tampering)
  T2: 업데이트 서버 가장 (Spoofing)
  T3: 업데이트 패킷 도청 (Information Disclosure)
  T4: 업데이트 방해 (Denial of Service)

T1 영향 평가:
  Safety: Severe (3) - 악성 코드로 로봇 오작동 → 환자 부상
  Financial: Major (2)
  → Impact = Severe

T1 공격 가능성:
  Elapsed Time: 수주 (3)
  Expertise: 전문가 (3)
  Window: 업데이트 시간대만 (2)
  Equipment: 표준 PC (2)
  → Feasibility = Medium

T1 위험 수준: High (Severe × Medium)
T1 대응: CAL 4 요구
  대응 방안:
    - 펌웨어 서명 검증 (RSA-4096 or ECDSA-P384)
    - UPTANE 보안 프레임워크
    - 코드 서명 인프라 (HSM 기반)
    - 롤백 방지 (Anti-Rollback Counter)
```

---

## 사이버 보안 대응 기술 (의료 로봇)

### Secure Communication

```
통신 보안 레이어:

Application:  TLS 1.3 (UDS, MQTT, REST API)
              DTLS 1.3 (UDP 기반 실시간 데이터)
              MACsec (IEEE 802.1AE, L2 암호화)
              SecOC (AUTOSAR, CAN/Ethernet 메시지 인증)

Network:      IPsec (IP 계층 암호화)
              VLAN 분리 (네트워크 분리)
              방화벽 (화이트리스트 기반)

Physical:     HSM (Hardware Security Module)
              TrustZone (ARM)
              SHE (Secure Hardware Extension)
```

### Intrusion Detection System (IDS)

```python
# 차량/로봇 네트워크 IDS 개념 코드
# (Python 예시 - 실제 구현은 C/C++ + 임베디드)

class NetworkIDS:
    """
    Ethernet 기반 로봇 네트워크 이상 탐지
    ISO/SAE 21434 요구: 비정상 트래픽 탐지 및 보고
    """

    def __init__(self):
        self.baselines = {
            'joint_control': {'rate': 1000, 'size': 64},   # 1kHz, 64B
            'diagnostics':   {'rate': 10,   'size': 1500},  # 10Hz, 1500B
            'video':         {'rate': 30,   'size': 65535}, # 30fps
        }
        self.alert_threshold = 2.0  # 기준의 2배 초과 시 알람

    def analyze_flow(self, flow_id: str, packet_rate: float,
                     packet_size: int) -> dict:
        """
        트래픽 플로우 분석 - Babbling Idiot 및 주입 공격 탐지
        """
        if flow_id not in self.baselines:
            return {'status': 'UNKNOWN_FLOW', 'severity': 'HIGH'}

        baseline = self.baselines[flow_id]
        rate_ratio = packet_rate / baseline['rate']
        size_ratio = packet_size / baseline['size']

        if rate_ratio > self.alert_threshold:
            return {
                'status': 'RATE_ANOMALY',
                'severity': 'HIGH',
                'detail': f'{flow_id} rate {packet_rate:.1f}/s '
                          f'(baseline: {baseline["rate"]}/s)',
                'action': 'BLOCK_AND_REPORT'
            }

        if size_ratio > self.alert_threshold:
            return {
                'status': 'SIZE_ANOMALY',
                'severity': 'MEDIUM',
                'detail': f'{flow_id} size {packet_size}B '
                          f'(baseline: {baseline["size"]}B)',
                'action': 'LOG_AND_ALERT'
            }

        return {'status': 'NORMAL'}

    def check_authentication(self, src_mac: str,
                              expected_vlan: int,
                              actual_vlan: int) -> bool:
        """VLAN hopping 공격 탐지"""
        if actual_vlan != expected_vlan:
            self.report_incident(
                severity='CRITICAL',
                event='VLAN_SPOOFING',
                detail=f'MAC {src_mac}: VLAN {actual_vlan} '
                       f'(expected {expected_vlan})'
            )
            return False
        return True

    def report_incident(self, severity: str, event: str, detail: str):
        """VSOC (Vehicle/Vehicle SOC)에 보안 이벤트 보고"""
        incident = {
            'timestamp': 'ISO8601_TIMESTAMP',
            'severity': severity,   # CRITICAL/HIGH/MEDIUM/LOW
            'event': event,
            'detail': detail,
            'system_id': 'ROBOT_R001'
        }
        # MQTT 또는 HTTPS로 VSOC에 전송
        # mqtt_client.publish('robot/R001/security/incident', incident)
        print(f"[SECURITY INCIDENT] {severity}: {event} - {detail}")
```

### Supply Chain Security

```
SBOM (Software Bill of Materials) 관리:

의료 로봇 SW 구성 예시:
┌───────────────────────────────────────────────────────────┐
│ Component          │ Version │ License   │ CVE 상태        │
├────────────────────┼─────────┼───────────┼────────────────┤
│ Linux Kernel       │ 5.15.83 │ GPLv2     │ CVE-2023-XXXX  │
│ ROS 2 (Humble)    │ 0.10.0  │ Apache 2  │ 없음            │
│ OpenSSL            │ 3.0.8   │ Apache 2  │ CVE-2023-YYYY  │
│ FastDDS            │ 2.9.1   │ Apache 2  │ 없음            │
│ paho-mqtt          │ 1.6.1   │ EPL 2.0   │ 없음            │
│ libcurl            │ 7.88.1  │ MIT       │ CVE-2023-ZZZZ  │
└───────────────────────────────────────────────────────────┘
→ CVE 발견 시 패치 → OTA 업데이트 → SUMS 관리 (R156 준수)

SBOM 표준:
  SPDX  : Linux Foundation, ISO/IEC 5962:2021
  CycloneDX: OWASP 기반, 기계 가독성 우수
```

---

## Vulnerability Management

```bash
# CVE 취약점 모니터링 예시
# NVD (National Vulnerability Database) API 활용

# SBOM 기반 취약점 스캔 (grype 도구)
grype sbom:robot_sbom.json --fail-on high

# 결과 예시:
# NAME      VERSION   TYPE   VULN ID         SEVERITY
# openssl   3.0.7     rpm    CVE-2023-0215   HIGH
# libcurl   7.83.0    rpm    CVE-2023-27538  MEDIUM

# 패키지 취약점 정기 스캔 스크립트
#!/bin/bash
SBOM_FILE=/etc/robot/sbom.json
REPORT_DIR=/var/log/security

grype sbom:$SBOM_FILE \
  --output json \
  --file $REPORT_DIR/cve_report_$(date +%Y%m%d).json

# 심각도 HIGH 이상 발견 시 VSOC 알림
HIGH_COUNT=$(jq '.matches | map(select(.vulnerability.severity == "High")) | length' \
  $REPORT_DIR/cve_report_$(date +%Y%m%d).json)

if [ "$HIGH_COUNT" -gt 0 ]; then
    curl -X POST https://vsoc.hospital.com/api/alert \
        -H "Authorization: Bearer $VSOC_TOKEN" \
        -d "{\"system\": \"robot_r001\", \"high_cve_count\": $HIGH_COUNT}"
fi
```

---

## Reference
- [ISO/SAE 21434:2021 - Road vehicles — Cybersecurity engineering](https://www.iso.org/standard/70918.html)
- [ENISA - Good Practices for Security of IoT](https://www.enisa.europa.eu/publications/good-practices-for-security-of-iot-1)
- [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework)
- [FDA - Cybersecurity in Medical Devices](https://www.fda.gov/medical-devices/digital-health-center-excellence/cybersecurity)
- [OWASP - Automotive Security](https://owasp.org/www-project-automotive/)
- [SPDX Specification](https://spdx.github.io/spdx-spec/)
