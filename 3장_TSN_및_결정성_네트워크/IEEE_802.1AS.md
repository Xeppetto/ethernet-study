# IEEE 802.1AS (gPTP: Generalized Precision Time Protocol)

## 개요
TSN(Time Sensitive Networking)의 기반이 되는 시간 동기화 표준입니다. 네트워크 전체가 동일한 시간을 공유해야 결정론적 통신(Deterministic Communication)이 가능합니다. IEEE 802.1AS는 Ethernet 네트워크에서 **서브마이크로초(< 1µs)** 수준의 시간 동기화를 제공합니다.

---

## PTP 계층 구조

```
IEEE 1588 (PTP: Precision Time Protocol)  ← 부모 표준
    │
    └── IEEE 802.1AS (gPTP)                ← L2 최적화 프로파일
            │
            ├── IEEE 802.1AS-2011          ← 1세대
            └── IEEE 802.1AS-2020         ← 현재 (이중화 GM 지원, Hot Standby)
```

### IEEE 1588 vs IEEE 802.1AS 비교
| 특성 | IEEE 1588 v2 | IEEE 802.1AS |
|------|-------------|--------------|
| 동작 계층 | L2 또는 L3 (IP) | L2 전용 (Ethernet) |
| 도메인 | 다중 도메인 | 단일 도메인 (기본), 다중 도메인 지원 |
| 경로 지연 | E2E 또는 P2P | P2P 전용 |
| 정밀도 | < 1µs | < 1µs (실측 수십~수백ns) |
| 차량/TSN 적합성 | 보통 | 우수 (TSN 최적화) |
| L3 지원 | ✓ | ✗ (L2 전용) |

---

## gPTP 메시지 타입

gPTP는 8가지 메시지 타입을 사용합니다. 이벤트 메시지는 하드웨어 타임스탬프가 필요합니다.

| 메시지 | 분류 | 방향 | 설명 |
|--------|------|------|------|
| **Sync** | 이벤트 | GM → Slave | 동기화 기준 시각 t1 전송. 하드웨어 타임스탬프 필수 |
| **Follow_Up** | General | GM → Slave | Two-Step 방식에서 t1의 정확한 값 전송. CorrectionField 포함 |
| **Pdelay_Req** | 이벤트 | 양방향 | 경로 지연 측정 요청. 전송 시각 t1 타임스탬프 |
| **Pdelay_Resp** | 이벤트 | 응답 | 요청 수신 시각 t2, 응답 전송 시각 t3 포함 |
| **Pdelay_Resp_Follow_Up** | General | 응답 | Two-Step Pdelay에서 t3 정확값 전송 |
| **Announce** | General | Master → Slave | GM 정보 브로드캐스트. BMCA에 사용. GM Priority, ClockClass 포함 |
| **Signaling** | General | 양방향 | 메시지 전송 주기 협상 등 파라미터 교환 |
| **Management** | General | 양방향 | 원격 설정 및 상태 조회 (pmc 도구 사용) |

### One-Step vs Two-Step Clock

```
One-Step Clock:
  Sync 전송 시 t1을 실시간으로 프레임에 직접 삽입
  ┌──────────────────────────────────────┐
  │  Sync  │  t1 (하드웨어가 직접 삽입)  │
  └──────────────────────────────────────┘
  장점: 메시지 수 절반, 지연 감소
  단점: 하드웨어 수준 타임스탬프 수정 능력 필요
  사용: 고성능 NIC, ASIC 기반 스위치

Two-Step Clock:
  Sync 전송 후 Follow_Up으로 t1 전달
  ┌───────────┐  ┌───────────────────────┐
  │  Sync (0) │  │  Follow_Up (t1 값)    │
  └───────────┘  └───────────────────────┘
  장점: 소프트웨어로 구현 가능
  단점: 메시지 2배, 소량 지연 증가
  사용: linuxptp ptp4l 기본 모드 (twoStepFlag=1)
```

---

## 동작 원리

### Best Master Clock Algorithm (BMCA)
네트워크 내에서 시간 기준이 되는 Grandmaster Clock(GM)을 자동 선출합니다.

```
선출 우선순위 (낮은 값이 더 우선):

1. Priority1           ← 관리자가 수동 설정 (0~255, 기본 128)
2. Clock Class         ← 시계 품질 등급
3. Clock Accuracy      ← 측정 정밀도
4. offsetScaledLogVariance ← 주파수 안정도
5. Priority2           ← 보조 우선순위 (기본 128)
6. Clock Identity      ← MAC 기반 고유 ID (최종 결정)

예: GM 후보 비교
  장치 A: Priority1=128, ClockClass=6 (GPS 동기화)
  장치 B: Priority1=128, ClockClass=7 (GPS 홀드오버)
  결과: 장치 A가 GM 선출 (ClockClass 6 < 7)
```

### ClockClass 값 의미

| ClockClass | 의미 | 예시 |
|-----------|------|------|
| 6 | PRC 동기화 (Primary Reference Clock 잠금) | GPS 수신 중 |
| 7 | PRC 잠금 후 홀드오버 가능 | GPS 잠금 직후 |
| 52 | GNSS 직접 동기화 (GPS/Galileo 등) | 안테나 수신 중 |
| 135 | GNSS 홀드오버 (신호 일시 손실) | GPS 신호 약화 |
| 187 | 외부 기준 없음, 로컬 발진기 | 독립 동작 중 |
| 193 | Slave Only 포트 (GM 불가) | slaveOnly=1 |
| 248 | 기본값 (외부 기준 없음) | linuxptp 기본 |
| 255 | Slave Only (BMCA 참여 불가) | 항상 슬레이브 |

### BMCA 포트 상태 머신

```
전원 ON
    │
    ▼
┌─────────────┐
│INITIALIZING │ ← 하드웨어 초기화 중
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  LISTENING  │ ← Announce 수신 대기 (announceReceiptTimeout 동안)
└──────┬──────┘
       │ BMCA 결과에 따라
  ┌────┴─────────────┐
  │                  │
  ▼                  ▼
┌──────────┐    ┌──────────┐
│PRE_MASTER│    │  SLAVE   │ ← 더 좋은 GM 발견
│ (대기)   │    │ (동기화) │
└────┬─────┘    └────┬─────┘
     │               │
     │ (대기 후)     ▼
     ▼          ┌──────────┐
┌──────────┐    │UNCALIB.  │ ← 초기 동기화 중
│  MASTER  │    │ (초기화) │
│ (GM 역할)│    └──────────┘
└──────────┘
     │
     ▼
┌──────────┐
│ PASSIVE  │ ← 동일 링크에 더 좋은 Master 존재 시
└──────────┘
     │
     ▼
┌──────────┐
│ DISABLED │ ← 관리자 비활성화
└──────────┘
```

### 메시지 흐름 및 경로 지연 측정

```
GM (Grandmaster)           Bridge (스위치)           Slave
      │                         │                      │
      │──── Sync ──────────────►│──────────────────────►│
      │  (t1: 전송 시각)        │  CF += 체류 시간      │ t2: 수신 시각
      │                         │                      │
      │──── Follow_Up ─────────►│──────────────────────►│
      │  (t1 정확값, CF 포함)   │                      │
      │                         │                      │
      │◄──────────────────────────────── Pdelay_Req ────│ t3: 전송
      │                         │◄──── Pdelay_Req ──────│
      │                         │──── Pdelay_Resp ─────►│ t4: 수신
      │                         │     (t2_p, t3_p 포함) │
      │                         │──── Pdelay_Resp_FU ──►│ (Two-Step)
      │                         │     (t3_p 정확값)     │

경로 지연(Link Delay) 계산:
  link_delay = ((t4 - t3) - (t3_p - t2_p)) / 2

Slave 시간 보정:
  offset = t2 - (t1 + link_delay) - CF_accumulated
  Slave 시계 += offset 조정

주파수 보정 (PI 서보):
  freq_adj = Kp × offset + Ki × Σoffset
```

### Transparent Clock (스위치 내 지연 보정)

TSN 스위치는 프레임이 스위치 내부에서 대기하는 시간을 **Correction Field (CF)**에 누적합니다.

```
일반 스위치 (문제):
  Sync 프레임 수신 → 큐 대기 (수 µs ~ ms) → 전송
  큐 대기 시간 불확실 → 동기화 오류 발생

Transparent Clock (해결):
  Sync 프레임 수신 (t_in 기록)
      │
      ▼
  큐 대기 (가변 시간)
      │
      ▼
  전송 직전 (t_out 기록)
      │
      ▼
  CorrectionField += (t_out - t_in)   ← 체류 시간 누적
      │
      ▼
  수신 측: CF 값만큼 오프셋 보정 → 정밀도 유지

다중 홉 CF 누적 예시:
  Hop 1 스위치: CF = 2000ns
  Hop 2 스위치: CF = 2000 + 1500 = 3500ns
  Hop 3 스위치: CF = 3500 + 2200 = 5700ns
  Slave: offset 계산 시 CF=5700ns 자동 보정

Boundary Clock (대안):
  PTP 도메인을 물리적으로 분리
  각 세그먼트에서 독립적으로 BMCA 실행
  Transparent Clock보다 복잡하지만 대규모 네트워크에 적합
```

---

## linuxptp를 이용한 PTP 설정

### 기본 설치 및 확인
```bash
# 설치
apt-get install linuxptp

# PTP 지원 확인
ethtool -T eth0
# Capabilities:
#   hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)  ← HW 타임스탬프 필수
#   hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
#   hardware-raw-clock    (SOF_TIMESTAMPING_RAW_HARDWARE)

# PHC (PTP Hardware Clock) 장치 확인
ls /dev/ptp*
# /dev/ptp0

# PHC 정보 확인
phc_ctl /dev/ptp0 caps
```

### 완전한 ptp4l 설정 파라미터

```ini
# /etc/ptp4l.conf

[global]
# ── 클럭 정체성 ──
priority1               128     # BMCA Priority1 (0=최우선, 255=최하위)
priority2               128     # BMCA Priority2 (동순위 시 결정)
clockClass              248     # 클럭 품질 등급 (6=GPS, 248=기본)
clockAccuracy           0xFE    # 정밀도 (0x20=100ns, 0x21=250ns, 0xFE=미상)
offsetScaledLogVariance 0xFFFF  # 주파수 안정도 (낮을수록 좋음)

# ── 포트 설정 ──
logAnnounceInterval     0       # Announce 주기 (2^0 = 1초)
logSyncInterval         -3      # Sync 주기 (2^-3 = 125ms)
                                # 차량용: -3 (125ms), 정밀: -5 (31.25ms)
logMinPdelayReqInterval -3      # Pdelay 주기 (2^-3 = 125ms)
announceReceiptTimeout  3       # Announce 미수신 허용 횟수

# ── 전송 설정 ──
transportSpecific       0x1     # 802.1AS (gPTP) = 0x1, 일반 PTP = 0x0
network_transport       L2      # L2 또는 UDPv4, UDPv6
delay_mechanism         P2P     # 802.1AS는 P2P 필수 (E2E 사용 금지)
twoStepFlag             1       # 0=One-Step, 1=Two-Step (기본)

# ── 슬레이브 설정 ──
slaveOnly               0       # 1=항상 슬레이브, 0=BMCA로 결정

# ── 타임스탬프 설정 ──
tx_timestamp_timeout    10      # TX 타임스탬프 대기 시간 (ms)
summary_interval        0       # 통계 출력 주기 (2^0=1초)

# ── 서보 (시간 조정 알고리즘) ──
step_threshold          0.000002 # 2µs 초과 시 계단식 조정 (초 단위)
first_step_threshold    0.000020 # 초기 20µs 초과 시 계단식 조정
max_frequency           900000000 # 최대 주파수 조정량 (ppb)
pi_proportional_const   0.0     # PI 제어기 비례 상수 (0=자동)
pi_integral_const       0.0     # PI 제어기 적분 상수 (0=자동)

[eth0]
# 인터페이스 이름 섹션으로 활성화
```

### GM(Grandmaster) 설정
```bash
# /etc/ptp4l-gm.conf
[global]
priority1               128
priority2               128
clockClass              6        # GPS 기준 시계 (최우선)
clockAccuracy           0x20     # 100ns 정밀도
offsetScaledLogVariance 0x4E5D   # GPS 안정도
transportSpecific       0x1      # 802.1AS (gPTP)
logAnnounceInterval     0        # 1초
logSyncInterval         -3       # 125ms
logMinPdelayReqInterval -3       # 125ms
tx_timestamp_timeout    10
slaveOnly               0
delay_mechanism         P2P

[eth0]

# GM 실행
ptp4l -f /etc/ptp4l-gm.conf -i eth0 --verbose
```

### Slave 설정
```bash
# /etc/ptp4l-slave.conf
[global]
slaveOnly               1
transportSpecific       0x1
delay_mechanism         P2P
logSyncInterval         -3

[eth0]

# Slave 실행
ptp4l -f /etc/ptp4l-slave.conf -i eth0 --verbose

# PHC → 시스템 시계 동기화 (phc2sys)
# -a: 자동 포트 검색
# -rr: PHC ↔ 시스템 시계 양방향 허용
phc2sys -a -rr --verbose

# 특정 장치 지정 방법
phc2sys -s /dev/ptp0 -c CLOCK_REALTIME -O -37 -n 1 --verbose
# -O -37: UTC-TAI 오프셋 (-37초, 2024년 기준)
```

### ts2phc - GPS/1PPS 동기화
```bash
# GPS 안테나 연결 시 GPS 시각을 PHC로 동기화

# /etc/ts2phc.conf
[global]
ts2phc.pulsewidth       100000000  # 1PPS 펄스 폭 (100ms)
leapfile                /usr/share/zoneinfo/leap-seconds.list

[eth0]
ts2phc.extts_polarity   rising     # 상승 엣지 트리거
ts2phc.extts_correction 0          # 1PPS 케이블 지연 보정 (ns)

[/dev/ttyS0]
ts2phc.master           1          # GPS 시리얼 포트 (NMEA)

# ts2phc 실행
ts2phc -f /etc/ts2phc.conf -s nmea -c eth0 --verbose
# -s nmea: NMEA GPS 입력
# -c eth0: 동기화 대상 PHC

# PHC 시각 직접 확인
phc_ctl /dev/ptp0 get
```

### 동기화 상태 확인
```bash
# pmc 명령으로 상태 조회
pmc -u -b 0 'GET CURRENT_DATA_SET'
# offsetFromMaster  -45     ← GM과의 오프셋 (ns 단위)
# meanPathDelay     234     ← 평균 경로 지연 (ns)
# stepsRemoved      1       ← GM까지의 홉 수

pmc -u -b 0 'GET TIME_STATUS_NP'
# master_offset  -45        ← 마스터와 오프셋 (ns)
# ingress_time   1234567890 ← 수신 시각
# cumulativeScaledRateOffset 0  ← 주파수 오프셋

pmc -u -b 0 'GET PORT_DATA_SET'
# portState: SLAVE          ← 현재 포트 상태

# 원격 장치 상태 조회 (홉=1)
pmc -u -b 1 'GET PORT_DATA_SET'

# offset 실시간 측정 및 통계
ptp4l -f /etc/ptp4l.conf -i eth0 -s 2>&1 | \
  grep "master offset" | \
  awk '{print $5}' | \
  python3 -c "
import sys, statistics
vals = [int(l) for l in sys.stdin]
print(f'Min: {min(vals)}ns  Max: {max(vals)}ns  Stdev: {statistics.stdev(vals):.1f}ns')
"
```

---

## 정밀도 검증

### 일반적인 정밀도 수준
| 구성 | 정밀도 | 비고 |
|------|--------|------|
| 소프트웨어 타임스탬프 | 수십 µs | 커널 스케줄러 지터 영향 |
| 하드웨어 타임스탬프 (단일 스위치) | < 1 µs | Intel I210/I225, NXP |
| 하드웨어 타임스탬프 (4홉) | < 1 µs | Transparent Clock 적용 |
| GPS + 하드웨어 | 수십 ns | ts2phc + 고품질 TCXO |
| 802.1AS-2020 목표 | < 30 ns | 전용 TSN 인프라 |
| OCXO + GPS | < 10 ns | 방위/항공 산업용 |

```bash
# 1PPS GPIO 비교로 정밀 측정 (오실로스코프 또는 TDC)
# 장치 A (GM): /dev/ptp0 외부 타임스탬프 출력
# 장치 B (Slave): 1PPS 입력 비교

# PHC 외부 타임스탬프 기능 활성화
echo 1 > /sys/class/ptp/ptp0/extts_enable

# ts2phc로 1PPS 기반 PHC 동기화 후 오프셋 로그 분석
ts2phc -f /etc/ts2phc.conf -s nmea -c eth0 -l 7 2>&1 | \
  grep "offset" | awk '{print $NF}'
```

---

## 다중 도메인 (Multiple PTP Domains)

하나의 네트워크에서 용도별로 독립된 시간 도메인을 운용할 수 있습니다.

```
도메인 0 (domainNumber=0): 차량 제어 네트워크
  GM: Sensor ECU (GPS, ClockClass=52)
  Slave: ADAS ECU, Safety ECU, Camera

도메인 1 (domainNumber=1): 인포테인먼트 네트워크
  GM: 내부 발진기 (ClockClass=248)
  Slave: 오디오 DSP, 디스플레이 컨트롤러

설정:
ptp4l -f /etc/ptp4l-domain0.conf -i eth0 -n 0  # 도메인 0
ptp4l -f /etc/ptp4l-domain1.conf -i eth1 -n 1  # 도메인 1 (별도 포트)

단일 스위치에서 다중 도메인 지원:
  도메인 0과 도메인 1 트래픽은 독립적으로 처리
  VLAN으로 추가 격리 가능
  transportSpecific=0x1 (gPTP)로 도메인 구분
```

---

## 이중화 GM (IEEE 802.1AS-2020 Hot Standby)

단일 GM 고장 시 대기 GM이 자동 인수하는 Hot Standby 구성입니다.

```
Primary GM ──── PTP ──── Network ──── Slaves
     │              BMCA                  │
Standby GM ─────────────────────────────┘
     │
  (Primary 고장 → announceReceiptTimeout 후 Standby가 MASTER 전환)

Switchover 시간 계산:
  announceReceiptTimeout = 3
  logAnnounceInterval = 0  (2^0 = 1초)
  → 최대 3 × 1초 = 3초 후 전환

빠른 Switchover 설정:
  logAnnounceInterval = -2  (2^-2 = 250ms)
  announceReceiptTimeout = 3
  → 최대 3 × 250ms = 750ms 후 전환

Primary GM 설정: priority1 = 1   (최우선)
Standby GM 설정: priority1 = 2   (Primary 고장 시 선출)
```

---

## TSN 내에서의 역할

IEEE 802.1AS는 다른 TSN 표준의 전제 조건입니다.

```
802.1AS (시간 동기화)
    │ 모든 장치가 동일한 시간 공유 (< 1µs 오차)
    ▼
802.1Qbv (TAS: Time Aware Shaper)
    │ 정확한 시간에 게이트 열고 닫기 (GCL 실행)
    ▼
802.1Qbu (Frame Preemption)
    │ 정확한 시간에 선점 결정
    ▼
결정론적 저지연 트래픽 전송 (< 100µs, Jitter < 1µs)
```

---

## Reference
- [IEEE 802.1AS-2020 - Timing and Synchronization](https://standards.ieee.org/ieee/802.1AS/7123/)
- [linuxptp 프로젝트](http://linuxptp.sourceforge.net/)
- [linuxptp ptp4l man page](https://man7.org/linux/man-pages/man8/ptp4l.8.html)
- [linuxptp phc2sys man page](https://man7.org/linux/man-pages/man8/phc2sys.8.html)
- [Avnu Alliance - gPTP Profile](https://avnu.org/knowledgebase/time-synchronization/)
- [IEEE 1588-2019 - Precision Clock Synchronization Protocol](https://standards.ieee.org/ieee/1588/6825/)
