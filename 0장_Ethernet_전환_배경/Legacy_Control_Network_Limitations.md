# 0.1 기존 제어 네트워크의 한계 (Legacy Control Network Limitations)

## 개요

수술 로봇, 재활 로봇, 산업 자동화 시스템은 지난 30년간 CAN, LIN, Fieldbus 계열의 전용 제어 네트워크 위에서 구축되어 왔습니다. 이 기술들은 탄생 당시 혁신이었지만, 오늘날 요구되는 **고속 데이터 처리**, **소프트웨어 중심 아키텍처**, **클라우드 연동**, **AI 기반 의사결정**을 지원하는 데 근본적인 구조적 한계를 가지고 있습니다.

이 문서는 레거시 제어 네트워크들의 기술적 특성과 한계를 정량적으로 분석하고, 왜 Ethernet 전환이 필연적인지에 대한 기술적 근거를 제시합니다.

---

## 1. CAN (Controller Area Network)

### 1.1 탄생 배경과 설계 철학

CAN은 1983년 Bosch가 자동차 내부 전자 제어장치(ECU) 간 통신을 위해 개발하였으며 1986년 공개 표준화되었습니다. 설계 철학의 핵심은 **신뢰성**과 **간결함**이었습니다.

- 제한된 하네스(배선) 복잡도: 점대점 배선 대신 버스 토폴로지로 ECU당 배선 절감
- 강인한 오류 감지: CRC, 비트 모니터링, 형식 검사 등 5종 오류 감지 메커니즘
- 우선순위 기반 접근: 메시지 ID로 중재(Arbitration) → 중요 메시지 우선 전달
- 저비용 구현: 전용 실리콘, 단순 인터페이스 → 대량 생산 적합

### 1.2 CAN 프로토콜 스택

```
CAN 프레임 구조 (Standard Frame, CAN 2.0A):
┌───────┬────┬─────────┬────┬──────────────────┬──────┬────┬────┐
│  SOF  │ ID │   CTL   │ R  │      Data        │ CRC  │ ACK│ EOF│
│  1bit │11b │  6bit   │    │  0 ~ 8 Byte      │15+1b │ 2b │ 7b │
└───────┴────┴─────────┴────┴──────────────────┴──────┴────┴────┘
SOF: Start of Frame
ID:  11비트 (Standard) / 29비트 (Extended, CAN 2.0B)
CTL: DLC(Data Length Code) - 데이터 길이 표시
CRC: 15비트 CRC + 1비트 구분자

CAN FD 프레임 (Flexible Data Rate):
  - 데이터부: 최대 64 Byte (CAN 2.0의 8배)
  - 데이터부 전송 속도: 최대 8 Mbps
  - 중재부 속도: 최대 1 Mbps (기존 CAN과 하위 호환)
```

### 1.3 CSMA/CD vs CAN의 비트 중재 (Arbitration)

```
CAN 버스 중재 (비파괴 방식):

노드 A: ID = 0b00000000001 (낮은 숫자 = 높은 우선순위)
노드 B: ID = 0b00000000010
노드 C: ID = 0b00000000011

동시 전송 시도:
  비트 1: A=0, B=0, C=0 → 버스 = 0 (Dominant) → 모두 계속
  비트 2: A=0, B=0, C=0 → 버스 = 0 → 모두 계속
  ...
  비트 11: A=1, B=0, C=1 → 버스 = 0
           ↑ B=0(Dominant) 감지 → A, C 전송 포기 → B가 버스 획득

결과: 가장 낮은 ID를 가진 메시지가 승리 → 우선순위 기반 충돌 없는 중재
장점: 메시지 손실 없음, 지연 예측 가능 (낮은 ID 메시지)
단점: 메시지 수가 많아질수록 높은 ID 메시지의 지연 보장 불가
```

### 1.4 CAN의 핵심 한계

#### 한계 1: 절대적 대역폭 부족

```
CAN 2.0:   최대 1 Mbps   (실효 ~700 Kbps, 프레임 오버헤드 포함)
CAN FD:    최대 8 Mbps   (데이터부 기준, 중재부 1Mbps)
CAN XL:    최대 20 Mbps  (최신 표준, 2021, 아직 보급 전)

비교:
  100BASE-TX Ethernet: 100 Mbps  (CAN의 100배)
  1000BASE-T Ethernet: 1 Gbps    (CAN의 1000배)
  10GBASE-T Ethernet:  10 Gbps   (CAN의 10,000배)

실제 의료 로봇 데이터 요구량 추정 (6-DOF 수술 로봇):
  관절 위치/속도/토크 × 6 축:       약 150 Byte/사이클
  카메라 영상 (4K, 30fps):           약 66 MB/s = 528 Mbps
  힘/토크 센서 (6축):               약 50 Byte/사이클
  관성 측정 장치 (IMU):             약 30 Byte/사이클
  안전 상태 메시지:                  약 20 Byte/사이클
  OTA/진단:                          가변 (수 Mbps 수준)

합계: 제어 데이터만 1kHz 기준 약 2.5 Mbps → CAN FD 한계에 근접
     영상 포함 시: 500 Mbps 이상 → CAN으로 절대 불가
```

#### 한계 2: 페이로드 크기 제한

```
CAN 2.0: 최대 8 Byte/프레임
CAN FD:  최대 64 Byte/프레임

문제 시나리오: OTA(Over-the-Air) 펌웨어 업데이트
  펌웨어 크기: 4 MB = 4,194,304 Byte
  CAN FD 기준: 4,194,304 / 64 = 65,536 프레임 필요
  각 프레임당 전송 시간(8Mbps): 64Byte × 8 / 8M = 64 µs
  순수 데이터 전송 시간: 65,536 × 64µs = 4.2초
  프레임 오버헤드 포함: 약 6~8초

  → 실제 자동차 OTA는 CAN FD로 수십 분 소요 (UDS 분할 전송)
  → Ethernet OTA: 100Mbps 기준 4MB = 0.32초

AI 모델 업데이트:
  추론 모델 크기: 수백 MB ~ 수 GB → CAN으로 실질적으로 불가
```

#### 한계 3: 노드 수와 버스 부하

```
CAN 버스 물리적 제한:
  최대 권장 노드 수: 110개 (종단 저항, 부하 용량 고려)
  실제 안전 권장:   30~40개

노드 증가 시 문제:
  버스 부하율이 높아지면 높은 ID 메시지의 지연 급증
  버스 부하 70% 이상: 낮은 우선순위 메시지 기아(Starvation) 발생

수술 로봇 내부 노드 예시:
  관절 제어 ECU × 6     = 6개
  안전 감시 ECU × 2     = 2개
  카메라 처리 ECU × 4   = 4개
  힘/토크 센서 × 6      = 6개
  IMU × 2               = 2개
  중앙 제어 ECU × 1     = 1개
  진단/OTA 인터페이스   = 1개
  ────────────────────────────
  합계: 22개 → 향후 확장 고려 시 40개 접근 → 관리 복잡도 급증
```

#### 한계 4: IP 프로토콜 미지원

```
CAN은 IP 주소 체계가 없습니다.
→ 직접 인터넷/클라우드 연결 불가
→ 원격 진단, 원격 제어, 클라우드 AI 연동 시:
   CAN → Gateway ECU → Ethernet → 클라우드
   (게이트웨이 ECU가 병목, 추가 지연 발생)

CAN over IP (Tunneling) 방법:
  CAN 메시지를 UDP/IP로 래핑하여 전송
  → 프로토콜 변환 오버헤드 + 지연 증가 + 실시간성 상실
  → 임시방편, 근본 해결 아님

의료기기 사이버보안:
  CAN에는 기본 보안 기능(인증, 암호화) 없음
  → CAN 버스 접근 시 모든 메시지 도청 가능 (시리얼 버스 특성)
  → ISO/SAE 21434, UNECE R155 준수 매우 어려움
  → TLS, PKI 기반 보안 적용 불가
```

---

## 2. LIN (Local Interconnect Network)

### 2.1 특성

LIN은 1999년 자동차 OEM 컨소시엄(BMW, VW, Audi, Daimler, Volvo)이 CAN의 저비용 보조 버스로 개발했습니다.

```
LIN 특성 요약:
  표준:    ISO 17987 (구: SAE J2602)
  속도:    최대 20 Kbps (일반 10~19.2 Kbps)
  토폴로지: 단일 마스터 + 최대 16 슬레이브 (버스)
  케이블:  단일 선 (Single Wire) + GND
  비용:    CAN 대비 1/10 수준
  용도:    창문 제어, 시트 조절, 조명, 소형 센서
```

### 2.2 LIN의 구조적 한계

```
LIN은 의료 로봇 제어에 사용될 수 없습니다:

1. 속도 한계: 20 Kbps → CAN의 1/50, Ethernet의 1/5000
   (64Byte 프레임 전송: 20Kbps 기준 25.6ms → 실시간 제어 불가)

2. 마스터 의존성: 슬레이브는 마스터 폴링에만 응답 → 자율적 이벤트 전송 불가

3. 오류 감지 약함: 8비트 체크섬만 사용 → CAN의 15비트 CRC보다 훨씬 약함

4. 확장성 없음: 최대 16 노드 → 복잡한 로봇 시스템에 태생적 부적합

→ LIN은 단순 보조 기능에만 사용 가능, 주 제어 네트워크로 절대 부적합
```

---

## 3. Fieldbus (필드버스) 계열

산업 자동화에서는 CAN/LIN 외에 다양한 Fieldbus가 사용됩니다.

### 3.1 주요 Fieldbus 비교

| 프로토콜 | 속도 | 토폴로지 | 최대 노드 | 용도 | 표준 |
|---------|------|---------|---------|------|------|
| PROFIBUS DP | 12 Mbps | 버스/링 | 126 | 공장 자동화 | IEC 61158 |
| DeviceNet | 500 Kbps | 버스 | 64 | 산업 기기 | IEC 62026-3 |
| Modbus RTU | 115 Kbps | 버스 | 247 | PLC 제어 | de facto |
| CC-Link | 10 Mbps | 버스 | 64 | 일본 공장 자동화 | IEC 61158-2 |
| EtherCAT | 100 Mbps | 링/버스 | 65535 | 모션 제어 | IEC 61158 |
| PROFINET RT | 100 Mbps | 스타/링 | 가변 | 공장 자동화 | IEC 61158 |

> **EtherCAT/PROFINET은 물리적으로 Ethernet을 사용하지만, 상위 프로토콜은 독자적입니다.** 표준 IP 스택과 호환되지 않아 클라우드 연동, 범용 진단 도구 활용에 제약이 있습니다.

### 3.2 Fieldbus의 공통 한계

#### 한계 1: 벤더 종속성 (Vendor Lock-in)

```
PROFIBUS 시스템 예:
  마스터 PLC:    Siemens S7-300 (PROFIBUS DP 마스터)
  슬레이브 드라이브: Siemens Sinamics (PROFIBUS DP 슬레이브)
  진단 도구:     Siemens TIA Portal (전용 소프트웨어)
  케이블:        DB9 커넥터, 120Ω 종단 저항 (전용 규격)

문제:
  → 다른 벤더 장비 혼용 시 호환성 문제
  → 진단 도구가 특정 벤더 소프트웨어에 종속
  → 엔지니어가 해당 Fieldbus 전문 교육 필요
  → 마스터-슬레이브 구조로 토폴로지 변경 어려움

Ethernet 전환 후:
  → 모든 표준 Ethernet 장비 혼용 가능
  → Wireshark 등 범용 도구로 진단
  → IP 기반 원격 접속/모니터링 가능
```

#### 한계 2: 토폴로지 유연성 부재

```
PROFIBUS 버스 토폴로지:
  PLC ── 드라이브1 ── 드라이브2 ── 센서1 ── 센서2 (선형 버스)

문제:
  중간 노드 추가/제거 시 버스 전체 중단 필요
  케이블 중간 단선 → 이후 모든 노드 통신 두절
  새 라인 추가 시 케이블 재배선 필요

Ethernet 스타 토폴로지:
  스위치 ─── 드라이브1
           ├─ 드라이브2
           ├─ 센서1
           └─ 새 센서 (포트 하나 더 연결로 즉시 추가)

→ 노드 추가/제거 시 해당 포트만 영향
→ 링크 장애가 다른 노드에 전파되지 않음
```

#### 한계 3: 실시간성의 아이러니 (EtherCAT 포함)

```
EtherCAT는 Ethernet 물리 계층을 사용하면서 100 µs 이하 사이클 타임을 달성하지만:

1. 독자 프로토콜: Ethernet 프레임을 사용하지만 IP/UDP 사용 안 함
   → 표준 Ethernet 스위치 통과 불가 (전용 EtherCAT 마스터/슬레이브 필요)
   → Wireshark로 분석 가능하지만 EtherCAT 디코더 필요

2. 마스터-슬레이브 구조 고착:
   → 마스터 ECU 단일 장애점(SPOF)
   → 마스터 없이 슬레이브끼리 통신 불가

3. IP 기반 서비스 불가:
   → REST API, MQTT, DDS 등 직접 사용 불가
   → 클라우드 데이터 수집 시 별도 Gateway 필요

4. 멀티벤더 혼재 어려움:
   → EtherCAT 슬레이브는 ESC(EtherCAT Slave Controller) 칩 필요
   → 일반 IP 장치와 동일 링에 혼재 불가

→ EtherCAT은 특정 고속 모션 제어에는 탁월하지만,
  소프트웨어 중심 아키텍처(SDV, SOA)로의 전환에는 부적합
```

---

## 4. 중앙집중화의 어려움

### 4.1 분산 ECU 구조의 문제

```
현재 자동차/산업 로봇의 ECU 분산 구조:

         CAN 버스 1 (파워트레인/모터 제어)
         ├── Engine ECU
         ├── Transmission ECU
         └── Motor Control ECU

         CAN 버스 2 (안전/섀시)
         ├── ABS ECU
         ├── ESP ECU
         └── EPS ECU

         CAN 버스 3 (바디/편의)
         ├── Body Control Module
         ├── Lighting ECU
         └── HVAC ECU

         LIN 버스 (부수 기능)
         ├── 시트 ECU
         └── 미러 ECU

         MOST/FlexRay (멀티미디어/안전)
         ├── 인포테인먼트 ECU
         └── ADAS ECU

수술 로봇 예시 (기존):
  CAN 버스 1: 관절 1~3 제어
  CAN 버스 2: 관절 4~6 제어
  CAN 버스 3: 안전 감시
  별도 비디오 케이블: 카메라 (아날로그 또는 독자 디지털)
  별도 USB/RS232: 진단 인터페이스
```

### 4.2 분산 구조의 비용과 복잡도

```
ECU 수 증가의 영향:

  ECU 1개당 비용: 하드웨어 + 소프트웨어 개발 + 검증
  ECU 100개 시스템: 100개 × (HW + SW + 검증) 비용

  하네스(배선) 비용:
    - 일반 승용차: 하네스 무게 40~60 kg, 비용 $800~1,500
    - SUV/트럭: 하네스 무게 70~100 kg
    - 수술 로봇: 로봇 팔 내부 배선 무게가 기동성, 내구성에 직접 영향

  소프트웨어 개발 복잡도:
    - ECU 수 증가 → 인터페이스 수 기하급수 증가 (n개 ECU: n×(n-1)/2 인터페이스)
    - ECU 펌웨어 관리: 각각 독립 개발 사이클
    - 통합 테스트: 모든 ECU 조합 검증 필요

Zonal Architecture (Ethernet 기반):
  Zone Controller 1: 물리적 영역 A 전담 → Zone 내 모든 기기 제어
  Zone Controller 2: 물리적 영역 B 전담
  Central High-Performance Computer: AI, 데이터 융합
  → ECU 수 대폭 감소, 배선 단순화, 소프트웨어 통합 관리
```

---

## 5. 데이터 중심 설계의 부재

### 5.1 신호 기반(Signal-based) vs 서비스 기반(Service-based) 아키텍처

```
레거시 CAN 기반 설계 (신호 기반):
  ECU A: CAN ID 0x100으로 엔진 RPM 10ms마다 브로드캐스트
  ECU B: CAN ID 0x101으로 차속 10ms마다 브로드캐스트
  ECU C: 0x100, 0x101을 구독하여 변속 판단

문제점:
  • 항상 전송 (수신자 필요 여부 무관) → 버스 부하
  • 신호 추가 시 ID 충돌 관리 필요 (DBC 파일 수동 관리)
  • 버전 관리 없음 → ECU 펌웨어 불일치 시 잘못된 데이터 해석
  • 생산자-소비자 관계 고정 (설계 시 하드코딩)
  • 암호화 불가 (IP 스택 없음)

Ethernet 기반 서비스 지향 아키텍처 (SOA):
  SOME/IP 서비스 예:
    서비스: JointPositionService
    메서드: GetCurrentPosition() → 요청 시에만 응답
    이벤트: OnPositionChanged() → 변화 시에만 발행
    구독:   SubscriberECU.subscribe(JointPositionService)

장점:
  • 필요한 데이터만 요청/수신 → 대역폭 효율
  • 서비스 발견(Service Discovery)으로 동적 연결
  • 인터페이스 버전 관리 가능
  • TLS로 암호화 가능
  • DDS(Data Distribution Service)로 실시간 Pub/Sub 패턴

DDS (Data Distribution Service) 장점:
  Topic: "robot/joint/state" → 생산자가 정의
  구독자: 필요한 Topic만 선택 구독
  QoS 정책: 실시간성, 신뢰성, 수명(Lifespan) 별도 설정
  → 동일 데이터를 다수 소비자에게 멀티캐스트 효율 전달
```

### 5.2 AI/ML 통합 불가능성

```
레거시 네트워크에서 AI 통합의 어려움:

요구 시나리오: 수술 로봇에 AI 기반 조직 인식 + 실시간 제어 융합

데이터 흐름 요구:
  4K 내시경 카메라: 4K×30fps = ~500 Mbps
  AI 추론 결과: 실시간 (100ms 이내)
  AI 모델 업데이트: 수 GB (학습 완료 후 배포)

CAN 기반 시스템에서:
  카메라 → 별도 LVDS/MIPI 케이블 → AI 처리 보드 → CAN → 로봇 팔
  문제:
    - 카메라와 AI 보드: 독자 케이블 (추가 배선)
    - AI → CAN 인터페이스: 게이트웨이 ECU 필요
    - 데이터 레이턴시: CAN 전달 지연 + 게이트웨이 변환 지연
    - 모델 업데이트: CAN으로 수 GB 전송 = 수 시간 소요 (불가)

Ethernet 기반 시스템에서:
  카메라 → 1Gbps Ethernet → AI 처리 보드 → Ethernet → 로봇 팔
  모든 구성 요소가 IP 네트워크로 연결 → 데이터 공유, 업데이트 자유
```

---

## 6. 정량적 비교 요약

| 항목 | CAN 2.0 | CAN FD | EtherCAT | 표준 Ethernet | TSN Ethernet |
|------|---------|--------|---------|--------------|-------------|
| 최대 대역폭 | 1 Mbps | 8 Mbps | 100 Mbps | 1 Gbps+ | 1 Gbps+ |
| 최대 페이로드 | 8 Byte | 64 Byte | 1498 Byte | 1500 Byte | 1500 Byte |
| IP 지원 | ✗ | ✗ | ✗ | ✓ | ✓ |
| 결정론적 지연 | △ (중재 의존) | △ | ✓ (전용 토폴로지) | ✗ | ✓ |
| 보안 (TLS) | ✗ | ✗ | ✗ | ✓ | ✓ |
| 클라우드 연동 | ✗ (GW 필요) | ✗ (GW 필요) | ✗ (GW 필요) | ✓ | ✓ |
| 범용 진단 도구 | ✗ | ✗ | △ | ✓ (Wireshark) | ✓ |
| OTA 지원 | △ (매우 느림) | △ (느림) | ✗ | ✓ | ✓ |
| AI 모델 업데이트 | ✗ | ✗ | ✗ | ✓ | ✓ |
| 최대 전송 거리 | 500m (1Mbps) | 40m (8Mbps) | 100m/link | 100m (구리) | 100m (구리) |

---

## 7. 결론: 전환의 불가피성

```
레거시 제어 네트워크의 한계는 개선으로 극복할 수 없습니다:

  CAN FD, CAN XL로 대역폭을 늘려도:
    → IP 스택 없음 → 클라우드/AI 연동 불가
    → 보안 기능 없음 → 의료기기 규제 준수 어려움
    → 페이로드 한계 (64 Byte) → 고해상도 영상 처리 불가

  PROFINET, EtherCAT으로 실시간성을 확보해도:
    → 독자 프로토콜 → 벤더 종속
    → IP 생태계 미활용 → 소프트웨어 중심 설계 불가
    → SOA, DDS, MQTT 직접 사용 불가

→ 근본적 해결: 표준 Ethernet을 제어 네트워크로 사용 가능하게 만드는 TSN
→ 다음 문서(0.2)에서 왜 Ethernet이 대안인지 분석
```

---

## Reference
- [Bosch CAN Specification 2.0](https://www.kvaser.com/wp-content/uploads/2017/08/can20spec.pdf)
- [ISO 11898 - Road vehicles – Controller area network (CAN)](https://www.iso.org/standard/74575.html)
- [ISO 17987 - Road vehicles – Local Interconnect Network (LIN)](https://www.iso.org/standard/61222.html)
- [IEC 61158 - Industrial communication networks – Fieldbus specifications](https://www.iec.ch/homepage)
- [EtherCAT Technology Group](https://www.ethercat.org/)
- [CAN FD - Bosch White Paper](https://www.bosch-semiconductors.com/media/ip_modules/pdf_1/can_fd_whitepaper.pdf)
- [McKinsey - Software-Defined Vehicle Report (2022)](https://www.mckinsey.com/industries/automotive-and-assembly/our-insights/capturing-value-in-the-software-defined-vehicle)
