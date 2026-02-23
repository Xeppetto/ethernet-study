# 0.2 왜 Ethernet인가 (Why Ethernet?)

## 개요

0.1에서 살펴본 레거시 제어 네트워크의 한계를 극복하기 위한 차세대 네트워크로 왜 하필 Ethernet이 선택되었을까요? Ethernet이 단순히 "빠른 네트워크"이기 때문이 아닙니다. 이 문서는 Ethernet이 가진 고유한 강점들이 현대 제어 시스템의 요구사항과 어떻게 정확히 맞물리는지를 다각도로 분석합니다.

---

## 1. 압도적인 고속성과 확장 가능한 대역폭

### 1.1 속도 발전 역사와 현재

```
Ethernet 속도 발전 (1980 → 현재):

1980: 10 Mbps (10BASE5, 동축 케이블)
1995: 100 Mbps (100BASE-TX, Fast Ethernet)
1999: 1 Gbps  (1000BASE-T, Gigabit Ethernet)
2002: 10 Gbps (10GBASE-SR/LR)
2016: 100BASE-T1, 1000BASE-T1 (차량용 SPE)
2020: 2.5/5 Gbps 보급형, 25/40/100 Gbps 데이터센터
2022: 400 Gbps (400GBASE-SR8)
2025+: 800 Gbps, Terabit Ethernet (연구 단계)

→ 40년 동안 10만 배 속도 향상, 동일한 프레임 구조 유지
```

### 1.2 대역폭이 실제로 어떤 차이를 만드는가

```
수술 로봇 시스템 요구 대역폭 (6-DOF 수술 로봇 예):

[제어 데이터] (1kHz 사이클):
  관절 위치/속도/토크 (6축 × 3 × 8B) = 144 B/ms → 1.152 Mbps
  안전 상태 모니터링                  =  20 B/ms → 0.160 Mbps
  IMU + 힘/토크 센서                  =  80 B/ms → 0.640 Mbps
  제어 데이터 소계:                              ≈ 2 Mbps

[영상 데이터] (실시간 처리):
  4K 내시경 (60fps, H.265 압축):               ≈ 60 Mbps
  AI 기반 조직 인식 카메라 (비압축):            ≈ 400 Mbps
  보조 카메라 (HD, 30fps):                      ≈ 50 Mbps
  영상 소계:                                    ≈ 510 Mbps

[진단/관리]:
  실시간 로그, 텔레메트리:                      ≈ 10 Mbps
  OTA 대기 (업데이트 시 피크):                  ≈ 100 Mbps

합계: ≈ 620 Mbps (피크)

네트워크별 가용성:
  CAN FD:         8 Mbps    → 불가 (부족분: ×77배)
  EtherCAT:     100 Mbps    → 불가 (영상 포함 시)
  100BASE-T1:   100 Mbps    → 제어 전용으로 적합, 영상은 별도
  1000BASE-T1: 1000 Mbps    → 모든 데이터 통합 가능 ✓
  10GBASE-T:  10000 Mbps    → 미래 확장 포함 여유 ✓
```

### 1.3 속도 등급별 역할 분담

```
의료 로봇 내부 Ethernet 계층 설계:

[센서 레이어] 10BASE-T1S (10 Mbps, 버스):
  ← CAN 대체 → 압력 센서, 온도 센서, 소형 컨트롤러
  특징: 최대 8개 노드, 단일 케이블, PoDL 전원 공급

[관절 제어 레이어] 100BASE-T1 (100 Mbps, P2P):
  ← 각 관절 ECU와 1:1 연결 →
  특징: 경량 단일 쌍 케이블, 15m, PoDL(50W)로 전원 공급
  데이터: 제어 명령 + 피드백 + 안전 상태 = 충분한 여유

[영상/고속 레이어] 1000BASE-T1 (1 Gbps, P2P):
  ← 카메라, 라이다, AI 처리 보드 →
  특징: 비압축 영상 전송 가능

[내부 백본] 2.5/5GBASE-T (2.5/5 Gbps):
  ← Zone Controller 간 연결 →
  특징: 다중 스트림 집선 처리

[병원망 연결] 10GBASE-T 또는 광섬유:
  ← 클라우드, OTA 서버, 원격 콘솔 →
  특징: 방화벽 경유, TLS 필수
```

---

## 2. 전 세계적 범용성과 생태계

### 2.1 수십 년간 축적된 기술 생태계

```
Ethernet 생태계의 규모:

반도체:
  NIC(Network Interface Card) 제조사: Intel, Marvell, Broadcom, Realtek, ...
  PHY 칩: 수십 개 제조사, 수백 종 모델
  스위치 칩: Marvell, Broadcom, Intel, Microchip, ...
  → 경쟁으로 인한 낮은 가격, 풍부한 선택지

소프트웨어:
  운영체제: Linux, Windows, macOS, RTOS 모두 표준 지원
  프로토콜 스택: TCP/IP, UDP, 오픈소스로 수십 년간 검증
  진단 도구: Wireshark (무료), tshark, tcpdump, iperf3, ...
  미들웨어: ROS 2 (DDS), SOME/IP, MQTT, OPC-UA, ...

인프라:
  스위치, 라우터: 소비자용(수만 원) ~ 산업용(수백만 원) 다양
  케이블 및 커넥터: 전 세계 어디서나 구매 가능
  클라우드 서비스: AWS, Azure, GCP 모두 Ethernet 기반

인력:
  전 세계 네트워크 엔지니어 수백만 명
  대학 교육 과정에 기본 포함
  → 인력 수급 용이, 교육 비용 절감
```

### 2.2 CAN 전문가 vs Ethernet 전문가 가용성

```
글로벌 기술 인력 비교 (대략적 추정):

CAN/Fieldbus 전문가:
  - 자동차 ECU 개발자 중 일부
  - 특정 산업 자동화 분야 엔지니어
  - 글로벌 수만~수십만 명 규모

Ethernet/IP 네트워크 전문가:
  - IT 인프라, 클라우드, 보안, IoT 등 포함
  - 글로벌 수백만 명 규모
  - 전문 교육 기관, 인증 체계 완비 (CCNA, Network+, ...)

→ Ethernet 기반 시스템으로 전환 시:
  SW 개발팀이 표준 네트워크 지식으로 기여 가능
  보안팀이 표준 보안 프레임워크(TLS, PKI) 직접 적용
  클라우드팀이 직접 의료 로봇 데이터 파이프라인 구축
```

### 2.3 표준화된 진단 도구의 위력

```
문제 시나리오: 로봇 제어 통신에서 간헐적 지연 발생

CAN 환경 진단:
  1. 전용 CAN 분석기 구매 (Vector CANalyzer: 수백만 원)
  2. DBC 파일 확보 (신호 정의, 벤더 제공 필요)
  3. CAN 버스에 별도 테스트 하네스 연결
  4. CANalyzer 라이센스 구독
  5. 전문 교육 이수

Ethernet 환경 진단 (동일 문제):
  1. Wireshark 설치 (무료, 3분)
  2. 의심 인터페이스에서 패킷 캡처 시작 (1분)
  3. PTP 타임스탬프 분석, Jitter 시각화 (즉시)
  4. 문제 패킷 필터링: tcp.analysis.retransmission (즉시)
  5. 원인 파악 후 설정 수정

# Wireshark 명령줄 진단 예시 (즉시 시작 가능)
tshark -i eth0 -w /tmp/capture.pcap -a duration:60
tshark -r /tmp/capture.pcap -T fields \
  -e frame.time_relative \
  -e ip.src -e ip.dst \
  -e udp.payload \
  -Y "someip" > analysis.txt

→ 전문 도구 없이 개발자 누구나 즉시 진단 가능
```

---

## 3. IP 기반 확장성

### 3.1 IP 주소 체계의 근본적 차이

```
CAN 주소 체계:
  메시지 ID (11비트 또는 29비트): 메시지를 식별, 노드 주소 아님
  노드 직접 주소 지정 불가 (멀티마스터 버스 특성)
  → "누구에게" 보내는지 명시 불가 (브로드캐스트 기반)

Ethernet + IP 주소 체계:
  IP 주소: 192.168.10.101/24 (노드 고유 주소)
  포트 번호: 50000 (서비스/애플리케이션 구분)
  → "192.168.10.101의 포트 50000번 서비스에 요청"이라는 명확한 통신

확장성 차이:
  CAN: 노드 추가 = 물리 버스 연결 + DBC 파일 수정 + 모든 ECU 재컴파일
  Ethernet: 노드 추가 = 케이블 연결 + IP 설정 → 즉시 통신 가능
```

### 3.2 글로벌 연결성

```
Ethernet + IP = 인터넷과 직접 연결 가능

시나리오: 원격 수술 로봇 지원

기존 CAN 기반:
  현장 로봇 (CAN) → Gateway → Ethernet → 인터넷 → 원격 지원 센터
  문제:
    - Gateway 병목 (지연, 대역폭)
    - 게이트웨이 장애 = 원격 지원 불가 (단일 장애점)
    - 보안: CAN 내부 보안 없음 → Gateway에서만 보안 처리
    - 프로토콜 변환 복잡도

Ethernet 기반 직접 연결:
  로봇 ECU (IP: 10.0.1.1) → TLS 터널 → 인터넷 → 원격 지원 (IP: 203.x.x.x)
  장점:
    - 각 ECU가 직접 보안 통신 (E2E TLS)
    - 원격 진단: SSH, HTTPS, WebSocket 직접 사용
    - 원격 업데이트: HTTPS 직접 (Gateway 불필요)
    - 원격 데이터 스트리밍: MQTT/DDS 직접 클라우드 전송

OTA (Over-the-Air) 비교:
  CAN 기반 OTA: 8 Mbps / 4 MB 펌웨어 = 4초 (이론), 실제 10~30초
  Ethernet OTA:  100 Mbps / 100 MB 모델 = 8초, 실제 10~15초
  → Ethernet은 AI 모델(GB 단위)까지 현실적인 OTA 가능
```

### 3.3 IP 멀티캐스트의 효율성

```
DDS(Data Distribution Service) + Ethernet 멀티캐스트:

시나리오: 관절 상태를 여러 소비자에게 동시 전달

CAN 기반 (브로드캐스트, 모든 노드 수신):
  관절 ECU → CAN 브로드캐스트 → [모션 제어 ECU, 안전 ECU, HMI, 진단 PC]
  문제: 모든 노드 인터럽트 발생 → CPU 부하 증가
        진단 PC가 바쁠 때 CAN 버스 부하 영향

DDS + IP 멀티캐스트:
  관절 ECU → UDP Multicast(239.255.0.1:7400) → [구독한 노드만]
  장점:
    - 구독하지 않은 노드에는 전달 안 됨 (IGMP Snooping)
    - 1번 전송으로 다수 수신자 동시 도달
    - QoS 정책으로 각 수신자별 신뢰성/지연 요구사항 설정
    - 생산자 1개 → 소비자 추가 시 생산자 코드 변경 불필요

# ROS 2 DDS Pub/Sub 예시
import rclpy
from std_msgs.msg import Float64MultiArray

def joint_state_publisher():
    node = rclpy.create_node('joint_state_pub')
    pub = node.create_publisher(Float64MultiArray, '/robot/joint/state', 10)
    # → 구독자가 1명이든 100명이든 코드 변경 없음
    # → Ethernet 멀티캐스트로 효율적 전달
```

---

## 4. 소프트웨어 중심 구조(Software-Defined)와의 적합성

### 4.1 소프트웨어 정의 차량(SDV) / 소프트웨어 정의 로봇

```
기존 하드웨어 중심 설계:
  기능 추가 = 새 ECU 추가 + 새 CAN 메시지 추가 + 모든 ECU 재컴파일 + 재검증

소프트웨어 중심 설계 (Ethernet 기반):
  기능 추가 = 새 소프트웨어 컨테이너 배포 + SOME/IP 서비스 등록

예시: 새 AI 기반 안전 감지 기능 추가

CAN 기반 (하드웨어 중심):
  1. 새 AI 처리 ECU 하드웨어 설계 (6개월)
  2. 새 CAN 메시지 ID 할당 및 DBC 업데이트
  3. 기존 ECU 모두 재컴파일/재플래시
  4. 시스템 통합 테스트 (재검증)
  총 소요: 12~18개월, 비용 수억 원

Ethernet + HPC (소프트웨어 중심):
  1. AI 모델 개발 및 테스트 (병렬 진행 가능)
  2. HPC에 도커 컨테이너로 배포
  3. SOME/IP 서비스 등록 (자동)
  4. 기존 서비스들이 새 서비스 발견 (Service Discovery 자동)
  총 소요: 수 주 ~ 수 개월, 비용 대폭 절감

OTA 소프트웨어 업데이트 비교:
  CAN: ECU 하나하나 UDS 프로토콜로 개별 업데이트 (수 시간)
  Ethernet: 컨테이너 이미지 롤링 업데이트 (수 분, 무중단)
```

### 4.2 가상화 및 컨테이너화

```
Ethernet 기반 HPC(High-Performance Computer)에서의 가상화:

물리 하드웨어 1대 (HPC):
  ┌─────────────────────────────────────────────────────────────┐
  │  하이퍼바이저 (Type 1: Xen, KVM)                            │
  │  ┌────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
  │  │ VM 1: 안전 OS  │  │ VM 2: 실시간 OS  │  │ VM 3: Linux  │ │
  │  │ (AUTOSAR AP)   │  │ (QNX/VxWorks)   │  │ (ROS 2, AI)  │ │
  │  │ Safety ECU 역할│  │ 모터 제어 역할  │  │ 데이터 분석  │ │
  │  └────────────────┘  └─────────────────┘  └──────────────┘ │
  │            ↕ 가상 Ethernet (VLAN 기반 격리)                  │
  └─────────────────────────────────────────────────────────────┘
  물리 Ethernet 포트 → 외부 네트워크

CAN 환경에서는 이 구조 불가:
  - 하이퍼바이저가 CAN 인터페이스를 VM에 직접 가상화하기 어려움
  - 실시간성 보장을 위한 CAN 스케줄링이 VM 격리와 충돌
  - VM 간 통신에 별도 CAN 인터페이스 필요
```

### 4.3 CI/CD 파이프라인과의 통합

```
소프트웨어 중심 로봇 개발 파이프라인:

개발자 PC → Git 저장소 → CI 서버 → HIL 테스트 → 배포
            (Ethernet)   (Ethernet)  (Ethernet)   (Ethernet OTA)

각 단계에서 Ethernet 역할:
  1. 코드 푸시: Git over HTTPS → Ethernet
  2. CI 빌드 트리거: Webhook → Ethernet
  3. 빌드 결과 배포: Docker Registry → Ethernet
  4. HIL 테스트 환경 설정: REST API → Ethernet
  5. 테스트 결과 수집: 로봇 센서 데이터 → Ethernet → CI 서버
  6. 프로덕션 배포: OTA over Ethernet

CAN 기반 시스템에서 CI/CD:
  → CI 서버에서 CAN 인터페이스 직접 접근 불가
  → 별도 하드웨어 인터페이스 장비 필요 (Vector, Peak, ...)
  → 빌드 결과를 CAN으로 직접 전달 불가 → 중간 변환 필요
  → 전체 파이프라인 자동화 매우 어려움
```

---

## 5. 표준 기반 보안 아키텍처 적용 가능성

### 5.1 Ethernet + IP 기반 보안 스택

```
의료기기 규제(IEC 62304, ISO 27001, HIPAA, UNECE R155) 요구 보안 기능:

1. 기밀성 (Confidentiality): 데이터 암호화
   Ethernet + TLS 1.3:
     → 모든 제어 데이터 256-bit AES 암호화
     → 인증서 기반 상호 인증 (mTLS)
     → 소켓 하나로 즉시 적용

   CAN: 암호화 표준 없음 → 별도 하드웨어 보안 모듈 필요 + 독자 구현

2. 무결성 (Integrity): 변조 감지
   Ethernet + TLS HMAC:
     → 모든 메시지 HMAC-SHA256 서명
     → 재전송 공격(Replay Attack) 방지 (Sequence Number)
     → 중간자 공격(MITM) 방지 (인증서 검증)

3. 가용성 (Availability): 서비스 지속성
   Ethernet FRER (802.1CB):
     → 두 경로로 동일 패킷 복제 전송 → 한 경로 장애 시 무중단
   CAN: 이중화 표준 없음 → 독자 구현 필요

4. 인증 (Authentication): 장치 신원 확인
   X.509 인증서 + PKI:
     → 각 ECU에 고유 인증서 발급
     → 부팅 시 인증서로 신원 확인 (Secure Boot)
     → 원격 인증서 갱신 (OCSP, CRL)
```

### 5.2 보안 사고 대응 역량

```
보안 침해 시나리오 대응 비교:

CAN 버스 해킹 (예: 외부 커넥터 통해 CAN 버스 접근):
  - 해킹 탐지: 매우 어려움 (CAN에 IDS 없음)
  - 격리: 버스 전체만 차단 가능 (세밀한 격리 불가)
  - 포렌식: CAN 로그 없음, 재현 어려움
  - 대응 시간: 수 주 ~ 수 개월 (취약한 ECU 특정 어려움)

Ethernet 기반 네트워크 해킹:
  - 탐지: IDS/IPS (Suricata, Snort) 실시간 모니터링
  - 격리: 의심 ECU의 VLAN 즉시 차단 (수 초)
  - 포렌식: Wireshark PCAP 로그 분석, 타임라인 재구성
  - 대응: TLS 인증서 즉시 폐기 (CRL), OTA로 패치 배포

UNECE R155 (사이버보안 관리 시스템) 준수:
  → "침해 탐지, 격리, 대응, 복구" 능력 요구
  → CAN 기반 시스템: 이 요구를 충족하기 매우 어려움
  → Ethernet 기반: 표준 도구로 충족 가능
```

---

## 6. 비용 구조의 역전

### 6.1 총 소유 비용(TCO) 비교

```
CAN 기반 시스템 비용 요소:

개발 비용:
  ├── 각 ECU별 별도 개발 팀 (독자 OS, RTOS 등)
  ├── CAN DBC 파일 관리 (신호 정의, 충돌 방지)
  ├── 전용 분석 도구 라이센스 (Vector, PEAK: 연간 수백만 원)
  ├── CAN Transceiver 칩 (ECU당 추가 비용)
  └── 통합 테스트: ECU 간 하네스 시뮬레이터 장비

운영 비용:
  ├── OTA 인프라: CAN Flash 전용 서버 + 소프트웨어
  ├── 원격 진단: 현장 방문 필요 (비접촉 진단 어려움)
  ├── 보안 업데이트: 전 ECU 수동 업데이트
  └── 고장 진단: 전문 장비 + 전문 엔지니어

Ethernet 기반 시스템 비용 요소:

개발 비용:
  ├── 표준 TCP/IP 스택 (오픈소스, 무료)
  ├── Wireshark (무료), tcpdump (무료)
  ├── ROS 2, SOME/IP, DDS (오픈소스 또는 저렴한 라이센스)
  ├── 일반 Ethernet PHY 칩 (대량 생산으로 저렴)
  └── 소프트웨어 시뮬레이션 환경 (Docker, VM으로 구성)

운영 비용:
  ├── OTA: 표준 HTTPS/도커 레지스트리 (클라우드 저렴)
  ├── 원격 진단: SSH, REST API, Wireshark 원격 세션
  ├── 보안 업데이트: 자동화 파이프라인으로 즉시 배포
  └── 모니터링: Prometheus, Grafana (오픈소스)
```

---

## 7. Ethernet이 유일한 정답인가?

Ethernet이 최선의 선택이지만 만능은 아닙니다. 객관적으로 평가합니다.

```
Ethernet의 약점과 보완책:

약점 1: 결정론적 지연 보장 불가 (표준 Ethernet)
  → 보완: TSN(IEEE 802.1Qbv TAS)으로 해결 → 3장에서 상세 설명

약점 2: 케이블과 커넥터의 취약성 (기계적 충격, 진동)
  → 보완: 산업용/차량용 커넥터(IEC 63171-6, H-MTD)
  → 보완: 광섬유(진동/EMI 강인)

약점 3: 물리 계층 전력 소비 (GigE PHY > CAN Transceiver)
  → 보완: EEE(Energy Efficient Ethernet) 활용 (단, TSN과 충돌 주의)
  → 보완: SPE(Single Pair Ethernet) + PoDL로 데이터+전력 통합

약점 4: EMI(전자기 간섭) 취약성
  → 보완: STP/FTP 실드 케이블, 광섬유(완전 면역)
  → 보완: ESD 보호, 적절한 그라운딩

약점 5: 프레임 오버헤드 (IP/UDP 헤더 = 28 Byte)
  → 1500 Byte MTU 기준 1.9% 오버헤드 → 실용적으로 무시 가능

결론:
  TSN Ethernet은 위 약점을 대부분 해결하며
  레거시 Fieldbus보다 훨씬 더 넓은 응용 범위를 커버합니다.
  단, 결정론적 보장을 위해서는 반드시 TSN 기술이 함께 적용되어야 합니다.
  → 이것이 "왜 Ethernet이 제어 네트워크가 되기 어려운가"의 핵심입니다.
  → 다음 문서(0.3)에서 이 한계를 상세히 분석합니다.
```

---

## Reference
- [Ethernet Alliance - Automotive Ethernet](https://ethernetalliance.org/technology/automotive-ethernet/)
- [OPEN Alliance - Automotive Ethernet Specification](https://www.opensig.org/)
- [IEEE 802.3 - Ethernet Standard History](https://standards.ieee.org/ieee/802.3/10422/)
- [OMG DDS Specification](https://www.omg.org/spec/DDS/1.4/PDF)
- [AUTOSAR AP - Adaptive Platform Specification](https://www.autosar.org/standards/adaptive-platform/)
- [McKinsey Global Institute - The Internet of Things (2015)](https://www.mckinsey.com/)
- [NIST SP 800-82 - Guide to ICS Security](https://csrc.nist.gov/publications/detail/sp/800-82/rev-3/final)
