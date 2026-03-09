# linuxptp - PTP/gPTP 설정 및 운용

## 개요
linuxptp는 IEEE 1588(PTP)과 IEEE 802.1AS(gPTP)를 Linux에서 구현하는 오픈소스 프로젝트(2011~, GPL v2)입니다. 핵심 구성요소는 `ptp4l`(PTP 프로토콜 데몬), `phc2sys`(PHC↔시스템 클록 동기화), `ts2phc`(GPS/1PPS 기반 PHC 동기화), `pmc`(관리 클라이언트)로 이루어집니다.

TSN 네트워크에서 GCL(Gate Control List)의 시간 슬롯이 모든 노드에서 동일한 절대 시각을 기준으로 동작하려면 **서브마이크로초(< 1µs) 수준의 시간 동기화**가 필수입니다. linuxptp는 이 요구사항을 Linux 시스템에서 달성하는 표준 솔루션입니다.

> **의료 로봇 관점**: 수술 로봇 시스템에서 linuxptp는 단순한 시계 동기화 이상의 역할을 합니다. 관절 제어 ECU들이 모두 동일한 TAI 기반 시각을 공유해야 taprio의 GCL 슬롯이 올바르게 정렬됩니다. 예를 들어, Grandmaster(HPC)의 "TAI 시각 T에서 TC7 게이트 열기" 명령이 Safety ECU에서도 정확히 동일한 T에 실행되어야 E-Stop 패킷이 스케줄된 슬롯에 전송됩니다. linuxptp 동기화 오차 > 1µs가 발생하면 GCL 정렬이 깨져 안전 트래픽이 BE 슬롯으로 밀릴 수 있습니다. IEEE 802.1AS 이론 배경은 **3장 IEEE_802.1AS.md**를 참조하세요.

---

## linuxptp 구성요소와 아키텍처

```
linuxptp 전체 아키텍처:

┌─────────────────────────────────────────────────────────────────┐
│                      Linux 시스템                                 │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ptp4l  (PTP 프로토콜 데몬)                                │  │
│  │  - IEEE 802.1AS gPTP 또는 IEEE 1588 PTP 구현              │  │
│  │  - BMCA (Best Master Clock Algorithm) 실행                │  │
│  │  - Sync/Follow_Up/Pdelay 메시지 처리                      │  │
│  │  - PHC (PTP Hardware Clock) 직접 제어                     │  │
│  └─────────────────────┬────────────────────────────────────┘  │
│                         │ HW 타임스탬핑                          │
│  ┌──────────────────────▼─────────────────────────────────┐    │
│  │  NIC PHC (/dev/ptp0)  NIC PHC (/dev/ptp1)              │    │
│  │  Intel i210/i225      Mellanox ConnectX                 │    │
│  └──────────────────────┬─────────────────────────────────┘    │
│                         │                                       │
│  ┌──────────────────────▼─────────────────────────────────┐    │
│  │  phc2sys  (PHC → 시스템 클록 동기화)                     │    │
│  │  - PHC → CLOCK_REALTIME 또는 CLOCK_TAI                  │    │
│  │  - PI 서보 알고리즘으로 점진적 조정                        │    │
│  └──────────────────────┬─────────────────────────────────┘    │
│                         │                                       │
│  ┌──────────────────────▼─────────────────────────────────┐    │
│  │  CLOCK_REALTIME (시스템 시계)                            │    │
│  │  CLOCK_TAI      (TAI = UTC + 37초, 2024 기준)           │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ts2phc (별도: GPS/1PPS → PHC 동기화)                           │
│  pmc    (관리/모니터링 클라이언트)                                │
└─────────────────────────────────────────────────────────────────┘
          │ Ethernet (gPTP/PTP 메시지)
    ┌─────▼──────────┐
    │  TSN Switch     │ ← Transparent Clock (체류 시간 보정)
    └─────────────────┘
```

---

## 설치 및 하드웨어 확인

```bash
# ── 설치 ──
sudo apt-get install linuxptp

# 버전 확인
ptp4l --version
# linuxptp-3.1.1

# ── NIC HW 타임스탬핑 지원 확인 (필수) ──
ethtool -T eth0

# 출력 예시 (Intel i225, HW 지원):
# Time stamping parameters for eth0:
# Capabilities:
#   hardware-transmit     ← TX HW 타임스탬핑 지원 (필수)
#   software-transmit
#   hardware-receive      ← RX HW 타임스탬핑 지원 (필수)
#   software-receive
#   hardware-raw-clock    ← PHC 기준 클록 (필수)
# PTP Hardware Clock: 0  ← /dev/ptp0에 해당
# Hardware Transmit Timestamp Modes:
#   off
#   on
# Hardware Receive Filter Modes:
#   none
#   ptpv2-l2-sync
#   ptpv2-l2-event

# 위 세 항목(hardware-transmit, hardware-receive, hardware-raw-clock) 없으면
# 소프트웨어 타임스탬핑 → 마이크로초 수준 (TSN 용도 부적합)

# ── PHC 장치 확인 ──
ls /dev/ptp*          # /dev/ptp0, /dev/ptp1 등
phc_ctl /dev/ptp0 caps
# ptp clock supports:
#   50 programmable alarms
#   3 external time stamp channels
#   3 programmable periodic signals
#   pulse per second

# PHC 현재 시각 읽기
phc_ctl /dev/ptp0 get
# phc_ctl[eth0]: clock time is 1700000000.123456789 TAI
# (TAI = UTC + 37초, 2024년 기준)
```

---

## NTP 비활성화 (PTP와 충돌 방지)

```bash
# PTP와 NTP/chrony를 동시에 실행하면 시스템 클록이 양쪽에서 조정되어 발산함
# → ptp4l 실행 전 반드시 NTP 비활성화

sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd

# chrony 사용 시
sudo systemctl stop chronyd
sudo systemctl disable chronyd

# 상태 확인
timedatectl
# System clock synchronized: yes
# NTP service: inactive   ← NTP 비활성화 확인

# 또는 timedatectl로 NTP 직접 비활성화
sudo timedatectl set-ntp false
```

---

## ptp4l 핵심 설정 파라미터

```ini
# /etc/linuxptp/ptp4l.conf - 공통 설정 파라미터 상세 설명

[global]
# ── 802.1AS 프로파일 (TSN 필수) ──
transportSpecific       0x1     # 0x1=IEEE 802.1AS, 0x0=일반 PTP
network_transport       L2      # L2(Ethernet), UDPv4, UDPv6
delay_mechanism         P2P     # P2P=802.1AS 필수 (E2E 사용 금지)
domainNumber            0       # PTP 도메인 번호 (0~127)

# ── 타임스탬핑 ──
time_stamping           hardware # hardware=HW TS (권장), software=SW TS
tx_timestamp_timeout    10       # TX 타임스탬프 대기 시간 (ms)
                                 # 이 시간 초과 → "timed out" 경고

# ── BMCA 파라미터 (클록 품질 결정) ──
priority1               128      # BMCA 1차 우선순위 (0=최우선, 255=최하위)
priority2               128      # BMCA 2차 우선순위 (동순위 시)
clockClass              248      # 클록 품질 등급 (6=GPS, 248=내부 발진기)
clockAccuracy           0xFE     # 정밀도 코드 (0x20=100ns, 0xFE=미상)
offsetScaledLogVariance 0xFFFF   # 주파수 안정도 (낮을수록 좋음)

# ── 메시지 간격 (2의 거듭제곱 초) ──
logAnnounceInterval     0        # 2^0 = 1초 (BMCA 정보 브로드캐스트)
logSyncInterval         -3       # 2^-3 = 125ms (TSN 표준)
                                 # 더 빠른 동기화: -5 (31.25ms)
logMinPdelayReqInterval -3       # 2^-3 = 125ms (경로 지연 측정)
announceReceiptTimeout  3        # Announce 미수신 허용 횟수
                                 # 3 × 1초 = 3초 후 BMCA 재실행

# ── 동기화 제어 ──
twoStepFlag             1        # 1=Two-Step (Follow_Up 별도 전송)
                                 # 0=One-Step (Sync에 직접 포함, HW 필요)
slaveOnly               0        # 1=항상 슬레이브, 0=BMCA로 자동 결정

# ── PI 서보 알고리즘 (시계 조정) ──
step_threshold          0.000002  # 2µs 초과 시 스텝(순간) 조정
                                  # 미만: 점진적 주파수 조정
first_step_threshold    0.000020  # 초기 20µs 초과 시 스텝 조정
max_frequency           900000000 # 최대 주파수 조정량 (ppb, parts-per-billion)
pi_proportional_const   0.0       # 0=자동 계산 (권장)
pi_integral_const       0.0       # 0=자동 계산 (권장)

# ── 로깅 ──
summary_interval        0         # 통계 출력 주기 (2^0=1초)
verbose                 1         # 자세한 로그 출력
logging_level           6         # 6=INFO, 7=DEBUG

[eth0]
# 인터페이스 섹션으로 ptp4l 활성화
```

---

## ptp4l 설정 - Grandmaster Clock

```ini
# /etc/linuxptp/gm.conf - Grandmaster 전용 설정
[global]
transportSpecific       0x1
domainNumber            0
priority1               1       # 최고 우선순위 → 반드시 GM이 됨
priority2               1
clockClass              6       # 6=GPS Primary Reference Clock
clockAccuracy           0x20    # 100ns 수준 정밀도
offsetScaledLogVariance 0x4E5D  # GPS 발진기 안정도
time_stamping           hardware
delay_mechanism         P2P
logSyncInterval         -3
logMinPdelayReqInterval -3
logAnnounceInterval     0
twoStepFlag             1
slaveOnly               0
tx_timestamp_timeout    10
summary_interval        0

[eth0]
```

```bash
# GM 실행
sudo ptp4l -f /etc/linuxptp/gm.conf -i eth0

# 로그 예시 (GM 선출 완료):
# ptp4l[0.000]: selected /dev/ptp0 as PTP clock
# ptp4l[1.234]: port 1: LISTENING to MASTER on INIT_COMPLETE
# ptp4l[2.456]: selected local clock 001122.fffe.334455 as best master
# ptp4l[3.678]: port 1: assuming the grand master role

# systemd 서비스 등록
cat > /etc/systemd/system/ptp4l-gm.service << 'EOF'
[Unit]
Description=PTP Grandmaster (ptp4l)
After=network.target

[Service]
ExecStart=/usr/sbin/ptp4l -f /etc/linuxptp/gm.conf -i eth0
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now ptp4l-gm
```

---

## ptp4l 설정 - Slave Clock

```ini
# /etc/linuxptp/slave.conf - Slave ECU 설정
[global]
transportSpecific       0x1
domainNumber            0
priority1               128     # 중간 우선순위 (GM에게 양보)
priority2               128
time_stamping           hardware
delay_mechanism         P2P
logSyncInterval         -3
logMinPdelayReqInterval -3
logAnnounceInterval     0
slaveOnly               1       # 항상 슬레이브 강제 (BMCA 불참)
step_threshold          0.000002
first_step_threshold    0.000020
tx_timestamp_timeout    10
summary_interval        0

[eth0]
```

```bash
# Slave 실행
sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0

# 정상 동기화 로그:
# ptp4l[12.789]: rms 45 max 123 freq -234 +/- 15 delay 456 +/- 5
#    ↑ RMS 오프셋(ns) ↑ 최대 오프셋(ns) ↑ 주파수 조정(ppb) ↑ 전파 지연(ns)
# rms < 1000 ns (< 1µs) → TSN 요구 충족
# rms < 500 ns → 의료 로봇 관절 제어 요구 충족

# 동기화 과정 단계:
# LISTENING → UNCALIBRATED (첫 Sync 수신) → SLAVE (동기화 완료)
```

---

## phc2sys - PHC → 시스템 클록 동기화

```bash
# ── 기본 실행 ──
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O -37 -m

# 옵션 설명:
#   -s eth0          : 소스 = eth0의 PHC
#   -c CLOCK_REALTIME: 대상 = 시스템 클록
#   -O -37           : TAI-UTC 오프셋 (-37초, 2024년 기준)
#                      음수인 이유: CLOCK_REALTIME(UTC) = TAI - 37s
#   -m               : 모니터링 로그 출력

# ── TAI 클록으로 직접 동기화 (추천: 애플리케이션에서 CLOCK_TAI 사용) ──
sudo phc2sys -s eth0 -c CLOCK_TAI -O 0 -m

# ── 자동 포트 감지 모드 ──
sudo phc2sys -a -rr -m
# -a  : ptp4l의 활성 포트 자동 감지
# -rr : PHC ↔ 시스템 클록 양방향 허용

# ── phc2sys 로그 예시 ──
# phc2sys[15.123]: CLOCK_REALTIME rms 42 max 89 freq -1234 +/- 23 delay 345 +/- 5
# → rms: 시스템 클록과 PHC 간 오프셋 RMS (ns)
# → 정상: rms < 1000 ns

# ── systemd 서비스 ──
cat > /etc/systemd/system/phc2sys.service << 'EOF'
[Unit]
Description=PHC to System Clock (phc2sys)
After=ptp4l.service
Requires=ptp4l.service

[Service]
ExecStart=/usr/sbin/phc2sys -s eth0 -c CLOCK_REALTIME -O -37 -m
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now phc2sys
```

---

## ts2phc - GPS/1PPS 기반 정밀 동기화

```bash
# GPS 안테나 연결 시 GPS 시각을 PHC로 동기화
# → GM의 ClockClass=6 (GPS Primary Reference) 달성

# /etc/linuxptp/ts2phc.conf
cat > /etc/linuxptp/ts2phc.conf << 'EOF'
[global]
ts2phc.pulsewidth    100000000   # 1PPS 펄스 폭 (100ms = 100,000,000 ns)
leapfile             /usr/share/zoneinfo/leap-seconds.list

[eth0]
ts2phc.extts_polarity   rising   # 상승 엣지에서 타임스탬핑
ts2phc.extts_correction 0        # 1PPS 케이블 지연 보정 (ns)

[/dev/ttyS0]
ts2phc.master        1           # GPS 시리얼 포트 (NMEA 프로토콜)
EOF

# ts2phc 실행
sudo ts2phc -f /etc/linuxptp/ts2phc.conf \
    -s nmea \      # NMEA GPS 입력
    -c eth0 \      # 동기화 대상 PHC
    --verbose

# ts2phc 로그:
# ts2phc[0.000]: nmea 1PPS source: /dev/ttyS0
# ts2phc[1.000]: eth0 offset -123 s2 freq +456
# → offset이 수십 ns 수준으로 수렴하면 GPS 동기화 완료

# 이후 ptp4l GM으로 네트워크에 시각 배포
# ptp4l의 clockClass를 6으로 설정하면 GPS 기반 GM 역할
```

---

## pmc (PTP Management Client) - 상태 조회

```bash
# ── 현재 오프셋 및 지연 조회 ──
sudo pmc -u -b 0 'GET CURRENT_DATA_SET'
# currentDataSet.offsetFromMaster    -45.3  ← GM과 오프셋 (ns)
# currentDataSet.meanPathDelay      456.7   ← 평균 전파 지연 (ns)
# currentDataSet.stepsRemoved         1     ← GM까지 홉 수

# ── 포트 상태 조회 ──
sudo pmc -u -b 0 'GET PORT_DATA_SET'
# portDataSet.portState       SLAVE   ← MASTER/SLAVE/PASSIVE/LISTENING
# portDataSet.peerMeanPathDelay  456  ← 이웃 장치까지 지연 (ns)

# ── GM 정보 조회 ──
sudo pmc -u -b 0 'GET GRANDMASTER_SETTINGS_NP'
# grandmasterSettings.clockClass       6    ← GPS 품질
# grandmasterSettings.timeSource    0x20    ← GPS
# grandmasterSettings.currentUtcOffset 37  ← TAI-UTC 오프셋

# ── 시간 상태 상세 조회 ──
sudo pmc -u -b 0 'GET TIME_STATUS_NP'
# timeStatusNP.master_offset    -45    ← 마스터 오프셋 (ns)
# timeStatusNP.gmPresent        true   ← GM 존재 여부

# ── 원격 장치(1홉 너머) 조회 ──
sudo pmc -u -b 1 'GET PORT_DATA_SET'

# ── 실시간 모니터링 (1초 간격) ──
watch -n 1 'sudo pmc -u -b 0 "GET CURRENT_DATA_SET"'

# ── ptp4l 로그에서 오프셋 스트림 분석 ──
sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0 2>&1 | \
    grep "rms" | \
    awk '{
        match($0, /rms ([0-9]+)/, a)
        match($0, /max ([0-9]+)/, b)
        match($0, /delay ([0-9]+)/, c)
        printf "rms=%sns max=%sns delay=%sns\n", a[1], b[1], c[1]
    }'
```

---

## 동기화 성능 측정 및 분석

```python
#!/usr/bin/env python3
"""
linuxptp 동기화 정밀도 자동 분석 도구
실행 조건: ptp4l와 phc2sys가 실행 중이어야 함
"""
import subprocess
import re
import time
import statistics
import sys

def get_pmc_value(key: str) -> float | None:
    """pmc 명령으로 특정 값 읽기"""
    result = subprocess.run(
        ['pmc', '-u', '-b', '0', 'GET CURRENT_DATA_SET'],
        capture_output=True, text=True, timeout=2
    )
    match = re.search(rf'{re.escape(key)}\s+([\-\d.]+)', result.stdout)
    return float(match.group(1)) if match else None

def check_port_state() -> str:
    """포트 상태 확인"""
    result = subprocess.run(
        ['pmc', '-u', '-b', '0', 'GET PORT_DATA_SET'],
        capture_output=True, text=True, timeout=2
    )
    match = re.search(r'portState\s+(\w+)', result.stdout)
    return match.group(1) if match else 'UNKNOWN'

def analyze_sync(samples: int = 100, interval: float = 0.5,
                 target_ns: float = 1000.0) -> dict:
    """
    동기화 품질 분석
    Args:
        samples: 측정 횟수
        interval: 측정 간격 (초)
        target_ns: 목표 정밀도 (ns)
    """
    print(f"동기화 분석 시작 ({samples}회 × {interval}초 간격)...")

    # 포트 상태 확인
    state = check_port_state()
    if state not in ('SLAVE', 'MASTER'):
        print(f"[경고] 포트 상태: {state} (SLAVE/MASTER 아님)")

    offsets = []
    delays = []

    for i in range(samples):
        offset = get_pmc_value('offsetFromMaster')
        delay = get_pmc_value('meanPathDelay')

        if offset is not None:
            offsets.append(abs(offset))
        if delay is not None:
            delays.append(delay)

        if (i + 1) % 10 == 0:
            print(f"  진행: {i+1}/{samples}", end='\r')

        time.sleep(interval)

    if not offsets:
        print("오프셋 데이터 없음 (ptp4l 실행 확인)")
        return {}

    result = {
        'port_state':  state,
        'sample_count': len(offsets),
        'mean_offset_ns': statistics.mean(offsets),
        'max_offset_ns': max(offsets),
        'min_offset_ns': min(offsets),
        'stdev_ns': statistics.stdev(offsets) if len(offsets) > 1 else 0,
        'mean_delay_ns': statistics.mean(delays) if delays else 0,
        'target_ns': target_ns,
        'pass': max(offsets) < target_ns,
    }

    print(f"\n{'='*55}")
    print(f"  linuxptp 동기화 품질 보고서")
    print(f"{'='*55}")
    print(f"  포트 상태:           {result['port_state']}")
    print(f"  샘플 수:             {result['sample_count']}")
    print(f"  평균 오프셋:         {result['mean_offset_ns']:.1f} ns")
    print(f"  최소 오프셋:         {result['min_offset_ns']:.1f} ns")
    print(f"  최대 오프셋:         {result['max_offset_ns']:.1f} ns")
    print(f"  표준편차(Jitter):    {result['stdev_ns']:.1f} ns")
    print(f"  평균 전파 지연:      {result['mean_delay_ns']:.1f} ns")
    print(f"  목표 정밀도:         < {result['target_ns']:.0f} ns")
    print(f"  결과:                {'PASS' if result['pass'] else 'FAIL'}")
    print(f"{'='*55}")

    # 경고 메시지
    if result['max_offset_ns'] > 10000:
        print(f"\n[경고] 최대 오프셋 {result['max_offset_ns']:.0f}ns > 10µs")
        print("  → HW 타임스탬핑 확인, 스위치 Transparent Clock 확인")
    elif result['max_offset_ns'] > 1000:
        print(f"\n[주의] 최대 오프셋 {result['max_offset_ns']:.0f}ns > 1µs")
        print("  → TSN 요구는 충족하나 의료 로봇 관절 제어에는 부족할 수 있음")

    return result

if __name__ == '__main__':
    target = float(sys.argv[1]) if len(sys.argv) > 1 else 1000.0
    analyze_sync(samples=100, interval=0.5, target_ns=target)
```

---

## 동기화 정밀도 목표 및 하드웨어 선택

```
환경별 요구 동기화 정밀도:

┌──────────────────────────┬──────────────┬───────────────────────────┐
│ 적용 환경                 │ 목표 정밀도  │ 달성 방법                  │
├──────────────────────────┼──────────────┼───────────────────────────┤
│ 일반 IT (NTP)            │ 1~100 ms     │ ntpd/chrony               │
│ 산업 자동화 (SW PTP)      │ < 10 µs      │ linuxptp + SW 타임스탬핑  │
│ TSN 802.1AS              │ < 1 µs       │ linuxptp + HW 타임스탬핑  │
│ 의료 로봇 관절 제어        │ < 500 ns     │ linuxptp + HW + TC 스위치 │
│ 방송/오디오 (AES67)       │ < 1 µs       │ linuxptp + 전용 NIC       │
│ 전력 계통 (IEC 61850-9-3) │ < 100 ns     │ GPS + 전용 하드웨어        │
│ 계측/방위 산업            │ < 10 ns      │ OCXO + GPS + FPGA 기반   │
└──────────────────────────┴──────────────┴───────────────────────────┘

권장 NIC (linuxptp HW 타임스탬핑):
  Intel i210 (1Gbps):    < 1µs, 오픈소스 드라이버 성숙, 저가
  Intel i225/i226 (2.5G): < 1µs + 하드웨어 TAS 오프로드 지원
  Meinberg LANTIME:      GPS 기반 GM, < 100ns (방송/전력 산업)
  Renesas RZ/N2:         ARM 기반 TSN SoC, FPGA 타임스탬핑
  FPGA (Xilinx/Intel):   < 10ns (초정밀 계측용)
```

---

## 이중화 GM (Hot Standby)

```bash
# IEEE 802.1AS-2020의 이중화 GM 설정
# Primary GM 장애 시 Standby가 자동 인수

# Primary GM (priority1 = 1)
# /etc/linuxptp/gm-primary.conf
# priority1 = 1  ← 최우선: 정상 시 항상 GM

# Standby GM (priority1 = 2)
# /etc/linuxptp/gm-standby.conf
# priority1 = 2  ← Primary 장애 시 GM 역할 인수

# 빠른 Failover 설정 (announceReceiptTimeout × logAnnounceInterval)
# logAnnounceInterval = -2 → 2^-2 = 250ms
# announceReceiptTimeout = 3
# → 최대 3 × 250ms = 750ms 후 Standby가 GM으로 전환

# 전환 시간 모니터링
pmc -u -b 0 'GET PORT_DATA_SET' | grep portState
# Primary 장애 시: SLAVE → LISTENING → MASTER (Standby 전환)
# 정상 복구 시: MASTER → PASSIVE (Standby가 다시 대기)
```

---

## 자주 발생하는 문제와 해결

```
문제 1: "timed out while polling for TX timestamp"
  원인: HW TX 타임스탬핑이 너무 느림 (혼잡, 버그)
  해결:
    tx_timestamp_timeout 값 증가 (10 → 50)
    NIC 드라이버 버전 확인 (최신 버전 권장)
    ethtool -C eth0 tx-usecs 0 (TX 인터럽트 지연 최소화)

문제 2: "port 1: LISTENING" 상태에서 변화 없음
  원인: gPTP 멀티캐스트 패킷이 스위치에서 차단됨
  확인: tcpdump -i eth0 -e ether proto 0x88F7
  해결:
    스위치에서 gPTP 멀티캐스트(01:80:C2:00:00:0E) 허용
    IGMP Snooping 비활성화 또는 멀티캐스트 예외 처리

문제 3: rms 값이 수십 µs 이상 (수렴 안 됨)
  원인 A: 소프트웨어 타임스탬핑 사용 중
    확인: ethtool -T eth0 | grep hardware-receive
    해결: time_stamping hardware 설정 확인

  원인 B: 스위치에 Transparent Clock 없음
    확인: pmc 'GET CURRENT_DATA_SET' → stepsRemoved 확인
    해결: TSN 스위치(TC 지원) 사용 또는 Boundary Clock 구성

  원인 C: CPU 절전 모드 (C-state 진입)
    해결:
      echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
      cpupower idle-set -D 1

문제 4: phc2sys 오프셋이 지속적으로 증가
  원인: TAI-UTC 오프셋 설정 오류
  확인: pmc 'GET GRANDMASTER_SETTINGS_NP' → currentUtcOffset
  해결: phc2sys -O -37 (37 = 현재 TAI-UTC 오프셋, 윤초 적용)
        leapfile 경로 확인: /usr/share/zoneinfo/leap-seconds.list

문제 5: 동기화 정밀도가 갑자기 나빠짐
  원인: 네트워크 혼잡으로 Sync 패킷 지연
  확인: Wireshark로 Sync 메시지 주기 확인
  해결:
    TSN TAS로 PTP 트래픽에 전용 슬롯 할당 (PCP=7)
    taprio의 GCL에 PTP 전송 슬롯 추가
```

---

## Reference
- [LinuxPTP Project](http://linuxptp.sourceforge.net/)
- [linuxptp GitHub (Richard Cochran)](https://github.com/richardcochran/linuxptp)
- [Red Hat - Configuring PTP using ptp4l](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_using-ptp-with-clock-synchronization_configuring-basic-system-settings)
- [IEEE 802.1AS-2020 - Timing and Synchronization](https://standards.ieee.org/ieee/802.1AS/7123/)
- [IEEE 1588-2019 - PTP Standard](https://standards.ieee.org/ieee/1588/6825/)
- [Avnu Alliance - gPTP Profile](https://avnu.org/knowledgebase/time-synchronization/)
- [Intel - i225/i226 TSN NIC Guide](https://www.intel.com/content/www/us/en/products/sku/184676/intel-ethernet-controller-i225lm/specifications.html)
- [OpenAvnu - linuxptp Integration](https://github.com/Avnu/OpenAvnu/tree/master/daemons/gptp)
