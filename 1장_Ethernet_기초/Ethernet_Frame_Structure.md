# Ethernet 프레임 구조 심화 (Ethernet Frame Structure - Deep Dive)

## 개요

Ethernet 프레임은 L2 통신의 기본 단위입니다. TSN(Time-Sensitive Networking)의 스케줄링, Frame Preemption, 지연 계산은 모두 프레임 구조와 크기에 직접적으로 의존합니다. 이 문서는 실무 설계와 디버깅에 필요한 Ethernet 프레임의 모든 필드를 깊이 있게 다룹니다.

---

## 1. 프레임 전체 구조 (Wire 레벨)

케이블 상에서 실제로 전송되는 순서대로 표현한 완전한 구조입니다.

```
Wire 전송 순서 (왼쪽 → 오른쪽)
┌──────────┬─────┬──────────┬──────────┬──────────────┬──────────────────┬──────┬──────┐
│ Preamble │ SFD │  Dst MAC │  Src MAC │  EtherType/  │     Payload      │ FCS  │ IFG  │
│  7 Byte  │1Byte│  6 Byte  │  6 Byte  │    Length    │  46 ~ 1500 Byte  │4 Byte│12byte│
│          │     │          │          │    2 Byte    │  (pad if < 46)   │      │      │
└──────────┴─────┴──────────┴──────────┴──────────────┴──────────────────┴──────┴──────┘
                 ◄─────────── 이 부분이 "Ethernet 프레임" (Preamble/SFD/IFG 제외) ──────────►
                 최소 64 Byte                              최대 1518 Byte (VLAN 없을 때)
```

> **핵심**: Preamble과 SFD는 프레임의 일부처럼 보이지만, IEEE 802.3 정의에서는 프레임 크기 계산에서 제외됩니다. IFG도 마찬가지입니다.

---

## 2. 각 필드 상세 설명

### 2.1 Preamble (7 Byte)

```
비트 패턴: 10101010 10101010 10101010 10101010 10101010 10101010 10101010
           (0xAA × 7)
```

**목적**: 수신 측 PHY(물리 계층)가 클록 동기화(Clock Recovery)를 수행할 시간을 제공합니다.

- 수신 측 PLL(Phase-Locked Loop)이 이 신호를 이용해 전송 클록에 동기를 맞춥니다.
- 100Mbps 기준: 7 × 8 / 100M = 0.56 µs 동안 전송됩니다.
- **TSN 함의**: Preamble도 링크 점유 시간에 포함됩니다. 대역폭 계산 시 고려해야 합니다.

### 2.2 SFD (Start Frame Delimiter, 1 Byte)

```
비트 패턴: 10101011  (0xAB)
                 ↑
          마지막 비트가 '1'로 바뀜 → 실제 프레임 시작 신호
```

**목적**: Preamble과 구분하여 실제 프레임의 시작 위치를 표시합니다.

### 2.3 Destination MAC Address (6 Byte)

```
┌──────────────────────────┬─────────────────────────────┐
│     OUI (3 Byte)         │   Device Identifier (3 Byte) │
│   (제조사 식별자)         │     (장치 고유 번호)          │
└──────────────────────────┴─────────────────────────────┘

첫 번째 바이트의 비트 의미:
  bit 0 (LSB): 0 = Unicast,  1 = Multicast/Broadcast
  bit 1:       0 = Globally Unique (OUI 기반), 1 = Locally Administered
```

**특수 목적지 MAC 주소:**

| MAC 주소 | 용도 | 관련 프로토콜 |
|---------|------|------------|
| `FF:FF:FF:FF:FF:FF` | L2 Broadcast | ARP, DHCP |
| `01:00:5E:xx:xx:xx` | IPv4 Multicast | IGMP, DDS, SOME/IP SD |
| `33:33:xx:xx:xx:xx` | IPv6 Multicast | NDP |
| `01:1B:19:00:00:00` | IEEE 1588 PTP Event | gPTP (TSN 시간 동기화) |
| `01:80:C2:00:00:0E` | LLDP | 토폴로지 발견 |
| `01:80:C2:00:00:00` | STP/RSTP BPDU | 루프 방지 |
| `01:80:C2:00:00:02` | IEEE 802.3x PAUSE | 흐름 제어 |
| `01:00:5E:00:01:81` | IEEE 1722 (AVTP) | 오디오/비디오 스트리밍 |

### 2.4 Source MAC Address (6 Byte)

항상 **Unicast 주소**여야 합니다 (bit 0 = 0). Multicast/Broadcast 주소는 송신지로 사용 불가.

```bash
# 현재 MAC 주소 확인
ip link show eth0 | grep "link/ether"
# 예: link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff

# OUI 조회
curl -s "https://api.macvendors.com/AA:BB:CC" 2>/dev/null
# 또는 로컬 OUI 데이터베이스 활용
```

### 2.5 EtherType / Length (2 Byte)

이 필드는 값의 범위에 따라 두 가지 의미를 가집니다.

```
값 < 0x0600 (1536):  IEEE 802.3 Length 필드 → 페이로드 크기(Byte) 표시
값 ≥ 0x0600 (1536):  DIX EtherType → 상위 프로토콜 식별자
```

**주요 EtherType 값:**

| EtherType | 16진수 | 프로토콜 | 용도 |
|-----------|--------|---------|------|
| 2048 | `0x0800` | IPv4 | 표준 IP 패킷 |
| 2054 | `0x0806` | ARP | MAC 주소 해석 |
| 34525 | `0x86DD` | IPv6 | 차세대 IP |
| 33024 | `0x8100` | 802.1Q VLAN | VLAN 태그 (TPID) |
| 34984 | `0x88A8` | 802.1ad | QinQ Provider Bridge |
| 34930 | `0x88B2` | EtherCAT | 모션 제어 필드버스 |
| 34962 | `0x88D2` | PROFINET RT | 공장 자동화 |
| 35063 | `0x88F7` | IEEE 1588v2 PTP | **시간 동기화 (TSN 핵심)** |
| 35020 | `0x88CC` | LLDP | 링크 레이어 발견 |
| 34824 | `0x8808` | Ethernet Flow Control | PAUSE 프레임 |
| 8784 | `0x2250` | IEEE 1722 AVTP | 오디오/비디오 전송 |

> **TSN 설계 포인트**: `0x88F7` (PTP)는 L2 멀티캐스트로 직접 전송됩니다. IP 스택을 거치지 않아 지연이 최소화됩니다.

### 2.6 Payload (데이터, 46 ~ 1500 Byte)

```
최솟값 = 46 Byte 이유:
  최소 프레임 크기 = 64 Byte (CSMA/CD 충돌 감지를 위한 최솟값)
  64 - (Dst MAC 6 + Src MAC 6 + Type 2 + FCS 4) = 64 - 18 = 46 Byte

페이로드 < 46 Byte인 경우: 패딩(0x00)으로 46 Byte까지 채움
실제 데이터 크기: 상위 프로토콜 헤더(IP Length 등)에서 확인
```

```
최댓값 = 1500 Byte 이유:
  역사적으로 DIX Ethernet(DEC-Intel-Xerox)이 정한 값
  MTU(Maximum Transmission Unit) = 1500 Byte (IP 계층 관점)
```

**페이로드 구성 예시 (IPv4/UDP/SOME-IP):**
```
Ethernet Payload (최대 1500 Byte)
├── IPv4 Header (20 Byte 최소)
│   └── UDP Header (8 Byte)
│       └── SOME/IP Header (16 Byte)
│           └── SOME/IP Payload (최대 1500-20-8-16 = 1456 Byte)
```

### 2.7 FCS (Frame Check Sequence, 4 Byte)

**CRC-32 (Cyclic Redundancy Check)**를 사용합니다.

```
검사 대상: Dst MAC + Src MAC + EtherType + Payload (Preamble/SFD/IFG 제외)
생성 다항식: x³² + x²⁶ + x²³ + x²² + x¹⁶ + x¹² + x¹¹ + x¹⁰ + x⁸ + x⁷ + x⁵ + x⁴ + x² + x + 1
            (= 0x04C11DB7)

검출 능력:
  ✓ 1비트 오류: 100% 검출
  ✓ 2비트 오류: 100% 검출
  ✓ 홀수 비트 오류: 100% 검출
  ✓ 32비트 이하 버스트 오류: 100% 검출
  △ 33비트 이상 버스트 오류: 확률적 검출 (99.99999977% 검출률)
```

```bash
# CRC 오류 통계 확인
ethtool -S eth0 | grep -E "crc|fcs|error"
# rx_crc_errors: FCS 불일치 → 케이블 손상, EMI 간섭, 커넥터 불량 의심

# Wireshark에서 FCS 검사 활성화
# Edit → Preferences → Protocols → Ethernet → Validate the Ethernet checksum
```

### 2.8 IFG (Inter Frame Gap, 최소 12 Byte = 96 비트 시간)

프레임과 프레임 사이의 **최소 휴지 시간**입니다.

```
목적:
  1. 수신 측 PHY가 프레임 처리 후 다음 프레임을 받을 준비 시간 확보
  2. 클록 복구(Clock Recovery)를 위한 전환 시간
  3. CSMA/CD 환경에서의 안정성 (현재는 레거시)

IFG 지속 시간:
  10 Mbps:   96 / 10M = 9.6 µs
  100 Mbps:  96 / 100M = 0.96 µs
  1 Gbps:    96 / 1G = 96 ns
  10 Gbps:   96 / 10G = 9.6 ns
```

> **TSN 설계 주의**: IFG는 대역폭 계산에 포함해야 합니다. 1Gbps 링크에서 최소 프레임(64byte)을 연속 전송 시 실제 처리량은 이론치의 ~94%에 불과합니다.

---

## 3. 프레임 크기와 크기별 의미

### 3.1 프레임 크기 범위

| 프레임 크기 (Byte) | 명칭 | 설명 |
|-------------------|------|------|
| < 64 | Runt Frame | 최소 크기 미달, 오류 프레임 → 폐기 |
| 64 | 최소 정상 프레임 | ARP Reply, ICMP(소형), VLAN 태그 없는 경우 |
| 65 ~ 1518 | 정상 프레임 | 일반 유니캐스트 데이터 |
| 1519 ~ 1522 | VLAN Tagged Max | 802.1Q 태그 포함 최대 크기 |
| 1523 ~ 1526 | QinQ Max | 이중 태그 포함 최대 크기 |
| > 1518 (표준) | Jumbo Frame | 비표준, 최대 9000 Byte (설정 필요) |
| > 1518 (IEEE 정의) | Giant/Oversized Frame | 오류 프레임 → 폐기 (Jumbo 미설정 시) |

### 3.2 최소 프레임 크기 64 Byte의 물리적 의미

```
CSMA/CD 기반 설계 (10Mbps, 최대 200m 케이블 세그먼트):

전파 지연: 200m / (200,000 km/s) = 200m / (2 × 10⁸ m/s) ≈ 1 µs (단방향)
왕복 지연: 2 µs (최악의 경우)
10Mbps에서 2µs = 2µs × 10Mbps = 20 bits → 안전 마진 포함 → 64 Byte (512 bits)

의미: 송신 노드가 최소 64 Byte를 전송하는 동안 충돌 발생 여부를 감지할 수 있어야 함
```

> **현대 의의**: Full Duplex 환경에서는 충돌이 없으므로 64 Byte 제한은 레거시입니다. 그러나 TSN에서 Frame Preemption(802.1Qbu) 구현 시 mPacket(최소 124 Byte)을 고려해야 합니다.

### 3.3 Jumbo Frame

```
표준 MTU:    1500 Byte (Ethernet payload)
Jumbo MTU:  9000 Byte (비표준, 구현 의존)

장점:
  ✓ 동일 데이터 전송 시 프레임 수 감소 → 오버헤드 절감
  ✓ CPU 인터럽트 횟수 감소 → 서버 처리량 향상
  ✓ 대용량 파일 전송 효율 향상

단점/위험:
  ✗ 경로 상 모든 장비가 지원해야 함 (일부 지원 안 하면 패킷 손실)
  ✗ 지연 증가: 9000 Byte @ 1Gbps = 72 µs (1518 Byte의 약 6배)
  ✗ TSN과 충돌: 9000 Byte 프레임은 TAS(802.1Qbv) 타임 슬롯 초과 가능
  ✗ 실시간 제어에 부적합

TSN 환경 권장: Jumbo Frame 사용 금지, 프레임 크기 최소화
```

---

## 4. VLAN Tagged 프레임 (IEEE 802.1Q)

```
표준 Ethernet 프레임 (1518 Byte max):
┌─────────────┬─────────────┬────────┬──────────────────┬──────┐
│  Dst MAC    │  Src MAC    │  Type  │     Payload      │ FCS  │
│   6 Byte    │   6 Byte    │ 2 Byte │  46 ~ 1500 Byte  │4 Byte│
└─────────────┴─────────────┴────────┴──────────────────┴──────┘

802.1Q Tagged 프레임 (1522 Byte max):
┌─────────────┬─────────────┬─────────────────────┬────────┬─────────────────┬──────┐
│  Dst MAC    │  Src MAC    │    802.1Q Tag        │  Type  │    Payload      │ FCS  │
│   6 Byte    │   6 Byte    │       4 Byte         │ 2 Byte │ 46 ~ 1496 Byte  │4 Byte│
└─────────────┴─────────────┴──────────┬──────────┴────────┴─────────────────┴──────┘
                                        │
                           ┌────────────▼───────────────┐
                           │  TPID (2B)  │  TCI (2B)   │
                           │  0x8100     │ PCP│DEI│VID  │
                           │             │ 3b │1b │12b  │
                           └────────────┴────┴───┴─────┘
                                          │    │   │
                                          │    │   └─ VLAN ID (0~4094)
                                          │    └───── Drop Eligible Indicator
                                          └────────── Priority Code Point (0~7, IEEE 802.1p)
```

**MTU 주의사항:**
```
문제 시나리오:
  호스트 MTU = 1500 Byte (기본값)
  VLAN 태그 4 Byte 추가 → 실제 프레임 = 1504 Byte Payload 전달 불가

해결책 1: 스위치가 VLAN 태그를 투명하게 처리 (Access Port에서 태그 추가/제거)
해결책 2: 호스트 MTU를 1496 Byte로 줄이기 (비권장)
해결책 3: 스위치 MTU를 1522 Byte 이상으로 설정
```

---

## 5. QinQ 프레임 (IEEE 802.1ad)

이중 VLAN 태그를 사용하는 구조입니다. 병원 네트워크에서 다중 로봇 팜 관리에 활용됩니다.

```
QinQ 프레임 (최대 1526 Byte):
┌──────┬──────┬───────────────┬───────────────┬──────┬─────────────┬──────┐
│ Dst  │ Src  │ Outer S-Tag   │ Inner C-Tag   │ Type │   Payload   │ FCS  │
│ MAC  │ MAC  │ (0x88A8)      │ (0x8100)      │      │             │      │
│ 6B   │ 6B   │ 4 Byte        │ 4 Byte        │ 2B   │46~1492 Byte │ 4B   │
└──────┴──────┴───────────────┴───────────────┴──────┴─────────────┴──────┘

활용 예:
  S-VLAN 100 = 병원 A 전체 로봇 팜
  C-VLAN 10  = 병원 A, 수술실 1, 안전 제어
  C-VLAN 20  = 병원 A, 수술실 1, 영상
```

---

## 6. 프레임 전송 지연 계산 (Serialization Delay)

**직렬화 지연(Serialization Delay)**: 프레임의 첫 비트부터 마지막 비트까지 링크에 올리는 데 걸리는 시간입니다.

```
공식: Serialization Delay = (Frame Size + Preamble + SFD + IFG) × 8 bits / Link Speed
     = (Frame Byte + 7 + 1 + 12) × 8 / Link Speed (bps)
     = (Frame Byte + 20) × 8 / Link Speed
```

### 6.1 속도별 직렬화 지연 비교표

| 프레임 크기 | 100 Mbps | 1 Gbps | 10 Gbps | 비고 |
|------------|---------|--------|---------|------|
| 64 Byte (최소) | 6.72 µs | 0.672 µs | 67.2 ns | ARP, 소형 제어 |
| 128 Byte | 11.84 µs | 1.184 µs | 118.4 ns | - |
| 256 Byte | 22.08 µs | 2.208 µs | 220.8 ns | - |
| 512 Byte | 42.56 µs | 4.256 µs | 425.6 ns | - |
| 1024 Byte | 83.52 µs | 8.352 µs | 835.2 ns | 일반 데이터 |
| 1518 Byte | 122.24 µs | 12.224 µs | 1.2224 µs | 표준 최대 |
| 9000 Byte (Jumbo) | 720.16 µs | 72.016 µs | 7.2 µs | Jumbo Frame |

> **TSN 설계 예시**: 1Gbps 링크, TAS Guard Band 계산 시 최대 프레임(1518 Byte) 직렬화 지연 12.2 µs를 반드시 고려해야 합니다.

### 6.2 지연 예산 (Latency Budget) 구성 요소

```
총 End-to-End 지연 = Σ (각 구간별 지연)

구간별 지연 종류:
  1. Serialization Delay    : 프레임을 링크에 올리는 시간
  2. Propagation Delay      : 신호가 케이블을 통과하는 시간 (≈ 5 ns/m for copper)
  3. Store-and-Forward Delay : 스위치가 전체 프레임 수신 후 전달 (= Serialization Delay)
  4. Queuing Delay           : 출력 큐에서 대기하는 시간 (비결정적, TSN으로 해결)
  5. Processing Delay        : 스위치 내부 MAC 주소 조회, 헤더 처리

예시 (1Gbps, 2-hop, 1518 Byte 프레임):
  링크 1 Serialization:   12.2 µs
  스위치 1 S&F:           12.2 µs  (수신 완료 후 전달)
  링크 1 Propagation:     0.05 µs  (10m 케이블)
  스위치 1 Processing:    1~5 µs   (스위치 성능 의존)
  링크 2 Serialization:   12.2 µs
  링크 2 Propagation:     0.05 µs
  ─────────────────────────────────
  총 지연:               ~38~42 µs (Queuing 없을 때)
  Queuing 추가 시:       수백 µs ~ 수 ms (TSN 없이)
```

---

## 7. PTP (IEEE 1588) 프레임과 타임스탬프

gPTP(IEEE 802.1AS)의 경우 L2 Ethernet 프레임으로 직접 전달되며, 하드웨어 타임스탬프가 핵심입니다.

```
PTP Event 프레임 (타임스탬프 필요):
  Dst MAC:    01:1B:19:00:00:00 (PTP 멀티캐스트)
  EtherType:  0x88F7
  PTP Header: messageType, domainNumber, sequenceId, ...
  Correction: 스위치 통과 시 Residence Time을 이 필드에 누적

하드웨어 타임스탬프 중요성:
  소프트웨어 타임스탬프: OS 스케줄링 지연으로 수 µs ~ 수십 µs 오차
  하드웨어 타임스탬프: PHY 레벨에서 캡처 → 수 ns 정밀도

```bash
# PTP 하드웨어 타임스탬프 지원 확인
ethtool -T eth0
# Time stamping parameters for eth0:
#   hardware-transmit     (TX 하드웨어 타임스탬프 지원)
#   hardware-receive      (RX 하드웨어 타임스탬프 지원)
#   hardware-raw-clock    (하드웨어 클록 사용 가능)
```

---

## 8. 프레임 분석: Wireshark 실습

### 8.1 주요 분석 필터

```
# 특정 EtherType 분석
eth.type == 0x88f7            # PTP 프레임 (TSN 시간 동기화)
eth.type == 0x8100            # VLAN Tagged 프레임
eth.type == 0x0800            # IPv4 프레임

# 프레임 크기 분석 (Jumbo 감지)
frame.len > 1518              # 표준 최대 크기 초과 프레임
frame.len < 64                # Runt Frame (오류)

# FCS 오류 감지 (FCS 검증 활성화 필요)
eth.fcs.status == "Bad"       # FCS 불일치 → 물리 계층 문제

# 브로드캐스트/멀티캐스트 비율 분석
eth.dst == ff:ff:ff:ff:ff:ff  # Broadcast
eth.dst[0] & 1 == 1           # 모든 Multicast (Broadcast 포함)

# Preamble/IFG를 포함한 Wire 통계
Statistics → I/O Graphs → 패킷 크기 분포 확인
```

### 8.2 프레임 캡처 및 통계 분석

```bash
# 특정 인터페이스의 프레임 크기 분포 수집
tshark -i eth0 -T fields -e frame.len -e eth.type 2>/dev/null | \
  awk '{size[$1]++; types[$2]++} END {for(s in size) print s, size[s]}' | sort -n

# 1초간 프레임 수 및 평균 크기 측정
tshark -i eth0 -a duration:10 -q -z io,stat,1 2>/dev/null

# CRC 오류 비율 모니터링
watch -n 1 'ethtool -S eth0 | grep -E "rx_crc|rx_errors|rx_frame"'
```

---

## 9. 산업/의료 로봇 적용 설계 지침

### 9.1 실시간 제어 트래픽 프레임 설계

```
의료 로봇 관절 제어 사이클 예시 (1ms 사이클):

제어 데이터 크기 결정:
  관절 위치 (float64 × 6) = 48 Byte
  관절 속도 (float64 × 6) = 48 Byte
  관절 토크 (float64 × 6) = 48 Byte
  타임스탬프 (uint64)      =  8 Byte
  상태 플래그 (uint32)     =  4 Byte
  ─────────────────────────────────
  실제 데이터:             156 Byte

헤더 포함 총 크기:
  UDP/IP:    156 + 8(UDP) + 20(IP) = 184 Byte
  Ethernet:  184 + 14(L2 헤더) + 4(VLAN) = 202 Byte
  (< 64 Byte 기준 없음, 최소 조건 충족)

1Gbps 직렬화 지연: (202 + 20) × 8 / 1G = 1.776 µs
1ms 사이클 대비: 0.18% 소비 → 허용 가능
```

### 9.2 안전 제어 프레임 무결성 요구사항

```
IEC 62304 / IEC 61508 관점에서 Ethernet 프레임의 안전 기능:

CRC-32 단독으로 충분한가?
  ✗ 불충분 (CRC는 우발적 오류 감지에 최적화)
  ✗ CRC는 악의적 변조 감지 불가
  ✗ 메시지 순서 오류 감지 불가
  ✗ 메시지 유실 감지 불가

추가 안전 메커니즘 (E2E Protection):
  ✓ 순서 번호 (Sequence Number): 패킷 유실/순서 오류 감지
  ✓ 타임스탬프: 오래된 메시지(Stale Data) 감지
  ✓ CRC 또는 체크섬: 상위 계층 추가 오류 검출
  ✓ HMAC/디지털 서명: 무결성 + 인증 (TLS 필요)
  → ISO 26262 E2E 프로파일 (PW-1a, PW-1b, PW-2, ...) 참조
```

---

## 10. 오류 프레임 분류 및 처리

| 오류 유형 | 정의 | 원인 | 스위치 처리 |
|---------|------|------|------------|
| Runt Frame | < 64 Byte | Duplex Mismatch, 충돌 잔재 | 무조건 폐기 |
| Giant Frame | > 1518 Byte (Jumbo 미설정) | 잘못된 MTU 설정 | 폐기 |
| FCS Error | CRC-32 불일치 | 케이블 불량, EMI 간섭, 커넥터 불량 | 폐기 |
| Alignment Error | 바이트 경계 비정렬 | PHY 오류, 케이블 불량 | 폐기 |
| Jabber | 매우 긴 전송 지속 | PHY 오작동 (Babbling Idiot) | PHY 차단 |

> **Jabber/Babbling Idiot**: 오작동한 노드가 정지 없이 프레임을 전송해 네트워크를 마비시키는 현상입니다. TSN의 PSFP(IEEE 802.1Qci)가 이를 방어합니다. 의료 로봇에서는 치명적 안전 위협이 될 수 있습니다.

---

## Reference
- [IEEE 802.3-2022 - Ethernet Standard](https://standards.ieee.org/ieee/802.3/10422/)
- [IEEE 802.1Q-2022 - VLAN Tagged Frame](https://standards.ieee.org/ieee/802.1Q/6844/)
- [IEEE 802.1ad - Provider Bridges (QinQ)](https://standards.ieee.org/ieee/802.1ad/3979/)
- [IEEE 802.1AS-2020 - gPTP for TSN](https://standards.ieee.org/ieee/802.1AS/6725/)
- [RFC 894 - A Standard for the Transmission of IP Datagrams over Ethernet Networks](https://datatracker.ietf.org/doc/html/rfc894)
- [Wireshark User's Guide - Ethernet](https://www.wireshark.org/docs/wsug_html_chunked/)
