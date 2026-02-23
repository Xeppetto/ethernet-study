# 흐름 제어와 버퍼 관리 (Flow Control & Buffer Management)

## 개요

Full Duplex Ethernet 환경에서도 여러 입력 포트의 트래픽이 하나의 출력 포트로 집중될 때 **혼잡(Congestion)**이 발생합니다. 혼잡은 패킷 손실, 지연 폭증, 지연 변동(Jitter)을 일으킵니다. 의료 로봇의 실시간 제어 루프에서 이는 직접적인 **기능 안전 위협**입니다.

이 문서는 혼잡 제어를 위한 흐름 제어 메커니즘, 큐 관리 알고리즘, 그리고 TSN이 이 문제를 어떻게 근본적으로 해결하는지를 다룹니다.

---

## 1. 혼잡 발생 원인과 영향

### 1.1 혼잡 발생 시나리오

```
[입력 집중(Ingress Aggregation) 문제]

포트 1 (1Gbps) ─────────────────────────────────────►
포트 2 (1Gbps) ──────────────────────────────────────►  출력 포트 (1Gbps)
포트 3 (1Gbps) ──────────────────────────────────────►  ← 3배 과부하!
                                                         출력 버퍼 포화 →
                                                         패킷 손실 발생

[구독 과부하(Oversubscription) 예시]
N개 입력 포트 × 1Gbps → 1개 출력 포트 1Gbps
  → N = 2 이상이면 구조적 혼잡 가능
  → 실무: 스위치 백플레인 용량이 전체 포트 용량 합보다 작을 경우 발생
```

### 1.2 혼잡의 영향

```
혼잡 시나리오별 영향:

1. 버퍼 포화 → Tail Drop (꼬리 폐기)
   - 버퍼 가득 차면 이후 모든 패킷 폐기
   - TCP: 재전송 → 혼잡 윈도우 축소 → 처리량 급감
   - UDP(제어 데이터): 패킷 유실 → 제어 루프 파손

2. 버퍼링 증가 → Latency 폭증
   - 10,000 Byte 버퍼 @ 1Gbps: 최대 80 µs 추가 지연
   - 1MB 버퍼 @ 1Gbps: 최대 8 ms 추가 지연 (Bufferbloat!)
   - 의료 로봇 1ms 제어 사이클에 치명적

3. 지연 변동(Jitter) 발생
   - 같은 트래픽 흐름도 버퍼 상태에 따라 지연이 달라짐
   - 서보 제어기: Jitter > 수십 µs → 진동/불안정

4. HOL (Head-of-Line) Blocking
   - 큐 앞의 대형 프레임이 뒤의 소형 긴급 프레임을 블로킹
```

---

## 2. IEEE 802.3x PAUSE 프레임 (Layer 2 흐름 제어)

### 2.1 동작 원리

수신 측이 버퍼 포화 임박 시 송신 측에게 전송 중지를 요청하는 메커니즘입니다.

```
수신 장치 (버퍼 80% 초과)              송신 장치
        │                                    │
        │──── PAUSE Frame ──────────────────►│
        │     Dst: 01:80:C2:00:00:01         │
        │     EtherType: 0x8808              │  ← 전송 일시 중지
        │     Opcode: 0x0001                 │  (quanta × 512 bit time)
        │     Pause Time: 0x0100 (256 quanta)│
        │                                    │
        │                    (256 quanta 후) │
        │◄─── PAUSE Frame (time=0) ──────────│
        │     (또는 자동 재개)               │  ← 전송 재개
```

**PAUSE 프레임 구조:**
```
Dst MAC:    01:80:C2:00:00:01 (MAC Control 멀티캐스트)
Src MAC:    발신 장치 MAC
EtherType:  0x8808 (Ethernet Flow Control)
Opcode:     0x0001 (PAUSE)
Pause Time: 0x0000 ~ 0xFFFF (Quanta 단위)
            1 Quanta = 512 bit time
            1Gbps에서 1 Quanta = 512 / 1G = 512 ns
            최대 65535 Quanta = ~33 ms

Pad:        42 Byte (64 Byte 최소 프레임 충족)
```

### 2.2 PAUSE의 근본적 문제점

```
문제 1: 모든 트래픽 일괄 차단 (무차별 블로킹)
  ┌─────────────────────────────────────────────────────────────┐
  │  링크 전체가 멈춤:                                           │
  │  긴급 안전 명령 ────────────────────────────────── 차단됨! │
  │  일반 데이터   ────────────────────────────────── 차단됨   │
  │  진단 데이터   ────────────────────────────────── 차단됨   │
  │  (모든 트래픽 클래스 구분 없이 중지)                         │
  └─────────────────────────────────────────────────────────────┘

문제 2: HOL (Head-of-Line) Blocking 악화
  수신 측이 PAUSE 전송 → 송신 측 전체 멈춤
  → 우선순위 큐 효과 무력화

문제 3: 데드락 위험
  A가 B에게 PAUSE → B도 A에게 PAUSE
  → 양방향 교착 상태 발생 가능

→ 결론: IEEE 802.3x PAUSE는 TSN 및 시간 민감 트래픽 환경에 부적합
```

---

## 3. PFC (Priority-based Flow Control) - IEEE 802.1Qbb

### 3.1 개념

PAUSE의 단점을 해결하기 위해 **우선순위별로 독립적인 흐름 제어**를 제공합니다.

```
PFC 없을 때 (PAUSE):        PFC 사용 시 (802.1Qbb):
  모든 트래픽 ─→ 차단         PCP 5 (안전 제어) ─→ 계속 전송 ✓
                              PCP 4 (실시간)    ─→ 계속 전송 ✓
                              PCP 2 (영상)      ─→ 일시 정지 (버퍼 포화)
                              PCP 0 (Best Effort) → 정지
```

### 3.2 PFC 프레임 구조

```
Dst MAC:    01:80:C2:00:00:01
EtherType:  0x8808
Opcode:     0x0101 (PFC, PAUSE Opcode와 다름)
Enable:     8비트 마스크 (우선순위 0~7 중 제어 대상)
Pause Time: 우선순위별 8개 × 2 Byte = 16 Byte
            예: Priority 2 = 0x0100 (256 quanta 정지)
                Priority 5 = 0x0000 (정지 해제)
```

### 3.3 PFC의 한계와 TSN으로의 전환

```
PFC가 해결하는 것:
  ✓ 우선순위별 독립 흐름 제어 (안전 트래픽 보호)
  ✓ 무차별 블로킹 방지

PFC가 해결 못하는 것:
  ✗ 확정적 지연 보장 불가 (반응형 제어 특성)
  ✗ PAUSE 전파 지연으로 여전히 순간적 패킷 손실 가능
  ✗ HOL Blocking 완전 해결 불가
  ✗ 사전에 계산된 타임 슬롯 보장 불가

TSN의 접근법: 사전 계획(Proactive) 기반 스케줄링
  → TAS(802.1Qbv): 시간 슬롯을 미리 할당하여 혼잡 자체를 방지
  → CBS(802.1Qav): 크레딧 기반 정형화로 버스트 억제
  → 흐름 제어 필요성 자체를 제거하는 방향
```

---

## 4. HOL (Head-of-Line) Blocking

### 4.1 발생 원리

```
단일 큐 시나리오 (FIFO):
                 출력 버퍼 (FIFO)
  시간 →      ┌──────────────────────────────────────┐
              │ [대형 프레임 9000B] [소형 64B] [소형 64B] │ → 출력
              └──────────────────────────────────────┘
                       ↑
               대형 프레임 전송 완료까지
               뒤의 소형 긴급 프레임은 대기

1Gbps에서 9000 Byte Jumbo Frame 전송 시간: ~72 µs
뒤의 64 Byte 제어 프레임: 72 µs 이상 대기 → HOL Blocking
```

### 4.2 다중 큐로 HOL Blocking 완화

```
다중 우선순위 큐 (8개 큐, 802.1p PCP 기반):
                 ┌─────────────────────────────────────┐
  PCP=7 큐  ────►│ [BPDU] [PTP]                        │→ 최우선 처리
  PCP=5 큐  ────►│ [안전명령 64B] [제어 202B]            │→ 우선 처리
  PCP=4 큐  ────►│ [실시간 데이터]                       │→ 2순위
  PCP=0 큐  ────►│ [진단] [로그] [대형 파일...]           │→ 최저 처리
                 └─────────────────────────────────────┘
                          ↓ 스케줄러
                       출력 포트
```

### 4.3 Frame Preemption으로 HOL Blocking 근본 해결

```
IEEE 802.1Qbu + IEEE 802.3br (Frame Preemption):

기존 (HOL Blocking):
  시간 ──►  [대형 Best-Effort 프레임 1518B]           [긴급 제어 64B]
                    72 µs 대기 후 전송 ──────────────────►

Frame Preemption 적용:
  시간 ──►  [대형 프레임 앞부분 124B] | [긴급 제어 64B] | [대형 프레임 나머지]
                선점 가능 (preemptable)   Express 프레임     중단 지점부터 재개
                                          (높은 우선순위)
mPacket 구조:
  선점 가능 프레임의 조각 = mPacket
  최소 mPacket: 60 Byte (Guard Band 포함)
  오버헤드: 6 Byte (CRC + mCRC) per fragment
```

---

## 5. 큐 스케줄링 알고리즘

### 5.1 Strict Priority (SP) - 엄격 우선순위

```
동작: 높은 우선순위 큐가 비어있을 때만 낮은 우선순위 큐 서비스

  PCP=7 ────► [BPDU/PTP] ─────────────────────────────────────► 항상 먼저
  PCP=5 ────► [안전 제어] ─── PCP=7 비어야 ───────────────────► 2순위
  PCP=0 ────► [Best Effort] ─── PCP=5, 7 모두 비어야 ─────────► 최저

장점:
  ✓ 구현 단순
  ✓ 긴급 트래픽 최우선 보장

단점:
  ✗ 낮은 우선순위 트래픽 기아(Starvation) 발생 가능
  ✗ 높은 우선순위 트래픽이 지속되면 낮은 우선순위는 영구 차단
```

### 5.2 WRR (Weighted Round Robin)

```
동작: 각 큐에 가중치(Weight) 부여 → 가중치 비율만큼 서비스

Weight 설정 예:
  PCP=5: Weight=8  → 전체 서비스의 8/(8+4+2+1) = 53%
  PCP=4: Weight=4  → 27%
  PCP=2: Weight=2  → 13%
  PCP=0: Weight=1  → 7%

  PCP=5 큐 ──► 8 프레임마다 서비스
  PCP=4 큐 ──► 4 프레임마다 서비스
  PCP=2 큐 ──► 2 프레임마다 서비스
  PCP=0 큐 ──► 1 프레임마다 서비스

장점:
  ✓ 낮은 우선순위도 최소 대역폭 보장 (Starvation 방지)

단점:
  ✗ 큰 프레임/작은 프레임 혼재 시 실제 비율 불균형
  ✗ 바이트 단위 공정성 미보장 (DWRR로 해결)
```

### 5.3 DWRR (Deficit Weighted Round Robin)

```
WRR의 바이트 단위 불공정 문제를 해결합니다.

동작:
  각 큐에 "크레딧(Quantum)"을 부여
  서비스 시 사용한 바이트만큼 크레딧 차감
  크레딧 0 이하: 다음 라운드까지 대기
  → 바이트 단위 공정한 대역폭 배분 실현

예:
  PCP=5: Quantum = 8000 Byte  (53% 비율)
  큰 프레임(9000B) 서비스 시: 크레딧 = -1000B → 다음 라운드에서 1000B 추가 서비스
```

### 5.4 SP + DWRR 하이브리드 (실무 권장)

```
TSN 미적용 실무 스위치 권장 구성:

PCP=7 ────► Strict Priority (제어 프레임, PTP)
PCP=5 ────► Strict Priority (안전 제어)
─────────────────────────────────────────────
PCP=4 ────► DWRR, Quantum=4000B
PCP=3 ────► DWRR, Quantum=3000B
PCP=2 ────► DWRR, Quantum=2000B
PCP=1 ────► DWRR, Quantum=1000B
PCP=0 ────► DWRR, Quantum=500B

설계 원칙:
  • SP 큐는 가능한 최소 트래픽만 (기아 위험)
  • SP 큐 총 대역폭 < 링크 용량의 30~40% 권장
  • DWRR 큐는 Quantum을 MTU보다 크게 설정 (최소 1 프레임 완전 전송 보장)
```

### 5.5 TSN TAS (Time-Aware Shaper, 802.1Qbv)

```
스케줄링 알고리즘의 최종 진화: 시간 기반 게이트 제어

각 큐에 게이트 개폐 스케줄(GCL: Gate Control List)을 설정:

시간 축 →
T=0      T=100µs   T=200µs   T=300µs  T=400µs (= T=0 반복)
│         │         │         │         │
│ PCP=5  ├─ OPEN ──┤ CLOSED  ├─ OPEN ──┤ ...
│ PCP=4  ├─ CLOSE─┤ OPEN ──┤─ CLOSE ─┤ ...
│ PCP=0  ├─ CLOSE─┤ CLOSE ─┤─ CLOSE ─┤ ...

결과:
  PCP=5 트래픽: T=0~100µs, T=200~300µs 구간에만 전송 가능 → 결정론적 보장
  다른 트래픽:  PCP=5 게이트 닫힌 시간에만 전송 허용

TSN의 혁신: 혼잡 자체를 사전 설계로 방지 → 흐름 제어 불필요
```

---

## 6. 버퍼 크기 설계

### 6.1 버퍼의 역할

```
버퍼가 너무 작으면:
  → 순간적 트래픽 버스트 시 즉시 패킷 손실
  → 특히 Store-and-Forward 스위치에서 수신 중 전송 완료 대기 불가

버퍼가 너무 크면 (Bufferbloat):
  → 지연 폭증: 1MB 버퍼 @ 1Gbps = 최대 8ms 추가 지연
  → TCP 혼잡 제어 오작동 (지연 기반 신호 왜곡)
  → TSN 타임 슬롯 계산 불일치
```

### 6.2 최소 버퍼 크기 계산

```
기본 원칙: 최대 프레임이 완전히 수신되는 동안 전송 준비 가능 크기

Store-and-Forward 스위치의 최소 입력 버퍼:
  최대 프레임 크기 × N (동시 입력 포트 수)
  = 1518 Byte × 포트 수

TSN 스위치에서 출력 큐 버퍼:
  Guard Band 기간 + 최대 프레임 서비스 시간 고려
  PCP=5 큐: 짧은 제어 프레임만 → 소형 버퍼 (예: 8KB)
  PCP=0 큐: Best Effort → 대형 버퍼 가능 (예: 512KB)
```

### 6.3 Linux 소켓 버퍼 설정

```bash
# 현재 소켓 버퍼 크기 확인
sysctl net.core.rmem_default    # 수신 기본 버퍼
sysctl net.core.wmem_default    # 송신 기본 버퍼
sysctl net.core.rmem_max        # 수신 최대 버퍼
sysctl net.core.wmem_max        # 송신 최대 버퍼

# 실시간 제어용 최적화 (소형 버퍼, 낮은 지연 우선)
sysctl -w net.core.rmem_default=131072   # 128KB (기본 212992)
sysctl -w net.core.wmem_default=131072

# NIC 링 버퍼 크기 확인
ethtool -g eth0
# Ring parameters for eth0:
#   RX:  1024  (현재)
#   TX:  1024  (현재)

# 실시간용 NIC 링 버퍼 크기 축소 (지연 감소)
ethtool -G eth0 rx 256 tx 256
# 주의: 너무 작으면 NIC 오버플로우 발생 → ethtool -S eth0 | grep miss
```

---

## 7. CBS (Credit-Based Shaper) - IEEE 802.1Qav

TSN의 비주기적 트래픽 제어를 위한 메커니즘입니다. AVB(Audio Video Bridging)에서 도입되었습니다.

### 7.1 동작 원리

```
크레딧(Credit) 개념:
  • 크레딧 > 0: 프레임 전송 허용
  • 크레딧 ≤ 0: 전송 대기 (Idle 상태라도 전송 금지)
  • 전송 중: 크레딧 빠르게 감소 (sendSlope)
  • 유휴 시: 크레딧 서서히 증가 (idleSlope = 허용 대역폭)

크레딧 변화 그래프:
           크레딧
            ▲  /←증가(idleSlope)
            │ /          ___
            │/ ←대기    /   ← 전송 대기
            ├──────────/─────── 시간
            │         │\ ← 전송 시작 (빠른 감소, sendSlope)
            │         │ \
```

### 7.2 CBS 파라미터 계산

```
설계 목표: 링크 대역폭의 최대 75% 할당 (나머지는 Best Effort 용)

CBS 큐 (PCP=4, 대역폭 40% 할당 예시, 1Gbps 링크):
  idleSlope  = 허용 대역폭     = 1Gbps × 40%  = 400 Mbps
  sendSlope  = 전송 시 감소율  = idleSlope - linkSpeed = 400M - 1000M = -600 Mbps
  hiCredit   = sendSlope × maxInterferenceSize / linkSpeed
             = 600M × 1518B × 8 / 1G = 7286 Byte (다른 CBS 큐 최대 프레임 고려)
  loCredit   = sendSlope × maxFrameSize / linkSpeed
             = 600M × 1518B × 8 / 1G = 7286 Byte

Linux tc-cbs 설정:
  tc qdisc add dev eth0 parent 1:4 handle 40: cbs \
    idleslope 400000 \     # 400 Kbps → 실제 kbps 단위
    sendslope -600000 \    # -600 Kbps
    hicredit 7286 \
    locredit -7286 \
    offload 1              # 하드웨어 CBS 사용
```

---

## 8. Linux tc (Traffic Control) 큐 설정 실습

### 8.1 다중 우선순위 큐 설정 (의료 로봇 예시)

```bash
# 기존 qdisc 제거
tc qdisc del dev eth0 root 2>/dev/null

# 루트 qdisc: MQPRIO (Multi-Queue Priority)
tc qdisc add dev eth0 root handle 1: mqprio \
  num_tc 4 \
  map 0 0 1 1 2 2 2 3 3 3 3 3 3 3 3 3 \
  queues 1@0 1@1 1@2 1@3 \
  hw 0

# 또는 prio qdisc (소프트웨어 기반)
tc qdisc add dev eth0 root handle 1: prio \
  bands 4 \
  priomap 3 3 2 2 1 1 0 0 3 3 3 3 3 3 3 3
#          PCP: 0 1 2 3 4 5 6 7 ...
# band 0 = 최고 우선순위 (PCP 6, 7)
# band 3 = 최저 우선순위 (PCP 0, 1)

# 큐별 대역폭 설정 (sfq: Stochastic Fair Queue)
tc qdisc add dev eth0 parent 1:1 handle 10: sfq perturb 10
tc qdisc add dev eth0 parent 1:4 handle 40: sfq perturb 10

# 설정 확인
tc qdisc show dev eth0
tc class show dev eth0
```

### 8.2 흐름 제어 모니터링

```bash
# 큐 폐기 통계 확인 (패킷 손실 지표)
tc -s qdisc show dev eth0 | grep -A 5 "Sent"
# Sent 1234567 bytes 89012 pkt (dropped 123, overlimits 456 requeues 0)
#                               ↑ 폐기 수 → 0이어야 정상 (실시간 시스템)

# PAUSE 프레임 통계 (NIC 지원 시)
ethtool -S eth0 | grep -E "pause|flow_control"
# rx_pause_frames: 수신한 PAUSE 요청 수
# tx_pause_frames: 전송한 PAUSE 요청 수

# PFC 통계 (PFC 지원 NIC)
ethtool -S eth0 | grep -E "pfc|priority"
# rx_pfc_frames_priority5: PCP=5 우선순위 PFC 수신 수

# 실시간 큐 길이 모니터링
watch -n 0.5 'tc -s qdisc show dev eth0 | grep -E "backlog|dropped"'
```

---

## 9. 의료 로봇 시스템 흐름 제어 설계 가이드라인

### 9.1 계층별 흐름 제어 전략

```
의료 로봇 네트워크 흐름 제어 계층:

[하드웨어/PHY 계층]
  • Auto-Negotiation: 항상 ON (속도/Duplex 자동 맞춤)
  • PAUSE 프레임: 비활성화 (의료 제어 네트워크에서 금지)
  • EEE(Energy Efficient Ethernet): 반드시 비활성화 (웨이크업 지연 위험)

[L2 스위치 계층]
  • TSN 스위치: TAS(802.1Qbv) + PSFP(802.1Qci)로 사전 혼잡 방지
  • Non-TSN 스위치: SP + DWRR 혼합, PAUSE 비활성화
  • 포트 격리: 안전 VLAN(PCP=5 이상)과 Best Effort 완전 분리

[소프트웨어/애플리케이션 계층]
  • 실시간 제어 루프: UDP + 애플리케이션 수준 시퀀스 번호
  • 타임아웃: 2~3 사이클 이내 패킷 미수신 시 Fail-safe 동작
  • 버퍼: 1~2 프레임 수준의 최소 버퍼 (지연 최소화)
```

### 9.2 흐름 제어 적용 금지 사항 (의료기기 안전)

```
❌ 금지 사항:
  1. 안전 제어 트래픽에 TCP 사용
     → 재전송이 제어 사이클 타임아웃 초과 가능
     → 혼잡 윈도우 감소 → 처리량 예측 불가

  2. 안전 링크에 PAUSE 프레임 활성화
     → 단일 노드 장애가 전체 안전망 블로킹 가능

  3. 안전 VLAN에 Best Effort 트래픽 혼재
     → 설계 시 VLAN 분리 철저히 할 것

  4. 하나의 물리 링크에 안전 + 비안전 트래픽 공유 (SIL 3 이상)
     → 완전 물리 분리 (separate NIC, separate switch) 권장

✓ 권장 사항:
  1. 안전 제어: UDP + 시퀀스 번호 + 타임아웃 기반 감시
  2. 실시간망: TSN 스위치 + TAS 스케줄링
  3. 트래픽 격리: VLAN + 물리적 분리 조합
  4. 모니터링: 주기적 큐 깊이 및 패킷 손실 감시
```

---

## Reference
- [IEEE 802.3x - MAC Control (PAUSE)](https://standards.ieee.org/ieee/802.3/10422/)
- [IEEE 802.1Qbb-2011 - Priority-based Flow Control (PFC)](https://standards.ieee.org/ieee/802.1Qbb/4345/)
- [IEEE 802.1Qav-2009 - Forwarding and Queuing for Time-Sensitive Streams (CBS)](https://standards.ieee.org/ieee/802.1Qav/3982/)
- [IEEE 802.1Qbv-2015 - Enhancements for Scheduled Traffic (TAS)](https://standards.ieee.org/ieee/802.1Qbv/6068/)
- [IEEE 802.1Qbu-2016 - Frame Preemption](https://standards.ieee.org/ieee/802.1Qbu/6069/)
- [Linux Traffic Control HOWTO](https://tldp.org/HOWTO/Traffic-Control-HOWTO/)
- [Linux tc-cbs man page](https://man7.org/linux/man-pages/man8/tc-cbs.8.html)
- [Bufferbloat - Jim Gettys](https://www.bufferbloat.net/projects/)
