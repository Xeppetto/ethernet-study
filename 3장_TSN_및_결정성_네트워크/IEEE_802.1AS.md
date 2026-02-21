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
            └── IEEE 802.1AS-2020         ← 현재 (이중화 GM 지원)
```

### IEEE 1588 vs IEEE 802.1AS 비교
| 특성 | IEEE 1588 v2 | IEEE 802.1AS |
|------|-------------|--------------|
| 동작 계층 | L2 또는 L3 (IP) | L2 전용 (Ethernet) |
| 도메인 | 다중 도메인 | 단일 도메인 (TSN) |
| 경로 지연 | E2E 또는 P2P | P2P 전용 |
| 정밀도 | < 1µs | < 1µs (실측 수십~수백ns) |
| 차량/TSN 적합성 | 보통 | 우수 (TSN 최적화) |

---

## 동작 원리

### Best Master Clock Algorithm (BMCA)
네트워크 내에서 시간 기준이 되는 Grandmaster Clock(GM)을 자동 선출합니다.

```
Priority1 (낮을수록 우선) → Clock Class → Clock Accuracy →
Offset Scaled Log Variance → Priority2 → Clock ID (최종 결정)

예: GM 후보 비교
  장치 A: Priority1=128, Clock Class=6 (GPS 동기화)
  장치 B: Priority1=128, Clock Class=7 (로컬 발진기)
  결과: 장치 A가 GM 선출 (Clock Class 6 < 7)
```

### 메시지 흐름

```
GM (Grandmaster)           Bridge (스위치)           Slave
      │                         │                      │
      │──── Sync ──────────────►│──────────────────────►│
      │  (t1: 전송 시각)        │                      │ t2: 수신 시각
      │                         │                      │
      │──── Follow_Up ─────────►│──────────────────────►│
      │  (t1 정확값 포함)       │                      │
      │                         │                      │
      │         ◄── Pdelay_Req ─────────────────────── │ t3: 전송
      │                         │◄──── Pdelay_Req ──────│
      │                         │──── Pdelay_Resp ─────►│ t4: 수신
      │                         │                      │
경로 지연(Link Delay) = ((t4 - t3) - (Pdelay_Resp 처리 시간)) / 2

Slave 시간 보정:
  offset = t2 - (t1 + Link_Delay)
  Slave 시계 += offset 조정
```

### Transparent Clock (스위치 내 지연 보정)

일반 스위치와 달리 TSN 스위치는 프레임을 처리하는 내부 지연을 보정합니다.

```
일반 스위치:
  Sync 프레임 수신 → 큐 대기 → 전송
  큐 대기 시간 = 불확실 → 동기화 오류 발생

Transparent Clock (TSN 스위치):
  Sync 프레임 수신 → 큐 대기 시간 측정 → Correction Field 업데이트 → 전송
  수신 측: Correction Field 참조하여 오류 보정
  결과: 스위치 홉이 증가해도 정밀도 유지
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
#   software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
#   software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
#   hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)  ← HW 타임스탬프
#   hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)  ← HW 타임스탬프
#   hardware-raw-clock    (SOF_TIMESTAMPING_RAW_HARDWARE)
```

### GM(Grandmaster) 설정
```bash
# /etc/ptp4l.conf (GM 설정)
[global]
priority1               128
priority2               128
clockClass              6      # GPS 기준 시계 (최우선)
clockAccuracy           0x20   # 100ns 정밀도
offsetScaledLogVariance 0xffff
transportSpecific       0x1    # 802.1AS (gPTP)
logAnnounceInterval     0      # 1초
logSyncInterval         -3     # 125ms
logMinPdelayReqInterval -3     # 125ms
tx_timestamp_timeout    10
slaveOnly               0      # GM은 Slave 아님
delay_mechanism         P2P    # 802.1AS는 P2P 필수

[eth0]
# 인터페이스 설정

# GM 실행
ptp4l -f /etc/ptp4l.conf -i eth0 --verbose
```

### Slave 설정
```bash
# /etc/ptp4l.conf (Slave 설정)
[global]
slaveOnly               1
transportSpecific       0x1    # 802.1AS
delay_mechanism         P2P

# Slave 실행
ptp4l -f /etc/ptp4l.conf -i eth0 --verbose

# 시스템 시계 동기화 (phc2sys)
phc2sys -a -rr --verbose  # PHC → 시스템 시계 동기화
```

### 동기화 상태 확인
```bash
# pmc 명령으로 상태 조회
pmc -u -b 0 'GET CURRENT_DATA_SET'
# 출력:
# offsetFromMaster  -45     ← GM과의 오프셋 (ns 단위)
# meanPathDelay     234     ← 평균 경로 지연 (ns)
# stepsRemoved      1       ← GM까지의 홉 수

# 실시간 모니터링
pmc -u -b 0 'GET TIME_STATUS_NP'
# master_offset  -45        ← 마스터와 오프셋 (ns)
# ingress_time   1234567890 ← 수신 시각
# cumulativeScaledRateOffset 0  ← 주파수 오프셋
```

---

## 정밀도 검증

```bash
# 두 장치 간 시간 오프셋 측정 (1PPS 신호 비교)
# 장치 A (GM): GPS 연결, 1PPS 출력
# 장치 B (Slave): 1PPS 입력 비교

# 소프트웨어 측정 (개략적)
# /sys/class/ptp/ptp0/clock_name 확인
cat /sys/class/ptp/ptp0/extts_enable

# ts2phc로 GPS 시간과 PHC 동기화
ts2phc -f /etc/ts2phc.conf -s nmea -c eth0
```

### 일반적인 정밀도 수준
| 구성 | 정밀도 |
|------|--------|
| 소프트웨어 타임스탬프 | 수십 µs |
| 하드웨어 타임스탬프 (단일 스위치) | < 1 µs |
| 하드웨어 타임스탬프 (4홉) | < 1 µs (Transparent Clock) |
| GPS + 하드웨어 | 수십 ns |

---

## 이중화 GM (IEEE 802.1AS-2020)

단일 GM 고장 시 대기 GM이 자동 인수하는 Hot Standby 구성입니다.

```
Primary GM ──── PTP ──── Network ──── Slaves
     │                                   │
Standby GM ─────────────────────────────┘
     │
  (Primary 고장 시 수 Sync 주기 내 인수)
  (수십ms 이내 switchover)
```

---

## TSN 내에서의 역할

IEEE 802.1AS는 다른 TSN 표준의 전제 조건입니다.

```
802.1AS (시간 동기화)
    │ 모든 장치가 동일한 시간 공유
    ▼
802.1Qbv (TAS: Time Aware Shaper)
    │ 정확한 시간에 게이트 열고 닫기
    ▼
결정론적 저지연 트래픽 전송
```

---

## Reference
- [IEEE 802.1AS-2020 - Timing and Synchronization](https://standards.ieee.org/ieee/802.1AS/7123/)
- [linuxptp 프로젝트](http://linuxptp.sourceforge.net/)
- [Avnu Alliance - gPTP Profile](https://avnu.org/knowledgebase/time-synchronization/)
- [IEEE 1588-2019 - Precision Clock Synchronization Protocol](https://standards.ieee.org/ieee/1588/6825/)
