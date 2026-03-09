# tc taprio - TSN 트래픽 제어 실습

## 개요
Linux Traffic Control(`tc`)의 `taprio` qdisc는 IEEE 802.1Qbv(TAS, Time-Aware Shaper)를 소프트웨어로 구현합니다. TAS는 TSN 아키텍처의 핵심으로, 네트워크 시간을 주기적인 슬롯으로 나누고 각 슬롯에서 허용되는 트래픽 클래스(TC)를 제어하여 **결정론적 지연(Deterministic Latency)**을 보장합니다.

`taprio`는 하드웨어 오프로드를 지원하는 NIC(Intel i225/i226, Microchip LAN9668 등)와 함께 사용하면 마이크로초 수준의 정밀한 스케줄링이 가능합니다. 소프트웨어 모드에서는 커널 스케줄러 지터로 인해 정밀도가 낮아지므로, 실제 운용에서는 하드웨어 오프로드가 권장됩니다.

> **의료 로봇 관점**: 수술 로봇 Ethernet 네트워크에서 taprio는 E-Stop 신호(< 100µs), 관절 제어(< 1ms), 센서 피드백(< 1ms), 영상 스트림, 진단 트래픽이 하나의 물리 링크를 공유할 때 우선순위별로 격리된 전송 슬롯을 보장합니다. GCL(Gate Control List)이 2ms 주기로 동작한다면 E-Stop은 처음 0.1ms 슬롯에서만 전송되어 **어떤 상황에서도 다른 트래픽의 방해를 받지 않습니다**. 이 결정론성이 IEC 62304 Class C 및 ISO 26262 ASIL D의 타이밍 요구사항을 충족하는 기술적 근거가 됩니다.

---

## tc 기본 개념 및 계층 구조

```
tc(Traffic Control) 전체 계층:

NIC TX Queue
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Root qdisc (Queueing Discipline)                   │
│  예: taprio, htb, prio, fq_codel                    │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │  Class 0   │  │  Class 1   │  │  Class 2   │    │
│  │  (TC0: BE) │  │  (TC5: Sensor)│ (TC7: Safety)│  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘    │
│        │               │               │            │
│  ┌─────▼──────┐  ┌─────▼──────┐  ┌─────▼──────┐   │
│  │  Leaf qdisc│  │  Leaf qdisc│  │  Leaf qdisc│   │
│  │  (fq, cbs) │  │  (cbs)     │  │  (pfifo)   │   │
│  └────────────┘  └────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────┘

트래픽 분류 흐름:
  패킷 도착 → tc filter (분류) → TC 결정 → Queue 진입
  taprio GCL → Gate Open 신호 → 해당 TC 패킷 전송
  Gate Close → 패킷은 다음 Open까지 대기

핵심 개념:
  qdisc   : 큐 규칙 (Queue Discipline)
  class   : qdisc 내 트래픽 클래스
  filter  : 패킷을 클래스로 분류하는 규칙
  handle  : qdisc/class 식별자 (예: 100:, 100:1)
```

---

## GCL(Gate Control List) 설계 방법론

taprio 설정 전 반드시 GCL을 설계해야 합니다. GCL 설계는 전송 요구사항 분석부터 시작합니다.

```
GCL 설계 절차:

1단계: 트래픽 분류 및 요구사항 정의
  트래픽 유형     TC  PCP  주기   최대 프레임  최대 지연  보장 대역
  ─────────────────────────────────────────────────────────────
  E-Stop 신호    TC7  7    1ms    64 Byte      < 100µs   최소 보장
  관절 제어      TC7  7    1ms    256 Byte     < 1ms     최소 보장
  힘/토크 피드백 TC5  5    2ms    128 Byte     < 1ms     CBS
  센서 스트림    TC5  5    10ms   1500 Byte    < 5ms     CBS
  영상 스트림    TC4  4    33ms   1500 Byte    < 100ms   CBS
  진단 (DoIP)   TC0  0    비정기  1500 Byte    < 500ms   BE
  OTA 업데이트  TC0  0    비정기  1500 Byte    무제한     BE

2단계: GCL 주기 결정
  기준: 가장 짧은 고우선순위 주기 = 1ms
  GCL 주기 = 1ms (1,000,000 ns) 또는 2ms

3단계: 슬롯 크기 계산
  전송 시간 = (헤더 + 페이로드) / 링크 속도 + IFG
  - 헤더: 26 Byte (Preamble 8 + Dst 6 + Src 6 + VLAN 4 + FCS 4 - Preamble 2)
  - 실제 Ethernet 최솟값: 64 Byte (Preamble 포함 84 Byte on wire)
  - IFG: 12 Byte (Inter-Frame Gap)

  E-Stop 256 Byte @ 1Gbps:
    전송 시간 = (256 + 38) / 1e9 * 8 = 2.352 µs
    IFG:       12 / 1e9 * 8  = 0.096 µs
    합계:      ≈ 2.5 µs

  GCL 2ms 주기 설계 예:
  ┌──────────────────────────────────────────────┐
  │ 시간(µs) 0    100  200  500  1000 1500 2000  │
  │ TC7(안전)  ████                               │ 0~100µs
  │ TC5(센서)       █████████                     │ 100~500µs
  │ TC4(영상)                ████████████         │ 500~1500µs
  │ TC0(BE)                              ████████ │ 1500~2000µs
  └──────────────────────────────────────────────┘

4단계: CBS 파라미터 계산 (TC5, TC4)
  CBS(Credit-Based Shaper) 파라미터:
    idleslope  = 보장 대역폭 (bps)
    sendslope  = idleslope - linkrate
    hicredit   = max_frame_size × (idleslope / linkrate)
    locredit   = max_frame_size × (sendslope / linkrate)

  TC5 (센서 스트림, 10Mbps 보장):
    idleslope  = 10,000,000
    sendslope  = 10,000,000 - 1,000,000,000 = -990,000,000
    hicredit   ≈ 1500 × (10M/1G)   = 15 Byte
    locredit   ≈ 1500 × (-990M/1G) = -1485 Byte
```

---

## taprio qdisc 설정

### 소프트웨어 모드 (개발/검증 환경)

```bash
# 기존 qdisc 초기화
sudo tc qdisc del dev eth0 root 2>/dev/null

# taprio 적용 (2ms GCL 주기)
# TC7: 안전 제어 (100µs), TC5: 센서 (400µs), TC0: BE (1500µs)
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $(date +%s%N) \
    sched-entry S 0x80 100000  \
    sched-entry S 0x20 400000  \
    sched-entry S 0x01 1500000 \
    flags 0x0 \
    clockid CLOCK_TAI

# 파라미터 상세 설명:
# num_tc 8           : TC0~TC7 총 8개 트래픽 클래스
# map 0 0 0 0 0 5 6 7: PCP(0~7) → TC 매핑
#   PCP 0,1,2,3,4 → TC0 (Best Effort)
#   PCP 5         → TC5 (Sensor, CBS 적용 예정)
#   PCP 6         → TC6 (Video)
#   PCP 7         → TC7 (Safety Critical)
# queues 1@0 1@1 ..  : 각 TC에 큐 1개씩, 큐 번호 0~7
# base-time          : GCL 시작 기준 시각 (TAI 나노초)
#   → PTP 동기화된 시스템에서는 다음 사이클 시작점 지정 권장
# sched-entry S [mask] [interval_ns]:
#   S: SetGates (지정된 게이트를 Open)
#   0x80 = 1000 0000 = TC7만 Open (100µs)
#   0x20 = 0010 0000 = TC5만 Open (400µs)
#   0x01 = 0000 0001 = TC0만 Open (1500µs)
#   → 총 주기: 100000 + 400000 + 1500000 = 2,000,000 ns = 2ms
# flags 0x0          : 소프트웨어 모드
# clockid CLOCK_TAI  : TAI 기준 (PTP와 동일, UTC+37초)
```

### 하드웨어 오프로드 (Intel i225/i226)

```bash
# Intel i225/i226은 하드웨어 TAS 오프로드 지원
# → NIC의 하드웨어 내부에서 GCL 직접 처리 → 수µs 정밀도

# flags 0x2 = TXTIME_OFFLOAD 활성화
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 4 \
    map 0 0 0 0 1 1 2 3 0 0 0 0 0 0 0 0 \
    queues 2@0 2@2 1@4 1@5 \
    base-time 1700000000000000000 \
    sched-entry S 0x08 100000  \
    sched-entry S 0x04 400000  \
    sched-entry S 0x03 1500000 \
    flags 0x2 \
    clockid CLOCK_TAI

# 파라미터 차이점 (HW 오프로드):
# num_tc 4:          Intel i225는 4개 TC 하드웨어 큐 지원
# queues 2@0 2@2 1@4 1@5:
#   TC0: 큐 2개 (인덱스 0,1)
#   TC1: 큐 2개 (인덱스 2,3)
#   TC2: 큐 1개 (인덱스 4)
#   TC3: 큐 1개 (인덱스 5)
# flags 0x2:         NIC 하드웨어가 GCL 처리 (< 1µs 정밀도)
# base-time:         절대 TAI 시각 (고정값 권장, PTP와 일치)

# HW 오프로드 지원 확인
ethtool -k eth0 | grep "tx-taprio-qopt-skip-sw"
# tx-taprio-qopt-skip-sw: off [fixed] → 미지원
# tx-taprio-qopt-skip-sw: off        → 지원 (설정으로 활성화 가능)

# NIC 하드웨어 큐 수 확인
ethtool -l eth0
# TX: 4  ← TC 수와 일치해야 HW 오프로드 가능
```

### base-time과 PTP 동기화 연동

```bash
# taprio의 base-time은 PTP Grandmaster와 동기화되어야 합니다.
# GCL이 동기화 없이 시작되면 각 노드가 서로 다른 슬롯에서 전송하여
# 동일 링크 상에서 충돌이 발생합니다.

# PTP 동기화 후 TAI 기준 base-time 계산 예시
#!/bin/bash
# 현재 TAI 시각 얻기 (PHC 직접 읽기)
PHC_TIME=$(phc_ctl /dev/ptp0 get | awk '{print $5}')
echo "현재 PHC (TAI): $PHC_TIME"

# GCL 주기(2ms = 2,000,000 ns)에 정렬된 다음 시작점 계산
PERIOD_NS=2000000
PHC_NS=$(echo "$PHC_TIME * 1e9" | bc | awk '{printf "%d", $1}')
REMAINDER=$((PHC_NS % PERIOD_NS))
BASE_TIME=$((PHC_NS - REMAINDER + PERIOD_NS * 2))  # 2 주기 후 시작

echo "base-time: $BASE_TIME"

tc qdisc replace dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $BASE_TIME \
    sched-entry S 0x80 100000  \
    sched-entry S 0x20 400000  \
    sched-entry S 0x01 1500000 \
    flags 0x2 \
    clockid CLOCK_TAI
```

---

## mqprio qdisc - 멀티큐 우선순위

taprio와 달리 시간 슬롯 없이 하드웨어 큐를 TC에 직접 매핑합니다. TAS 없는 환경에서 우선순위 기반 스케줄링에 사용합니다.

```bash
# Intel i225는 4개 TX 큐 지원 → 4개 TC 매핑 가능
sudo tc qdisc add dev eth0 parent root handle 100 mqprio \
    num_tc 4 \
    map 0 0 0 0 1 1 2 3 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 \
    hw 1    # 1 = 하드웨어 큐 직접 사용

# PCP → TC 매핑 의미:
#   PCP 0,1,2,3 → TC0 → HW Queue 0 (Best Effort)
#   PCP 4,5     → TC1 → HW Queue 1 (Sensor)
#   PCP 6       → TC2 → HW Queue 2 (Control)
#   PCP 7       → TC3 → HW Queue 3 (Safety)

# 확인
tc qdisc show dev eth0
# qdisc mqprio 100: root tc 4 map 0 0 0 0 1 1 2 3 0 0 0 0 0 0 0 0
#                   queues:(0:0) (1:1) (2:2) (3:3)
#                   mode:channel
#                   shaper:dcb
```

---

## CBS (Credit-Based Shaper) - IEEE 802.1Qav

CBS는 오디오/비디오 스트리밍 트래픽의 대역폭을 보장합니다. taprio의 하위 qdisc로 사용되어 ST(Scheduled Traffic) 슬롯 안에서도 대역폭을 제어합니다.

```bash
# TC5에 CBS 적용 (taprio 100:6 클래스의 하위 qdisc)
# 10Mbps 보장, 링크 속도 1Gbps

# CBS 파라미터 계산:
#   idleslope  = 보장 대역폭 = 10,000,000 bps
#   sendslope  = idleslope - linkrate = 10M - 1G = -990,000,000 bps
#   hicredit   = maxFrameSize × (idleslope / linkrate)
#              = 1500 × (10/1000) = 15 Byte
#   locredit   = maxFrameSize × (sendslope / linkrate)
#              = 1500 × (-990/1000) = -1485 Byte

sudo tc qdisc add dev eth0 parent 100:6 handle 110 cbs \
    idleslope 10000    \
    sendslope -990000  \
    hicredit  15       \
    locredit  -1485    \
    offload   1        # 하드웨어 오프로드 (지원 NIC)

# 파라미터 단위 주의: tc cbs는 kbps 단위 사용!
# idleslope  10000  = 10,000 kbps = 10 Mbps
# sendslope -990000 = -990,000 kbps = -990 Mbps

# TC4 (영상 스트림, 50Mbps 보장):
sudo tc qdisc add dev eth0 parent 100:5 handle 120 cbs \
    idleslope  50000    \
    sendslope  -950000  \
    hicredit   75       \
    locredit   -7125    \
    offload    1

# CBS 동작 확인
tc -s qdisc show dev eth0
# qdisc cbs 110: parent 100:6 ...
#   Sent 102400 bytes 512 pkt (dropped 0)
#   idleslope 10000 sendslope -990000 hicredit 15 locredit -1485
```

---

## tc filter - 트래픽 분류

패킷을 TC에 분류하는 필터입니다. u32(IP 기반)와 flower(MAC/VLAN 기반)가 주로 사용됩니다.

```bash
# ── u32 필터 (IP/포트 기반) ──

# SOME/IP 제어 명령 (UDP 30490) → TC7
sudo tc filter add dev eth0 \
    parent 100: \
    protocol ip \
    prio 1 \
    u32 \
    match ip protocol 17 0xff \
    match ip dport 30490 0xffff \
    action skbedit priority 7

# DoIP 진단 (TCP 13400) → TC5
sudo tc filter add dev eth0 \
    parent 100: \
    protocol ip \
    prio 2 \
    u32 \
    match ip protocol 6 0xff \
    match ip dport 13400 0xffff \
    action skbedit priority 5

# DDS RTPS (UDP 7400-7500) → TC4
sudo tc filter add dev eth0 \
    parent 100: \
    protocol ip \
    prio 3 \
    u32 \
    match ip protocol 17 0xff \
    match ip dport 7400 0xff00 \
    action skbedit priority 4

# ── flower 필터 (VLAN/MAC 기반) ──

# VLAN 10 (Safety Network) PCP 7 → TC7
sudo tc filter add dev eth0 \
    parent 100: \
    protocol 802.1Q \
    prio 1 \
    flower \
    vlan_id 10 \
    vlan_prio 7 \
    action skbedit priority 7

# E-Stop 전용 MAC 주소 → TC7 (최고 우선순위)
sudo tc filter add dev eth0 \
    parent 100: \
    protocol 802.1Q \
    prio 0 \
    flower \
    dst_mac 01:80:c2:00:00:0e \
    action skbedit priority 7

# 필터 목록 확인
tc filter show dev eth0 parent 100:

# 필터 통계 (매칭된 패킷 수)
tc -s filter show dev eth0 parent 100:
```

---

## SO_TXTIME - 정밀 전송 타이밍

```bash
# SO_TXTIME 소켓 옵션으로 패킷 전송 시각을 나노초 단위로 지정
# taprio + SO_TXTIME 조합으로 GCL 슬롯에 정확히 맞춰 전송

# 지원 확인 (hardware-transmit 필요)
ethtool -T eth0 | grep "hardware-transmit"

# C 코드 개념 (실제 구현 예시)
cat << 'EOF' > /tmp/txtime_concept.c
#include <linux/net_tstamp.h>
#include <sys/socket.h>

// SO_TXTIME 설정
struct sock_txtime txtime_opt = {
    .clockid = CLOCK_TAI,    // PTP와 동일한 TAI 기준
    .flags   = 0,
};
setsockopt(fd, SOL_SOCKET, SO_TXTIME, &txtime_opt, sizeof(txtime_opt));

// 메시지 전송 시각 지정 (cmsg)
// SCM_TXTIME: TAI 나노초 단위 전송 시각
// 예: GCL 슬롯 시작 + 여유 시간(guard band) 이후로 설정
uint64_t txtime = ptp_tai_now() + GUARD_BAND_NS;
// sendmsg()로 전송 → NIC이 txtime까지 대기 후 전송
EOF
echo "SO_TXTIME 개념 코드 저장됨: /tmp/txtime_concept.c"

# cyclictest로 taprio 스케줄링 정밀도 측정
sudo cyclictest -p 99 -t 4 -n -i 1000 -l 100000
# T: 0 ( ...) P:99 I:1000 C:100000 Min: 12 Act: 18 Avg: 23 Max: 156
# Min/Avg/Max 단위: µs
# Max < 100µs: 의료 로봇 E-Stop 요구 충족
```

---

## 전체 TSN 스택 설정 스크립트

```bash
#!/bin/bash
# ============================================================
# 의료 로봇 TSN 스택 설정 스크립트
# - VLAN 설정 (10:Safety, 20:Control, 30:Sensor)
# - taprio GCL 적용 (2ms 주기)
# - CBS 대역폭 보장 (TC5, TC4)
# - tc filter 트래픽 분류
# ============================================================

set -euo pipefail

IFACE="eth0"
GCL_PERIOD_NS=2000000   # 2ms GCL 주기
LINK_SPEED_KBPS=1000000 # 1Gbps

log() { echo "[TSN] $*"; }

# 1. 기존 설정 초기화
log "기존 tc/VLAN 설정 초기화..."
tc qdisc del dev $IFACE root 2>/dev/null || true
ip link del ${IFACE}.10 2>/dev/null || true
ip link del ${IFACE}.20 2>/dev/null || true
ip link del ${IFACE}.30 2>/dev/null || true

# 2. 인터페이스 초기화
ip link set $IFACE down
ip link set $IFACE promisc off
ip link set $IFACE up
ethtool -A $IFACE autoneg off tx off rx off 2>/dev/null || true  # Flow Control 비활성화

# 3. VLAN 인터페이스 생성
log "VLAN 설정..."
ip link add link $IFACE name ${IFACE}.10 type vlan id 10
ip link add link $IFACE name ${IFACE}.20 type vlan id 20
ip link add link $IFACE name ${IFACE}.30 type vlan id 30

# VLAN 우선순위 맵 (egress: 내부 우선순위 → PCP)
ip link set ${IFACE}.10 type vlan egress-qos-map 7:7 6:6 5:5
ip link set ${IFACE}.20 type vlan egress-qos-map 5:5 4:4 3:3
ip link set ${IFACE}.30 type vlan egress-qos-map 4:4 3:3

ip addr add 192.168.10.1/24 dev ${IFACE}.10
ip addr add 192.168.20.1/24 dev ${IFACE}.20
ip addr add 192.168.30.1/24 dev ${IFACE}.30

ip link set ${IFACE}.10 up
ip link set ${IFACE}.20 up
ip link set ${IFACE}.30 up

# 4. PTP 동기화 후 base-time 계산
log "PTP 동기화 대기 (10초)..."
sleep 10  # 실제 환경에서는 pmc로 SLAVE 상태 확인 후 진행

if command -v phc_ctl >/dev/null 2>&1; then
    PHC_NS=$(phc_ctl $IFACE get 2>/dev/null | awk '{printf "%.0f", $5*1e9}')
else
    PHC_NS=$(date +%s%N)  # fallback: 시스템 시각
fi
REMAINDER=$((PHC_NS % GCL_PERIOD_NS))
BASE_TIME=$((PHC_NS - REMAINDER + GCL_PERIOD_NS * 3))  # 3 주기 후 시작

# 5. taprio GCL 설정
log "taprio GCL 설정 (base-time: $BASE_TIME)..."
tc qdisc add dev $IFACE parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $BASE_TIME \
    sched-entry S 0x80  100000  \  # TC7: Safety (0~100µs)
    sched-entry S 0x20  400000  \  # TC5: Sensor (100~500µs)
    sched-entry S 0x10  500000  \  # TC4: Video  (500~1000µs)
    sched-entry S 0x01  1000000 \  # TC0: BE     (1000~2000µs)
    flags 0x2 \
    clockid CLOCK_TAI

# 6. CBS 설정 (TC5: 10Mbps, TC4: 50Mbps)
log "CBS 설정..."
tc qdisc add dev $IFACE parent 100:6 handle 110 cbs \
    idleslope 10000 sendslope -990000 hicredit 15 locredit -1485 offload 0
tc qdisc add dev $IFACE parent 100:5 handle 120 cbs \
    idleslope 50000 sendslope -950000 hicredit 75 locredit -7125 offload 0

# 7. tc filter 설정
log "트래픽 분류 필터 설정..."
# Safety VLAN (PCP 7) → TC7
tc filter add dev $IFACE parent 100: protocol 802.1Q prio 1 flower \
    vlan_id 10 vlan_prio 7 action skbedit priority 7

# SOME/IP 제어 (UDP 30490) → TC7
tc filter add dev $IFACE parent 100: protocol ip prio 2 u32 \
    match ip protocol 17 0xff \
    match ip dport 30490 0xffff \
    action skbedit priority 7

# 센서 VLAN (PCP 5) → TC5
tc filter add dev $IFACE parent 100: protocol 802.1Q prio 3 flower \
    vlan_id 30 vlan_prio 5 action skbedit priority 5

# DoIP 진단 (TCP 13400) → TC0 (BE)
tc filter add dev $IFACE parent 100: protocol ip prio 10 u32 \
    match ip protocol 6 0xff \
    match ip dport 13400 0xffff \
    action skbedit priority 0

log "TSN 스택 설정 완료!"
tc qdisc show dev $IFACE
```

---

## 설정 검증 및 모니터링

```bash
# ── qdisc 상태 확인 ──
tc qdisc show dev eth0
# qdisc taprio 100: root refcnt 2 num_tc 8 ...
#   base-time 1700000000100000000 cycle-time 2000000
#   tc 0 count 1 offset 0
#   ...

# ── 통계 포함 상세 확인 ──
tc -s qdisc show dev eth0
# Sent N bytes M pkt (dropped D, overlimits O requeues R)
# dropped: GCL 게이트 닫혀 폐기된 패킷 (0이어야 정상)

# ── 트래픽 클래스별 통계 ──
tc -s class show dev eth0

# ── Wireshark로 GCL 동작 검증 ──
# TC7 트래픽이 2ms 주기의 0~100µs 구간에만 집중되는지 확인
tshark -r capture.pcap \
    -Y "vlan.priority == 7" \
    -T fields \
    -e frame.time_epoch \
    -e frame.len | \
awk '{
    t_ns = int($1 * 1e9)
    slot = t_ns % 2000000   # 2ms 주기 내 위치 (ns)
    printf "슬롯 내 위치: %d ns (%.0fµs)\n", slot, slot/1000
}' | sort -n | head -20
# → 모든 TC7 패킷이 0~100,000 ns 구간에 집중되면 GCL 정상 동작

# ── PTP와 GCL 정렬 확인 ──
pmc -u -b 0 'GET CURRENT_DATA_SET'
# offsetFromMaster -45 → 오프셋이 GCL 슬롯 크기(100µs)보다 훨씬 작으면 정상
```

---

## 일반적인 실수와 해결 방법

```
문제 1: "Error: Failed to find qdisc with specified classid"
  원인: parent 클래스 지정 오류
  해결: tc qdisc show 로 handle 확인 후 정확한 parent 지정
        tc qdisc show dev eth0 | grep handle

문제 2: taprio 설정 후 패킷이 모두 TC0으로 분류됨
  원인: VLAN 태그가 없거나 tc filter가 없음
  확인: tcpdump -i eth0 -e -c 10 | grep vlan
  해결: ip link set ${IFACE}.10 type vlan egress-qos-map 7:7 설정

문제 3: "Error: Specified queuecount is too large"
  원인: NIC HW 큐 수 < num_tc
  확인: ethtool -l eth0 | grep TX
  해결: num_tc를 NIC 큐 수 이하로 줄이거나 flags 0x0 (SW 모드) 사용

문제 4: GCL 주기가 PTP와 맞지 않아 지터 발생
  원인: base-time이 PTP 동기화 이전에 설정됨
  해결:
    1. ptp4l 동기화 완료 확인 (pmc GET CURRENT_DATA_SET)
    2. PTP 동기화 후 PHC 시각 기반 base-time 재계산
    3. tc qdisc replace로 base-time 업데이트

문제 5: CBS locredit/hicredit 계산 오류
  증상: 특정 트래픽이 전혀 전송 안 됨 또는 과도한 대역 사용
  해결:
    idleslope = 보장 대역폭 (kbps)
    sendslope = idleslope - 링크 속도 (kbps)
    hicredit  = ceil(maxFrameSize * idleslope / linkSpeed)  (Bytes)
    locredit  = floor(maxFrameSize * sendslope / linkSpeed) (Bytes, 음수)
```

---

## Reference
- [Linux Kernel - tc-taprio(8)](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
- [Linux Kernel - tc-cbs(8)](https://man7.org/linux/man-pages/man8/tc-cbs.8.html)
- [Linux Kernel - tc-flower(8)](https://man7.org/linux/man-pages/man8/tc-flower.8.html)
- [Linux Kernel - tc-u32(8)](https://man7.org/linux/man-pages/man8/tc-u32.8.html)
- [IEEE 802.1Qbv - Time-Aware Shaper Specification](https://standards.ieee.org/ieee/802.1Qbv/6068/)
- [IEEE 802.1Qav - Credit-Based Shaper](https://standards.ieee.org/ieee/802.1Qav/3684/)
- [OpenAvnu - TSN Linux Setup Guide](https://github.com/Avnu/OpenAvnu)
- [Intel - TSN Configuration Guide for i225/i226](https://www.intel.com/content/www/us/en/developer/articles/technical/time-sensitive-networking.html)
- [Marvell - 88Q5072 TSN Switch Configuration Guide](https://www.marvell.com/products/automotive-solutions/88q5072.html)
