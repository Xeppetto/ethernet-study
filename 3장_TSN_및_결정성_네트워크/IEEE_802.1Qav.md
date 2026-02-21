# IEEE 802.1Qav (CBS: Credit-Based Shaper)

## 개요
IEEE 802.1Qav는 **Credit-Based Shaper(CBS)**를 정의하는 TSN 표준입니다. 802.1Qbv TAS가 정밀한 시간 슬롯 스케줄링으로 결정론적 지연을 보장한다면, CBS는 **대역폭 예약과 버스트 제어**로 오디오/비디오 스트리밍과 같은 등시성(Isochronous) 트래픽에 적합합니다.

원래 IEEE 802.1BA(AVB: Audio Video Bridging)의 일부로 개발되었으며, TSN에서 계속 활용됩니다.

---

## CBS 동작 원리

### 크레딧 메커니즘

```
CBS 개념도:
  크레딧(Credit) = 전송 허용 여부 결정
  ┌─────────────────────────────────────────────────────────┐
  │  Credit                                                  │
  │    │                                                    │
  │  +128 ┤ 크레딧 최대값 (idleSlope에 따라 충전)            │
  │    │   ─────────────────────────────────────────────    │
  │    0   프레임 전송 조건: Credit ≥ 0                     │
  │    │   ─────────────────────────────────────────────    │
  │  -128 ┤ 크레딧 최소값 (프레임 전송 중 감소)              │
  └─────────────────────────────────────────────────────────┘

상태 A: Credit ≥ 0 → 큐에 프레임 있으면 즉시 전송
         프레임 전송 중: Credit 감소 (sendSlope)
상태 B: Credit < 0 → 전송 불가, 대기
         대기 중: Credit 증가 (idleSlope)
```

### CBS 파라미터
| 파라미터 | 설명 | 계산 |
|---------|------|------|
| idleSlope | 대기 시 크레딧 증가율 | 예약 대역폭 (bps) |
| sendSlope | 전송 시 크레딧 감소율 | idleSlope - 링크속도 |
| hiCredit | 최대 크레딧 | 버스트 크기 제한 |
| loCredit | 최소 크레딧 | -hiCredit (대칭) |

---

## TAS vs CBS 비교

| 특성 | TAS (802.1Qbv) | CBS (802.1Qav) |
|------|----------------|----------------|
| 동작 방식 | 시간 슬롯 기반 게이트 제어 | 크레딧 기반 대역폭 제어 |
| 지연 특성 | 결정론적 (< 1µs Jitter) | 확률론적 (< 2ms, 예측 가능) |
| 설정 복잡도 | 높음 (GCL, 시간 동기화 필수) | 낮음 (대역폭 % 설정) |
| 주요 용도 | 안전 제어, 정밀 제어 | 오디오/비디오, 센서 스트리밍 |
| 시간 동기화 | 802.1AS 필수 | 802.1AS 권장 (필수 아님) |
| 대역폭 보장 | 정확한 슬롯 | 평균 대역폭 보장 |

---

## AVB (Audio Video Bridging) 프로토콜 스택

CBS는 AVB 프로토콜 스택의 핵심입니다.

```
┌───────────────────────────────────────────────────────┐
│  Application                                          │
│  (오디오/비디오 스트리밍 앱)                           │
├───────────────────────────────────────────────────────┤
│  AVTP (Audio Video Transport Protocol)                │
│  IEEE 1722 - 오디오/비디오 데이터 캡슐화              │
├───────────────────────────────────────────────────────┤
│  SRP (Stream Reservation Protocol)                    │
│  IEEE 802.1Qat - 대역폭 예약                          │
├───────────────────────────────────────────────────────┤
│  CBS (Credit-Based Shaper)                            │
│  IEEE 802.1Qav - 트래픽 쉐이핑                        │
├───────────────────────────────────────────────────────┤
│  gPTP (IEEE 802.1AS)                                  │
│  시간 동기화                                           │
├───────────────────────────────────────────────────────┤
│  Ethernet (L2)                                        │
└───────────────────────────────────────────────────────┘
```

---

## SRP (Stream Reservation Protocol) - 대역폭 예약

CBS 트래픽을 위해 네트워크 경로의 대역폭을 사전 예약합니다.

```
Talker (스트리머)               Network                  Listener (수신자)
      │                           │                          │
      │──── Advertise ────────────►│───────────────────────►│
      │    (스트림 요청, 대역폭 정보│                          │
      │     Latency Class 정보)    │                          │
      │                           │                          │
      │◄─── Ready ─────────────────│◄──────────────────────  │
      │    (대역폭 예약 성공)       │   (Listener가 Accept)    │
      │                           │                          │
      │──── AVTP 데이터 스트림 ────►│──────────────────────►│
           (CBS로 쉐이핑)

SRP 메시지: MSRP (Multiple Stream Registration Protocol)
포트: 이더넷 멀티캐스트 (01:80:C2:00:00:E)
```

---

## Linux에서 CBS 설정

```bash
# CBS qdisc 설정 (eth0, Class A 스트림)
# Class A: 2ms 지연, 75% 대역폭 최대
# Class B: 50ms 지연, 75% 대역폭 최대

# 1. MQ (Multi-Queue) qdisc 설정
tc qdisc add dev eth0 root handle 1: mqprio \
    num_tc 3 \
    map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 \
    queues 1@0 1@1 2@2 \
    hw 0

# 2. CBS qdisc 추가 (Class A, TC0 = 높은 우선순위)
tc qdisc replace dev eth0 parent 1:1 cbs \
    idleslope 49152 \     # 49.152 Mbps (예약 대역폭)
    sendslope -950848 \   # 1000M - 49152 = 950848 감소율
    hicredit 348 \        # 최대 크레딧
    locredit -1392 \      # 최소 크레딧
    offload 0             # 소프트웨어 모드

# 3. CBS qdisc 추가 (Class B, TC1)
tc qdisc replace dev eth0 parent 1:2 cbs \
    idleslope 9216 \      # 9.216 Mbps
    sendslope -990784 \
    hicredit 56 \
    locredit -224 \
    offload 0

# 설정 확인
tc qdisc show dev eth0
```

### CBS 파라미터 계산 방법
```python
# CBS 파라미터 계산 예시
def calc_cbs_params(link_rate_bps, reserved_bps, max_frame_size_bytes):
    """
    link_rate_bps: 링크 속도 (예: 1_000_000_000 = 1Gbps)
    reserved_bps: 예약 대역폭 (예: 100_000_000 = 100Mbps)
    max_frame_size_bytes: 최대 프레임 크기 (예: 1518)
    """
    idle_slope = reserved_bps   # idleSlope = 예약 대역폭
    send_slope = reserved_bps - link_rate_bps  # 음수

    # hiCredit: 비CBS 프레임 최대 전송 중 대기하는 동안 충전되는 크레딧
    # (최악 경우: 비CBS 프레임 1개가 전송 중일 때)
    non_cbs_frame_transmission_time = (max_frame_size_bytes * 8) / link_rate_bps
    hi_credit = idle_slope * non_cbs_frame_transmission_time

    # loCredit: CBS 프레임 전송 중 소비되는 최대 크레딧
    cbs_frame_transmission_time = (max_frame_size_bytes * 8) / link_rate_bps
    lo_credit = send_slope * cbs_frame_transmission_time

    print(f"idleSlope: {idle_slope / 1e6:.3f} Mbps")
    print(f"sendSlope: {send_slope / 1e6:.3f} Mbps")
    print(f"hiCredit:  {hi_credit * 1e6:.0f} bits (약 {hi_credit*1e9:.0f} ns)")
    print(f"loCredit:  {lo_credit * 1e6:.0f} bits")

calc_cbs_params(1_000_000_000, 100_000_000, 1518)
# 출력:
# idleSlope: 100.000 Mbps
# sendSlope: -900.000 Mbps
# hiCredit:  12144 bits (약 12144 ns)
# loCredit:  -13662 bits
```

---

## 의료 로봇에서의 CBS 활용

```
수술 로봇 CBS 트래픽 구성:

Class A (2ms 지연 보장):
  - 조인트 위치 피드백 스트리밍 (500Hz, 10Mbps)
  - 힘/토크 센서 데이터 (1kHz, 5Mbps)
  예약 대역폭: 15Mbps

Class B (50ms 지연 허용):
  - 내시경 오디오 스트리밍
  - 환자 모니터링 데이터
  예약 대역폭: 5Mbps

Best Effort:
  - 내시경 영상 (UDP, 적응형 비트레이트)
  - 진단/OTA 트래픽

TSN 스위치 CBS 설정:
  Class A: idleSlope = 15Mbps, hiCredit 계산
  Class B: idleSlope = 5Mbps, hiCredit 계산
  나머지: Best Effort (약 980Mbps)

CBS + VLAN 조합:
  VLAN 10, PCP=5 → Class A CBS 처리
  VLAN 20, PCP=3 → Class B CBS 처리
  VLAN 30, PCP=0 → Best Effort
```

---

## Reference
- [IEEE 802.1Qav-2009 - Forwarding and Queuing for Time-Sensitive Streams](https://standards.ieee.org/ieee/802.1Qav/3684/)
- [IEEE 1722-2016 - AVTP (Audio Video Transport Protocol)](https://standards.ieee.org/ieee/1722/5971/)
- [Linux tc-cbs man page](https://man7.org/linux/man-pages/man8/tc-cbs.8.html)
- [Avnu Alliance - AVB/TSN](https://avnu.org/)
