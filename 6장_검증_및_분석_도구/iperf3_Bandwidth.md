# iperf3 - 대역폭 및 성능 측정

## 개요
iperf3는 네트워크 대역폭(Bandwidth), 지연(Latency), 패킷 손실(Packet Loss)을 정밀하게 측정하는 오픈소스 도구입니다. 의료 로봇 Ethernet 네트워크의 실제 처리량을 검증하고, TSN 설정 후 예약된 대역폭이 올바르게 동작하는지 확인하는 데 사용합니다.

---

## 설치 및 기본 사용법

```bash
# 설치
sudo apt-get install iperf3

# 기본 구조: 서버(Server)와 클라이언트(Client) 쌍
# 서버 시작
iperf3 -s
# iperf3: I will be a server, listening on 5201
# Server listening on 5201

# TCP 클라이언트 (기본: 10초, 1 스트림)
iperf3 -c 192.168.1.100

# UDP 클라이언트 (100Mbps 지정)
iperf3 -c 192.168.1.100 -u -b 100M

# 양방향 (Bidirectional)
iperf3 -c 192.168.1.100 --bidir
```

---

## TCP 대역폭 측정

```bash
# 기본 TCP 테스트 (1Gbps 링크)
iperf3 -c 192.168.1.100 -t 30 -i 1

# 출력 예시:
# [ ID] Interval       Transfer     Bitrate         Retr  Cwnd
# [  5]   0.00-1.00   sec   112 MBytes   942 Mbits/sec    0   3.12 MBytes
# [  5]   1.00-2.00   sec   112 MBytes   941 Mbits/sec    0   3.12 MBytes
# - - - - - - - - - - - - - - - - - - - - - - - - -
# [ ID] Interval       Transfer     Bitrate         Retr
# [  5]   0.00-30.00  sec  3.27 GBytes   937 Mbits/sec    0     sender
# [  5]   0.00-30.00  sec  3.27 GBytes   936 Mbits/sec         receiver
#
# 분석:
#   Retr (Retransmit) = 0 → 패킷 재전송 없음 (우수)
#   937 Mbits/sec ≈ 1Gbps 링크의 93.7% 활용률

# 병렬 스트림 (TCP 처리량 최대화)
iperf3 -c 192.168.1.100 -P 4 -t 30

# TCP 창 크기 조정
iperf3 -c 192.168.1.100 -w 256K -t 30

# Reverse 모드 (서버→클라이언트 방향)
iperf3 -c 192.168.1.100 -R
```

---

## UDP 대역폭 및 패킷 손실 측정

```bash
# UDP 100Mbps 테스트 (의료 로봇 영상 스트림 시뮬)
iperf3 -c 192.168.1.100 -u -b 100M -t 30 -i 1

# 출력 예시:
# [ ID] Interval       Transfer     Bitrate         Jitter    Lost/Total
# [  5]   0.00-1.00   sec  11.9 MBytes  99.9 Mbits/sec  0.021 ms  0/8622 (0%)
# [  5]   1.00-2.00   sec  11.9 MBytes  99.9 Mbits/sec  0.018 ms  0/8622 (0%)
# - - - - - - - - - - - - -
# [  5]   0.00-30.00  sec   357 MBytes  99.9 Mbits/sec  0.019 ms  0/258660 (0%)
#
# 분석:
#   Jitter: 0.019 ms (19µs) → 매우 낮음 (TSN 효과)
#   Lost/Total: 0/258660 (0%) → 패킷 손실 없음

# 패킷 크기 설정 (MTU 테스트)
iperf3 -c 192.168.1.100 -u -b 500M -l 1400  # 1400 Byte 페이로드
iperf3 -c 192.168.1.100 -u -b 500M -l 64    # 64 Byte (소형 제어 패킷)

# 1Gbps 최대 UDP 부하 (DoS 테스트 - 내부 망만)
iperf3 -c 192.168.1.100 -u -b 1G -t 10
```

---

## Jitter 측정 (TSN 검증)

```bash
# TSN 전후 Jitter 비교 측정

# Before TSN: 표준 Ethernet
# 10MHz 스트림 (100Kpps @ 100Byte)
iperf3 -c 192.168.1.100 -u -b 80M -l 100 -t 60 --json \
  > before_tsn_jitter.json

# After TSN: taprio + CBS 설정 후
iperf3 -c 192.168.1.100 -u -b 80M -l 100 -t 60 --json \
  > after_tsn_jitter.json

# Python으로 비교 분석
python3 << 'EOF'
import json

def parse_jitter(filename):
    with open(filename) as f:
        data = json.load(f)
    intervals = data['intervals']
    jitters = [i['sum']['jitter_ms'] for i in intervals if 'jitter_ms' in i['sum']]
    return jitters

before = parse_jitter('before_tsn_jitter.json')
after = parse_jitter('after_tsn_jitter.json')

import statistics
print("=== Jitter 비교 ===")
print(f"Before TSN - 평균: {statistics.mean(before):.3f}ms, 최대: {max(before):.3f}ms")
print(f"After TSN  - 평균: {statistics.mean(after):.3f}ms, 최대: {max(after):.3f}ms")
improvement = (1 - statistics.mean(after)/statistics.mean(before)) * 100
print(f"Jitter 개선율: {improvement:.1f}%")
EOF

# 전형적인 결과:
# Before TSN - 평균: 2.341ms, 최대: 15.234ms
# After TSN  - 평균: 0.021ms, 최대:  0.089ms
# Jitter 개선율: 99.1%
```

---

## VLAN별 대역폭 검증

```bash
# VLAN 인터페이스 생성 (테스트 환경)
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 192.168.10.1/24 dev eth0.10
sudo ip addr add 192.168.20.1/24 dev eth0.20
sudo ip link set eth0.10 up
sudo ip link set eth0.20 up

# VLAN 10 (Safety Network) 대역폭 측정
iperf3 -c 192.168.10.100 -u -b 10M -t 30 \
  --bind 192.168.10.1 \
  --json > vlan10_result.json

# VLAN 20 (Control Network) 대역폭 측정
iperf3 -c 192.168.20.100 -u -b 100M -t 30 \
  --bind 192.168.20.1 \
  --json > vlan20_result.json

# 동시 부하 테스트 (VLAN 격리 확인)
# VLAN 20에 높은 부하 → VLAN 10 영향 없음 확인
iperf3 -c 192.168.20.100 -u -b 900M -t 30 &  # 배경 부하
iperf3 -c 192.168.10.100 -u -b 10M -t 30 \
  --json > vlan10_under_load.json             # Safety 측정
wait

# VLAN 격리 확인
python3 -c "
import json
with open('vlan10_result.json') as f:
    normal = json.load(f)['end']['sum']['jitter_ms']
with open('vlan10_under_load.json') as f:
    under_load = json.load(f)['end']['sum']['jitter_ms']
print(f'VLAN10 Jitter 정상: {normal:.3f}ms')
print(f'VLAN10 Jitter 부하: {under_load:.3f}ms')
print(f'TSN VLAN 격리: {\"OK\" if under_load < normal * 1.5 else \"FAIL\"}')"
```

---

## 멀티스트림 혼합 부하 (의료 로봇 시뮬)

```bash
#!/bin/bash
# 의료 로봇 실제 트래픽 패턴 시뮬레이션

ROBOT_IP="192.168.1.100"

# 1. Safety Stream: 1kHz 64Byte (E-Stop, 관절 제어)
#    대역폭: 1000 × 64 × 8 = 512 Kbps
iperf3 -c $ROBOT_IP -u -b 512K -l 64 -t 60 \
  --bind 192.168.10.1 &

# 2. Sensor Stream: 1kHz 1500Byte (토크, 위치 센서)
#    대역폭: 1000 × 1500 × 8 = 12 Mbps
iperf3 -c $ROBOT_IP -u -b 12M -l 1500 -t 60 \
  --bind 192.168.20.1 &

# 3. Video Stream: 30fps H.264 (수술 카메라)
#    대역폭: 50 Mbps (압축)
iperf3 -c $ROBOT_IP -u -b 50M -l 1400 -t 60 &

# 4. Background: 진단/로그 (Best Effort)
iperf3 -c $ROBOT_IP -b 100M -t 60 &

wait
echo "멀티스트림 테스트 완료"
```

---

## JSON 출력 분석 스크립트

```python
#!/usr/bin/env python3
"""iperf3 JSON 결과 분석 및 보고서 생성"""
import json
import sys
from pathlib import Path

def analyze_iperf3_json(json_file: str) -> dict:
    """iperf3 JSON 파일 파싱 및 핵심 지표 추출"""
    with open(json_file) as f:
        data = json.load(f)

    end = data.get('end', {})
    streams = end.get('streams', [{}])
    sender = end.get('sum_sent', end.get('sum', {}))
    receiver = end.get('sum_received', sender)

    # UDP 결과
    if 'jitter_ms' in sender:
        return {
            'protocol': 'UDP',
            'duration_s': end.get('sum', {}).get('seconds', 0),
            'bitrate_mbps': sender.get('bits_per_second', 0) / 1e6,
            'jitter_ms': sender.get('jitter_ms', 0),
            'lost_packets': sender.get('lost_packets', 0),
            'total_packets': sender.get('packets', 0),
            'loss_rate_pct': sender.get('lost_percent', 0),
        }
    # TCP 결과
    else:
        return {
            'protocol': 'TCP',
            'duration_s': sender.get('seconds', 0),
            'bitrate_mbps': sender.get('bits_per_second', 0) / 1e6,
            'retransmits': sender.get('retransmits', 0),
            'max_rtt_ms': end.get('streams', [{}])[0].get('sender', {}).get('max_rtt', 0) / 1000,
            'min_rtt_ms': end.get('streams', [{}])[0].get('sender', {}).get('min_rtt', 0) / 1000,
            'mean_rtt_ms': end.get('streams', [{}])[0].get('sender', {}).get('mean_rtt', 0) / 1000,
        }

def print_report(metrics: dict, name: str = ""):
    print(f"\n{'='*50}")
    print(f"  iperf3 결과 보고서{' - ' + name if name else ''}")
    print(f"{'='*50}")
    for k, v in metrics.items():
        if isinstance(v, float):
            print(f"  {k:<25}: {v:.3f}")
        else:
            print(f"  {k:<25}: {v}")

    # 의료 로봇 요구사항 평가
    print(f"\n  [요구사항 평가]")
    if metrics['protocol'] == 'UDP':
        jitter_ok = metrics['jitter_ms'] < 0.1  # < 100µs
        loss_ok = metrics['loss_rate_pct'] < 0.001  # < 0.001%
        print(f"  Jitter < 100µs:       {'PASS' if jitter_ok else 'FAIL'}")
        print(f"  패킷 손실 < 0.001%:   {'PASS' if loss_ok else 'FAIL'}")
    elif metrics['protocol'] == 'TCP':
        retx_ok = metrics['retransmits'] == 0
        rtt_ok = metrics['mean_rtt_ms'] < 1.0
        print(f"  재전송 없음:          {'PASS' if retx_ok else 'FAIL'}")
        print(f"  평균 RTT < 1ms:       {'PASS' if rtt_ok else 'FAIL'}")

if __name__ == '__main__':
    for json_file in sys.argv[1:]:
        metrics = analyze_iperf3_json(json_file)
        print_report(metrics, name=Path(json_file).stem)
```

---

## 네트워크 성능 튜닝 확인

```bash
# 현재 NIC 설정 확인
ethtool eth0 | grep -E "Speed|Duplex|Link"
# Speed: 1000Mb/s
# Duplex: Full
# Link detected: yes

# iperf3로 MTU Jumbo Frame 효과 테스트
# 표준 MTU (1500)
iperf3 -c 192.168.1.100 -t 10 -M 1500

# Jumbo Frame (9000)
sudo ip link set eth0 mtu 9000
iperf3 -c 192.168.1.100 -t 10 -M 9000

# GRO/GSO 설정 영향 테스트
ethtool -K eth0 gro off gso off
iperf3 -c 192.168.1.100 -t 10
# → GRO/GSO 비활성화 시 CPU 사용률 증가, 처리량 감소 확인

# 결과 비교
# MTU 1500: ~940 Mbps
# MTU 9000: ~970 Mbps (Jumbo Frame 효과)
```

---

## Reference
- [iperf3 Official](https://iperf.fr/)
- [iperf3 GitHub](https://github.com/esnet/iperf)
- [iperf3 Man Page](https://software.es.net/iperf/invoking.html)
- [Linux Network Performance](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
