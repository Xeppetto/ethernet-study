# linuxptp - PTP/gPTP 설정 및 운용

## 개요
linuxptp는 IEEE 1588(PTP)와 IEEE 802.1AS(gPTP)를 Linux에서 구현하는 오픈소스 프로젝트입니다. `ptp4l`(PTP 데몬)과 `phc2sys`(시스템 클록 동기화)로 구성됩니다. TSN 네트워크에서 마이크로초 수준의 시간 동기화를 달성하는 데 필수적입니다.

---

## 구성요소 역할

```
linuxptp 아키텍처:

┌──────────────────────────────────────────────────────────────┐
│                    Linux 시스템                               │
│                                                              │
│  ┌──────────────┐          ┌──────────────────────────────┐  │
│  │   ptp4l      │          │        phc2sys               │  │
│  │              │          │                              │  │
│  │ Grandmaster  │          │  NIC PHC → System Clock 동기화│  │
│  │ 또는 Slave   │◄────────►│  (CLOCK_REALTIME, TAI)       │  │
│  │ BMCA 실행    │          │                              │  │
│  └──────┬───────┘          └──────────────────────────────┘  │
│         │ Hardware Timestamping                               │
│  ┌──────┴───────────────────────────┐                         │
│  │   NIC PHC (PTP Hardware Clock)   │                         │
│  │   Intel i210/i225, Mellanox 등   │                         │
│  └──────────────────────────────────┘                         │
│         │                                                    │
└─────────┼────────────────────────────────────────────────────┘
          │ Ethernet (gPTP 메시지)
    ┌─────▼────────┐
    │ TSN Switch   │ ← Transparent Clock (TC): 지연 보정
    └──────────────┘
```

---

## 하드웨어 타임스탬핑 확인

```bash
# NIC의 타임스탬핑 지원 확인
ethtool -T eth0

# 출력 예시 (Intel i225):
# Time stamping parameters for eth0:
# Capabilities:
#         hardware-transmit     ← TX HW 타임스탬핑
#         software-transmit     ← TX SW 타임스탬핑
#         hardware-receive      ← RX HW 타임스탬핑
#         software-receive      ← RX SW 타임스탬핑
#         hardware-raw-clock    ← PHC (PTP HW Clock)
# PTP Hardware Clock: 0        ← /dev/ptp0에 해당
# Hardware Transmit Timestamp Modes:
#         off
#         on
# Hardware Receive Filter Modes:
#         none
#         ptpv1-l4-sync
#         ptpv2-l4-sync
#         ptpv2-l2-sync
#         ptpv2-l2-event

# PHC 장치 확인
ls /dev/ptp*
# /dev/ptp0  /dev/ptp1

# PHC 현재 시각 읽기
phc_ctl /dev/ptp0 get
# phc_ctl[eth0]: clock time is 1700000000.123456789
```

---

## ptp4l 설정 (Grandmaster Clock)

```ini
# /etc/linuxptp/gm.conf - Grandmaster Clock 설정

[global]
# 802.1AS 프로파일
transportSpecific   1       # 1 = IEEE 802.1AS
domainNumber        0       # 도메인 번호

# BMCA 우선순위
priority1           1       # 최고 우선순위 → GM이 됨
priority2           1
clockClass          6       # 6 = GPS 잠금 (최고 품질)

# 타임스탬핑 모드
time_stamping       hardware

# P2P 지연 측정 (802.1AS 필수)
delay_mechanism     P2P

# 메시지 간격
logSyncInterval     -3      # 2^-3 = 8 Hz (125ms 주기)
logMinPdelayReqInterval -3
logAnnounceInterval 1       # 2^1 = 2초

verbose             1
logging_level       7

[eth0]
```

```bash
# ptp4l 실행 (Grandmaster)
sudo ptp4l -f /etc/linuxptp/gm.conf -i eth0

# 로그 예시:
# ptp4l[0.000]: selected /dev/ptp0 as PTP clock
# ptp4l[1.234]: port 1: LISTENING to MASTER on INIT_COMPLETE
# ptp4l[2.456]: selected local clock 001122.fffe.334455 as best master
# ptp4l[3.678]: port 1: assuming the grand master role
```

---

## ptp4l 설정 (Slave Clock)

```ini
# /etc/linuxptp/slave.conf - Slave 설정

[global]
transportSpecific   1
domainNumber        0

priority1           128     # 중간 우선순위 (Slave 역할)
priority2           128

time_stamping       hardware
delay_mechanism     P2P

logSyncInterval     -3
logMinPdelayReqInterval -3
logAnnounceInterval 1

# 동기화 임계치
step_threshold      1.0     # 1초 이상 차이 시 스텝 조정
first_step_threshold 0.00002 # 첫 번째 스텝: 20µs

[eth0]
```

```bash
# ptp4l 실행 (Slave)
sudo ptp4l -f /etc/linuxptp/slave.conf -i eth0

# 로그 예시 (동기화 완료 후):
# ptp4l[12.789]: rms 45.3 ns max 123.5 ns freq -234 +/- 15 delay 456 +/- 5 ns
#    ↑ RMS offset  ↑ 최대 offset   ↑ 주파수 조정      ↑ 전파 지연
# → rms < 1000 ns (1µs) 달성 시 정상
```

---

## phc2sys - PHC → 시스템 시계 동기화

```bash
# 기본 실행 (PHC → CLOCK_REALTIME)
sudo phc2sys -s eth0 -c CLOCK_REALTIME \
  -O 37 \       # TAI-UTC 오프셋 (2024년 기준 37초)
  -m            # 로그 출력

# 옵션 설명:
#   -s eth0              : 소스 = eth0의 PHC
#   -c CLOCK_REALTIME    : 대상 = 시스템 클록
#   -O 37                : TAI-UTC 오프셋 (37초)
#   -m                   : 로그 출력

# systemd 서비스 등록
# /etc/systemd/system/phc2sys.service
[Unit]
Description=PHC to System Clock Synchronization
After=ptp4l.service

[Service]
ExecStart=/usr/sbin/phc2sys -s eth0 -c CLOCK_REALTIME -O 37 -m
Restart=always

[Install]
WantedBy=multi-user.target

sudo systemctl enable --now phc2sys
```

---

## pmc (PTP Management Client) - 모니터링

```bash
# Grandmaster 정보 조회
sudo pmc -u -b 0 'GET GRANDMASTER_SETTINGS_NP'
# clockClass      6       ← GPS 동기화
# timeSource      0x20    ← GPS
# currentUtcOffset 37

# 포트 상태 조회
sudo pmc -u -b 0 'GET PORT_DATA_SET'
# portState               MASTER          ← 또는 SLAVE
# peerMeanPathDelay       456             ← ns 단위 전파 지연

# 현재 타임오프셋 모니터링
sudo pmc -u -b 0 'GET CURRENT_DATA_SET'
# offsetFromMaster        -45.3           ← -45.3 ns 오프셋
# meanPathDelay           456.7           ← 456.7 ns 전파 지연
# stepsRemoved            1               ← GM까지 홉 수

# 실시간 모니터링 (1초 간격)
watch -n 1 'sudo pmc -u -b 0 "GET CURRENT_DATA_SET"'
```

---

## 동기화 성능 측정

```python
# linuxptp 동기화 정밀도 분석
import subprocess
import re
import time
import statistics

def get_offset_ns():
    """pmc를 통해 현재 오프셋 읽기 (나노초)"""
    result = subprocess.run(
        ['pmc', '-u', '-b', '0', 'GET CURRENT_DATA_SET'],
        capture_output=True, text=True
    )
    match = re.search(r'offsetFromMaster\s+([\-\d.]+)', result.stdout)
    if match:
        return float(match.group(1))
    return None

def analyze_sync(samples=100, interval=0.1):
    offsets = []
    for _ in range(samples):
        offset = get_offset_ns()
        if offset is not None:
            offsets.append(abs(offset))
        time.sleep(interval)

    if not offsets:
        print("오프셋 데이터 없음 (ptp4l 실행 확인)")
        return

    print(f"샘플 수: {len(offsets)}")
    print(f"평균 오프셋: {statistics.mean(offsets):.1f} ns")
    print(f"최대 오프셋: {max(offsets):.1f} ns")
    print(f"표준편차(Jitter): {statistics.stdev(offsets):.1f} ns")
    print(f"TSN 요구 (<1µs): {'OK' if max(offsets) < 1000 else 'FAIL'}")

analyze_sync(samples=100, interval=0.1)
```

---

## 동기화 정밀도 목표

```
환경별 요구 정밀도:

┌──────────────────────────┬──────────────────────────────┐
│ 환경                     │ 목표 동기화 정밀도            │
├──────────────────────────┼──────────────────────────────┤
│ 일반 IT (NTP)            │ 1~100 ms                     │
│ 산업 자동화 (일반 PTP)    │ < 10 µs (SW 타임스탬핑)      │
│ TSN (IEEE 802.1AS)       │ < 1 µs (HW 타임스탬핑)       │
│ 의료 로봇 관절 제어        │ < 500 ns                    │
│ 방송/오디오 (AES67)       │ < 1 µs                      │
│ 전력 계통 (IEC 61850-9-3) │ < 100 ns                    │
└──────────────────────────┴──────────────────────────────┘

NIC 선택:
  Intel i210/i225/i226: < 1µs, linuxptp 지원 우수
  Meinberg LANTIME:     GPS 기반 GM, < 100ns
  FPGA 기반:            < 10ns (초정밀)
```

---

## NTP 비활성화 (PTP와 충돌 방지)

```bash
# NTP 중지 (PTP와 함께 사용 금지)
sudo systemctl stop systemd-timesyncd
sudo systemctl disable systemd-timesyncd
sudo timedatectl set-ntp false

# 상태 확인
timedatectl
# System clock synchronized: yes
# NTP service: inactive   ← NTP 비활성화 확인
```

---

## Reference
- [LinuxPTP Project](http://linuxptp.sourceforge.net/)
- [linuxptp GitHub](https://github.com/richardcochran/linuxptp)
- [Red Hat - Configuring PTP with ptp4l](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/using-ptp-with-clock-synchronization_configuring-basic-system-settings)
- [IEEE 1588-2019 - PTP Standard](https://standards.ieee.org/ieee/1588/6825/)
- [IEEE 802.1AS-2020](https://standards.ieee.org/ieee/802.1AS/7121/)
