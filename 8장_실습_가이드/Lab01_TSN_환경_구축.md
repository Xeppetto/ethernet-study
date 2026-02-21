# Lab 01: TSN 환경 구축 및 기본 설정

## 개요
이 실습에서는 Linux 환경에서 TSN(Time-Sensitive Networking) 기본 환경을 구축합니다. PC 2대(또는 VM)를 Ethernet으로 연결하고, PTP 시간 동기화, VLAN 설정, TAS 스케줄링을 단계별로 실습합니다.

**필요 환경:**
- Ubuntu 22.04 LTS (또는 22.04 RT)
- Intel i210/i225/i226 NIC (하드웨어 타임스탬핑 지원)
- Ethernet 케이블 (직결 또는 TSN 스위치)
- linuxptp, iproute2, ethtool 패키지

---

## 사전 준비

```bash
# 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y \
    linuxptp \
    ethtool \
    iproute2 \
    tcpdump \
    wireshark \
    python3-pip \
    net-tools \
    iperf3

# Python 패키지
pip3 install scapy paho-mqtt

# NIC 지원 확인
ethtool -T eth0
# Capabilities에 hardware-transmit, hardware-receive 확인

# 커널 버전 확인 (5.4+ 권장)
uname -r

# PREEMPT_RT 커널 (선택 사항)
sudo apt-get install linux-image-$(uname -r)-rt 2>/dev/null || \
    echo "RT 커널 별도 빌드 필요"
```

---

## Step 1: 네트워크 인터페이스 설정

```bash
# ── Node A (Grandmaster) ──────────────────────────────────────
# 인터페이스 확인
ip link show

# IP 설정 (정적)
sudo ip addr add 192.168.100.1/24 dev eth0
sudo ip link set eth0 up

# VLAN 생성
# VLAN 10: Safety (PCP 7)
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 192.168.10.1/24 dev eth0.10
sudo ip link set eth0.10 up

# VLAN 20: Control (PCP 5)
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.20.1/24 dev eth0.20
sudo ip link set eth0.20 up

# ── Node B (Slave) ────────────────────────────────────────────
sudo ip addr add 192.168.100.2/24 dev eth0
sudo ip link set eth0 up

sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 192.168.10.2/24 dev eth0.10
sudo ip link set eth0.10 up

sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.20.2/24 dev eth0.20
sudo ip link set eth0.20 up

# 연결 확인
ping -c 3 192.168.100.2
ping -c 3 192.168.10.2  # VLAN 10
ping -c 3 192.168.20.2  # VLAN 20
```

---

## Step 2: PTP/gPTP 시간 동기화

```bash
# ── Node A (Grandmaster) ──────────────────────────────────────
# ptp4l 설정 파일
sudo cat > /etc/linuxptp/gm.conf << 'EOF'
[global]
transportSpecific   1
domainNumber        0
priority1           1
priority2           1
clockClass          6
time_stamping       hardware
delay_mechanism     P2P
logSyncInterval     -3
logMinPdelayReqInterval -3
logAnnounceInterval 1
verbose             1

[eth0]
EOF

# Grandmaster 실행
sudo ptp4l -f /etc/linuxptp/gm.conf -i eth0 &

# PHC → 시스템 클록 동기화
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 37 -m &

# ── Node B (Slave) ────────────────────────────────────────────
sudo cat > /etc/linuxptp/slave.conf << 'EOF'
[global]
transportSpecific   1
domainNumber        0
priority1           128
priority2           128
time_stamping       hardware
delay_mechanism     P2P
logSyncInterval     -3
logMinPdelayReqInterval -3
logAnnounceInterval 1
verbose             1

[eth0]
EOF

sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0 &
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 37 -m &

# ── 검증 ─────────────────────────────────────────────────────
# 10초 후 동기화 확인
sleep 10
sudo pmc -u -b 0 'GET CURRENT_DATA_SET'
# offsetFromMaster가 ±1000ns(1µs) 이내면 성공

# Wireshark로 PTP 확인
sudo tshark -i eth0 -Y "ptp" -c 20 \
    -T fields -e frame.time_relative -e ptp.v2.messagetype
```

---

## Step 3: TAS (Time-Aware Shaper) 설정

```bash
# ── Node A (Grandmaster 겸 TAS 설정) ──────────────────────────

# GCL 설계: 1ms 주기
# [0.0~0.5ms] TC7 Open (Safety/E-Stop)
# [0.5~0.8ms] TC5 Open (Control)
# [0.8~1.0ms] TC0 Open (Best Effort)

# 기존 qdisc 제거
sudo tc qdisc del dev eth0 root 2>/dev/null

# TAI 기준 시각 계산 (현재 시각 기준)
CURRENT_TAI=$(python3 -c "
import time
# TAI = UTC + 37초
tai = time.time() + 37
# 다음 1초 경계로 올림
next_s = (int(tai) + 1) * 1e9
print(int(next_s))
")

# taprio 설정
sudo tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 0 0 0 0 5 6 7 0 0 0 0 0 0 0 0 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time $CURRENT_TAI \
    sched-entry S 0x80 500000  \
    sched-entry S 0x20 300000  \
    sched-entry S 0x01 200000  \
    flags 0x0 \
    clockid CLOCK_TAI

echo "[TAS] 설정 완료"
tc qdisc show dev eth0

# 필터 추가 (VLAN PCP → TC 매핑)
sudo tc filter add dev eth0 parent 100: protocol 802.1Q prio 1 flower \
    vlan_prio 7 action skbedit priority 7  # Safety

sudo tc filter add dev eth0 parent 100: protocol 802.1Q prio 2 flower \
    vlan_prio 5 action skbedit priority 5  # Control
```

---

## Step 4: 검증 및 측정

```bash
# ── 지연 측정 (iperf3) ────────────────────────────────────────
# Node B에서 서버 실행
iperf3 -s &

# Node A에서 측정 (VLAN 10 Safety)
iperf3 -c 192.168.10.2 -u -b 1M -l 64 -t 30 \
    --bind 192.168.10.1 --json > lab01_safety_result.json

# Node A에서 측정 (Best Effort 동시 부하)
iperf3 -c 192.168.100.2 -b 900M -t 30 &
iperf3 -c 192.168.10.2 -u -b 1M -l 64 -t 30 \
    --bind 192.168.10.1 --json > lab01_safety_under_load.json

# ── 결과 분석 ────────────────────────────────────────────────
python3 << 'EOF'
import json

def print_udp_stats(filename, label):
    with open(filename) as f:
        data = json.load(f)
    end_sum = data['end']['sum']
    print(f"\n=== {label} ===")
    print(f"  대역폭: {end_sum['bits_per_second']/1e6:.2f} Mbps")
    print(f"  Jitter: {end_sum['jitter_ms']:.3f} ms")
    print(f"  손실:   {end_sum['lost_percent']:.3f}%")

print_udp_stats('lab01_safety_result.json', 'TAS 없이 (순수 측정)')
print_udp_stats('lab01_safety_under_load.json', 'TAS 있음 (부하 하)')
EOF

# ── Wireshark 캡처 (선택) ────────────────────────────────────
sudo tshark -i eth0 -a duration:30 -w /tmp/lab01_capture.pcap
# → Wireshark에서 Statistics > I/O Graph로 GCL 패턴 시각화
```

---

## 실습 결과 체크리스트

```
Lab 01 완료 기준:

PTP 동기화:
  ☐ offsetFromMaster < ±1000ns 달성
  ☐ ptp4l 로그에서 SLAVE 상태 확인
  ☐ phc2sys 실행 확인

VLAN:
  ☐ VLAN 10, 20 ping 성공
  ☐ Wireshark에서 802.1Q 태그 확인

TAS:
  ☐ taprio qdisc 설정 완료
  ☐ TC7 필터 동작 확인

성능 검증:
  ☐ Safety VLAN Jitter < 1ms (TAS 없이)
  ☐ Safety VLAN Jitter < 0.1ms (TAS 있음, 선택)
  ☐ 부하 하에서 Safety 대역폭 보장 확인

고민해 볼 것:
  Q1. TAS 슬롯 크기를 줄이면 Jitter는 어떻게 변하나요?
  Q2. PTP 동기화가 안 된 상태에서 TAS를 쓰면 어떤 문제가 생기나요?
  Q3. Guard Band 없이 TAS를 쓰면 어떤 위험이 있나요?
```

---

## 트러블슈팅

```bash
# 문제 1: ptp4l "port 1: LISTENING" 상태에서 안 넘어감
# 원인: NIC가 PTP 하드웨어 타임스탬핑 미지원
ethtool -T eth0 | grep "hardware-transmit"
# 없으면: time_stamping software 로 변경
# → 정밀도 낮아짐 (수십 µs)

# 문제 2: taprio "RTNETLINK answers: Operation not supported"
# 원인: 커널 버전 부족 (5.0 미만) 또는 NIC 미지원
uname -r  # 5.4+ 필요
modprobe ifb  # 필요 시

# 문제 3: VLAN 패킷 미수신
# 원인: NIC VLAN stripping 설정
ethtool -K eth0 rx-vlan-filter off
ethtool -K eth0 rx-vlan-offload off

# 문제 4: phc2sys 오프셋 큰 경우
# CLOCK_TAI 기준 확인
cat /sys/class/net/eth0/ptp/ptp0/clock_name
timedatectl | grep "Universal time"
```
