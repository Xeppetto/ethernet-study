# Single Pair Ethernet (SPE)

## 개요
전통적인 Ethernet(RJ45)은 4쌍(8선)의 구리선을 사용하지만, 차량·산업·의료 로봇 환경에서는 경량화와 비용 절감을 위해 **단일 쌍(1쌍=2선)**으로 통신하는 SPE(Single Pair Ethernet) 기술이 도입되었습니다.

SPE는 기존 Ethernet과 동일한 IP 프로토콜 스택을 사용하면서 L1(물리 계층)만 변경됩니다. 이를 통해 CAN, LIN 등 레거시 버스를 IP 기반 네트워크로 대체할 수 있습니다.

---

## SPE 표준 비교

| 표준 | IEEE 규격 | 속도 | 거리 | 토폴로지 | Power over Data Line | 용도 |
|------|-----------|------|------|----------|----------------------|------|
| 10BASE-T1S | 802.3cg | 10 Mbps | 25 m | 멀티드롭(버스) | 선택적(PoDL) | 센서/액추에이터 버스 |
| 10BASE-T1L | 802.3cg | 10 Mbps | 1000 m | P2P | PoDL (최대 52W) | 필드 장비 (공장/빌딩) |
| 100BASE-T1 | 802.3bw | 100 Mbps | 15 m | P2P | PoDL (최대 50W) | 차량 카메라/센서 |
| 1000BASE-T1 | 802.3bp | 1 Gbps | 15 m | P2P | PoDL | 차량 고속 링크 |
| 2.5/5GBASE-T1 | 802.3ch | 2.5/5 Gbps | 15 m | P2P | 미지원 | 차세대 차량 백본 |

---

## 10BASE-T1S (멀티드롭 버스)

기존 CAN 버스를 대체할 수 있는 유일한 SPE 표준입니다.

### 토폴로지
```
     ┌──────────────────────────────────────────────┐
     │                    버스 케이블 (최대 25m)        │
     │                                              │
  ───┼────┬──────────────┬──────────────┬───────────┼───
  종단    │              │              │          종단
  저항   ECU #1         ECU #2         ECU #3     저항
 (100Ω) (IP 기반)      (IP 기반)      (IP 기반)  (100Ω)
         최대 8개 노드 연결 가능
```

### PLCA (Physical Layer Collision Avoidance)
멀티드롭에서 동시 전송 충돌을 방지하는 메커니즘입니다. CAN의 CSMA/CD 대신 **라운드로빈 타임슬롯 방식**으로 동작합니다.

```
Cycle T=0       T=1       T=2       T=3       ...
        │         │         │         │
  Beacon│         │         │         │
 ───────►         │         │         │
  ECU#0  ─전송─►  │         │         │
         ECU#1  ─전송─►     │         │
                   ECU#2  ─전송─►     │
                           ECU#3  ─전송─►
```

- **Beacon**: ECU #0(Coordinator)이 매 사이클 시작 신호 전송
- **Transmit Opportunity**: 각 노드는 자신의 순서에만 전송
- 결과: 최대 지연 = 노드 수 × 타임슬롯 크기 (예측 가능)

### 10BASE-T1S vs CAN 비교
| 항목 | 10BASE-T1S | CAN FD |
|------|-----------|--------|
| 속도 | 10 Mbps | 최대 8 Mbps |
| 프로토콜 스택 | IP 기반 (표준 네트워크) | CAN 전용 |
| 노드 수 | 최대 8개 | 최대 32개 |
| 거리 | 최대 25m | 최대 40m (1Mbps) |
| 진단 | DoIP, 표준 도구 사용 가능 | 전용 도구 필요 |
| 보안 | TLS/IPsec 적용 가능 | 제한적 |
| 전력 공급 | PoDL (선택) | 불가 |

---

## 100BASE-T1 (차량/로봇용 포인트-투-포인트)

### 특징
- **3레벨 PAM-3 변조**: 기존 100BASE-TX의 MLT-3과 달리, 단일 쌍으로 Full Duplex 실현
- **에코 캔슬레이션**: 동일 쌍에서 동시 송수신 가능하게 하는 신호 처리 기법
- **커넥터**: IEC 63171-6 (H-MTD, MATE-AX 등 차량용 커넥터)

### 신호 변조 방식 비교
```
100BASE-TX (일반 Ethernet, 4쌍):
  TX+/TX- 쌍: 송신
  RX+/RX- 쌍: 수신
  나머지 2쌍: 기가비트 이상에서 사용

100BASE-T1 (단일 쌍):
  단일 쌍: 송신 + 수신 동시 (에코 캔슬레이션으로 분리)
  3레벨 신호: -1, 0, +1 (PAM-3)
```

---

## PoDL (Power over Data Line)

SPE 케이블로 데이터와 전력을 동시에 공급하는 기능입니다.

```
PSE (Power Sourcing Equipment)          PD (Powered Device)
  예: ECU, 스위치 포트                    예: 센서, 카메라, 액추에이터
         │                                      │
         │──── 데이터 + 전력 (SPE 1쌍) ─────────►│
         │                                      │
         │  전압: 12V ~ 50V DC                   │
         │  전력: 클래스에 따라 0.5W ~ 52W        │
```

### PoDL 클래스
| 클래스 | 최대 전력 | 활용 사례 |
|--------|---------|-----------|
| 10 | 0.5W | 저전력 센서 |
| 11 | 2W | MCU가 없는 단순 센서 |
| 12 | 5W | 소형 카메라 |
| 13 | 15W | 고성능 센서 |
| 15 | 52W | 소형 액추에이터 |

---

## 케이블과 커넥터

### 케이블 사양
```
SPE 케이블 (STP: Shielded Twisted Pair)
  ┌─────────────────────────────────┐
  │  ┌───┐           ┌───────────┐  │
  │  │ + │──────────►│ Shield    │  │
  │  │ - │◄──────────│ (실드)    │  │
  │  └───┘           └───────────┘  │
  │   꼬임 쌍         외부 실드     │
  └─────────────────────────────────┘
  임피던스: 100Ω ±15%
  최대 정전용량: 100 pF/m
```

### 커넥터 표준
| 커넥터 | 표준 | 적용 환경 |
|--------|------|-----------|
| IEC 63171-1 | 산업용 | IP20 (실내) |
| IEC 63171-2 | M8 | 소형 산업 |
| IEC 63171-5 | M12 | IP65/67 방수 |
| IEC 63171-6 | H-MTD/MATE-AX | 차량/의료 로봇 |

---

## 의료 로봇 적용 시나리오

### 수술 로봇 내부 배선 최적화
```
기존 구조 (CAN + 표준 Ethernet 혼용):
  Main ECU ──CAN────► 조인트 제어 #1 (4쌍 케이블)
  Main ECU ──CAN────► 조인트 제어 #2 (4쌍 케이블)
  Main ECU ──GigE───► 카메라 (4쌍 케이블)
  총 케이블 무게: 상대적으로 높음

SPE 기반 신구조:
  Main ECU ──100BASE-T1──► 조인트 제어 #1 (1쌍, PoDL로 전원 공급)
  Main ECU ──100BASE-T1──► 조인트 제어 #2 (1쌍, PoDL)
  Main ECU ──1000BASE-T1──► 카메라 (1쌍)
  10BASE-T1S 버스──► 센서 그룹 (1쌍, 최대 8개 센서 연결)

  이점: 케이블 수 감소, 무게 최대 50% 절감, IP 기반 통합 관리
```

### 진단 개선 효과
```
CAN 버스 진단:
  - 전용 CANalyzer, Vector CANalyzer 필요
  - 복잡한 DBC 파일 관리 필요

SPE (IP 기반) 진단:
  - Wireshark로 패킷 캡처 가능
  - DoIP (ISO 13400)로 표준화된 진단
  - iperf3로 대역폭 측정
  - SSH로 원격 접속 가능
```

---

## Linux에서 SPE 인터페이스 설정

```bash
# SPE 인터페이스 링크 속도 확인 (100BASE-T1)
ethtool eth1
# Settings for eth1:
#   Speed: 100Mb/s
#   Duplex: Full
#   Port: MII
#   Auto-negotiation: off  (100BASE-T1은 자동협상 미지원)

# 10BASE-T1S PLCA 설정 확인
ethtool --show-plca eth2
# PLCA node id: 1
# PLCA enabled: yes
# PLCA burst count: 0
# PLCA burst timer: 128 (0x80)
# PLCA max burst count: 0

# PoDL 전원 상태 확인 (PSE 측)
ethtool --show-pse eth1
# PSE Allocated power: 15W
# PSE PD class: Class 13
```

---

## Reference
- [IEEE 802.3cg-2019 - 10 Mb/s over Single Balanced Pair (10BASE-T1S, 10BASE-T1L)](https://standards.ieee.org/ieee/802.3cg/7438/)
- [IEEE 802.3bw-2015 - 100BASE-T1](https://standards.ieee.org/ieee/802.3bw/5447/)
- [IEEE 802.3bp-2016 - 1000BASE-T1](https://standards.ieee.org/ieee/802.3bp/5614/)
- [IEEE 802.3ch-2020 - Multi-Gig Automotive Ethernet](https://standards.ieee.org/ieee/802.3ch/6714/)
- [OPEN Alliance - Automotive Ethernet Specifications](https://www.opensig.org/about/specifications/)
- [IEC 63171 - Connectors for SPE](https://webstore.iec.ch/publication/62405)
