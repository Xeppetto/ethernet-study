# tc taprio - TSN 트래픽 제어 실습

## 개요
Linux Traffic Control(tc)의 `taprio` qdisc는 IEEE 802.1Qbv(TAS, Time-Aware Shaper)를 소프트웨어로 구현합니다. 하드웨어 오프로드를 지원하는 NIC와 함께 사용하면 마이크로초 수준의 정밀한 스케줄링이 가능합니다. `tc`는 TSN 외에도 `cbs`(CBS, Credit-Based Shaper), `mqprio`, `fq`(Flow Queue), `htb` 등 다양한 qdisc를 포함합니다.

---

## tc 기본 개념

```
tc 계층 구조:

NIC tx Queue
    │
    ▼
┌───────────────────────────────────────┐
│ qdisc (Queueing Discipline)           │
│  예: taprio, cbs, mqprio, htb, fq    │
│                                       │
│  ┌──────────┐  ┌──────────┐          │
│  │  class   │  │  class   │ ...      │
│  │  (TC0)   │  │  (TC1)   │          │
│  └────┬─────┘  └────┬─────┘          │
│       │              │               │
│  ┌────▼─────┐  ┌────▼─────┐         │
│  │  filter  │  │  filter  │          │
│  │  (분류)  │  │  (분류)  │          │
│  └──────────┘  └──────────┘          │
└───────────────────────────────────────┘

트래픽 분류 흐름:
  패킷 → qdisc → 우선순위(PCP/DSCP) → TC 매핑 → Queue → 전송
```

---

## taprio qdisc - TAS 구현

### 기본 설정

```bash
# taprio GCL (Gate Control List) 설정 예시
# 2ms 주기: TC7(1ms 제어) + TC5(0.5ms 센서) + TC0(0.5ms BE)

# 기존 qdisc 제거
sudo tc qdisc del dev eth0 root 2>/dev/null

# taprio 적용
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $(date +%s%N) \
    sched-entry S 0x80 1000000 \
    sched-entry S 0x20 500000  \
    sched-entry S 0x01 500000  \
    flags 0x0 \
    clockid CLOCK_TAI

# 파라미터 설명:
# num_tc 8             : 트래픽 클래스 8개 (TC0~TC7)
# map 0 0 0 0 0 5 6 7 : PCP 0~4 → TC0, PCP 5 → TC5, PCP 6 → TC6, PCP 7 → TC7
# queues 1@0 ...       : 각 TC에 큐 1개씩 할당
# base-time            : GCL 시작 기준 시각 (나노초, TAI)
# sched-entry S 0x80 1000000 : TC7(0x80=bit7) Open, 1ms 유지
# sched-entry S 0x20 500000  : TC5(0x20=bit5) Open, 0.5ms 유지
# sched-entry S 0x01 500000  : TC0(0x01=bit0) Open, 0.5ms 유지
# flags 0x0            : 소프트웨어 모드 (0x2 = TX 오프로드)
# clockid CLOCK_TAI    : TAI 기준 시계 (PTP와 연동)
```

### 하드웨어 오프로드 설정

```bash
# Intel i225/i226 하드웨어 TAS 설정 (flags 0x2)
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 4 \
    map 0 0 0 0 1 1 2 3 0 0 0 0 0 0 0 0 \
    queues 2@0 2@2 1@4 1@5 \
    base-time 1700000000000000000 \
    sched-entry S 0x08 100000  \
    sched-entry S 0x04 200000  \
    sched-entry S 0x03 700000  \
    flags 0x2 \
    clockid CLOCK_TAI
# flags 0x2 = TXTIME_OFFLOAD: NIC 하드웨어가 GCL 직접 처리 → 정밀도 향상

# 설정 확인
tc qdisc show dev eth0
tc -s qdisc show dev eth0     # 통계 포함
```

---

## mqprio qdisc - 멀티큐 우선순위

```bash
# mqprio: 하드웨어 큐를 TC에 직접 매핑
# Intel i225는 4개 TX 큐 지원

sudo tc qdisc add dev eth0 parent root handle 100 mqprio \
    num_tc 4 \
    map 0 0 0 0 1 1 2 3 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 \
    hw 1    # 하드웨어 큐 사용

# 802.1Q CoS(PCP)와 TC 매핑:
# PCP 0,1,2,3 → TC0 (BE)
# PCP 4,5     → TC1 (Sensor)
# PCP 6       → TC2 (Control)
# PCP 7       → TC3 (Safety)
```

---

## CBS (Credit-Based Shaper) 설정

```bash
# TC5 (Sensor Stream) CBS 설정 - IEEE 802.1Qav
# 가정: 10Mbps 대역폭 보장, 링크 1Gbps

# CBS 파라미터 계산:
# idleslope = sendrate = 10,000,000 bps
# sendslope = idleslope - linkrate = 10,000,000 - 1,000,000,000 = -990,000,000
# hicredit = max_frame_size * (idleslope / linkrate) ≈ 1500 * 0.01 = 15
# locredit = max_frame_size * (sendslope / linkrate) ≈ 1500 * (-0.99) ≈ -1485

# TC5에 CBS 적용 (taprio의 하위 qdisc)
sudo tc qdisc add dev eth0 parent 100:6 cbs \
    idleslope 10000  \
    sendslope -990000 \
    hicredit 12 \
    locredit -1440 \
    offload 1

# CBS 통계 확인
tc -s qdisc show dev eth0
```

---

## 필터 (tc filter) - 트래픽 분류

```bash
# u32 필터 - IP/포트 기반 분류
# SOME/IP (UDP 30490) → TC7 (최고 우선순위)
sudo tc filter add dev eth0 parent 100: protocol ip prio 1 u32 \
    match ip protocol 17 0xff \
    match ip dport 30490 0xffff \
    action skbedit priority 7    # TC7로 분류

# flower 필터 - MAC/VLAN 기반 분류
sudo tc filter add dev eth0 parent 100: protocol 802.1Q prio 2 flower \
    vlan_id 10 \
    vlan_prio 7 \
    action skbedit priority 7

# DoIP (TCP 13400) → TC5
sudo tc filter add dev eth0 parent 100: protocol ip prio 3 u32 \
    match ip protocol 6 0xff \
    match ip dport 13400 0xffff \
    action skbedit priority 5

# 필터 확인
tc filter show dev eth0 parent 100:
```

---

## 지연 측정 도구 (sock_txtime)

```bash
# SO_TXTIME 소켓 옵션 - 정확한 전송 시각 지정
# taprio + SO_TXTIME 조합으로 ns 수준 전송 제어

# txtime 지원 NIC 확인
ethtool -T eth0 | grep "hardware-transmit"

# Linux 소켓 SO_TXTIME 예시 (개념 - C 코드)
# struct sock_txtime {
#     __kernel_clockid_t clockid;  /* CLOCK_TAI */
#     __u32 flags;
# };
# setsockopt(fd, SOL_SOCKET, SO_TXTIME, &txtime, sizeof(txtime));
# cmsg: SCM_TXTIME = 전송 시각 (TAI 나노초)

# cyclictest로 스케줄링 지연 측정 (PREEMPT_RT 커널)
sudo cyclictest -p 99 -t 4 -n -i 1000 -l 10000
# T: 0 (  0) I: 1000 C:  10000 Min:     15 Act:     21 Avg:     23 Max:     156
# → Min/Avg/Max latency in µs
```

---

## 전체 TSN 스택 설정 스크립트

```bash
#!/bin/bash
# 의료 로봇 TSN 설정 스크립트
# - ptp4l: 시간 동기화
# - taprio: TAS (GCL)
# - cbs: CBS 대역폭 보장

IFACE="eth0"

# 1. 기존 설정 초기화
tc qdisc del dev $IFACE root 2>/dev/null
ip link set $IFACE down
ip link set $IFACE up

# 2. VLAN 설정 (VLAN 10: Safety, VLAN 20: Control, VLAN 30: Sensor)
ip link add link $IFACE name ${IFACE}.10 type vlan id 10
ip link add link $IFACE name ${IFACE}.20 type vlan id 20
ip addr add 192.168.10.1/24 dev ${IFACE}.10
ip addr add 192.168.20.1/24 dev ${IFACE}.20

# 3. taprio 설정 (주기: 2ms = 2,000,000 ns)
BASE=$(date +%s)000000000  # 현재 시각 기준 (TAI 변환 필요)

tc qdisc add dev $IFACE parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $BASE \
    sched-entry S 0x80 1000000 \
    sched-entry S 0x20 500000  \
    sched-entry S 0x01 500000  \
    flags 0x0 \
    clockid CLOCK_TAI

# 4. TC7 필터 (Safety E-Stop)
tc filter add dev $IFACE parent 100: protocol 802.1Q prio 1 flower \
    vlan_id 10 vlan_prio 7 \
    action skbedit priority 7

# 5. TC5 필터 (Sensor stream) + CBS
tc filter add dev $IFACE parent 100: protocol ip prio 2 u32 \
    match ip dport 30490 0xffff \
    action skbedit priority 5

tc qdisc add dev $IFACE parent 100:6 cbs \
    idleslope 10000 sendslope -990000 \
    hicredit 12 locredit -1440 offload 0

echo "[TSN] 설정 완료"
tc qdisc show dev $IFACE
```

---

## 설정 검증 및 모니터링

```bash
# qdisc 상태 및 통계
tc -s qdisc show dev eth0
# qdisc taprio 100: root refcnt 2 ...
#  Sent 12345 bytes 100 pkt (dropped 0, overlimits 0 requeues 0)
#  backlog 0b 0p requeues 0

# 트래픽 클래스별 통계
tc -s class show dev eth0

# 필터 통계
tc -s filter show dev eth0 parent 100:

# Wireshark로 TAS 동작 검증
# 캡처 후 Statistics → I/O Graph
# 시간 축에서 GCL 슬롯별 트래픽 패턴 확인
tshark -r capture.pcap \
    -Y "vlan.priority == 7" \
    -T fields -e frame.time_relative -e frame.len \
    | awk '{t=int($1*1000)%2; printf "%d %s %s\n", t, $1, $2}'
# → 2ms 주기에서 0~1ms에만 TC7 트래픽 집중 확인
```

---

## Reference
- [Linux Kernel - tc-taprio](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
- [Linux Kernel - tc-cbs](https://man7.org/linux/man-pages/man8/tc-cbs.8.html)
- [Linux Kernel - tc-flower](https://man7.org/linux/man-pages/man8/tc-flower.8.html)
- [OpenAvnu - TSN Linux Setup](https://github.com/Avnu/OpenAvnu)
- [Intel - TSN Linux Driver Notes](https://www.intel.com/content/www/us/en/developer/articles/technical/time-sensitive-networking.html)
