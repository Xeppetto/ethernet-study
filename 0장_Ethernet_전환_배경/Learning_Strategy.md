# 0.5 이 문서의 학습 전략: 어떻게 읽어야 최대 효과를 얻는가

## 개요

이 문서는 **"의료 로봇을 위한 Ethernet/TSN 네트워크 엔지니어"** 를 목표로 하는
학습자를 위해 설계되었습니다.

단순한 Ethernet 교과서가 아닙니다.
각 장은 **"왜 이것이 필요한가"** → **"어떻게 동작하는가"** → **"의료 로봇에서 무엇을 조심해야 하는가"**
의 흐름으로 연결되어 있습니다.

이 절은 다음을 안내합니다:

> **"이 문서를 어떤 순서로, 어떤 목적으로, 어떻게 읽어야 최단 경로로 실무 역량을 갖출 수 있는가?"**

---

## 1. 전체 학습 맵

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    의료 로봇 Ethernet/TSN 학습 로드맵                     │
│                                                                         │
│  [0장] 왜 Ethernet인가?         : 동기 부여, 문제 정의                   │
│    ↓                                                                    │
│  [1장] Ethernet 기초            : 기술 기반 구축                         │
│    ↓                                                                    │
│  [2장] 네트워크 장비와 구조      : 하드웨어 이해                          │
│    ↓                                                                    │
│  [3장] TSN (Time-Sensitive Net) : 결정성 해법 (핵심 장)                  │
│    ↓                                                                    │
│  [4장] 산업용 Ethernet 프로토콜 : 실무 적용                              │
│    ↓                                                                    │
│  [5장] 기능 안전과 사이버 보안   : 규제 및 안전                           │
│    ↓                                                                    │
│  [6장] ROS 2와 DDS             : 소프트웨어 스택                         │
│    ↓                                                                    │
│  [7장] 산업 사례와 미래 트렌드   : 실제 적용 사례                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 장별 상세 학습 목표와 연결 관계

### 0장: Ethernet 전환 배경 (지금 읽고 있는 장)

**학습 목표:**
- 왜 기존 CAN/Fieldbus로는 부족한지 이해
- 왜 Standard Ethernet만으로도 부족한지 이해
- 의료 로봇에서 네트워크가 왜 생사를 가르는지 이해
- 이 문서를 어떻게 읽을지 방향 설정

**핵심 개념:**
```
CAN 한계 → Ethernet 필요성 → Ethernet 한계 → TSN 필요성 → 의료 안전 요구
```

**읽은 후 자가 점검:**
- [ ] CAN 2.0과 CAN FD의 최대 속도 차이를 말할 수 있다
- [ ] Ethernet이 CAN 대비 5가지 장점을 설명할 수 있다
- [ ] Best-Effort가 왜 제어 네트워크에 위험한지 설명할 수 있다
- [ ] IEC 62304 Class C가 무엇인지 설명할 수 있다

---

### 1장: Ethernet 기초

**학습 목표:**
- Ethernet 프레임의 각 필드 역할 이해
- OSI 7계층 모델과 실제 네트워크 동작 연결
- VLAN, QoS 개념 기반 구축
- TSN을 위한 선행 지식 완성

**핵심 파일과 순서:**

```
1-1. OSI_7_Layer.md
     └ OSI 모델, LLC/MAC 하위계층, Collision/Broadcast Domain
     └ 선행: 없음 | 후행: 모든 1장 파일

1-2. Ethernet_Frame_Structure.md  ← 가장 중요
     └ Preamble/SFD/IFG, EtherType, FCS, 직렬화 지연
     └ 선행: OSI_7_Layer.md | 후행: Flow_Control, TSN 이해

1-3. MAC_IP_Difference.md
     └ L2 MAC vs L3 IP, ARP, Routing 기초
     └ 선행: OSI_7_Layer.md | 후행: VLAN, 네트워크 장비

1-4. VLAN_IEEE_802.1Q.md
     └ VLAN 태깅, QinQ, PCP 우선순위, TSN 연동
     └ 선행: Ethernet_Frame_Structure.md | 후행: TSN, QoS

1-5. Full_Duplex_Switching.md
     └ 스위칭, 직렬화 지연, WRR/DWRR, STP/RSTP
     └ 선행: MAC_IP_Difference.md | 후행: QoS, TSN 스위치

1-6. Flow_Control_and_Buffer.md
     └ PAUSE/PFC, HOL Blocking, CBS, 큐 스케줄링
     └ 선행: Full_Duplex_Switching.md | 후행: 3장 TAS/CBS

1-7. Ethernet_Speed_Standards.md
     └ 10M~400G 표준, EEE 위험, TSN PHY 요구사항
     └ 선행: Ethernet_Frame_Structure.md | 후행: 하드웨어 선택

1-8. Single_Pair_Ethernet.md
     └ 10BASE-T1S, 100BASE-T1, PLCA, PoDL
     └ 선행: Ethernet_Speed_Standards.md | 후행: 차량/의료 적용

1-9. Determinism_and_Safety_Basics.md  ← 1장 마무리
     └ 비결정성 원인, 네트워크 FMEA, Fail-safe, IEC 62304
     └ 선행: Flow_Control, Speed_Standards | 후행: 3장 TSN
```

**읽은 후 자가 점검:**
- [ ] Ethernet 프레임을 처음부터 끝까지 바이트 단위로 그릴 수 있다
- [ ] 1GbE에서 64-byte 프레임의 직렬화 지연을 계산할 수 있다
- [ ] PCP 3비트와 8개 우선순위 큐의 관계를 설명할 수 있다
- [ ] HOL Blocking이 무엇이고 어떻게 해결하는지 설명할 수 있다
- [ ] EEE를 TSN 환경에서 반드시 끄는 이유를 설명할 수 있다

---

### 2장: 네트워크 장비와 구조

**학습 목표:**
- 스위치, 라우터, 허브의 차이 이해
- TSN 지원 스위치의 하드웨어 요구사항 이해
- 케이블, SFP, 광섬유 선택 기준 이해
- 수술실 네트워크 물리 구조 설계 능력

**1장과의 연결:**
```
1장: "VLAN이 논리적 분리"   → 2장: "스위치가 VLAN을 어떻게 구현"
1장: "직렬화 지연 공식"     → 2장: "실제 스위치 포트별 큐 구조"
1장: "STP/RSTP 설명"       → 2장: "스위치 포트별 STP 상태 기계"
1장: "EEE 비활성화 필요"   → 2장: "어느 CLI 명령어로 비활성화하나"
```

**읽은 후 자가 점검:**
- [ ] TSN 스위치 선택 시 필수 체크리스트를 말할 수 있다
- [ ] SFP, SFP+, QSFP 차이를 설명하고 적절한 상황에 적용할 수 있다
- [ ] 광섬유 vs 구리선 선택 기준을 설명할 수 있다

---

### 3장: TSN (Time-Sensitive Networking) ← 핵심 장

**학습 목표:**
- TSN 6대 표준의 역할 이해
- TAS (Time-Aware Shaper) 동작 원리와 설정 방법
- gPTP 시간 동기화 구현
- 의료 로봇에서 TSN 적용 설계

**1장과의 연결 (필수 선행 지식):**

```
1장 개념                    3장에서 심화되는 내용
─────────────────────────────────────────────────────────────
Ethernet 프레임 구조      → PTP 프레임 (EtherType 0x88F7) 처리
IFG, Preamble 타이밍      → Guard Band 계산 (TAS 설정)
직렬화 지연 공식           → 사이클 타임 설계 (GCL 프로그래밍)
VLAN PCP 우선순위         → TAS 큐 매핑 (PCP→큐→Gate)
HOL Blocking             → Frame Preemption (802.1Qbu) 필요성
CBS (Credit-Based Shaper)→ 802.1Qav 심화 (SR Class A/B)
Fail-safe, FMEA          → FRER (802.1CB) 설계
IEC 62304 요구사항        → TSN 네트워크 검증 방법론
```

**3장 내 학습 순서:**

```
3-1. TSN 개요와 표준 체계
     └ IEEE 802.1AS/Qbv/Qav/Qbu/Qci/CB 개요
     └ 선행: 1장 전체

3-2. gPTP (802.1AS) 시간 동기화
     └ Grand Master Clock, Transparent Clock, PDelay
     └ 선행: PTP 프레임 구조, 하드웨어 타임스탬프 이해

3-3. TAS (802.1Qbv) Time-Aware Shaper
     └ GCL, Cycle Time, Guard Band, Hold-and-Release
     └ 선행: 직렬화 지연, 큐 스케줄링 (1장), gPTP (3-2)

3-4. CBS (802.1Qav) Credit-Based Shaper
     └ idleSlope, sendSlope, SR Class A/B
     └ 선행: CBS 개념 (1장), 대역폭 계산

3-5. Frame Preemption (802.1Qbu)
     └ Express Frame vs Preemptable Frame, LLDP 협상
     └ 선행: IFG, HOL Blocking (1장), TAS (3-3)

3-6. PSFP (802.1Qci) 와 FRER (802.1CB)
     └ Per-Stream Filtering, Sequence Encoding, Path Selection
     └ 선행: VLAN, 네트워크 이중화 (1-2장)
```

**읽은 후 자가 점검:**
- [ ] TAS의 GCL을 설계하고 사이클 타임을 계산할 수 있다
- [ ] gPTP에서 Transparent Clock의 역할을 설명할 수 있다
- [ ] FRER로 50µs 내 경로 전환이 가능한 이유를 설명할 수 있다
- [ ] 의료 로봇에서 SR Class A vs Class B 트래픽을 분류할 수 있다

---

### 4장: 산업용 Ethernet 프로토콜

**학습 목표:**
- EtherCAT, PROFINET, EtherNet/IP 비교 이해
- TSN과 기존 산업 프로토콜의 공존 방법
- 의료 로봇에서 레거시 프로토콜 브리지 설계

**3장과의 연결:**
```
3장: TSN 표준 기반           → 4장: 각 산업 프로토콜이 TSN을 어떻게 활용
3장: 결정성 보장 메커니즘    → 4장: EtherCAT 분산 클록 vs TSN gPTP 비교
3장: 대역폭 예약 (CBS)       → 4장: PROFINET IRT vs TSN Qbv 비교
```

**읽은 후 자가 점검:**
- [ ] EtherCAT이 표준 IP와 호환되지 않는 이유를 설명할 수 있다
- [ ] PROFINET RT, IRT, TSN의 지연 성능 차이를 나열할 수 있다
- [ ] 신규 의료 로봇 설계에서 TSN vs EtherCAT을 선택하는 기준을 말할 수 있다

---

### 5장: 기능 안전과 사이버 보안 ← 의료 로봇 필수

**학습 목표:**
- IEC 62304 Class A/B/C 적용 방법
- ISO 14971 위험 관리와 네트워크 FMEA
- 사이버 보안 (IEC 62443, FDA 가이드라인)
- 인증 프로세스 이해

**0장 및 1장과의 연결:**
```
0장 0.4: IEC 62304 소개      → 5장: 구체적 요구사항 구현
1장: 네트워크 FMEA 예시      → 5장: 완전한 위험 관리 문서
1장: Fail-safe 설계 원칙     → 5장: 안전 기능 구현 (SIL/ASIL)
3장: FRER 이중화             → 5장: 안전 무결성 레벨 달성 방법
```

**읽은 후 자가 점검:**
- [ ] 네트워크 장애에 대한 FMEA 표를 직접 작성할 수 있다
- [ ] IEC 62304 Class C 소프트웨어 개발 체크리스트를 설명할 수 있다
- [ ] FDA 사이버보안 가이드라인의 핵심 요구사항 5가지를 말할 수 있다

---

### 6장: ROS 2와 DDS

**학습 목표:**
- ROS 2의 DDS 미들웨어 이해
- DDS QoS 정책과 Ethernet QoS 연계
- TSN 위에서 ROS 2 실행 최적화

**3장 및 4장과의 연결:**
```
3장: TSN 결정성 보장         → 6장: ROS 2가 TSN 위에서 실시간 동작
4장: DDS vs 산업 프로토콜   → 6장: DDS Publisher/Subscriber 설계
1장: VLAN, QoS, DSCP       → 6장: ROS 2 DDS QoS ↔ Ethernet QoS 매핑
```

**읽은 후 자가 점검:**
- [ ] ROS 2에서 RELIABLE vs BEST_EFFORT QoS 선택 기준을 설명할 수 있다
- [ ] DDS DEADLINE QoS가 제어 루프 타이밍을 어떻게 보장하는지 설명할 수 있다

---

### 7장: 산업 사례와 미래 트렌드

**학습 목표:**
- 실제 의료 로봇, 자동차, 제조 사례 분석
- 5G + TSN 융합 트렌드
- DetNet, URLLC 미래 기술
- 커리어 관점에서 중요한 기술 스택

**이전 장과의 연결:**
- 전체 내용의 실무 적용 사례
- 기술 선택의 트레이드오프 실제 경험

---

## 3. 학습자 유형별 권장 경로

### 유형 A: Ethernet 초보자 (제어 공학 배경)

```
목표: 제어 엔지니어에서 네트워크-인식 제어 엔지니어로
예상 학습 기간: 4~6주 (하루 2시간 기준)

권장 경로:
Week 1: 0장 전체 (배경 이해) + 1장 OSI, Frame
Week 2: 1장 나머지 (VLAN, QoS, Flow Control, Speed)
Week 3: 2장 (하드웨어) + 1장 Determinism 복습
Week 4: 3장 (TSN) - 가장 중요, 2회 이상 읽기 권장
Week 5: 5장 (안전) + 4장 (산업 프로토콜)
Week 6: 6장 (ROS 2) + 7장 (사례)

주의사항:
  ✓ 1장을 충분히 이해한 후 3장으로 이동
  ✓ 3장 TAS 설정 예제를 직접 실습 (가상 환경)
  ✗ EtherCAT 경험이 있다고 TSN을 건너뛰지 말 것
```

### 유형 B: 네트워크 엔지니어 (IT/데이터센터 배경)

```
목표: IT 네트워크 전문가에서 실시간 제어 네트워크 전문가로
예상 학습 기간: 2~4주

권장 경로:
Week 1: 0장 (동기 이해) → 1장 Determinism 먼저
         → 1장 Flow Control, VLAN 복습 (알지만 RT 관점에서 재해석)
Week 2: 3장 전체 (TSN이 핵심 신규 내용)
Week 3: 5장 (규제 환경 - 의료 특화) + 4장
Week 4: 6장 + 7장

주의사항:
  ✓ TCP/IP 지식은 있지만 실시간 제어 요구사항이 다름을 인식
  ✓ 의료 규제 (IEC 62304)에 집중 - IT와 가장 다른 부분
  ✗ "Best-Effort로 충분하다"는 IT 관성을 버릴 것
  ✗ STP 재수렴(30-50초)은 의료 로봇에서 절대 허용 불가
```

### 유형 C: 소프트웨어 엔지니어 (임베디드/ROS 배경)

```
목표: 소프트웨어 개발자에서 시스템 레벨 네트워크 이해 개발자로
예상 학습 기간: 3~5주

권장 경로:
Week 1: 0장 + 1장 Frame Structure (소켓 프로그래밍과 연결)
Week 2: 1장 QoS, Flow Control (Linux tc 명령어 중심)
Week 3: 3장 (TSN, linuxptp, tc-taprio 실습)
Week 4: 6장 (ROS 2 DDS QoS 심화) + 5장
Week 5: 2장 + 4장 + 7장

주의사항:
  ✓ Linux networking 스택 (socket, tc, ethtool) 실습 병행
  ✓ linuxptp 설치 및 ptp4l 실습 강력 권장
  ✓ Wireshark로 실제 프레임 캡처 및 분석
  ✗ 소프트웨어 타이머로 실시간을 흉내내는 함정 주의
```

### 유형 D: 의료기기 규제/인증 전문가

```
목표: 기술 이해 기반 규제 전문가
예상 학습 기간: 2~3주 (기술 심도 선택적)

권장 경로:
Week 1: 0장 전체 + 1장 Determinism
Week 2: 5장 (안전/규제 핵심) + 0장 0.4 상세 복습
Week 3: 3장 (TSN 개요 수준) + 7장 사례

주의사항:
  ✓ 정량적 요구사항 (지연, 지터 수치) 이해 중요
  ✓ FMEA 표 작성 방법 실습
  ✗ 모든 기술 세부사항 암기 필요 없음 - 개념과 영향도 이해 중심
```

---

## 4. 반드시 알아야 할 핵심 개념 30개

이 문서를 읽고 나면 다음 30개 개념을 자신 있게 설명할 수 있어야 합니다:

```
Ethernet 기초 (1-10):
  1.  Ethernet 프레임 구조 (7개 필드 + IFG)
  2.  직렬화 지연 계산 공식
  3.  EtherType과 주요 값 (0x0800, 0x88F7, 0x8100)
  4.  VLAN 태그 구조 (TPID, PCP, DEI, VID)
  5.  VLAN Hopping 공격과 방어
  6.  HOL (Head-of-Line) Blocking
  7.  PFC (Priority Flow Control)
  8.  CBS (Credit-Based Shaper) 원리
  9.  Collision Domain vs Broadcast Domain
  10. EEE (Energy Efficient Ethernet) 위험

TSN 핵심 (11-20):
  11. gPTP (802.1AS) 동기화 원리
  12. TAS (802.1Qbv) Gate Control List
  13. Guard Band 계산
  14. Frame Preemption (802.1Qbu)
  15. PSFP (802.1Qci) 스트림 필터링
  16. FRER (802.1CB) 이중화
  17. SR Class A vs Class B
  18. WCRT (Worst-Case Response Time) 분석
  19. Hardware Timestamping 필요성
  20. cyclictest와 latency 측정

의료/안전 (21-30):
  21. IEC 62304 Class A/B/C 차이
  22. ISO 14971 위험 관리 5단계
  23. 네트워크 FMEA 작성 방법
  24. Fail-Safe vs Fail-Operational
  25. Watchdog Timer 네트워크 적용
  26. IEC 62443 Zone & Conduit 모델
  27. FDA 사이버보안 가이드라인 핵심
  28. SBOM (Software Bill of Materials)
  29. 직렬화 지연이 수술 정밀도에 미치는 영향
  30. Best-Effort vs TSN 지연 비교 수치
```

---

## 5. 실습 환경 구성 가이드

### 5.1 소프트웨어 실습 (무료, 즉시 가능)

```bash
# Ubuntu 22.04 LTS 권장

# 1. Linux 네트워크 스택 실습
sudo apt install -y iproute2 ethtool wireshark tcpdump

# 2. linuxptp (gPTP 구현)
sudo apt install -y linuxptp
sudo ptp4l -i eth0 -m -f /etc/linuxptp/gPTP.cfg &
sudo phc2sys -s eth0 -c CLOCK_REALTIME -m &

# 3. Linux Traffic Control (TSN 유사 QoS)
# taprio qdisc (TAS 유사)
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1000000000 \
    sched-entry S 0x80 100000 \
    sched-entry S 0x78 900000 \
    flags 0x2

# 4. 네트워크 지연 측정
sudo apt install -y rt-tests
sudo cyclictest -m -p 99 -i 1000 -d 0 -n -h 100 -l 10000

# 5. 패킷 손실 시뮬레이션 (테스트용)
sudo tc qdisc add dev eth0 root netem delay 1ms 100us loss 0.1%

# 6. Wireshark 필터로 PTP 프레임 분석
# wireshark 필터: eth.type == 0x88f7

# 7. ROS 2 Humble 설치 (Ubuntu 22.04)
sudo apt install -y ros-humble-desktop
```

### 5.2 하드웨어 실습 (권장)

```
최소 구성 (약 30-50만원):
  ● Raspberry Pi 4B × 2대 (PTP 지원, $35×2)
  ● TSN 지원 스위치 (저예산: Marvell 88Q5050 기반)
  ● Cat6A UTP 케이블 × 3m
  ● Linux RT-Preempt 커널 적용

권장 구성 (약 100-200만원):
  ● Intel NUC × 2대 (Intel I225 NIC, PTP 하드웨어 지원)
  ● TSN 스위치: Cisco IE-3400H 또는 Moxa PT-7528
  ● 오실로스코프 (지터 측정용)

확인 가능 항목:
  ✓ gPTP 동기화 정밀도 측정 (< 100ns 목표)
  ✓ TAS 활성화 전후 지연/지터 비교
  ✓ Frame Preemption 효과 측정
  ✓ 네트워크 이중화 (FRER) 절체 시간 측정
```

---

## 6. 추가 학습 자원

### 6.1 공식 표준 문서

```
IEEE 표준 (유료, 기관/대학 접근 가능):
  ● IEEE 802.1AS-2020 (gPTP)
  ● IEEE 802.1Qbv-2015 (TAS)
  ● IEEE 802.1Qbu-2016 (Frame Preemption)
  ● IEEE 802.1CB-2017 (FRER)
  ● IEEE 802.1Qci-2017 (PSFP)

IEC 표준 (유료):
  ● IEC 62304:2006/AMD1:2015 (의료 SW)
  ● IEC 62443-3-3:2013 (산업 보안)
  ● ISO 14971:2019 (위험 관리)
```

### 6.2 무료 학습 자료

```
IEEE TSN 공식 자료:
  ● IEEE 802.1 TSN 태스크 그룹 공개 자료
    https://1.ieee802.org/tsn/

Linux TSN 구현:
  ● linuxptp 프로젝트: http://linuxptp.sourceforge.net/
  ● Linux Foundation Networking: https://wiki.linuxfoundation.org/networking/

Wireshark 분석:
  ● IEEE 1588 PTP 해석 플러그인 포함
  ● SampleCaptures에서 PTP, TSN 패킷 샘플 제공

ROS 2 실시간:
  ● ROS 2 Real-time Guide: https://docs.ros.org/en/rolling/Tutorials/Advanced/Real-Time-Programming.html
  ● micro-ROS (임베디드 실시간): https://micro.ros.org/

의료기기 규제:
  ● FDA 의료기기 사이버보안 가이드라인 (2023) - 무료 다운로드
  ● IMDRF 가이드라인 - 무료 다운로드
  ● IEC 62304 요약 튜토리얼 (각종 컨설팅 회사 무료 제공)
```

### 6.3 추천 서적

```
네트워크 기초:
  ● "Computer Networks" - Tanenbaum & Wetherall (5판)
  ● "TCP/IP Illustrated" - Stevens (Vol 1-3)

실시간 시스템:
  ● "Real-Time Systems" - Jane Liu
  ● "Embedded Real-Time Systems" - Rajib Mall

TSN 전문:
  ● "Time-Sensitive Networking for Flexible Automation" - Andreas Kern (2020)
  ● IEEE TSN Application Guide (무료, IEEE 802.1 공개)

의료 로봇:
  ● "Surgical Robotics: Science and Systems" - MIT Press
  ● "Medical Robotics" - Nabil Simaan (Springer)

안전 시스템:
  ● "Safety-Critical Computer Systems" - Neil Storey
  ● "Functional Safety" - IEC 61508 설명서
```

---

## 7. 이 문서의 활용 방법

### 7.1 오프라인 학습 시

```
장별 읽기 → 자가 점검 → 실습 → 다음 장
                ↓
          이해 부족 시 이전 장 참조 → 재확인
```

### 7.2 프로젝트 진행 중 참고 시

```
문제 상황            참조 위치
──────────────────────────────────────────────────────
지연이 크다          → 1장 직렬화 지연, 3장 TAS
패킷이 손실된다       → 1장 Flow Control, 3장 FRER
PTP 동기화 안된다    → 3장 gPTP, 2장 하드웨어 타임스탬프
규제 문서 작성       → 5장, 0장 0.4
VLAN 설계           → 1장 VLAN, 5장 보안
ROS 2 지연 이슈     → 6장 DDS QoS, 3장 TSN 연동
```

### 7.3 면접 준비 시

```
포지션별 핵심 주제:
  네트워크 엔지니어: 1장 + 3장 + 2장 (하드웨어 구성)
  임베디드 개발자:   1장 + 3장 + 6장 (Linux PTP, tc)
  안전 엔지니어:     5장 + 0장 0.4 + 3장 FRER
  시스템 설계자:     0장 전체 + 3장 + 5장 + 7장
```

---

## 8. 마치며: 이 여정의 의미

```
당신이 이 문서를 통해 배우는 것은 단순한 네트워크 기술이 아닙니다.

Ethernet 프레임 하나가 수술실 로봇 팔에 도달하는 과정,
1ms의 지연이 외과의의 손 감각에 미치는 영향,
패킷 하나의 손실이 환자 안전에 어떤 위험을 초래하는지—

이 모든 것을 이해하는 엔지니어가 되는 여정입니다.

기술은 도구입니다.
그 도구를 언제, 왜, 어떻게 사용해야 하는지 아는 엔지니어—
그것이 이 문서가 지향하는 최종 목표입니다.

─────────────────────────────────────
"In medical robotics, reliability is not a feature.
 It is a prerequisite."
─────────────────────────────────────
```

---

## 참고: 장별 학습 시간 추정

```
장     제목                        분량    권장 시간
──────────────────────────────────────────────────
0장    Ethernet 전환 배경           5개 절  4~6시간
1장    Ethernet 기초               10개 절  10~15시간 ← 가장 많은 투자
2장    네트워크 장비                6개 절   6~8시간
3장    TSN                        8개 절  12~18시간 ← 가장 어려운 장
4장    산업용 Ethernet 프로토콜    6개 절   6~8시간
5장    기능 안전과 사이버 보안      7개 절   8~10시간
6장    ROS 2와 DDS                5개 절   6~8시간
7장    산업 사례와 미래             4개 절   4~6시간
──────────────────────────────────────────────────
합계                                       56~79시간
                          (하루 2시간 기준 약 4~7주)
```

---

*이 절로 0장 '소개 및 배경'을 마칩니다.*
*다음 단계: [1장 Ethernet 기초](../1장_Ethernet_기초/OSI_7_Layer.md) 로 이동하세요.*
