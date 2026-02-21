# IEEE 802.1Qbu (Frame Preemption)

## 개요
IEEE 802.1Qbu는 **프레임 선점(Frame Preemption)** 기능을 정의합니다. 긴 프레임이 전송 중일 때 중요한 프레임이 도착하면, 진행 중인 프레임 전송을 **중단(Preempt)**하고 중요 프레임을 먼저 전송한 후 다시 이어서 전송합니다.

물리 계층 표준 IEEE 802.3br(Interspersing Express Traffic)과 함께 동작합니다.

---

## 왜 Frame Preemption이 필요한가?

### 가드 밴드(Guard Band) 문제
TAS(802.1Qbv)만 사용하는 경우, Best Effort 프레임이 전송 중일 때 제어 트래픽 슬롯이 시작되면 **가드 밴드 동안 아무것도 전송할 수 없습니다**.

```
TAS만 사용 (가드 밴드 낭비):

시간 ──────────────────────────────────────────────────────►
      │←─────── 100µs 제어 슬롯 ───────────────────►│
      │←── 가드밴드(12µs) ──►│                       │
      │                      │                       │
BE 전송 중 ──────────────────►중단. 가드밴드 동안 낭비 │제어 전송

1Gbps에서 12µs = 1500 Byte 프레임 1개 손실 대역폭
가드 밴드가 많을수록 실효 대역폭 감소
```

### Frame Preemption 해결책
```
Frame Preemption 적용:

시간 ──────────────────────────────────────────────────────►
      │←────────────── 100µs 제어 슬롯 ─────────────►│
      │                                              │
BE ─────────────►│중단!│제어 전송 완료│BE 이어서 전송─►

가드 밴드 불필요: BE 프레임을 중단하고 즉시 제어 전송
→ 가드 밴드 낭비 없음 → 대역폭 효율 향상
```

---

## 프레임 선점 동작 원리

### Express 프레임과 Preemptable 프레임

```
Express (eMAC): 선점 불가, 항상 우선 전송
  - 일반적으로 높은 우선순위 (TC4~TC7)
  - TAS로 스케줄된 제어 트래픽

Preemptable (pMAC): 선점 가능
  - 일반적으로 낮은 우선순위 (TC0~TC3)
  - Best Effort, 영상 트래픽
```

### 선점 과정 (802.3br mPacket)

```
pMAC 프레임 전송 중 Express 프레임 도착:

① pMAC 프레임 전송 중 (최소 64 Byte 전송 후 중단 가능)

프리앰블  SFD  mSFD  데이터...    중단!
───────── ─── ─────  ────────────── ↓

② Express 프레임 즉시 전송
    ┌──────────────────────────────┐
    │ Express 프레임 완전 전송      │
    └──────────────────────────────┘

③ pMAC 프레임 이어서 전송 (Continuation)
  - 특수 프리앰블(mSFD=0xD5)으로 조각임을 표시
  - CRC 재계산하여 전송

수신 측:
  조각들을 재조립 → 원본 pMAC 프레임 복원
```

### mPacket (Preemption Fragment) 구조
```
일반 Ethernet 프레임:
  [7B Preamble][1B SFD][DST MAC][SRC MAC][Type][Payload][4B FCS]

첫 조각 (Initial mPacket):
  [7B Preamble][1B mSFD(0x7C)][DST MAC][SRC MAC][Type][Payload 일부][4B mCRC]

중간/마지막 조각 (Continuation mPacket):
  [7B Preamble][1B mSFD(0xD5)][Payload 나머지][4B mCRC/FCS]

SMD(Start mPacket Delimiter):
  0x7C: Initial mPacket (첫 조각)
  0xD5: Continuation mPacket (이어지는 조각)
```

---

## Frame Preemption 장점 수치화

```python
# TAS 지연 개선 계산
def calc_preemption_benefit(link_gbps=1, max_frame_bytes=1518):
    """
    Frame Preemption으로 단축되는 최대 지연 계산
    """
    # 최대 선점 전 대기 시간 (가드 밴드 제거)
    max_frame_time_us = (max_frame_bytes * 8) / (link_gbps * 1e9) * 1e6

    print(f"링크 속도: {link_gbps}Gbps")
    print(f"최대 프레임 전송 시간: {max_frame_time_us:.2f} µs")
    print(f"\nTAS 가드 밴드 크기 (Preemption 없을 때):")
    print(f"  = 최대 프레임 전송 시간 = {max_frame_time_us:.2f} µs")
    print(f"\nFrame Preemption 적용 후 최대 지연:")
    min_preemptable_bytes = 64  # 최소 선점 가능 크기
    max_interruption_us = (min_preemptable_bytes * 8) / (link_gbps * 1e9) * 1e6
    print(f"  = 최소 preemptable 단편 = {max_interruption_us:.2f} µs")
    print(f"\n지연 감소: {max_frame_time_us - max_interruption_us:.2f} µs 절약")

calc_preemption_benefit(1, 1518)
# 출력:
# 링크 속도: 1Gbps
# 최대 프레임 전송 시간: 12.14 µs
# TAS 가드 밴드 크기: 12.14 µs
# Frame Preemption 후 최대 지연: 0.51 µs
# 지연 감소: 11.63 µs 절약
```

---

## Linux에서 Frame Preemption 설정

Frame Preemption은 하드웨어(PHY/MAC) 지원이 필요합니다.

```bash
# Frame Preemption 지원 확인
ethtool --show-frame-preemption eth0
# frame-preemption: enabled
# min-frag-size: 128

# Frame Preemption 설정
ethtool --set-frame-preemption eth0 preemptible-queues-mask 0x0f
# bit 0~3 = TC0~TC3 (Preemptable)
# bit 4~7 = TC4~TC7 (Express)

# 최소 조각 크기 설정
ethtool --set-frame-preemption eth0 min-frag-size 64

# 설정 확인
ethtool --show-frame-preemption eth0
# Preemptible queues: 0x0f (TC0~TC3)
# Express queues: 0xf0 (TC4~TC7)
# Min frag size: 64 bytes
```

---

## TAS + CBS + Preemption 조합 전략

```
최적 TSN 구성 (의료 로봇 권장):

TC7 (Priority 7) → Express + TAS GCL
  안전 E-Stop, 비상 알람
  지연: < 10µs (선점 즉시 전송)

TC5,6 (Priority 5,6) → Express + TAS GCL
  관절 제어 명령/피드백
  지연: < 100µs (TAS 슬롯 내)

TC3,4 (Priority 3,4) → Preemptable + CBS
  센서 스트리밍 (힘, 위치)
  지연: < 2ms (CBS 보장)

TC0,1,2 (Priority 0,1,2) → Preemptable + Best Effort
  영상, 진단, OTA
  Express 프레임에 의해 선점됨

결과:
  - TC7이 도착하면 TC0~TC4 즉시 선점 → 최소 지연
  - 가드 밴드 제거 → 대역폭 낭비 없음
  - CBS로 센서 스트리밍 대역폭 보장
```

---

## 지원 하드웨어

| 제품 | 제조사 | Preemption 지원 |
|------|--------|---------------|
| KSZ9563 | Microchip | 802.3br 지원 |
| SJA1110 | NXP | 완전 TSN + Preemption |
| i210/i225 | Intel | Preemption 지원 |
| RTL9031/9071 | Realtek | Preemption 지원 |
| TAS5970 | Texas Instruments | Preemption 지원 |

---

## Reference
- [IEEE 802.1Qbu-2016 - Frame Preemption](https://standards.ieee.org/ieee/802.1Qbu/5954/)
- [IEEE 802.3br-2016 - Interspersing Express Traffic](https://standards.ieee.org/ieee/802.3br/5953/)
- [Linux ethtool Frame Preemption](https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html)
- [TSN Task Group - Frame Preemption](https://1.ieee802.org/tsn/802-1qbu/)
