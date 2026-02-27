# IEEE 802.1Qbv (TAS: Time Aware Shaper)

## 개요
일반 Ethernet은 Best Effort 방식으로 트래픽이 몰리면 지연이 발생합니다. IEEE 802.1Qbv TAS(Time Aware Shaper)는 **시간 기반 스케줄링**으로 각 트래픽 종류에 전용 시간 슬롯을 할당하여 **결정론적 지연(Deterministic Latency)**을 보장합니다.

전제 조건: IEEE 802.1AS 시간 동기화가 먼저 구성되어 있어야 합니다.

---

## 기본 개념

### 우선순위 큐 (Priority Queue)
Ethernet 스위치는 출력 포트마다 8개의 우선순위 큐(TC0~TC7)를 가집니다.

```
입력 프레임 → [분류] → 큐 TC7 (최고 우선순위) ─┐
                    → 큐 TC6              ─┤
                    → 큐 TC5 (제어 트래픽) ─┤
                    → 큐 TC4              ─┤  출력 포트
                    → 큐 TC3              ─┤
                    → 큐 TC2              ─┤
                    → 큐 TC1              ─┤
                    → 큐 TC0 (Best Effort) ─┘

VLAN PCP → Traffic Class 기본 매핑 (IEEE 802.1Q):
  PCP 7 → TC7   PCP 6 → TC6   PCP 5 → TC5   PCP 4 → TC4
  PCP 3 → TC3   PCP 2 → TC2   PCP 0 → TC1   PCP 1 → TC0
```

### TAS 게이트 제어

```
TAS 동작 원리:
각 큐마다 게이트(Gate)가 있어 열림(Open)/닫힘(Closed) 상태 제어

시간 →  T=0   100µs  1ms   1.1ms  2ms
                │      │      │      │
TC7(제어) ─ OPEN─────CLOSE──OPEN──CLOSE──
TC0(BE)   ─ CLOSE────OPEN───CLOSE─OPEN──

→ TC7은 T=0~100µs 구간만 전송 가능 (정확한 시간 보장)
→ TC0은 TC7이 닫혀 있을 때만 전송 가능 (잔여 대역폭 사용)
```

---

## GCL (Gate Control List)

TAS의 핵심 설정 테이블입니다.

```
GCL 구조:
  cycle_time: 전체 주기 (예: 1,000,000 ns = 1ms)
  base_time:  GCL 시작 기준 시각 (TAI 절대값 나노초)
  list_length: GCL 항목 수

GCL 항목 (각 슬롯):
  gate_mask: 어떤 TC를 열지 (8비트, 비트0=TC0, 비트7=TC7)
  interval:  슬롯 지속 시간 (나노초)

예시 (Cycle Time: 1ms):
┌─────────────────┬──────────────────┬──────────────────────────────────┐
│  Gate States    │  Duration        │  설명                            │
│  TC7~TC0 (8bit) │  (nanoseconds)   │                                  │
├─────────────────┼──────────────────┼──────────────────────────────────┤
│  1000 0000 (0x80)│  100,000 ns     │  TC7(제어) 전용 100µs            │
│  0111 1111 (0x7F)│  875,000 ns     │  TC6~TC0 나머지 875µs            │
│  1000 0000 (0x80)│  25,000 ns      │  Guard Band (TC7만 허용, 실제 무전송)│
└─────────────────┴──────────────────┴──────────────────────────────────┘
Total: 100µs + 875µs + 25µs = 1000µs (= 1ms Cycle)
```

### Guard Band (가드 밴드) 계산

Best Effort 프레임이 Scheduled Traffic 시간대로 침범하는 것을 방지합니다.

```
Guard Band 크기 계산:
  guard_band_ns = (max_frame_size_bytes × 8) / link_rate_bps × 1e9

  1Gbps, 1518 Byte 최대 프레임:
    guard_band = (1518 × 8) / 1,000,000,000 × 1,000,000,000
               = 12,144 ns ≈ 12.1µs

  100Mbps, 1518 Byte:
    guard_band = 121,440 ns ≈ 121µs

Guard Band 없을 때 문제:
  TC0 프레임(1518 Byte @ 1Gbps = 12.1µs) 전송 중 TC7 게이트 열림
  → TC7이 12.1µs 지연 발생

Guard Band 적용:
  TC7 게이트 열리기 12.1µs 전에 TC0 게이트 닫음
  → TC7 항상 정확한 시간에 전송

Frame Preemption(802.1Qbu) 적용 시:
  Guard Band 불필요 (BE 프레임을 중단하고 즉시 제어 전송 가능)
  → 가드 밴드 낭비 제거, 유효 대역폭 증가
```

### 최악 지연(Worst-Case Latency) 계산

```
TAS 적용 시 TC7의 최악 지연:
  WCL = Cycle_Time + max_TX_time_TC7

  Cycle_Time = 1ms
  max_TX_time_TC7 = TC7 슬롯 내 최대 프레임 전송 시간
  WCL = 1ms + 수µs ≈ 1ms (결정론적)

TAS 없는 경우 (Best Effort):
  WCL ≈ N × max_frame_size / link_rate  (N=경쟁 프레임 수)
  1Gbps, N=10, 1518B: WCL ≈ 121µs ~ 수ms (비결정론적)

다중 홉 최악 지연:
  WCL_total = Σ(WCL_i)  for each switch hop i
  스위치 4홉, 각 1ms WCL: WCL_total ≈ 4ms
```

---

## Linux tc taprio로 TAS 구현

### taprio flags 설명

| flags 값 | 모드 | 설명 |
|---------|------|------|
| `0x0` | Software 모드 | 커널 소프트웨어가 GCL 실행. 정밀도 낮음 (수 µs 오차) |
| `0x1` | TX Launch Time | TX 시각 기반 전송. 일부 HW 필요 |
| `0x2` | Full Hardware Offload | NIC이 GCL 전체를 HW로 실행. 최고 정밀도 |
| `0x3` | TX Launch Time + HW | TX 시각 + HW 오프로드 조합 |

```bash
# taprio qdisc 설정 (1ms 주기, TC7=100µs 제어, TC0=900µs BE)
tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1600000000000000000 \
    sched-entry S 0x80 100000 \
    sched-entry S 0x7F 875000 \
    sched-entry S 0x80 25000 \
    clockid CLOCK_TAI \
    flags 0x2                    # Full Hardware Offload (Intel I225-V 등)

# base-time 설정 방법 (현재 TAI 시각 기반)
# TAI = UTC + 37초 (2024년 기준 leapseconds 누적)
CURRENT_TAI=$(phc_ctl /dev/ptp0 get 2>/dev/null | awk '{print $5}')
# 또는
CURRENT_TAI=$(date +%s%N)  # 대략적인 계산용

# 설정 확인
tc qdisc show dev eth0

# GCL 상태 및 통계 확인
tc -s qdisc show dev eth0

# taprio 제거
tc qdisc del dev eth0 root
```

### CLOCK_TAI 동기화 (중요)

```bash
# TAI 시간이 PHC와 동기화되어 있어야 taprio base-time이 정확함

# phc2sys로 PHC → CLOCK_TAI 동기화
phc2sys -s /dev/ptp0 -c CLOCK_TAI -O 0 --verbose &

# TAI 시각 확인
timedatectl | grep "TAI offset"
# TAI offset: +37 seconds

# base-time 계산 (다음 사이클 경계)
python3 -c "
import time
CYCLE_NS = 1_000_000  # 1ms
tai_ns = int(time.time() * 1e9) + 37_000_000_000  # 대략
base_time = (tai_ns // CYCLE_NS + 1000) * CYCLE_NS  # 1000사이클 후 시작
print(f'base-time {base_time}')
"
```

### VLAN PCP → Traffic Class 매핑

```bash
# VLAN 인터페이스 생성 및 PCP 매핑 설정
ip link add link eth0 name eth0.10 type vlan id 10
ip link set eth0.10 type vlan \
    egress-qos-map 7:7 6:6 5:5 4:4 3:3 2:2 1:1 0:0
# egress-qos-map: SO_PRIORITY → VLAN PCP 매핑

# 소켓 우선순위 → TC 매핑 확인
cat /proc/net/vlan/eth0.10  # egress map 확인
```

### 소켓 우선순위 설정

```c
/* 애플리케이션에서 소켓 우선순위 설정 */
int priority = 7;  /* TC7 매핑 (VLAN PCP=7로 태깅됨) */
setsockopt(sockfd, SOL_SOCKET, SO_PRIORITY, &priority, sizeof(priority));

/* RAW 소켓으로 VLAN PCP 직접 설정 */
struct sockaddr_ll sa = {
    .sll_family   = AF_PACKET,
    .sll_protocol = htons(ETH_P_8021Q),
    .sll_ifindex  = if_nametoindex("eth0"),
};
/* VLAN 태그: PCP=7, DEI=0, VID=10 */
uint16_t vlan_tag = htons((7 << 13) | (0 << 12) | 10);
```

---

## 하드웨어 TAS 지원

### 지원 하드웨어
| NIC/스위치 | 제조사 | TAS 지원 | 최대 GCL 항목 | flags |
|-----------|--------|---------|------------|-------|
| I225-V (igc) | Intel | Full HW Offload | 128 | 0x2 |
| I210 (igb) | Intel | SW 기반 | - | 0x0 |
| KSZ9563 | Microchip | Full HW Offload | 16 | 0x2 |
| SJA1110 | NXP | Full HW Offload | 1024 | 0x2 |
| RTL9031 | Realtek | Full HW Offload | - | 0x2 |

```bash
# Intel I225-V (igc) TAS 확인
ethtool -k eth0 | grep "tx-sched"
# tx-sched-offload: on  ← HW TAS 지원

# 하드웨어 TX 큐 수 확인
ethtool -l eth0
# Current hardware settings:
#   RX: 4
#   TX: 4  ← TAS에서 TC별로 독립 큐 필요
```

---

## TAS 적용 전후 지연 비교

```
TAS 없는 경우 (Best Effort):
  제어 명령 전송 요청
      │
      ▼
  1518 Byte 영상 데이터 전송 중 (12.1µs @ 1Gbps)
  다음 영상 데이터 (또 12.1µs 대기)
  결과: 수십µs ~ 수ms 불규칙 지연

TAS 적용:
  제어 명령 전송 요청
      │
      ▼
  TC7 게이트 열림 시각까지 대기 (최대 1주기 = 1ms)
      │
      ▼ (정확히 TC7 슬롯에)
  제어 명령 전송 완료
  결과: 최대 지연 = 1주기(1ms) + 전송 시간, Jitter < 1µs
```

---

## 대역폭 계산

```python
# TAS 대역폭 할당 계산 예시
cycle_time_us = 1000  # 1ms
tc7_slot_us = 100     # 100µs

# TC7 최대 전송량 계산 (1Gbps)
max_bytes_per_cycle = (tc7_slot_us * 1e-6) * (1e9 / 8)  # 12,500 Byte
# → 1주기 100µs에 약 8개의 1518B 표준 프레임 전송 가능

# TC7 유효 대역폭
tc7_bandwidth_mbps = (tc7_slot_us / cycle_time_us) * 1000  # 100 Mbps
print(f"TC7 대역폭: {tc7_bandwidth_mbps} Mbps")
print(f"TC0 대역폭: {(1 - tc7_slot_us/cycle_time_us) * 1000} Mbps")

# 가드 밴드 포함 계산
guard_band_us = 12.144  # 1518B @ 1Gbps
available_be_us = cycle_time_us - tc7_slot_us - guard_band_us
print(f"BE 유효 슬롯: {available_be_us:.1f}µs ({available_be_us/cycle_time_us*100:.1f}%)")
```

---

## 의료 로봇 TAS 설정 예시

```
수술 로봇 TSN 스위치 GCL 설계:
Cycle Time: 2ms (500Hz 제어 주기)

Entry 1: [TC7 OPEN]  50µs   → 안전 알람 전송 (ASIL-D, 최우선)
Entry 2: [TC5 OPEN]  150µs  → 관절 제어 명령/피드백 (ASIL-B)
Entry 3: [TC3 OPEN]  300µs  → 센서 데이터 (힘, 위치)
Entry 4: [TC0 OPEN]  1500µs → 영상, 진단, OTA (Best Effort)
                    ────────
Total:              2000µs = 2ms Cycle

보장되는 지연:
  안전 알람: < 2ms (1 Cycle 내 전송 보장)
  제어 명령: < 2ms
  센서 데이터: < 2ms
  영상 트래픽: Best Effort (허용 범위 내)

taprio 명령:
tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1600000000000000000 \
    sched-entry S 0x80 50000 \
    sched-entry S 0x20 150000 \
    sched-entry S 0x08 300000 \
    sched-entry S 0x01 1500000 \
    clockid CLOCK_TAI \
    flags 0x2
```

---

## Wireshark로 TAS 검증

```bash
# 트래픽 스케줄링 검증용 필터
# 1. VLAN 우선순위 확인
vlan.priority == 7

# 2. 제어 트래픽 타임스탬프 분석
# Wireshark → Statistics → I/O Graphs
# → Delta Time 열로 프레임 간 간격 측정 → Jitter 확인

# 3. 지터 측정 (tcpdump + Python 분석)
tcpdump -i eth0 -w capture.pcap vlan and vlan pcp 7
# → Python/pandas로 타임스탬프 분포 분석
python3 -c "
import subprocess, struct
# pcap 분석 → Delta Time 측계 → Jitter = stdev(delta)
"
```

---

## Reference
- [IEEE 802.1Qbv-2015 - Enhancements for Scheduled Traffic](https://standards.ieee.org/ieee/802.1Qbv/5742/)
- [Linux tc-taprio man page](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
- [TSN Task Group - Time Aware Shaper](https://1.ieee802.org/tsn/802-1qbv/)
- [Avnu Alliance - Automotive TSN Profile](https://avnu.org/automotive/)
- [Intel I225 TSN Application Note](https://www.intel.com/content/www/us/en/products/sku/184676/intel-ethernet-controller-i225lm/specifications.html)
