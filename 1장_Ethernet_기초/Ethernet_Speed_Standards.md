# Ethernet 속도 표준 및 물리 계층 (PHY)

## 개요
Ethernet은 1970년대 10Mbps에서 시작하여 현재 400Gbps 이상으로 발전했습니다. 산업 및 차량 환경에서는 다양한 속도 표준이 공존하므로, 각 표준의 특성과 적용 사례를 이해하는 것이 중요합니다.

---

## Ethernet 속도 표준 발전 역사

```
1980  →  10BASE5 (두꺼운 동축 케이블, 최초 Ethernet)
1985  →  10BASE2 (얇은 동축 케이블)
1990  →  10BASE-T (UTP 케이블, RJ45 커넥터 시작)
1995  →  100BASE-TX (Fast Ethernet, 100Mbps)
1999  →  1000BASE-T (Gigabit Ethernet, UTP 4쌍)
2002  →  10GBASE-SR/LR (10Gbps, 광섬유)
2016  →  100BASE-T1, 1000BASE-T1 (차량용 SPE)
2019  →  10BASE-T1S, 10BASE-T1L (산업용 SPE)
2020  →  2.5GBASE-T1, 5GBASE-T1 (차량용 고속)
2022  →  400GBASE-SR8 (데이터센터)
2025+ →  800Gbps, Terabit Ethernet (연구 단계)
```

---

## 주요 Ethernet 표준 비교

### 일반 Ethernet (사무/데이터센터)

| 표준 | IEEE | 속도 | 매체 | 최대 거리 | 커넥터 |
|------|------|------|------|---------|--------|
| 10BASE-T | 802.3i | 10 Mbps | UTP Cat3 | 100 m | RJ45 |
| 100BASE-TX | 802.3u | 100 Mbps | UTP Cat5 | 100 m | RJ45 |
| 1000BASE-T | 802.3ab | 1 Gbps | UTP Cat5e | 100 m | RJ45 |
| 2.5GBASE-T | 802.3bz | 2.5 Gbps | UTP Cat5e | 100 m | RJ45 |
| 5GBASE-T | 802.3bz | 5 Gbps | UTP Cat6 | 100 m | RJ45 |
| 10GBASE-T | 802.3an | 10 Gbps | UTP Cat6a | 100 m | RJ45 |

### 광섬유 Ethernet

| 표준 | IEEE | 속도 | 매체 | 최대 거리 |
|------|------|------|------|---------|
| 100BASE-FX | 802.3u | 100 Mbps | MMF | 2 km |
| 1000BASE-SX | 802.3z | 1 Gbps | MMF | 550 m |
| 1000BASE-LX | 802.3z | 1 Gbps | SMF | 5 km |
| 10GBASE-SR | 802.3ae | 10 Gbps | MMF | 300 m |
| 10GBASE-LR | 802.3ae | 10 Gbps | SMF | 10 km |
| 40GBASE-SR4 | 802.3ba | 40 Gbps | MMF | 100 m |
| 100GBASE-SR4 | 802.3bm | 100 Gbps | MMF | 100 m |

### 차량/산업용 SPE Ethernet

| 표준 | IEEE | 속도 | 거리 | 토폴로지 | PoDL |
|------|------|------|------|---------|------|
| 10BASE-T1S | 802.3cg | 10 Mbps | 25 m | 멀티드롭 | 선택 |
| 10BASE-T1L | 802.3cg | 10 Mbps | 1000 m | P2P | 52W |
| 100BASE-T1 | 802.3bw | 100 Mbps | 15 m | P2P | 50W |
| 1000BASE-T1 | 802.3bp | 1 Gbps | 15 m | P2P | 선택 |
| 2.5GBASE-T1 | 802.3ch | 2.5 Gbps | 15 m | P2P | 미지원 |
| 5GBASE-T1 | 802.3ch | 5 Gbps | 15 m | P2P | 미지원 |

---

## 물리 계층 (PHY) 이해

PHY 칩은 MAC(Media Access Controller)과 물리 매체 사이에서 신호 변환을 담당합니다.

### PHY 내부 구조
```
   MAC 측                              케이블 측
┌────────┐   MII/GMII/RGMII   ┌──────────────────────┐
│  MAC   │◄──────────────────►│  PHY 칩               │
│(L2 처리)│                    │  ┌──────┐  ┌───────┐ │
└────────┘                    │  │PCS   │  │PMD    │ │────► 케이블
                              │  │(코딩) │  │(변조) │ │◄─── 케이블
                              │  └──────┘  └───────┘ │
                              └──────────────────────┘
MII 인터페이스 종류:
  MII (Media Independent Interface): 100Mbps
  GMII (Gigabit MII): 1Gbps
  RGMII (Reduced GMII): 1Gbps, 핀 수 절반
  SGMII (Serial GMII): 1Gbps, 직렬화
  XGMII (10G MII): 10Gbps
```

### 신호 변조 방식
| 표준 | 변조 방식 | 설명 |
|------|---------|------|
| 100BASE-TX | MLT-3 | 3레벨 전환 방식 |
| 1000BASE-T | PAM-5 | 5레벨 펄스 진폭 변조 |
| 10GBASE-T | LDPC + DSQ128 | 저밀도 패리티 검사 + 128 DSQ |
| 100BASE-T1 | PAM-3 | 3레벨, 단일 쌍 |
| 1000BASE-T1 | PAM-3 | 3레벨, 단일 쌍, 고속 |

---

## 자동 협상 (Auto-Negotiation)

### 동작 순서
```
장치 A                                    장치 B
   │                                          │
   │◄──── Fast Link Pulse (FLP) 교환 ────────►│
   │      (지원 능력 광고 비트맵)              │
   │                                          │
   │  A 능력: 10/100/1000 Mbps, Full/Half     │
   │  B 능력: 100/1000 Mbps, Full             │
   │                                          │
   │  협상 결과: 1000 Mbps Full Duplex         │
   └──────── 1000BASE-T 링크 수립 ────────────┘
```

### 협상 우선순위 (높을수록 선택됨)
```
1. 1000BASE-T Full Duplex    ← 최고 우선순위
2. 1000BASE-T Half Duplex
3. 100BASE-TX Full Duplex
4. 100BASE-T2 Full Duplex
5. 100BASE-TX Half Duplex
6. 10BASE-T Full Duplex
7. 10BASE-T Half Duplex      ← 최저 우선순위
```

---

## 케이블 카테고리와 주의사항

| 카테고리 | 최대 대역폭 | 지원 표준 | 실링 |
|---------|-----------|---------|------|
| Cat 5 | 100 MHz | 100BASE-TX | UTP |
| Cat 5e | 100 MHz | 1000BASE-T | UTP/STP |
| Cat 6 | 250 MHz | 5GBASE-T | UTP/STP |
| Cat 6a | 500 MHz | 10GBASE-T | UTP/STP |
| Cat 7 | 600 MHz | 10GBASE-T (100m) | S/FTP |
| Cat 8 | 2000 MHz | 40GBASE-T | S/FTP |

### 케이블 배선 규칙
```
TIA-568B 배선 표준 (표준 다이렉트 케이블):
핀 1: 흰/주황 (TX+)   핀 1: 흰/주황 (TX+)
핀 2: 주황   (TX-)   핀 2: 주황   (TX-)
핀 3: 흰/초록 (RX+)   핀 3: 흰/초록 (RX+)
핀 4: 파랑            핀 4: 파랑
핀 5: 흰/파랑         핀 5: 흰/파랑
핀 6: 초록   (RX-)   핀 6: 초록   (RX-)
핀 7: 흰/갈색         핀 7: 흰/갈색
핀 8: 갈색            핀 8: 갈색

※ 현대 스위치는 Auto-MDI/MDIX 지원으로 크로스/다이렉트 자동 감지
```

---

## 실무: PHY 문제 진단

```bash
# PHY 상태 확인 (ethtool)
ethtool eth0
# Supported link modes:
#   10baseT/Half 10baseT/Full
#   100baseT/Half 100baseT/Full
#   1000baseT/Full
# Advertised link modes: (위와 동일)
# Speed: 1000Mb/s
# Duplex: Full
# Auto-negotiation: on
# Link detected: yes

# PHY 레지스터 직접 읽기 (고급)
ethtool -d eth0 | head -20

# 케이블 진단 (일부 PHY 지원)
ethtool --cable-test eth0
# test started for device eth0
# cable test started

# 링크 이벤트 모니터링
ip monitor link
```

### 흔한 물리 계층 문제
| 증상 | 원인 | 확인 방법 |
|------|------|-----------|
| 링크 자주 끊김 | 케이블 불량/불완전 접촉 | `dmesg | grep eth0` 링크 이벤트 |
| 1000M → 100M 강등 | 케이블 Cat5 미달 | `ethtool eth0` Speed 확인 |
| CRC 오류 증가 | 케이블 손상/EMI | `ethtool -S eth0 \| grep rx_crc` |
| Duplex Mismatch | 한쪽 강제 설정 | `ethtool eth0` Duplex 확인 |
| 링크 안 올라옴 | MDI/MDIX 미지원 | 크로스 케이블 시도 |

---

## 속도별 적용 사례 (산업/의료)

```
센서 레이어 (저속, 다수 연결):
  10BASE-T1S (10Mbps, 멀티드롭) ← CAN 대체

관절/모듈 제어 (중속, P2P):
  100BASE-T1 (100Mbps, SPE) ← 경량, PoDL

영상/고속 센서:
  1000BASE-T1 (1Gbps, SPE) ← 카메라, 라이다

로봇 내부 백본:
  1000BASE-T / 2.5GBASE-T (표준 Ethernet)

클라우드/병원망 연결:
  10GBASE-T 이상 (고속, 광섬유 권장)
```

---

---

## 지연 예산 (Latency Budget) 계산

의료 로봇 제어망 설계 시 End-to-End 지연 예산을 사전에 계산해야 합니다.

### 지연 구성 요소와 공식

```
총 지연 = Σ (직렬화 지연 + 전파 지연) + Σ 스위치 처리 지연 + 큐잉 지연

1. 직렬화 지연 (Serialization Delay)
   T_ser = (Frame_Byte + 20) × 8 / Link_Speed_bps
   (20 Byte = Preamble 7 + SFD 1 + IFG 12)

2. 전파 지연 (Propagation Delay)
   구리 케이블: ≈ 5 ns/m  (신호 속도 ≈ 200,000 km/s)
   광섬유:      ≈ 5 ns/m  (굴절률에 따라 4~5 ns/m)

3. Store-and-Forward 스위치 지연
   T_sf = T_ser (입력 포트 직렬화 지연과 동일)

4. 큐잉 지연
   Best-Effort: 0 ~ 수 ms (혼잡 의존, 비결정적)
   TSN TAS 적용: 사전 계산된 최대값 (결정론적)
```

### 속도별 직렬화 지연 빠른 참조표

| 프레임 크기 | 100 Mbps | 1 Gbps | 10 Gbps | 용도 |
|------------|---------|--------|---------|------|
| 64 Byte   | 6.72 µs | 0.672 µs | 67.2 ns | ARP, 소형 제어 |
| 256 Byte  | 22.1 µs | 2.21 µs  | 221 ns  | 관절 제어 데이터 |
| 512 Byte  | 42.6 µs | 4.26 µs  | 426 ns  | 센서 묶음 |
| 1518 Byte | 122 µs  | 12.2 µs  | 1.22 µs | 표준 최대 |

### 의료 로봇 2-hop 지연 예산 예시

```
구성: 제어기 → TSN 스위치 → 관절 ECU (2 링크, 1Gbps, 256B 프레임)

  링크 1 직렬화: (256+20)×8/1G = 2.208 µs
  링크 1 전파:   20m × 5ns/m  = 0.100 µs
  SW1 S&F:       2.208 µs     (수신 완료 대기)
  SW1 처리:      1.000 µs     (TSN 스위치 내부)
  링크 2 직렬화: 2.208 µs
  링크 2 전파:   10m × 5ns/m  = 0.050 µs
  ─────────────────────────────────────────
  최소 지연 합:  7.774 µs     (큐잉 없을 때)

1ms 제어 사이클 대비: 7.774/1000 = 0.78% → 충분한 여유
TSN TAS WCRT 목표:   < 15 µs (안전 마진 포함)
```

---

## EEE (Energy Efficient Ethernet) - TSN 위험 요소

### EEE 개요 (IEEE 802.3az)

```
목적: 유휴 시 PHY를 LPI(Low Power Idle) 모드로 전환 → 50~80% 전력 절감

동작:
  트래픽 없음 → PHY LPI 진입
  트래픽 재개 → Wake-up 후 정상 동작

EEE 웨이크업 지연 (Tw):
  100BASE-TX: Tw ≈ 16 µs
  1000BASE-T: Tw ≈ 16.5 µs
  10GBASE-T:  Tw ≈ 4.48 µs
  100BASE-T1: 미지원 (설계적으로 EEE 없음)
```

### TSN에서 EEE 위험성

```
문제 시나리오:
  T=0: TAS 게이트 OPEN → 안전 제어 패킷 전송 시도
  T=0~16µs: PHY 웨이크업 중 → 전송 불가!
  T=16µs: PHY 준비 → 이미 타임슬롯 초과
  결과: 안전 제어 패킷 지연 또는 손실 → 제어 루프 파손

→ TSN 환경에서 EEE는 반드시 비활성화
```

```bash
# EEE 상태 확인 및 비활성화
ethtool --show-eee eth0
# EEE status: enabled  ← 비활성화 필요

ethtool --set-eee eth0 eee off

# 설정 확인
ethtool --show-eee eth0
# EEE status: disabled  ← 정상
```

---

## TSN PHY 요구사항

### 필수 기능

```
1. 하드웨어 타임스탬핑 (IEEE 1588v2)
   - RX/TX 각각 SFD 수신/전송 시점에 하드웨어 클록 캡처
   - 정밀도 요구: < 10 ns (일반 달성: < 1 ns)
   - 소프트웨어 타임스탬프로는 수 µs 오차 → PTP 정밀도 불가

   확인:
   ethtool -T eth0 | grep -E "hardware-transmit|hardware-receive"

2. EEE 비지원 또는 비활성화 가능
   → TSN 인증 PHY는 대부분 EEE 미지원 또는 비활성화 옵션 제공

3. TAS 하드웨어 지원 (권장)
   - 소프트웨어 TAS: OS 스케줄링 지터 포함 (수 µs 오차)
   - 하드웨어 TAS: PHY/MAC 레벨 정밀 게이트 제어 (ns 수준)

4. PHY 처리 지연 일정성 (Constant PHY Latency)
   - PTP 투명 클록(TC) 구현 시 PHY 지연을 Correction 필드에 누적
   - 일정하지 않으면 시간 동기화 오차 증가
```

### 속도별 TSN 지원 PHY 예시

| 표준 | 칩 예시 | TSN 기능 | 용도 |
|------|--------|---------|------|
| 100BASE-T1 | NXP TJA1103 | PTP TS, TAS, SGMII | 차량/로봇 관절 |
| 1000BASE-T1 | Marvell 88Q2221 | PTP TS, TAS | 고속 제어 |
| 1000BASE-T | Intel I210 | PTP TS, TAS (S/W) | PC 기반 제어기 |
| 1000BASE-T | Intel I225-V | PTP TS, TAS (H/W) | 최신 PC 제어기 |
| 2.5GBASE-T1 | Marvell 88Q4364 | PTP TS | 차세대 차량 |

```bash
# Intel I210/I225 TSN 기능 확인
ethtool -T eth0          # 타임스탬핑 지원 확인
ethtool -k eth0 | grep -E "hw-tc-offload|tx-scheduling"
# hw-tc-offload: on  ← 하드웨어 TAS 지원 표시
```

---

## Reference
- [IEEE 802.3-2022 - Ethernet Standard (Complete)](https://standards.ieee.org/ieee/802.3/10422/)
- [IEEE 802.3az - Energy Efficient Ethernet (EEE)](https://standards.ieee.org/ieee/802.3az/4270/)
- [IEEE 802.3cg - 10BASE-T1S/T1L](https://standards.ieee.org/ieee/802.3cg/7438/)
- [IEEE 802.3bw - 100BASE-T1](https://standards.ieee.org/ieee/802.3bw/5447/)
- [IEEE 1588-2019 - Precision Time Protocol (PTP)](https://standards.ieee.org/ieee/1588/6825/)
- [TIA-568 - Commercial Building Telecommunications Cabling Standard](https://www.tiaonline.org/)
- [Ethernet Alliance - Technology Roadmap](https://ethernetalliance.org/technology/ethernet-roadmap/)
- [Linux PTP Project - linuxptp](https://linuxptp.sourceforge.net/)
