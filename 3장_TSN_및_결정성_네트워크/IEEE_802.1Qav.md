# IEEE 802.1Qav (CBS: Credit-Based Shaper)

## 개요
IEEE 802.1Qav는 **Credit-Based Shaper(CBS)**를 정의하는 TSN 표준입니다. 802.1Qbv TAS가 정밀한 시간 슬롯 스케줄링으로 결정론적 지연을 보장한다면, CBS는 **대역폭 예약과 버스트 제어**로 오디오/비디오 스트리밍과 같은 등시성(Isochronous) 트래픽에 적합합니다.

원래 IEEE 802.1BA(AVB: Audio Video Bridging)의 일부로 개발되었으며, TSN에서 계속 활용됩니다.

---

## CBS 동작 원리

### 크레딧 메커니즘

```
CBS 개념도:
  크레딧(Credit) = 전송 허용 여부 결정
  ┌─────────────────────────────────────────────────────────┐
  │  Credit                                                  │
  │    │                                                    │
  │  hiCredit ┤ 크레딧 최대값 (idleSlope에 따라 충전)       │
  │    │       ─────────────────────────────────────────    │
  │    0       프레임 전송 조건: Credit ≥ 0                │
  │    │       ─────────────────────────────────────────    │
  │  loCredit ┤ 크레딧 최소값 (프레임 전송 중 감소)         │
  └─────────────────────────────────────────────────────────┘

상태 A: Credit ≥ 0 → 큐에 프레임 있으면 즉시 전송
         프레임 전송 중: Credit 감소 (sendSlope 속도로)
상태 B: Credit < 0 → 전송 불가, 대기
         대기 중: Credit 증가 (idleSlope 속도로)
상태 C: 큐가 비어 있음 → Credit = 0 유지 (증가 없음)
```

### CBS 파라미터 정의

| 파라미터 | 단위 | 설명 | 계산 공식 |
|---------|------|------|---------|
| **idleSlope** | bps | 대기 시 크레딧 증가율 | = 예약 대역폭 |
| **sendSlope** | bps | 전송 시 크레딧 감소율 | = idleSlope − 링크속도 (음수) |
| **hiCredit** | bits | 최대 크레딧 | = idleSlope × (max_non_cbs_frame / link_rate) |
| **loCredit** | bits | 최소 크레딧 | = sendSlope × (max_cbs_frame / link_rate) |

### CBS 파라미터 계산

```python
def calc_cbs_params(link_rate_bps, reserved_bps, max_frame_bytes, interfering_frame_bytes=1518):
    """
    link_rate_bps:        링크 속도 (예: 1_000_000_000 = 1Gbps)
    reserved_bps:         예약 대역폭 (예: 100_000_000 = 100Mbps)
    max_frame_bytes:      CBS 스트림 최대 프레임 크기 (예: 1518)
    interfering_frame_bytes: 비CBS 트래픽 최대 프레임 크기 (기본 1518)
    """
    idle_slope = reserved_bps
    send_slope = reserved_bps - link_rate_bps  # 음수

    # hiCredit: 비CBS 프레임 1개 전송 중 충전되는 최대 크레딧
    non_cbs_tx_time = (interfering_frame_bytes * 8) / link_rate_bps
    hi_credit = int(idle_slope * non_cbs_tx_time)

    # loCredit: CBS 프레임 전송 중 소비되는 최대 크레딧
    cbs_tx_time = (max_frame_bytes * 8) / link_rate_bps
    lo_credit = int(send_slope * cbs_tx_time)

    print(f"idleSlope: {idle_slope / 1e6:.3f} Mbps  ({idle_slope} bps)")
    print(f"sendSlope: {send_slope / 1e6:.3f} Mbps  ({send_slope} bps)")
    print(f"hiCredit:  {hi_credit} bits  ({hi_credit / link_rate_bps * 1e6:.0f} µs)")
    print(f"loCredit:  {lo_credit} bits  ({lo_credit / link_rate_bps * 1e6:.0f} µs)")
    return idle_slope, send_slope, hi_credit, lo_credit

# Class A 예시: 1Gbps 링크, 49.152Mbps 예약
calc_cbs_params(1_000_000_000, 49_152_000, 1518)
# 출력:
# idleSlope:  49.152 Mbps  (49152000 bps)
# sendSlope: -950.848 Mbps (-950848000 bps)
# hiCredit:  593 bits  (0 µs)
# loCredit: -11523 bits  (-9 µs)
```

---

## TAS vs CBS 비교

| 특성 | TAS (802.1Qbv) | CBS (802.1Qav) |
|------|----------------|----------------|
| 동작 방식 | 시간 슬롯 기반 게이트 제어 | 크레딧 기반 대역폭 제어 |
| 지연 특성 | 결정론적 (< 1µs Jitter) | 확률론적 (< 2ms, 예측 가능) |
| 설정 복잡도 | 높음 (GCL, 시간 동기화 필수) | 낮음 (대역폭 % 설정) |
| 주요 용도 | 안전 제어, 정밀 제어 | 오디오/비디오, 센서 스트리밍 |
| 시간 동기화 | 802.1AS 필수 | 802.1AS 권장 (필수 아님) |
| 대역폭 보장 | 정확한 슬롯 | 평균 대역폭 보장 |
| AVB 표준 | - | IEEE 802.1BA 핵심 |

---

## AVB 스트림 클래스 (Class A / Class B)

| 항목 | Class A | Class B |
|------|---------|---------|
| 최대 지연 | **2ms** (7홉 기준) | **50ms** (7홉 기준) |
| Observation Interval | 125µs | 250µs |
| 최대 대역폭 | 링크의 75% | 링크의 75% |
| PCP 기본값 | 3 | 2 |
| 용도 | 오디오, 제어 | 비디오, 보조 스트림 |

---

## AVB (Audio Video Bridging) 프로토콜 스택

CBS는 AVB 프로토콜 스택의 핵심입니다.

```
┌───────────────────────────────────────────────────────┐
│  Application                                          │
│  (오디오/비디오 스트리밍 앱)                           │
├───────────────────────────────────────────────────────┤
│  AVTP (Audio Video Transport Protocol)                │
│  IEEE 1722 - 오디오/비디오 데이터 캡슐화              │
│  EtherType 0x22F0                                     │
├───────────────────────────────────────────────────────┤
│  SRP (Stream Reservation Protocol)                    │
│  IEEE 802.1Qat - 대역폭 예약                          │
├───────────────────────────────────────────────────────┤
│  CBS (Credit-Based Shaper)                            │
│  IEEE 802.1Qav - 트래픽 쉐이핑                        │
├───────────────────────────────────────────────────────┤
│  gPTP (IEEE 802.1AS)                                  │
│  시간 동기화 (미디어 동기에 필요)                      │
├───────────────────────────────────────────────────────┤
│  Ethernet (L2)                                        │
└───────────────────────────────────────────────────────┘
```

---

## SRP (Stream Reservation Protocol) - 대역폭 예약

CBS 트래픽을 위해 네트워크 경로의 대역폭을 사전 예약합니다.

```
Talker (스트리머)               Network                  Listener (수신자)
      │                           │                          │
      │── MSRP Advertise ────────►│──────────────────────►  │
      │   (스트림 ID, 대역폭,      │                          │
      │    Latency Class,         │                          │
      │    MaxFrameSize 포함)     │                          │
      │                           │                          │
      │◄── MSRP Ready ────────────│◄── MSRP Ready ──────────│
      │    (대역폭 예약 완료)      │    (Listener Accept)     │
      │                           │                          │
      │── AVTP 데이터 스트림 ─────►│──────────────────────►  │
           (CBS로 쉐이핑)

MSRP (Multiple Stream Registration Protocol):
  MRP 기반 (IEEE 802.1Qat)
  이더넷 멀티캐스트: 01:80:C2:00:00:0E
  스위치가 홉별 대역폭 가용성 확인 후 예약

Stream ID 구조:
  [6Byte Talker MAC][2Byte Stream ID]  = 8Byte 고유 식별자
```

---

## Linux에서 CBS 설정

```bash
# CBS qdisc 설정 (eth0, Class A 스트림)
# 1Gbps 링크, Class A에 49.152Mbps 예약

# 1. MQ (Multi-Queue) qdisc 설정
tc qdisc add dev eth0 root handle 1: mqprio \
    num_tc 3 \
    map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 \
    queues 1@0 1@1 2@2 \
    hw 0

# 2. CBS qdisc 추가 (Class A, TC0 = 높은 우선순위)
tc qdisc replace dev eth0 parent 1:1 cbs \
    idleslope 49152 \
    sendslope -950848 \
    hicredit 348 \
    locredit -1392 \
    offload 0

# 3. CBS qdisc 추가 (Class B, TC1)
tc qdisc replace dev eth0 parent 1:2 cbs \
    idleslope 9216 \
    sendslope -990784 \
    hicredit 56 \
    locredit -224 \
    offload 0

# 설정 확인
tc qdisc show dev eth0
tc -s qdisc show dev eth0  # 통계 포함

# CBS 제거
tc qdisc del dev eth0 root
```

### CBS + TAS 조합 설정

```bash
# TAS가 우선: TAS 슬롯 외 시간에 CBS로 대역폭 보장
# 권장 계층: TAS (TC7) > CBS (TC5,6) > BE (TC0~4)

tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1600000000000000000 \
    sched-entry S 0x80 100000 \  # TC7 전용 100µs
    sched-entry S 0x7F 900000 \  # 나머지 900µs (CBS 포함)
    clockid CLOCK_TAI \
    flags 0x2

# TC5에 CBS 추가 (TAS 비전용 시간에 50Mbps 보장)
tc qdisc add dev eth0 parent 100:6 cbs \
    idleslope 50000000 \
    sendslope -950000000 \
    hicredit 1524 \
    locredit -14424 \
    offload 0
```

---

## 의료 로봇에서의 CBS 활용

```
수술 로봇 CBS 트래픽 구성:

Class A (2ms 지연 보장):
  - 조인트 위치 피드백 스트리밍 (500Hz, 10Mbps)
  - 힘/토크 센서 데이터 (1kHz, 5Mbps)
  예약 대역폭: 15Mbps
  idleSlope = 15,000,000 bps
  sendSlope = -985,000,000 bps (15M - 1G)

Class B (50ms 지연 허용):
  - 내시경 오디오 스트리밍
  - 환자 모니터링 데이터
  예약 대역폭: 5Mbps

Best Effort:
  - 내시경 영상 (UDP, 적응형 비트레이트)
  - 진단/OTA 트래픽

CBS + VLAN 조합:
  VLAN 10, PCP=5 → Class A CBS 처리
  VLAN 20, PCP=3 → Class B CBS 처리
  VLAN 30, PCP=0 → Best Effort
```

---

## Reference
- [IEEE 802.1Qav-2009 - Forwarding and Queuing for Time-Sensitive Streams](https://standards.ieee.org/ieee/802.1Qav/3684/)
- [IEEE 1722-2016 - AVTP (Audio Video Transport Protocol)](https://standards.ieee.org/ieee/1722/5971/)
- [Linux tc-cbs man page](https://man7.org/linux/man-pages/man8/tc-cbs.8.html)
- [Linux tc-mqprio man page](https://man7.org/linux/man-pages/man8/tc-mqprio.8.html)
- [Avnu Alliance - AVB/TSN](https://avnu.org/)
