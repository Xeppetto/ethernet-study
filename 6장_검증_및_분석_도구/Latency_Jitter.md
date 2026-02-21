# Latency와 Jitter 분석

## 개요
의료 로봇 Ethernet 통신에서 Latency(지연 시간)와 Jitter(지연 변동)는 안전성과 직결되는 핵심 성능 지표입니다. 수술 로봇은 제어 루프 주기 1ms, Jitter < 50µs 수준이 요구되며, 이를 측정하고 원인을 분석하는 체계적인 방법이 필요합니다.

---

## Latency 구성 요소

```
End-to-End Latency 분해:

송신 애플리케이션
    │ Processing Delay (애플리케이션 처리)
    │   - OS 스케줄링 지연 (일반 Linux: 100µs~ms)
    │   - PREEMPT_RT 패치: < 50µs
    │
    ▼
NIC TX Queue
    │ Serialization (Transmission) Delay
    │   - 패킷 크기 / 링크 속도
    │   - 64 Byte @ 1Gbps = 0.512 µs
    │   - 1518 Byte @ 1Gbps = 12.14 µs
    │
    ▼
Ethernet Cable
    │ Propagation Delay
    │   - 빛의 속도 × 2/3 (구리 케이블 굴절률)
    │   - ≈ 5 ns/m → 10m = 50 ns
    │
    ▼
Switch Ingress
    │ Store-and-Forward Delay (기본 스위치)
    │   - 전체 프레임 수신 후 전달: 최대 12.14 µs
    │   Cut-Through: 헤더만 보고 즉시 전달: < 1 µs
    │
    ▼
Switch Queueing Delay
    │   - Best Effort: 0 ~ 수ms (혼잡 시)
    │   - TAS (taprio): GCL 슬롯 대기 (최대 슬롯 주기)
    │
    ▼
수신 NIC
    │ Receive Processing
    │   - 인터럽트 처리, DMA
    │
    ▼
수신 애플리케이션

총 지연 (전형적인 TSN 네트워크):
  Processing:   < 50 µs (RT-Linux)
  Serialization: 12 µs (1518B @ 1Gbps)
  Propagation:  < 1 µs (< 200m)
  Switch:       < 10 µs (Cut-Through + TAS)
  ─────────────────────────────────────
  합계:         < 100 µs (TSN 목표)
```

---

## Jitter 원인 분석

```
Jitter 발생 원인:

1. OS 스케줄링 불규칙성
   일반 Linux:    스케줄 지연 100µs ~ 수ms
   PREEMPT_RT:   < 50µs
   실시간 OS:     < 10µs
   FPGA/bare-metal: < 1µs

2. 네트워크 큐잉 (Queueing Delay)
   혼잡 없음:    Jitter ≈ 0
   혼잡 있음:    Jitter = 최대 큐 길이 × 전송 시간

3. 인터럽트 처리 (IRQ)
   소프트웨어 IRQ 병합 (coalescing): 지연 감소 but Jitter 증가
   IRQ 어피니티 설정으로 CPU 코어 고정

4. CPU 주파수 변동 (C-state, P-state)
   절전 모드 진입 시 복귀 지연: 수십 ~ 수백 µs

5. 메모리 캐시 미스
   NUMA 시스템에서 원격 메모리 접근: 수십 ns 추가
```

---

## 측정 도구

### cyclictest (PREEMPT_RT 지연 측정)

```bash
# 설치
sudo apt-get install rt-tests

# 기본 실행 (스레드 4개, 우선순위 99, 1000µs 주기, 10000회)
sudo cyclictest -p 99 -t 4 -n -i 1000 -l 10000

# 출력 예시:
# T: 0 (12345) P:99 I:1000 C: 10000 Min:     12 Act:     18 Avg:     21 Max:    156
# T: 1 (12346) P:99 I:1000 C: 10000 Min:     13 Act:     19 Avg:     22 Max:    178
# ↑ 스레드  ↑ PID         ↑ 주기(µs)       ↑Min/Act/Avg/Max (µs)

# 히스토그램 생성 (분포 분석)
sudo cyclictest -p 99 -t 1 -n -i 1000 -l 100000 \
  --histogram=200 \
  --histofile=/tmp/latency_hist.txt

# 히스토그램 플롯 (gnuplot)
gnuplot << 'EOF'
set terminal png size 1200,600
set output '/tmp/latency_hist.png'
set title "Scheduling Latency Distribution"
set xlabel "Latency (µs)"
set ylabel "Occurrences"
plot '/tmp/latency_hist.txt' using 1:2 with linespoints title "Thread 0"
EOF
```

### ping (기본 RTT 측정)

```bash
# 고정밀 ping (마이크로초 단위)
ping -i 0.001 -c 1000 192.168.1.100

# 결과:
# 1000 packets transmitted, 1000 received, 0% packet loss, time 999ms
# rtt min/avg/max/mdev = 0.254/0.312/1.234/0.045 ms

# iperf3 기반 RTT (Wireshark와 병행)
iperf3 -c 192.168.1.100 -u -b 100M -t 10 --get-server-output
```

### Hardware Timestamping 기반 정밀 측정

```bash
# ptp4l offset 로그 수집 (링크 파트너 간 편도 지연)
sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0 2>&1 | \
  grep "rms" | tee /tmp/ptp_latency.log

# 로그 분석
awk '/rms/{
    split($0, a)
    rms=a[2]; max=a[4]
    sum_rms+=rms; sum_max+=max; n++
} END{
    printf "평균 RMS: %.1f ns, 평균 MAX: %.1f ns, 샘플: %d\n",
           sum_rms/n, sum_max/n, n
}' /tmp/ptp_latency.log
```

---

## 지연 분석 Python 도구

```python
#!/usr/bin/env python3
"""
Latency/Jitter 분석 도구
PCAP 파일에서 SOME/IP 또는 DoIP 메시지 지연 측정
"""
import csv
import statistics
import subprocess
import sys

def analyze_pcap_latency(pcap_file: str, protocol_filter: str = "someip") -> dict:
    """
    Wireshark/tshark로 PCAP 분석 → 지연 통계 계산

    Args:
        pcap_file: PCAP 파일 경로
        protocol_filter: Wireshark 디스플레이 필터
    """
    # tshark로 타임스탬프 추출
    result = subprocess.run([
        'tshark', '-r', pcap_file,
        '-Y', protocol_filter,
        '-T', 'fields',
        '-e', 'frame.time_epoch',
        '-e', 'frame.len',
        '-e', 'ip.src',
        '-E', 'separator=\t'
    ], capture_output=True, text=True)

    if result.returncode != 0:
        print(f"오류: {result.stderr}")
        return {}

    # 타임스탬프 파싱
    timestamps = []
    frame_sizes = []
    for line in result.stdout.strip().split('\n'):
        if not line:
            continue
        parts = line.split('\t')
        if len(parts) >= 2:
            try:
                timestamps.append(float(parts[0]))
                frame_sizes.append(int(parts[1]))
            except ValueError:
                continue

    if len(timestamps) < 2:
        print("데이터 부족")
        return {}

    # 패킷 간격 계산 (Jitter)
    intervals_ms = [(timestamps[i+1] - timestamps[i]) * 1000
                    for i in range(len(timestamps) - 1)]

    # 통계 계산
    stats = {
        'count': len(timestamps),
        'duration_s': timestamps[-1] - timestamps[0],
        'avg_interval_ms': statistics.mean(intervals_ms),
        'min_interval_ms': min(intervals_ms),
        'max_interval_ms': max(intervals_ms),
        'stdev_ms': statistics.stdev(intervals_ms),
        'jitter_ms': statistics.stdev(intervals_ms),
        'pkt_rate_hz': len(timestamps) / (timestamps[-1] - timestamps[0]),
        'avg_frame_bytes': statistics.mean(frame_sizes),
    }

    # 분위수
    sorted_intervals = sorted(intervals_ms)
    n = len(sorted_intervals)
    stats['p95_ms'] = sorted_intervals[int(n * 0.95)]
    stats['p99_ms'] = sorted_intervals[int(n * 0.99)]
    stats['p999_ms'] = sorted_intervals[int(n * 0.999)]

    return stats


def print_report(stats: dict, target_ms: float = 1.0):
    """성능 보고서 출력"""
    print("=" * 55)
    print("  Latency/Jitter 분석 보고서")
    print("=" * 55)
    print(f"  패킷 수:           {stats['count']:,}")
    print(f"  측정 기간:         {stats['duration_s']:.1f} s")
    print(f"  패킷 레이트:       {stats['pkt_rate_hz']:.1f} Hz")
    print(f"  평균 프레임 크기:   {stats['avg_frame_bytes']:.0f} Byte")
    print("-" * 55)
    print(f"  평균 간격:         {stats['avg_interval_ms']:.3f} ms")
    print(f"  최소 간격:         {stats['min_interval_ms']:.3f} ms")
    print(f"  최대 간격:         {stats['max_interval_ms']:.3f} ms")
    print(f"  표준편차 (Jitter): {stats['stdev_ms']:.3f} ms")
    print(f"  P95:               {stats['p95_ms']:.3f} ms")
    print(f"  P99:               {stats['p99_ms']:.3f} ms")
    print(f"  P99.9:             {stats['p999_ms']:.3f} ms")
    print("-" * 55)
    ok = stats['max_interval_ms'] <= target_ms * 2
    print(f"  목표 ({target_ms}ms ±100%): {'PASS' if ok else 'FAIL'}")
    print("=" * 55)


if __name__ == '__main__':
    pcap_file = sys.argv[1] if len(sys.argv) > 1 else 'capture.pcap'
    stats = analyze_pcap_latency(pcap_file, 'someip')
    if stats:
        print_report(stats, target_ms=1.0)
```

---

## 시스템 튜닝 (지연 최소화)

```bash
# 1. PREEMPT_RT 커널 설치 (실시간 패치)
# Ubuntu 22.04 기준
sudo apt-get install linux-image-$(uname -r)-rt linux-headers-$(uname -r)-rt
# 재부팅 후 RT 커널 선택

# RT 커널 확인
uname -r
# 5.15.0-76-realtime  ← PREEMPT_RT 확인

# 2. CPU 절전 비활성화
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
sudo cpupower frequency-set -g performance

# C-state 비활성화 (CPU 절전 단계)
sudo cpupower idle-set -D 1   # C1까지만 허용

# 3. IRQ 어피니티 (NIC IRQ를 특정 CPU에 고정)
# NIC IRQ 번호 확인
cat /proc/interrupts | grep eth0

# CPU 3에 IRQ 43 고정 (0x8 = bit3)
echo 8 | sudo tee /proc/irq/43/smp_affinity

# 4. 네트워크 인터럽트 병합 설정 (낮을수록 지연 감소)
ethtool -C eth0 rx-usecs 0 rx-frames 1
ethtool -C eth0 tx-usecs 0 tx-frames 1

# 5. 소켓 버퍼 최적화
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.core.netdev_max_backlog=30000

# 6. RPS (Receive Packet Steering) 설정
echo ff | sudo tee /sys/class/net/eth0/queues/rx-0/rps_cpus
```

---

## 지연 목표 (의료 로봇)

```
의료 로봇 Latency/Jitter 요구사항:

┌──────────────────────┬────────────┬────────────┬─────────────┐
│ 트래픽 유형          │ 목표 지연  │ Jitter 허용│ 기술        │
├──────────────────────┼────────────┼────────────┼─────────────┤
│ 관절 제어 명령       │ < 1 ms     │ < 50 µs    │ TAS + RT    │
│ 힘/토크 피드백       │ < 1 ms     │ < 50 µs    │ TAS         │
│ 햅틱 피드백          │ < 1 ms     │ < 10 µs    │ TAS + FPGA  │
│ E-Stop 신호          │ < 100 µs   │ < 10 µs    │ TAS TC7     │
│ 영상 스트림          │ < 100 ms   │ < 10 ms    │ CBS (QoS)   │
│ 진단 (DoIP)         │ < 500 ms   │ 제한 없음  │ BE          │
│ 로그 전송            │ < 1 s      │ 제한 없음  │ BE          │
│ OTA 업데이트         │ 제한 없음  │ 제한 없음  │ BE          │
└──────────────────────┴────────────┴────────────┴─────────────┘

기술 스택:
  FPGA/ASIc:   < 1 µs (하드웨어 직접 제어)
  PREEMPT_RT:  < 50 µs (리눅스 RT 패치)
  TSN TAS:     결정론적 스케줄링 (지터 최소화)
  TSN CBS:     대역폭 보장 (유연한 지연)
```

---

## Reference
- [Cisco - Understanding Jitter in Packet Voice Networks](https://www.cisco.com/c/en/us/support/docs/voice/voice-quality/18902-jitter-packet-voice.html)
- [RFC 3393 - IP Packet Delay Variation Metric](https://datatracker.ietf.org/doc/html/rfc3393)
- [PREEMPT_RT Patch](https://wiki.linuxfoundation.org/realtime/start)
- [cyclictest - rt-tests](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)
- [Linux Network Performance Tuning](https://access.redhat.com/sites/default/files/attachments/20150325_network_performance_tuning.pdf)
