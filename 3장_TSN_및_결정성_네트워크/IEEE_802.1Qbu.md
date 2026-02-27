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
BE 전송 중 ──────────────────►│중단. 낭비             │제어 전송

1Gbps에서 12µs = 1518 Byte 프레임 1개 손실 대역폭
가드 밴드가 많을수록 실효 대역폭 감소
```

### Frame Preemption 해결책

```
Frame Preemption 적용:

시간 ──────────────────────────────────────────────────────►
      │←────────────── 100µs 제어 슬롯 ─────────────►│
      │                                              │
BE ─────────────►│중단!│제어 전송 완료│BE 이어서 전송─►

가드 밴드 불필요: BE 프레임 즉시 중단 → 제어 즉시 전송
→ 가드 밴드 낭비 없음 → 대역폭 효율 향상
→ 지연 단축: 12µs → 0.5µs (최소 조각 기준)
```

---

## Express MAC (eMAC) vs Preemptable MAC (pMAC)

```
eMAC (Express MAC):
  - 선점 불가, 항상 완전 전송
  - 일반적으로 높은 우선순위 (TC4~TC7)
  - TAS로 스케줄된 제어 트래픽
  - 표준 Ethernet SFD(0xD5) 사용

pMAC (Preemptable MAC):
  - 선점 가능 (전송 중 중단됨)
  - 일반적으로 낮은 우선순위 (TC0~TC3)
  - Best Effort, 영상 트래픽
  - mPacket 형식 사용
```

---

## 프레임 선점 동작 원리 (802.3br mPacket)

### SMD (Start mPacket Delimiter) 값

| SMD 코드 | 16진수 값 | 의미 |
|---------|---------|------|
| SMD-S | `0x7C` | 선점 가능 프레임 시작 (Initial mPacket) |
| SMD-C0 | `0x61` | 연속 조각 (FragCount = 0) |
| SMD-C1 | `0x52` | 연속 조각 (FragCount = 1) |
| SMD-C2 | `0x9E` | 연속 조각 (FragCount = 2) |
| SMD-C3 | `0x2A` | 연속 조각 (FragCount = 3) |
| SMD-V | `0x07` | 검증 요청 (Verify) |
| SMD-R | `0x19` | 검증 응답 (Respond) |

FragCount는 0~3으로 순환합니다 (조각 번호 추적용).

### 선점 과정

```
① pMAC 프레임 전송 중 (최소 64 Byte = 512bit 전송 후 선점 가능)

일반 Ethernet:   [Preamble 7B][SFD 1B][DST 6B][SRC 6B][Type 2B][Data...][FCS 4B]

pMAC Initial:    [Preamble 7B][SMD-S 1B][DST 6B][SRC 6B][Type 2B][Data...][mCRC 4B]
                                ↑ 0x7C

② eMAC 프레임 도착 → 즉시 선점 신호 → pMAC 전송 중단
   중단 위치에서 mCRC 계산하여 첨부 → 조각 완성

③ eMAC 프레임 완전 전송
   [Preamble 7B][SFD 0xD5][DST][SRC][Type][Data][FCS]

④ pMAC 조각 이어서 전송 (Continuation mPacket)
   [Preamble 7B][SMD-Cx 1B][Data 나머지][FCS 4B]
                  ↑ 0x61/0x52/0x9E/0x2A (FragCount 기반)

수신 측:
  SMD 값 검사 → 조각 인식 → 재조립 → 원본 프레임 복원
  mCRC로 각 조각 무결성 확인
```

### 검증 핸드셰이크 (Preemption 활성화 전)

```
선점 기능을 활성화하기 전 수신 측이 지원 여부를 확인합니다.

송신 측                          수신 측
    │                               │
    │── [SMD-V (0x07) 검증 요청] ──►│
    │                               │ (preemption 지원 확인)
    │◄── [SMD-R (0x19) 검증 응답] ──│
    │                               │
    │  (응답 수신 시 선점 활성화)   │
    │── pMAC 선점 가능 프레임 전송 ─►│

LLDP 기반 능력 광고:
  송신 측이 LLDP TLV로 Frame Preemption 지원 능력 알림
  IEEE 802.3br TLV:
    addFragSize: 최소 조각 크기 (0=64B, 1=128B, 2=192B, 3=256B)
    verifyDisableTx: 검증 비활성화 여부
    verifyTime: 검증 응답 대기 시간 (ms)
```

---

## Frame Preemption 장점 수치화

```python
def calc_preemption_benefit(link_gbps=1, max_be_frame_bytes=1518, min_frag_bytes=64):
    """
    Frame Preemption으로 단축되는 최대 지연 계산
    """
    link_bps = link_gbps * 1e9

    # TAS 가드 밴드 크기 (Preemption 없을 때)
    guard_band_us = (max_be_frame_bytes * 8) / link_bps * 1e6

    # Preemption 후 최대 인터럽션 (최소 조각 전송 시간)
    min_frag_us = (min_frag_bytes * 8) / link_bps * 1e6

    print(f"링크 속도: {link_gbps}Gbps")
    print(f"가드 밴드 (Preemption 없음): {guard_band_us:.2f}µs")
    print(f"최대 인터럽션 (Preemption 적용): {min_frag_us:.2f}µs")
    print(f"지연 감소: {guard_band_us - min_frag_us:.2f}µs 절약")
    print(f"대역폭 회복: 1ms 사이클에서 {guard_band_us/1000*100:.1f}% → {min_frag_us/1000*100:.1f}%")

calc_preemption_benefit(1, 1518, 64)
# 출력:
# 링크 속도: 1Gbps
# 가드 밴드 (Preemption 없음): 12.14µs
# 최대 인터럽션 (Preemption 적용): 0.51µs
# 지연 감소: 11.63µs 절약
# 대역폭 회복: 1ms 사이클에서 1.2% → 0.05%
```

---

## Linux에서 Frame Preemption 설정

Frame Preemption은 하드웨어(PHY/MAC) 지원이 필요합니다.

```bash
# Frame Preemption 지원 확인
ethtool --show-frame-preemption eth0
# frame-preemption: enabled
# queues preemptible:  0xf  (TC0~TC3)
# queues express:      0xf0 (TC4~TC7)
# min-frag-size: 128

# Frame Preemption 설정
# preemptible-queues-mask: 어떤 TC를 선점 가능으로 설정
ethtool --set-frame-preemption eth0 \
    preemptible-queues-mask 0x0f  # TC0~TC3를 pMAC으로
# 결과: TC0~TC3 = Preemptable, TC4~TC7 = Express

# 최소 조각 크기 설정 (64 or 128 or 192 or 256)
ethtool --set-frame-preemption eth0 min-frag-size 64

# 검증 비활성화 (상대방 지원 확인 없이 선점 활성화)
ethtool --set-frame-preemption eth0 disable-verify on

# 최종 설정 확인
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

taprio + frame-preemption 조합:
  tc qdisc add dev eth0 ... taprio ... flags 0x2
  ethtool --set-frame-preemption eth0 preemptible-queues-mask 0x0f
```

---

## 지원 하드웨어

| 제품 | 제조사 | Preemption 지원 |
|------|--------|---------------|
| KSZ9563 | Microchip | 802.3br 지원 |
| SJA1110 | NXP | 완전 TSN + Preemption |
| I210/I225 | Intel | Preemption 지원 |
| RTL9031/9071 | Realtek | Preemption 지원 |
| TJA1103 | NXP | 100BASE-T1 Preemption |

---

## Reference
- [IEEE 802.1Qbu-2016 - Frame Preemption](https://standards.ieee.org/ieee/802.1Qbu/5954/)
- [IEEE 802.3br-2016 - Interspersing Express Traffic](https://standards.ieee.org/ieee/802.3br/5953/)
- [Linux ethtool Frame Preemption](https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html)
- [TSN Task Group - Frame Preemption](https://1.ieee802.org/tsn/802-1qbu/)
