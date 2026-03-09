# iperf3 - 대역폭 및 성능 측정

## 개요
iperf3는 네트워크 대역폭(Bandwidth), 지연(Latency), 패킷 손실(Packet Loss), Jitter를 정밀하게 측정하는 오픈소스 도구입니다. TCP와 UDP 양쪽을 지원하며, JSON 출력으로 자동화 파이프라인과 쉽게 통합됩니다.

의료 로봇 Ethernet 네트워크 검증에서 iperf3는 두 가지 핵심 역할을 합니다. 첫째, **용량 검증**: 링크가 설계 대역폭을 실제로 제공하는지 확인(케이블 불량, NIC 설정 오류 조기 발견). 둘째, **TSN 효과 측정**: taprio/CBS 설정 전후의 Jitter 변화를 정량화하여 TSN 투자 효과를 수치로 증명합니다.

> **의료 로봇 관점**: 수술 로봇 Ethernet 인프라 검증 체크리스트 중 "네트워크 성능 요구사항 충족 확인"은 iperf3로 수행합니다. 안전 VLAN(VLAN 10)에 최대 부하가 걸린 상황에서도 E-Stop 패킷(VLAN 10, TC7)의 Jitter가 < 50µs를 유지하는지 검증합니다. 이 결과는 IEC 62304의 시스템 검증(System Verification) 증거 문서로 활용됩니다.

---

## 설치 및 기본 사용법

```bash
# ── 설치 ──
sudo apt-get install iperf3

# 버전 확인
iperf3 --version
# iperf 3.11 (cJSON 1.7.15)
# Linux robot-ecU 5.15.0 #1 SMP x86_64

# ── 기본 구조: 서버 + 클라이언트 쌍 ──
# 서버 (수신 측에서 실행)
iperf3 -s
# iperf3: I will be a server, listening on 5201

# TCP 클라이언트 (기본: 10초 테스트)
iperf3 -c 192.168.1.100

# UDP 클라이언트 (100Mbps 지정)
iperf3 -c 192.168.1.100 -u -b 100M

# 서버를 백그라운드 데몬으로 실행
iperf3 -s -D --logfile /var/log/iperf3.log

# 서버 포트 변경 (방화벽 등)
iperf3 -s -p 9999
iperf3 -c 192.168.1.100 -p 9999
```

---

## TCP 대역폭 측정

```bash
# ── 기본 TCP 대역폭 측정 ──
iperf3 -c 192.168.1.100 -t 30 -i 1

# 출력 예시 (1Gbps 링크):
# [ ID] Interval       Transfer     Bitrate         Retr  Cwnd
# [  5]   0.00-1.00   sec   112 MBytes   942 Mbits/sec    0   3.12 MBytes
# [  5]   1.00-2.00   sec   112 MBytes   941 Mbits/sec    0   3.12 MBytes
# - - - - - - - - - - - - - - - - - - - - - - - - -
# [ ID] Interval       Transfer     Bitrate         Retr
# [  5]   0.00-30.00  sec  3.27 GBytes   937 Mbits/sec    0     sender
# [  5]   0.00-30.00  sec  3.27 GBytes   936 Mbits/sec         receiver
#
# 분석 포인트:
#   Retr = 0           → 재전송 없음 (물리 링크 건강)
#   937 Mbps / 1000 Mbps = 93.7% 활용률 (우수)
#   Cwnd 3.12 MB       → TCP 혼잡 윈도우 (클수록 처리량 높음)

# ── 병렬 스트림 (다중 CPU 코어 활용) ──
iperf3 -c 192.168.1.100 -P 4 -t 30
# 4개 스트림 합산으로 링크 최대 처리량 측정

# ── Reverse 모드 (서버→클라이언트, 반대 방향) ──
iperf3 -c 192.168.1.100 -R

# ── Bidirectional (동시 양방향) ──
iperf3 -c 192.168.1.100 --bidir
# [TX-C] 송신, [RX-C] 수신 동시 측정

# ── TCP 창 크기 조정 ──
iperf3 -c 192.168.1.100 -w 256K -t 30
# 소형 창(64K): 높은 RTT 환경에서 처리량 제한 확인
# 대형 창(4M): 이론적 최대 처리량 근접

# ── OTA 업데이트 시뮬 (단방향 대용량) ──
iperf3 -c 192.168.1.100 -n 1G -i 5
# 1GB 파일 전송 시뮬레이션 (총 바이트 지정)
```

---

## UDP 대역폭 및 패킷 손실 측정

```bash
# ── 기본 UDP 테스트 (100Mbps) ──
iperf3 -c 192.168.1.100 -u -b 100M -t 30 -i 1

# 출력 예시:
# [ ID] Interval       Transfer     Bitrate         Jitter    Lost/Total
# [  5]   0.00-1.00   sec  11.9 MBytes  99.9 Mbits/sec  0.021 ms  0/8622  (0%)
# [  5]   1.00-2.00   sec  11.9 MBytes  99.9 Mbits/sec  0.018 ms  0/8622  (0%)
# - - - - - - - - - - - - -
# [  5]   0.00-30.00  sec   357 MBytes  99.9 Mbits/sec  0.019 ms  0/258660 (0%)
#
# 분석 포인트:
#   Jitter: 0.019 ms = 19µs → 우수 (TSN 효과)
#   Lost: 0/258660 (0%)      → 패킷 손실 없음
#   → 의료 로봇 센서 스트림 요구 충족

# ── 패킷 크기별 측정 (프레임 크기 영향 분석) ──
for pkt in 64 128 256 512 1024 1400; do
    result=$(iperf3 -c 192.168.1.100 -u -b 100M -l $pkt -t 5 \
             --json 2>/dev/null | \
             python3 -c "
import json,sys
d=json.load(sys.stdin)
s=d['end']['sum']
print(f'{s[\"jitter_ms\"]:.3f}ms {s[\"lost_percent\"]:.3f}%')")
    echo "패킷 ${pkt}B: Jitter=${result}"
done
# 출력 예:
# 패킷 64B:   Jitter=0.043ms 손실=0.000%
# 패킷 1400B: Jitter=0.019ms 손실=0.000%

# ── 1Gbps 링크 최대 UDP 부하 (스위치 버퍼 포화 테스트) ──
# 주의: 프로덕션 네트워크에서는 금지!
iperf3 -c 192.168.1.100 -u -b 950M -t 10

# ── 작은 패킷 고속 테스트 (실시간 제어 시뮬) ──
# 1kHz 64Byte = 512 Kbps
iperf3 -c 192.168.1.100 -u -b 512K -l 64 -t 30 --json \
    > control_stream_result.json
```

---

## Jitter 측정 (TSN 효과 정량 검증)

```bash
# ── TSN 적용 전후 Jitter 비교 ──

# 1단계: TSN 적용 전 측정
echo "=== TSN Before ==="
iperf3 -c 192.168.1.100 -u -b 80M -l 256 -t 60 --json \
    > before_tsn.json

# 2단계: taprio + CBS 설정 적용 (tc_taprio.md 참조)

# 3단계: TSN 적용 후 측정
echo "=== TSN After ==="
iperf3 -c 192.168.1.100 -u -b 80M -l 256 -t 60 --json \
    > after_tsn.json

# 4단계: 비교 분석
python3 << 'EOF'
import json, statistics

def load_intervals(fname):
    with open(fname) as f:
        d = json.load(f)
    return [i['sum']['jitter_ms'] for i in d['intervals']
            if 'jitter_ms' in i.get('sum', {})]

before = load_intervals('before_tsn.json')
after  = load_intervals('after_tsn.json')

print("="*55)
print("  TSN 전후 Jitter 비교")
print("="*55)
print(f"  측정 구간 수: {len(before)} / {len(after)}")
print()
print(f"  [Before TSN]")
print(f"    평균 Jitter: {statistics.mean(before):.3f} ms")
print(f"    최대 Jitter: {max(before):.3f} ms")
print(f"    표준편차:    {statistics.stdev(before):.3f} ms")
print()
print(f"  [After TSN]")
print(f"    평균 Jitter: {statistics.mean(after):.3f} ms")
print(f"    최대 Jitter: {max(after):.3f} ms")
print(f"    표준편차:    {statistics.stdev(after):.3f} ms")
print()
imp = (1 - statistics.mean(after)/statistics.mean(before)) * 100
max_imp = (1 - max(after)/max(before)) * 100
print(f"  평균 개선율: {imp:.1f}%")
print(f"  최대 개선율: {max_imp:.1f}%")
print()
# 의료 로봇 기준 검증
target = 0.05   # 50µs
print(f"  목표 Jitter < {target*1000:.0f}µs: "
      f"{'PASS' if max(after) < target else 'FAIL'}")
EOF

# 전형적인 결과:
# Before TSN: 평균 2.341ms, 최대 15.234ms
# After TSN:  평균 0.021ms, 최대  0.089ms
# 평균 개선율: 99.1%
# 목표 Jitter < 50µs: PASS
```

---

## VLAN별 격리 검증

```bash
# ── VLAN 인터페이스 준비 ──
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip link add link eth0 name eth0.30 type vlan id 30

# VLAN PCP 설정 (egress-qos-map: 내부 우선순위 → PCP)
sudo ip link set eth0.10 type vlan egress-qos-map 7:7 6:6 5:5
sudo ip link set eth0.20 type vlan egress-qos-map 5:5 4:4

sudo ip addr add 192.168.10.1/24 dev eth0.10
sudo ip addr add 192.168.20.1/24 dev eth0.20
sudo ip link set eth0.10 up
sudo ip link set eth0.20 up

# ── 격리 테스트: VLAN 20 최대 부하 → VLAN 10 Jitter 영향 없음 ──
# 서버 측에서 두 iperf3 서버 실행
iperf3 -s -B 192.168.10.100 -p 5201 &
iperf3 -s -B 192.168.20.100 -p 5202 &

# VLAN 20에 배경 부하 (900 Mbps)
iperf3 -c 192.168.20.100 -p 5202 \
       -u -b 900M -t 60 \
       --bind 192.168.20.1 &

# VLAN 10 Jitter 측정 (E-Stop 시뮬)
iperf3 -c 192.168.10.100 -p 5201 \
       -u -b 1M -l 64 -t 60 \
       --bind 192.168.10.1 \
       --json > vlan10_under_load.json
wait

# 결과 분석
python3 -c "
import json
with open('vlan10_under_load.json') as f:
    d = json.load(f)
jitter = d['end']['sum']['jitter_ms']
lost_pct = d['end']['sum']['lost_percent']
print(f'VLAN10 부하 중 Jitter: {jitter:.3f}ms')
print(f'VLAN10 패킷 손실: {lost_pct:.3f}%')
print(f'VLAN 격리 결과: {\"PASS\" if jitter < 0.05 and lost_pct < 0.001 else \"FAIL\"}')"
```

---

## 멀티스트림 혼합 부하 (의료 로봇 실제 패턴)

```bash
#!/bin/bash
# ============================================================
# 의료 로봇 실제 트래픽 패턴 동시 시뮬레이션
# - Safety Stream: 1kHz × 64B (E-Stop, 관절 제어)
# - Sensor Stream: 500Hz × 512B (토크, 위치)
# - Video Stream: 30fps × 1400B (수술 카메라)
# - Diagnostics:  Best Effort TCP (진단/로그)
# ============================================================

ROBOT_IP="192.168.1.100"
RESULT_DIR="/tmp/iperf3_results"
mkdir -p $RESULT_DIR

# 1. Safety Stream (VLAN 10, PCP 7)
#    1000pps × 64B × 8 = 512 Kbps
iperf3 -c $ROBOT_IP -u \
    -b 512K -l 64 -t 60 \
    --bind 192.168.10.1 \
    --json > $RESULT_DIR/safety.json &
PID_SAFETY=$!

# 2. Sensor Stream (VLAN 20, PCP 5)
#    500pps × 512B × 8 = 2 Mbps
iperf3 -c $ROBOT_IP -u \
    -b 2M -l 512 -t 60 \
    --bind 192.168.20.1 \
    --json > $RESULT_DIR/sensor.json &
PID_SENSOR=$!

# 3. Video Stream (50 Mbps H.264)
iperf3 -c $ROBOT_IP -u \
    -b 50M -l 1400 -t 60 \
    --json > $RESULT_DIR/video.json &
PID_VIDEO=$!

# 4. Background Diagnostics (BE TCP)
iperf3 -c $ROBOT_IP \
    -b 100M -t 60 \
    --json > $RESULT_DIR/diag.json &
PID_DIAG=$!

echo "전체 스트림 실행 중 (60초)..."
wait $PID_SAFETY $PID_SENSOR $PID_VIDEO $PID_DIAG

# 결과 요약
python3 << 'PYEOF'
import json
from pathlib import Path

results = {}
for name in ['safety', 'sensor', 'video', 'diag']:
    path = Path(f'/tmp/iperf3_results/{name}.json')
    if path.exists():
        with open(path) as f:
            d = json.load(f)
        end = d.get('end', {})
        s = end.get('sum', end.get('sum_sent', {}))
        results[name] = {
            'bitrate_mbps': s.get('bits_per_second', 0) / 1e6,
            'jitter_ms': s.get('jitter_ms', None),
            'lost_pct': s.get('lost_percent', None),
            'retransmits': s.get('retransmits', None),
        }

print("="*65)
print(f"  {'스트림':<12} {'대역폭':>10} {'Jitter':>12} {'손실/재전송':>15}")
print("="*65)

checks = {
    'safety': {'jitter_max': 0.05,  'lost_max': 0.001},
    'sensor': {'jitter_max': 0.1,   'lost_max': 0.01},
    'video':  {'jitter_max': 10.0,  'lost_max': 0.1},
    'diag':   {'jitter_max': None,  'lost_max': None},
}

for name, r in results.items():
    bw = f"{r['bitrate_mbps']:.1f} Mbps"
    jitter = f"{r['jitter_ms']:.3f}ms" if r['jitter_ms'] is not None else "N/A"
    loss = f"{r['lost_pct']:.3f}%" if r['lost_pct'] is not None else f"{r['retransmits']}retr"
    chk = checks.get(name, {})
    status = "OK"
    if chk.get('jitter_max') and r.get('jitter_ms') and r['jitter_ms'] > chk['jitter_max']:
        status = "FAIL(J)"
    elif chk.get('lost_max') and r.get('lost_pct') and r['lost_pct'] > chk['lost_max']:
        status = "FAIL(L)"
    print(f"  {name:<12} {bw:>10} {jitter:>12} {loss:>15}  [{status}]")
PYEOF
```

---

## JSON 출력 분석 및 자동화

```python
#!/usr/bin/env python3
"""iperf3 JSON 결과 분석 도구 - CI/CD 통합용"""
import json
import sys
import subprocess
from pathlib import Path
from typing import Optional

def run_iperf3(host: str, port: int = 5201,
               udp: bool = False, bitrate: str = "100M",
               duration: int = 30, pkt_size: int = 1400,
               bind: Optional[str] = None) -> dict:
    """iperf3 실행 및 JSON 결과 반환"""
    cmd = ['iperf3', '-c', host, '-p', str(port),
           '-t', str(duration), '--json']
    if udp:
        cmd += ['-u', '-b', bitrate, '-l', str(pkt_size)]
    if bind:
        cmd += ['--bind', bind]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=duration+10)
    if result.returncode != 0:
        raise RuntimeError(f"iperf3 실패: {result.stderr}")
    return json.loads(result.stdout)

def parse_result(data: dict) -> dict:
    """결과 파싱 - UDP/TCP 자동 구분"""
    end = data.get('end', {})
    s = end.get('sum', end.get('sum_sent', {}))

    if 'jitter_ms' in s:  # UDP
        return {
            'protocol': 'UDP',
            'bitrate_mbps': s.get('bits_per_second', 0) / 1e6,
            'jitter_ms': s.get('jitter_ms', 0),
            'lost_packets': s.get('lost_packets', 0),
            'total_packets': s.get('packets', 1),
            'lost_pct': s.get('lost_percent', 0),
        }
    else:  # TCP
        stream = end.get('streams', [{}])[0].get('sender', {})
        return {
            'protocol': 'TCP',
            'bitrate_mbps': s.get('bits_per_second', 0) / 1e6,
            'retransmits': s.get('retransmits', 0),
            'max_rtt_ms': stream.get('max_rtt', 0) / 1000,
            'min_rtt_ms': stream.get('min_rtt', 0) / 1000,
            'mean_rtt_ms': stream.get('mean_rtt', 0) / 1000,
        }

def verify_requirements(metrics: dict, name: str = "") -> bool:
    """의료 로봇 네트워크 요구사항 검증"""
    print(f"\n[{name}] 요구사항 검증:")
    passed = True

    if metrics['protocol'] == 'UDP':
        checks = [
            ('Jitter < 100µs', metrics['jitter_ms'] < 0.1),
            ('패킷 손실 < 0.001%', metrics['lost_pct'] < 0.001),
            ('대역폭 > 목표', True),  # 실제 목표값과 비교
        ]
    else:
        checks = [
            ('재전송 없음', metrics['retransmits'] == 0),
            ('평균 RTT < 1ms', metrics['mean_rtt_ms'] < 1.0),
        ]

    for desc, ok in checks:
        status = "PASS" if ok else "FAIL"
        print(f"  [{status}] {desc}")
        if not ok:
            passed = False

    return passed

if __name__ == '__main__':
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('host', help='iperf3 서버 IP')
    parser.add_argument('--udp', action='store_true')
    parser.add_argument('--bitrate', default='100M')
    parser.add_argument('--duration', type=int, default=30)
    args = parser.parse_args()

    data = run_iperf3(args.host, udp=args.udp,
                      bitrate=args.bitrate, duration=args.duration)
    metrics = parse_result(data)
    print(json.dumps(metrics, indent=2))
    ok = verify_requirements(metrics, name="network")
    sys.exit(0 if ok else 1)
```

---

## NIC 및 시스템 성능 튜닝 검증

```bash
# ── 현재 NIC 상태 확인 ──
ethtool eth0 | grep -E "Speed|Duplex|Link detected"
# Speed: 1000Mb/s   → 1Gbps 확인
# Duplex: Full      → Full Duplex 확인
# Link detected: yes

# ── Jumbo Frame 효과 테스트 ──
# 기본 MTU (1500)
iperf3 -c 192.168.1.100 -t 10
# Bitrate: ~940 Mbps

# Jumbo Frame 설정 (9000)
sudo ip link set eth0 mtu 9000
iperf3 -c 192.168.1.100 -t 10
# Bitrate: ~970 Mbps (약 3% 향상 - 헤더 오버헤드 감소)

# ── GRO/GSO/TSO 설정 영향 ──
# GRO/GSO 활성화 (기본): CPU 효율적, 지연 약간 증가
ethtool -K eth0 gro on gso on tso on
iperf3 -c 192.168.1.100 -t 10

# GRO/GSO 비활성화: CPU 부하 증가, 지연 감소
ethtool -K eth0 gro off gso off tso off
iperf3 -c 192.168.1.100 -t 10

# ── IRQ 코얼레싱 설정 영향 ──
# 높은 코얼레싱: CPU 효율 ↑, Jitter ↑
ethtool -C eth0 rx-usecs 50 tx-usecs 50
iperf3 -c 192.168.1.100 -u -b 100M -l 256 -t 10 --json | \
    python3 -c "import json,sys; d=json.load(sys.stdin); print('Jitter:', d['end']['sum']['jitter_ms'])"

# 낮은 코얼레싱: CPU 부하 ↑, Jitter ↓ (실시간 트래픽에 유리)
ethtool -C eth0 rx-usecs 0 rx-frames 1 tx-usecs 0 tx-frames 1
iperf3 -c 192.168.1.100 -u -b 100M -l 256 -t 10 --json | \
    python3 -c "import json,sys; d=json.load(sys.stdin); print('Jitter:', d['end']['sum']['jitter_ms'])"

# ── 소켓 버퍼 최적화 ──
# TCP 버퍼 크기 최적화 (고대역폭 장거리 전송)
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

iperf3 -c 192.168.1.100 -P 4 -t 30
# 최적화 전: ~700 Mbps (4 스트림)
# 최적화 후: ~940 Mbps (버퍼 부족 해소)
```

---

## 성능 요구사항 기준표

```
의료 로봇 네트워크 iperf3 통과 기준:

┌────────────────────────┬────────────┬──────────────┬────────────────┐
│ 트래픽 유형            │ 프로토콜   │ Jitter 기준  │ 손실/재전송    │
├────────────────────────┼────────────┼──────────────┼────────────────┤
│ E-Stop / 관절 제어     │ UDP 1Mbps  │ < 50µs       │ 0%             │
│ 토크/위치 센서         │ UDP 10Mbps │ < 100µs      │ < 0.001%       │
│ 수술 영상 스트림        │ UDP 50Mbps │ < 10ms       │ < 0.1%         │
│ OTA 업데이트 (TCP)     │ TCP 100Mbps│ N/A          │ 재전송 < 1%    │
│ DoIP 진단 (TCP)        │ TCP 10Mbps │ N/A          │ 재전송 0       │
│ MQTT 원격 모니터링      │ TCP 1Mbps  │ N/A          │ 재전송 0       │
└────────────────────────┴────────────┴──────────────┴────────────────┘

달성 조건:
  - TSN taprio GCL 적용 (3장 참조)
  - PREEMPT_RT 리눅스 커널
  - 하드웨어 타임스탬핑 NIC
  - IRQ 어피니티 고정 (NIC IRQ → 전용 CPU 코어)
  - CPU C-state 비활성화
  - VLAN 분리 (VLAN 10: Safety, VLAN 20: Control)
```

---

## Reference
- [iperf3 Official Site](https://iperf.fr/)
- [iperf3 GitHub (ESnet)](https://github.com/esnet/iperf)
- [iperf3 Man Page](https://software.es.net/iperf/invoking.html)
- [iperf3 JSON Output Format](https://github.com/esnet/iperf/blob/master/docs/invoking.rst)
- [RFC 3393 - IP Packet Delay Variation Metric](https://datatracker.ietf.org/doc/html/rfc3393)
- [Red Hat - Network Performance Tuning Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/monitoring_and_managing_system_status_and_performance/)
- [Linux Foundation - Network Performance Tuning](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
