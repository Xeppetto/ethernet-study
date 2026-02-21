# IEEE 802.1Qbv (TAS: Time Aware Shaper)

## 개요
일반 Ethernet은 Best Effort 방식으로 트래픽이 몰리면 지연이 발생합니다. IEEE 802.1Qbv TAS(Time Aware Shaper)는 **시간 기반 스케줄링**으로 각 트래픽 종류에 전용 시간 슬롯을 할당하여 **결정론적 지연(Deterministic Latency)**을 보장합니다.

전제 조건: IEEE 802.1AS 시간 동기화가 먼저 구성되어 있어야 합니다.

---

## 기본 개념

### 우선순위 큐 (Priority Queue)
Ethernet 스위치는 출력 포트마다 8개의 우선순위 큐(TC0~TC7)를 가집니다.

```
입력 프레임 → [분류] → 큐 TC7 (최고 우선순위) ─┐
                    → 큐 TC6              ─┤
                    → 큐 TC5 (제어 트래픽) ─┤
                    → 큐 TC4              ─┤  출력 포트
                    → 큐 TC3              ─┤
                    → 큐 TC2              ─┤
                    → 큐 TC1              ─┤
                    → 큐 TC0 (Best Effort) ─┘
```

### TAS 게이트 제어

```
TAS 동작 원리:
각 큐마다 게이트(Gate)가 있어 열림(Open)/닫힘(Closed) 상태 제어

시간 →  T=0   100µs  1ms   1.1ms  2ms
                │      │      │      │
TC7(제어) ─ OPEN─────CLOSE──OPEN──CLOSE──
TC0(BE)   ─ CLOSE────OPEN───CLOSE─OPEN──

→ TC7은 T=0~100µs 구간만 전송 가능 (정확한 시간 보장)
→ TC0은 TC7이 닫혀 있을 때만 전송 가능 (잔여 대역폭 사용)
```

---

## GCL (Gate Control List)

TAS의 핵심 설정 테이블입니다.

```
GCL 예시 (Cycle Time: 1ms):
┌─────────────────┬──────────────────┬──────────────────────────────────┐
│  Gate States    │  Duration        │  설명                            │
│  TC7~TC0 (8bit) │  (nanoseconds)   │                                  │
├─────────────────┼──────────────────┼──────────────────────────────────┤
│  1000 0000 (0x80)│  100,000 ns     │  TC7(제어) 전용 100µs            │
│  0111 1111 (0x7F)│  875,000 ns     │  TC6~TC0 나머지 875µs            │
│  1000 0000 (0x80)│  25,000 ns      │  Guard Band (TC7만 허용, 실제 무전송)│
└─────────────────┴──────────────────┴──────────────────────────────────┘
Total: 100µs + 875µs + 25µs = 1000µs (= 1ms Cycle)
```

### Guard Band (가드 밴드)
Best Effort 프레임이 Scheduled Traffic 시간대로 침범하는 것을 방지합니다.

```
Guard Band 없을 때 문제:
  TC0 프레임(1500 Byte, 1Gbps에서 12µs) 전송 중 TC7 게이트 열림
  → 12µs 만큼 TC7 지연 발생

Guard Band 적용:
  TC7 게이트 열리기 12µs 전에 TC0 게이트 닫음
  → TC7 항상 정확한 시간에 전송
```

---

## Linux tc taprio로 TAS 구현

```bash
# taprio qdisc 설정 (1ms 주기, TC7=100µs 제어, TC0=900µs BE)
tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1600000000000000000 \
    sched-entry S 0x80 100000 \
    sched-entry S 0x7F 875000 \
    sched-entry S 0x80 25000 \
    clockid CLOCK_TAI \
    flags 0x2

# 설정 확인
tc qdisc show dev eth0

# GCL 상태 확인
tc -s qdisc show dev eth0
```

### 소켓 우선순위 설정
```c
/* 애플리케이션에서 소켓 우선순위 설정 */
int priority = 7;  /* TC7 매핑 */
setsockopt(sockfd, SOL_SOCKET, SO_PRIORITY, &priority, sizeof(priority));

/* VLAN PCP와 TC 매핑 확인 */
/* /proc/net/vlan/eth0.10 의 egress map 참조 */
```

---

## TAS 적용 전후 지연 비교

```
TAS 없는 경우 (Best Effort):
  제어 명령 전송 요청
      │
      ▼
  1500 Byte 영상 데이터 전송 중 (12µs @ 1Gbps)
      │
      ▼ (+12µs 대기)
  다음 영상 데이터 전송 중 (또 12µs 대기)
      │
  결과: 수십µs ~ 수ms 불규칙 지연

TAS 적용:
  제어 명령 전송 요청
      │
      ▼
  TC7 게이트 열림 시각까지 대기 (최대 900µs = 1주기 내)
      │
      ▼ (정확히 TC7 슬롯에)
  제어 명령 전송 완료
  결과: 최대 지연 = 1주기(1ms) + 전송 시간, Jitter < 1µs
```

---

## 대역폭 계산

```python
# TAS 대역폭 할당 계산 예시
cycle_time_us = 1000  # 1ms
tc7_slot_us = 100     # 100µs

# TC7 최대 프레임 크기 계산 (1Gbps)
max_frame_bytes = (tc7_slot_us * 1e-6) * (1e9 / 8)  # 12,500 bytes
# → 1Gbps 링크에서 100µs = 12,500 Byte 전송 가능

# 실제 Ethernet 프레임 최대 크기: 1518 Byte
# → 1주기 100µs에 약 8개의 표준 프레임 전송 가능

# TC7 유효 대역폭
tc7_bandwidth_mbps = (tc7_slot_us / cycle_time_us) * 1000  # 100 Mbps
print(f"TC7 대역폭: {tc7_bandwidth_mbps} Mbps")
print(f"TC0 대역폭: {(1 - tc7_slot_us/cycle_time_us) * 1000} Mbps")
```

---

## 의료 로봇 TAS 설정 예시

```
수술 로봇 TSN 스위치 GCL 설계:
Cycle Time: 2ms (500Hz 제어 주기)

Entry 1: [TC7 OPEN]  50µs   → 안전 알람 전송 (ASIL-D, 최우선)
Entry 2: [TC5 OPEN]  150µs  → 관절 제어 명령/피드백 (ASIL-B)
Entry 3: [TC3 OPEN]  300µs  → 센서 데이터 (힘, 위치)
Entry 4: [TC0 OPEN]  1500µs → 영상, 진단, OTA (Best Effort)
                    ────────
Total:              2000µs = 2ms Cycle

보장되는 지연:
  안전 알람: < 2ms (1 Cycle 내 전송 보장)
  제어 명령: < 2ms
  센서 데이터: < 2ms
  영상 트래픽: Best Effort (허용 범위 내)
```

---

## Wireshark로 TAS 검증

```bash
# 트래픽 스케줄링 검증용 필터
# 1. VLAN 우선순위 확인
vlan.priority == 7

# 2. 제어 트래픽 타임스탬프 분석
# Wireshark → Statistics → I/O Graphs
# → Delta Time 열로 프레임 간 간격 측정

# 3. 지터 측정 (tcpdump + 분석)
tcpdump -i eth0 -w capture.pcap vlan and vlan pcp 7
# → 이후 Python/pandas로 타임스탬프 분포 분석
```

---

## Reference
- [IEEE 802.1Qbv-2015 - Enhancements for Scheduled Traffic](https://standards.ieee.org/ieee/802.1Qbv/5742/)
- [Linux tc-taprio man page](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
- [TSN Task Group - Time Aware Shaper](https://1.ieee802.org/tsn/802-1qbv/)
- [Avnu Alliance - Automotive TSN Profile](https://avnu.org/automotive/)
