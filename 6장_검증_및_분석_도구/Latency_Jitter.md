# Latency와 Jitter 분석

## 개요

의료 로봇 Ethernet 통신에서 Latency(지연 시간)와 Jitter(지연 변동)는 시스템 안전성과 성능을 결정하는 핵심 지표입니다. 수술 로봇의 관절 제어 루프는 1kHz(1ms 주기)로 동작하며, 이 주기에서의 Jitter가 50µs를 초과하면 제어 안정성이 저하되고, 극단적인 경우 안전 인터록이 발동됩니다.

Latency와 Jitter는 단순한 성능 지표를 넘어 **안전 기능 요구사항(SRS)**에 직접 반영됩니다. E-Stop 신호는 100µs 이내에 전달되어야 하며, 관절 제어 명령은 1ms 주기를 보장해야 합니다. 이를 달성하기 위해 PREEMPT_RT 커널, TSN 스케줄링(IEEE 802.1Qbv), 하드웨어 타임스탬핑, 시스템 튜닝의 조합이 필요합니다.

> **의료 로봇 관점**: 수술 로봇에서 Jitter는 의사의 손 움직임이 로봇 관절에 얼마나 정확하게 전달되는지를 결정합니다. 10µs Jitter는 0.001mm 수준의 위치 오차를 유발하여 허용 가능하지만, 500µs Jitter는 0.5mm 오차로 이어져 정밀 수술에서 안전 문제를 야기할 수 있습니다. IEC 62304 Class C 소프트웨어에서 Latency/Jitter 요구사항은 SRS에 정량적으로 명시되어야 하며, HIL 테스트로 검증해야 합니다. 본 문서는 `HIL.md`, `tc_taprio.md`(TSN 스케줄링)와 함께 읽으면 측정-튜닝-검증의 전체 흐름을 이해할 수 있습니다.

---

## Latency 구성 요소 (분해 분석)

End-to-End Latency는 단일 값이 아닌 여러 구성 요소의 합입니다. 각 요소를 이해하면 병목 지점을 정확히 파악하고 최적화 방향을 결정할 수 있습니다.

```
End-to-End Latency 분해:

송신 애플리케이션
    │
    │ ① Application Processing Delay (애플리케이션 처리 지연)
    │     · 제어 루프 계산 (역기구학, PID 등)
    │     · 일반 Linux 스케줄링: 100µs ~ 수ms (불규칙)
    │     · PREEMPT_RT: < 50µs (실시간 보장)
    │     · FPGA/bare-metal: < 1µs
    │
    ▼
NIC TX Queue
    │
    │ ② Queueing Delay (큐잉 지연)
    │     · 일반 OS: 여러 스레드가 큐 경쟁 → 수백µs
    │     · SO_TXTIME + tc ETF: 정확한 전송 시각 예약
    │     · TSN TAS (Qbv): 게이트 열림 시각까지 대기
    │       → 최악: 슬롯 주기 - 1 (예: GCL 1ms 주기 → 최대 1ms 대기)
    │
    ▼
NIC Hardware
    │
    │ ③ Serialization (Transmission) Delay (직렬화 지연)
    │     · 패킷 크기 / 링크 속도
    │     · 64 Byte   @ 1Gbps  = 0.512 µs
    │     · 256 Byte  @ 1Gbps  = 2.048 µs
    │     · 1518 Byte @ 1Gbps  = 12.14 µs
    │     · 64 Byte   @ 100Mbps = 5.12  µs (구형 Ethernet)
    │
    ▼
Ethernet Cable
    │
    │ ④ Propagation Delay (전파 지연)
    │     · 빛의 속도 × 2/3 (구리 케이블 굴절률 ≈ 0.64~0.68c)
    │     · ≈ 5 ns/m
    │     · 10m = 50 ns, 100m = 500 ns (무시 가능)
    │
    ▼
Switch Ingress
    │
    │ ⑤ Switch Processing Delay (스위치 처리 지연)
    │     Store-and-Forward: 전체 프레임 수신 후 전달
    │       → 최소 12.14 µs (1518B @ 1Gbps) + 내부 처리
    │     Cut-Through: 헤더만 보고 즉시 전달
    │       → < 1 µs (CRC 검증 없음, 손상 패킷 전달 위험)
    │     Express Frame (IEEE 802.1Qbu preemption):
    │       → 진행 중 프레임 중단 후 고우선순위 전달
    │       → 추가 지연: 최소 프레임 크기(64B) 전송 시간 = 512 ns
    │
    ▼
Switch Queueing Delay
    │
    │ ⑥ Queueing Delay at Switch (스위치 큐 지연)
    │     Best Effort: 0 ~ 수ms (혼잡 시 큰 패킷 큐 대기)
    │     TSN TAS: GCL 슬롯에 의해 제어 → 결정론적
    │     CBS (Credit-Based Shaper): 대역폭 보장, 일부 Jitter
    │
    ▼
수신 NIC
    │
    │ ⑦ Receive Processing Delay (수신 처리 지연)
    │     · 인터럽트 발생 → CPU 컨텍스트 전환
    │     · DMA 전송 → 소켓 버퍼
    │     · 인터럽트 병합(coalescing): 지연 증가 but CPU 부하 감소
    │
    ▼
수신 애플리케이션

총 지연 비교 (1518B 패킷, 1홉):
┌───────────────────┬─────────────┬─────────────┬──────────────┐
│ 구성 요소         │ 일반 Linux  │ PREEMPT_RT  │ TSN 최적화   │
├───────────────────┼─────────────┼─────────────┼──────────────┤
│ ① App 처리        │ 100µs~1ms   │ < 50µs      │ < 50µs       │
│ ② TX 큐잉         │ 수십µs      │ 수십µs      │ ETF < 1µs    │
│ ③ 직렬화          │ 12.14µs     │ 12.14µs     │ 12.14µs      │
│ ④ 전파            │ 50ns        │ 50ns        │ 50ns         │
│ ⑤ 스위치          │ 12µs        │ 12µs        │ < 1µs (CT)   │
│ ⑥ SW 큐잉         │ 0~수ms      │ 0~수ms      │ < 10µs (TAS) │
│ ⑦ RX 처리         │ 수십µs      │ < 10µs      │ < 10µs       │
├───────────────────┼─────────────┼─────────────┼──────────────┤
│ 합계 (전형적)     │ > 1ms       │ < 200µs     │ < 100µs      │
└───────────────────┴─────────────┴─────────────┴──────────────┘
```

---

## Jitter 원인 분석

Jitter는 패킷 간격의 통계적 변동을 의미합니다. 주기적인 제어 신호에서 Jitter는 제어 성능과 직결됩니다.

```
Jitter 발생 원인별 크기:

1. OS 스케줄링 불규칙성 (가장 큰 원인)
   일반 Linux (CFS):   100µs ~ 수ms  ← 제어에 부적합
   PREEMPT_RT 패치:   < 50µs         ← 경량 제어 가능
   실시간 OS (FreeRTOS, VxWorks): < 10µs
   FPGA/bare-metal: < 1µs            ← 최고 정밀도

   원인: 커널 비선점 구간(스핀락, IRQ 비활성화)이
         실시간 스레드를 지연시킴
         → PREEMPT_RT: 대부분의 락을 Mutex로 변환, 선점 허용

2. 네트워크 큐잉 Jitter
   혼잡 없음:    Jitter ≈ 0
   혼잡 있음:    Jitter = 큰 패킷 직렬화 시간
                 예: 10개의 1518B 패킷 큐 대기 = 121µs

   TSN 해결책: PSFP(Per-Stream Filtering) + TAS(Time-Aware Shaper)
               → 우선순위 트래픽은 큐 독립적으로 전송

3. IRQ 처리 경합 (Interrupt Latency)
   IRQ 병합(Interrupt Coalescing): 여러 패킷을 묶어 한 번에 인터럽트
     → 처리 효율 up, Jitter up (패킷마다 다른 지연)
   최적화: rx-usecs=0, rx-frames=1 (인터럽트 병합 비활성)

4. CPU 주파수/절전 상태 (C-state/P-state)
   절전 모드 C2/C3 복귀: 수십 ~ 수백µs
   해결: cpupower idle-set -D 1 (C1까지만 허용)
         scaling_governor=performance (고정 주파수)

5. NUMA 메모리 접근 (다소켓 서버)
   원격 NUMA 접근: 로컬보다 40~100ns 더 느림
   해결: numactl --cpunodebind=0 --membind=0 ./robot_controller

6. Cache Effects (캐시 미스)
   L3 캐시 미스: 수십ns 추가 지연
   해결: CPU 어피니티로 동일 코어에 고정 (캐시 warm-up 유지)
```

---

## 측정 도구

### cyclictest (PREEMPT_RT 스케줄링 지연 측정)

cyclictest는 Linux 실시간 커널의 스케줄링 지연을 µs 단위로 측정하는 표준 도구입니다. rt-tests 패키지에 포함되어 있습니다.

```bash
# 설치
sudo apt-get install rt-tests

# 기본 실행 (4개 스레드, 우선순위 99, 1ms 주기, 100,000회)
sudo cyclictest \
  --mlockall \
  --smp \
  --priority=99 \
  --interval=1000 \
  --loops=100000

# 출력 해석:
# T: 0 (PID=1234) P:99 I:1000 C:100000 Min:12 Act:18 Avg:21 Max:156
#                                         ↑      ↑     ↑     ↑
#                                        최소  현재  평균  최대 (µs)
# → Max < 50µs: PREEMPT_RT 목표 달성
# → Max > 500µs: 절전/인터럽트 문제 → 튜닝 필요

# 히스토그램 생성 (분포 분석, 정규성 확인)
sudo cyclictest \
  --mlockall \
  --priority=99 \
  --interval=1000 \
  --loops=1000000 \
  --histogram=500 \
  --histofile=/tmp/latency_hist.txt

# 히스토그램 gnuplot 시각화
gnuplot << 'PLOT_EOF'
set terminal png size 1400,700
set output '/tmp/latency_hist.png'
set title "Scheduling Latency Distribution (1,000,000 samples)"
set xlabel "Latency (µs)"
set ylabel "Occurrences (log scale)"
set logscale y
set grid
set style line 1 lc rgb '#0066cc' lw 2
plot '/tmp/latency_hist.txt' using 1:2 with linespoints ls 1 \
     title "RT Thread Latency"
PLOT_EOF

echo "히스토그램 → /tmp/latency_hist.png"

# 부하 시 측정 (실제 동작 환경 시뮬레이션)
# 1. 백그라운드 부하 생성
stress-ng --cpu 4 --io 2 --vm 1 --vm-bytes 1G &

# 2. cyclictest 실행 (부하 상태에서 측정)
sudo cyclictest --mlockall --priority=99 --interval=1000 --loops=100000

# 3. 부하 제거
kill %1
```

### Hardware Timestamping 기반 정밀 측정

소프트웨어 타임스탬핑은 OS 스케줄링 영향을 받아 µs 이상의 오차가 발생합니다. 하드웨어 타임스탬핑은 NIC에서 직접 패킷 도착/전송 시각을 기록하여 ns 수준의 정밀도를 제공합니다.

```bash
# NIC 하드웨어 타임스탬핑 지원 확인
ethtool -T eth0
# 출력에서 확인:
# Capabilities:
#   hardware-transmit     (TX 하드웨어 타임스탬프)
#   software-transmit     (TX 소프트웨어 타임스탬프)
#   hardware-receive      (RX 하드웨어 타임스탬프)
#   hardware-raw-clock    (원시 하드웨어 클록)

# SO_TIMESTAMPING 소켓 옵션으로 타임스탬프 수집
# (C 코드 예시)
# int flags = SOF_TIMESTAMPING_TX_HARDWARE |
#             SOF_TIMESTAMPING_RX_HARDWARE |
#             SOF_TIMESTAMPING_RAW_HARDWARE;
# setsockopt(sock, SOL_SOCKET, SO_TIMESTAMPING, &flags, sizeof(flags));

# ptp4l로 하드웨어 타임스탬프 기반 PTP 오프셋 모니터링
sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0 2>&1 | \
  grep "rms\|offset" | tee /tmp/ptp_hw_ts.log &

# 실시간 PTP 오프셋 및 Jitter 분석
watch -n 1 'tail -20 /tmp/ptp_hw_ts.log | grep rms | \
  awk "{print \"오프셋 RMS:\", \$2, \"ns  최대:\", \$4, \"ns\"}"'

# ptp4l 로그 통계 분석
python3 << 'PYEOF'
import re, statistics

with open('/tmp/ptp_hw_ts.log') as f:
    offsets = [int(m.group(1)) for line in f
               if (m := re.search(r'rms\s+(\d+)', line))]

if offsets:
    print(f"샘플 수: {len(offsets)}")
    print(f"평균 RMS: {statistics.mean(offsets):.1f} ns")
    print(f"최대 RMS: {max(offsets)} ns")
    print(f"표준편차: {statistics.stdev(offsets):.1f} ns")
    print(f"P99: {sorted(offsets)[int(len(offsets)*0.99)]} ns")
PYEOF
```

### 네트워크 왕복 지연 측정 (RTT)

```bash
# 1. ping - 간단한 ICMP RTT (소프트웨어 타임스탬프)
ping -i 0.001 -c 10000 192.168.10.2 | tee /tmp/ping_result.txt

# 결과 분석
tail -1 /tmp/ping_result.txt
# rtt min/avg/max/mdev = 0.254/0.312/1.234/0.045 ms
#        ↑min  ↑avg  ↑max   ↑표준편차(Jitter)

# 2. hping3 - UDP/TCP RTT 측정
sudo hping3 --udp -p 5001 -i u100 -c 1000 192.168.10.2
# u100 = 100µs 간격 (10kHz)

# 3. sockperf - 고정밀 소켓 지연 측정 (µs)
# 서버
sockperf server --ip 0.0.0.0 --port 11111

# 클라이언트 (1kHz, 10000 패킷, 64B)
sockperf ping-pong \
  --ip 192.168.10.2 --port 11111 \
  --udp --msg-size 64 \
  --time 10 --rate-limit 1000

# 결과: percentile 99 latency in usec
# 50.000%  = 245.875 usec
# 99.000%  = 312.500 usec
# 99.900%  = 456.250 usec
# 99.990%  = 1234.000 usec
```

---

## 지연 분석 Python 도구

```python
#!/usr/bin/env python3
"""
Latency/Jitter 종합 분석 도구
PCAP 파일 또는 실시간 소켓에서 지연/지터 측정 및 분석
IEC 62304 / ISO 26262 성능 검증 보고서 생성
"""
import csv
import json
import statistics
import subprocess
import sys
import socket
import time
from datetime import datetime
from typing import Optional


def analyze_pcap_latency(pcap_file: str,
                          protocol_filter: str = "someip",
                          target_ms: float = 1.0) -> dict:
    """
    Wireshark/tshark로 PCAP 분석 → 지연 통계 계산

    패킷 간격(Jitter)과 절대 타임스탬프를 분석하여
    TSN 효과 전후 비교, 이상값 탐지 등에 활용
    """
    result = subprocess.run([
        'tshark', '-r', pcap_file,
        '-Y', protocol_filter,
        '-T', 'fields',
        '-e', 'frame.time_epoch',
        '-e', 'frame.len',
        '-e', 'ip.src',
        '-e', 'frame.time_delta',
        '-E', 'separator=\t'
    ], capture_output=True, text=True)

    if result.returncode != 0:
        print(f"오류: {result.stderr}")
        return {}

    timestamps, frame_sizes, inter_pkt_ms = [], [], []

    for line in result.stdout.strip().split('\n'):
        if not line:
            continue
        parts = line.split('\t')
        if len(parts) >= 4:
            try:
                ts = float(parts[0])
                size = int(parts[1])
                delta_ms = float(parts[3]) * 1000  # 초 → ms
                timestamps.append(ts)
                frame_sizes.append(size)
                if delta_ms > 0:
                    inter_pkt_ms.append(delta_ms)
            except (ValueError, IndexError):
                continue

    if len(timestamps) < 10:
        print(f"데이터 부족 ({len(timestamps)}개 패킷)")
        return {}

    sorted_ipd = sorted(inter_pkt_ms)
    n = len(sorted_ipd)

    stats = {
        'filter': protocol_filter,
        'pcap_file': pcap_file,
        'count': len(timestamps),
        'duration_s': round(timestamps[-1] - timestamps[0], 3),
        'pkt_rate_hz': round(len(timestamps) / (timestamps[-1] - timestamps[0]), 1),
        'avg_frame_bytes': round(statistics.mean(frame_sizes), 1),
        # 패킷 간격 통계 (Jitter)
        'avg_ipd_ms': round(statistics.mean(inter_pkt_ms), 4),
        'min_ipd_ms': round(min(inter_pkt_ms), 4),
        'max_ipd_ms': round(max(inter_pkt_ms), 4),
        'stdev_ms': round(statistics.stdev(inter_pkt_ms), 4),
        'p50_ms': round(sorted_ipd[int(n * 0.50)], 4),
        'p95_ms': round(sorted_ipd[int(n * 0.95)], 4),
        'p99_ms': round(sorted_ipd[int(n * 0.99)], 4),
        'p999_ms': round(sorted_ipd[min(int(n * 0.999), n-1)], 4),
        # 이상값 (목표의 2배 초과)
        'outlier_count': sum(1 for v in inter_pkt_ms if v > target_ms * 2),
        'outlier_pct': round(
            sum(1 for v in inter_pkt_ms if v > target_ms * 2) / n * 100, 3
        ),
        'target_ms': target_ms,
        'pass': max(inter_pkt_ms) < target_ms * 3,  # 3배 이내 → 허용
    }

    return stats


def print_report(stats: dict):
    """성능 분석 보고서 출력 (IEC 62304 증거용)"""
    ts = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    passed = "PASS ✓" if stats.get('pass') else "FAIL ✗"

    print(f"\n{'='*60}")
    print(f"  Latency/Jitter 분석 보고서  [{ts}]")
    print(f"{'='*60}")
    print(f"  파일:          {stats['pcap_file']}")
    print(f"  필터:          {stats['filter']}")
    print(f"  패킷 수:       {stats['count']:,}")
    print(f"  측정 기간:     {stats['duration_s']} s")
    print(f"  패킷 레이트:   {stats['pkt_rate_hz']} Hz")
    print(f"  평균 프레임:   {stats['avg_frame_bytes']} Byte")
    print(f"{'-'*60}")
    print(f"  평균 간격:     {stats['avg_ipd_ms']:.4f} ms")
    print(f"  최소 간격:     {stats['min_ipd_ms']:.4f} ms")
    print(f"  최대 간격:     {stats['max_ipd_ms']:.4f} ms")
    print(f"  표준편차:      {stats['stdev_ms']:.4f} ms  ← Jitter")
    print(f"  P50 (중앙값):  {stats['p50_ms']:.4f} ms")
    print(f"  P95:           {stats['p95_ms']:.4f} ms")
    print(f"  P99:           {stats['p99_ms']:.4f} ms")
    print(f"  P99.9:         {stats['p999_ms']:.4f} ms")
    print(f"{'-'*60}")
    print(f"  목표 주기:     {stats['target_ms']} ms")
    print(f"  이상값(>2x):   {stats['outlier_count']}개 ({stats['outlier_pct']}%)")
    print(f"  종합 판정:     {passed}")
    print(f"{'='*60}\n")


def compare_tsn_effect(before_pcap: str, after_pcap: str,
                        filter_str: str = "someip"):
    """
    TSN 도입 전후 지연 비교 분석
    - before: 일반 Ethernet
    - after: TSN TAS 활성화
    """
    print("\n[TSN 효과 비교 분석]")
    before = analyze_pcap_latency(before_pcap, filter_str)
    after = analyze_pcap_latency(after_pcap, filter_str)

    if not before or not after:
        print("PCAP 분석 실패")
        return

    metrics = ['avg_ipd_ms', 'max_ipd_ms', 'stdev_ms', 'p99_ms', 'p999_ms']
    labels  = ['평균 간격', '최대 간격', '표준편차(Jitter)', 'P99', 'P99.9']

    print(f"\n{'지표':<20} {'TSN 이전':>12} {'TSN 이후':>12} {'개선율':>10}")
    print("-" * 56)
    for m, lbl in zip(metrics, labels):
        b, a = before.get(m, 0), after.get(m, 0)
        improve = (b - a) / b * 100 if b > 0 else 0
        print(f"{lbl:<20} {b:>10.4f}ms {a:>10.4f}ms {improve:>8.1f}%")

    print(f"\n결론: TSN 적용으로 Jitter(표준편차) "
          f"{before.get('stdev_ms', 0):.4f}ms → "
          f"{after.get('stdev_ms', 0):.4f}ms 감소")


def export_json_report(stats: dict, path: str = "/tmp/latency_report.json"):
    """CI/CD 파이프라인용 JSON 리포트"""
    stats['generated_at'] = datetime.now().isoformat()
    with open(path, 'w') as f:
        json.dump(stats, f, indent=2, ensure_ascii=False)
    print(f"[JSON 리포트] {path} 저장")


if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("사용법: latency_analyze.py <pcap_file> [filter] [target_ms]")
        print("예시:   latency_analyze.py capture.pcap someip 1.0")
        sys.exit(1)

    pcap = sys.argv[1]
    filt = sys.argv[2] if len(sys.argv) > 2 else "someip"
    tgt  = float(sys.argv[3]) if len(sys.argv) > 3 else 1.0

    stats = analyze_pcap_latency(pcap, filt, tgt)
    if stats:
        print_report(stats)
        export_json_report(stats)
```

---

## 시스템 튜닝 (지연 최소화)

### PREEMPT_RT 커널 설치

```bash
# Ubuntu 22.04 LTS 기준 PREEMPT_RT 설치

# 방법 1: 패키지 관리자 (Ubuntu Pro 또는 메인라인)
sudo apt-get install linux-realtime    # Ubuntu 22.04 RT 패키지

# 방법 2: 소스 빌드 (커스텀 설정)
# 커널 소스 + RT 패치 다운로드
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.83.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/projects/rt/5.15/\
     patch-5.15.83-rt55.patch.xz

# 적용
tar xf linux-5.15.83.tar.xz && cd linux-5.15.83
xzcat ../patch-5.15.83-rt55.patch.xz | patch -p1

# 설정 (Fully Preemptible Kernel 선택)
make menuconfig
# Kernel Features → Preemption Model → Fully Preemptible Kernel (RT)

# RT 커널 확인
uname -r
# 5.15.83-rt55  ← PREEMPT_RT 확인

# 또는
cat /sys/kernel/realtime  # 1이면 RT 커널
```

### CPU 및 OS 튜닝

```bash
#!/bin/bash
# RT 시스템 튜닝 스크립트 (의료 로봇 ECU 최적화)
# 부팅 시 실행: /etc/rc.local 또는 systemd service

echo "=== RT 시스템 튜닝 적용 ==="

# 1. CPU 성능 모드 (절전 비활성)
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cpupower frequency-set -g performance

# 2. C-state 제한 (절전 단계 최소화)
cpupower idle-set -D 1            # C1까지만 허용 (C2/C3 비활성)
# 또는 커널 파라미터: intel_idle.max_cstate=1

# 3. IRQ 어피니티 (NIC IRQ를 특정 CPU에 격리)
# NIC 이름에 맞게 수정 필요
NIC=eth0
for IRQ in $(grep $NIC /proc/interrupts | awk -F: '{print $1}'); do
    echo "NIC IRQ $IRQ → CPU 3"
    echo 8 > /proc/irq/$IRQ/smp_affinity   # 0x8 = CPU 3
done

# 4. 인터럽트 병합 최소화 (지연 우선, CPU 효율 희생)
ethtool -C $NIC rx-usecs 0 rx-frames 1
ethtool -C $NIC tx-usecs 0 tx-frames 1

# 5. NIC 오프로드 최적화 (지연 vs 처리량 트레이드오프)
# TSN 정밀 타이밍: 일부 오프로드 비활성
ethtool -K $NIC tso off gso off    # 세그멘테이션 오프로드 off
ethtool -K $NIC gro off            # GRO off (수신 지연 감소)
# 단, LRO는 필요에 따라 유지

# 6. 소켓 버퍼 최적화
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.core.netdev_max_backlog=30000
sysctl -w net.core.netdev_budget=600       # RX 패킷 예산 증가

# 7. CPU 격리 (RT 프로세스 전용 코어)
# 커널 파라미터에 추가: isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3
# 확인:
cat /sys/devices/system/cpu/isolated  # 2-3 표시

# 격리된 CPU에 RT 프로세스 고정
taskset -c 2 chrt -f 99 ./robot_joint_controller &
taskset -c 3 chrt -f 98 ./robot_safety_monitor &

# 8. 메모리 잠금 (페이지 폴트 방지)
# 프로그램에서: mlockall(MCL_CURRENT | MCL_FUTURE)
# 또는 ulimit -l unlimited

echo "=== 튜닝 완료 ==="

# 확인
cyclictest --mlockall --priority=99 --interval=1000 --loops=10000 --quiet
```

### SO_TXTIME + ETF (Earliest TxTime First) 스케줄러

TSN 환경에서 정확한 전송 시각을 예약하여 Jitter를 최소화합니다.

```c
// SO_TXTIME 소켓 설정 (C 코드 개념)
// 패킷마다 정확한 전송 시각을 지정하여 결정론적 전송 달성

#include <linux/net_tstamp.h>
#include <linux/if_ether.h>

int setup_txtime_socket(int sock) {
    // ETF qdisc 설정 (커널 명령)
    // tc qdisc add dev eth0 parent 100:1 etf
    //    clockid CLOCK_TAI offload delta 500000

    struct sock_txtime txtime = {
        .clockid = CLOCK_TAI,  // TAI 시계 사용 (gPTP와 동기)
        .flags   = 0,
    };
    return setsockopt(sock, SOL_SOCKET, SO_TXTIME,
                      &txtime, sizeof(txtime));
}

// 패킷 전송 시각 예약
void send_with_txtime(int sock, void *data, size_t len,
                      uint64_t txtime_ns) {
    // cmsg로 TX 시각 전달
    char control[CMSG_SPACE(sizeof(uint64_t))];
    struct msghdr msg = {0};
    struct cmsghdr *cmsg;
    struct iovec iov = {.iov_base = data, .iov_len = len};

    msg.msg_iov     = &iov;
    msg.msg_iovlen  = 1;
    msg.msg_control = control;
    msg.msg_controllen = sizeof(control);

    cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type  = SCM_TXTIME;
    cmsg->cmsg_len   = CMSG_LEN(sizeof(uint64_t));
    *((uint64_t *)CMSG_DATA(cmsg)) = txtime_ns;

    sendmsg(sock, &msg, 0);
}
```

---

## 지연 목표 (의료 로봇 SRS)

```
의료 로봇 Latency/Jitter 요구사항 (SRS 기반):

┌───────────────────┬──────────┬─────────┬─────────────┬─────────┐
│ 트래픽 유형       │ 목표 지연 │ Jitter  │ 기술        │ ASIL    │
├───────────────────┼──────────┼─────────┼─────────────┼─────────┤
│ 관절 제어 명령    │ < 1 ms   │< 50 µs  │ TAS + RT    │ ASIL D  │
│ 힘/토크 피드백    │ < 1 ms   │< 50 µs  │ TAS         │ ASIL D  │
│ 햅틱 피드백       │ < 500 µs │< 10 µs  │ TAS + FPGA  │ ASIL C  │
│ E-Stop 신호       │< 100 µs  │< 10 µs  │ TAS TC7     │ ASIL D  │
│ PTP Sync 메시지   │< 100 µs  │< 1 µs   │ TAS TC7     │ -       │
│ 영상 스트림       │< 100 ms  │< 10 ms  │ CBS         │ QM      │
│ 진단 (DoIP)      │< 500 ms  │ 무제한  │ BE          │ QM      │
│ OTA 업데이트      │ 무제한   │ 무제한  │ BE          │ QM      │
└───────────────────┴──────────┴─────────┴─────────────┴─────────┘

기술별 달성 가능 지연:
  FPGA/ASIC:      < 1 µs    (하드웨어 직접 제어)
  PREEMPT_RT:     < 50 µs   (Linux RT 패치)
  TSN TAS:        결정론적  (Jitter 최소화)
  TSN CBS:        대역폭 보장, 일부 Jitter 허용
  Best Effort:    수ms      (제어에 부적합)
```

---

## TSN 도입 전후 측정 비교

```bash
# TSN TAS 도입 전 측정 (일반 Ethernet)
# 시나리오: 관절 제어 1kHz + 영상 100Mbps 동시 전송
tshark -i eth0 -f "udp dst port 5001" \
  -w /tmp/capture_before_tsn.pcap -a duration:30

# TSN TAS 설정 후 측정 (tc_taprio.md 참조)
tc qdisc replace dev eth0 parent root handle 100 taprio \
  num_tc 3 \
  map 2 2 1 0 0 0 0 0 0 0 0 0 0 0 0 0 \
  queues 1@0 1@1 1@2 \
  base-time $(date +%s%N) \
  sched-entry S 04 500000 \   # TC2 (E-Stop): 500µs
  sched-entry S 02 300000 \   # TC1 (제어): 300µs
  sched-entry S 01 200000 \   # TC0 (영상): 200µs
  clockid CLOCK_TAI

tshark -i eth0 -f "udp dst port 5001" \
  -w /tmp/capture_after_tsn.pcap -a duration:30

# 비교 분석
python3 latency_analyze.py /tmp/capture_before_tsn.pcap someip 1.0
python3 latency_analyze.py /tmp/capture_after_tsn.pcap someip 1.0

# 전형적인 결과:
# TSN 이전: Jitter(stdev) = 2.345ms, Max = 12.567ms
# TSN 이후: Jitter(stdev) = 0.032ms, Max = 0.987ms
```

---

## 트러블슈팅

### 문제 1: cyclictest Max 지연이 수ms로 증가

```
증상: cyclictest 실행 중 Max 지연이 50µs 목표를 크게 초과
      (예: 3000µs = 3ms)

원인 분석 우선순위:
  1. CPU C-state 진입 (가장 흔한 원인)
     확인: powertop → CPU idle states C2/C3 진입률 확인
     해결: cpupower idle-set -D 1

  2. IRQ 스톰 (대량 인터럽트 발생)
     확인: watch -n 1 'cat /proc/interrupts'
     해결: IRQ 어피니티로 RT 코어 분리

  3. 메모리 스왑 발생
     확인: vmstat 1 | head -10
     해결: swapoff -a 또는 mlockall()

  4. NUMA 교차 접근
     확인: numastat -p $(pidof robot_controller)
     해결: numactl --cpunodebind=0 --membind=0

  5. 커널 RCU 처리
     확인: dmesg | grep rcu
     해결: 커널 파라미터 rcu_nocbs=<RT_CPUs>
```

### 문제 2: TSN TAS 적용 후에도 Jitter 높음

```
증상: tc taprio 설정 후에도 관절 제어 Jitter가 500µs 초과

원인 분석:
  1. base-time 정렬 미흡
     → GCL이 PTP 시간과 정렬되지 않아 슬롯 위반
     확인: tc -d qdisc show dev eth0 | grep base-time
     해결: base-time을 현재 TAI 시각 + 1초 정렬 값으로 재설정

  2. 애플리케이션 전송 시각이 GCL 슬롯 밖
     → 애플리케이션이 제어 주기와 GCL 슬롯을 맞추지 않음
     해결: SO_TXTIME + ETF로 정확한 전송 시각 예약

  3. 다른 노드의 CBS 트래픽이 슬롯 침범
     확인: tshark 캡처 후 Wireshark Statistics → I/O Graph로 슬롯 위반 탐지
     해결: CBS idleslope 재계산 (tc_taprio.md 참조)

  4. NIC 드라이버가 ETF/TAS 오프로드 미지원
     확인: ethtool -T eth0 | grep hardware
     해결: 소프트웨어 모드(tc taprio flags 0x0) 사용 또는 NIC 교체
```

### 문제 3: sockperf P99.9 지연이 목표의 10배

```
증상: sockperf로 측정 시 P50은 250µs로 양호하나
      P99.9가 2500µs로 목표(< 1ms) 초과

원인:
  Tail Latency는 드물게 발생하는 최악 시나리오를 나타냄
  · GC(Garbage Collection): Java/Go 사용 시
  · 메모리 페이지 폴트: 초기화 안 된 메모리 접근
  · 인터럽트 코얼레싱: 여러 패킷 묶어 처리 시 일부 패킷 지연

해결:
  # C/C++: mlockall로 메모리 잠금
  mlockall(MCL_CURRENT | MCL_FUTURE)

  # 사전 워밍 (페이지 폴트 예방)
  memset(large_buffer, 0, sizeof(large_buffer))  // 초기화

  # 인터럽트 코얼레싱 비활성
  ethtool -C eth0 rx-usecs 0 rx-frames 1

  # RT 스레드 우선순위 최고로 설정
  struct sched_param sp = {.sched_priority = 99};
  sched_setscheduler(0, SCHED_FIFO, &sp)
```

---

## Reference
- [Cisco - Understanding Jitter in Packet Voice Networks](https://www.cisco.com/c/en/us/support/docs/voice/voice-quality/18902-jitter-packet-voice.html)
- [RFC 3393 - IP Packet Delay Variation Metric](https://datatracker.ietf.org/doc/html/rfc3393)
- [PREEMPT_RT Wiki](https://wiki.linuxfoundation.org/realtime/start)
- [cyclictest - rt-tests](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)
- [Linux Network Performance Tuning](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
- [Intel - Network Latency Performance](https://www.intel.com/content/www/us/en/developer/articles/guide/open-source-firmware-performance-guide-reducing-latency-in-network-applications.html)
- [sockperf - Network Performance Measurement](https://github.com/Mellanox/sockperf)
- [IEEE 802.1Qbv - Time-Aware Shaper](https://standards.ieee.org/ieee/802.1Qbv/3975/)
- [SO_TXTIME and ETF qdisc](https://www.kernel.org/doc/html/latest/networking/etf.html)
