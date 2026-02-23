# 결정성 네트워크와 의료 안전 기초 (Determinism & Safety Basics)

## 개요

이 문서는 **1장(Ethernet 기초)에서 3장(TSN)으로**, 그리고 **5장(기능 안전)으로** 연결되는 핵심 브릿지입니다. Ethernet의 기본 개념을 이해했다면, 이제 "왜 일반 Ethernet으로는 의료 로봇을 안전하게 제어할 수 없는가?"라는 질문에 답해야 합니다.

---

## 1. 결정성(Determinism)이란?

### 1.1 정의

```
결정성 시스템 (Deterministic System):
  동일한 입력과 동일한 초기 조건에서
  항상 동일한 결과를 동일한 시간 안에 생성하는 시스템

비결정성 시스템 (Non-deterministic):
  출력이 정확하더라도 '언제' 나올지 보장 없음
```

### 1.2 네트워크 결정성의 세 가지 요건

```
1. 지연 결정성 (Latency Determinism)
   - 패킷 전달 지연의 최댓값(Worst-Case Execution Time)이 사전에 계산되고 보장됨
   - 예: "제어 명령은 항상 500 µs 이내에 도착한다"

2. 지터 결정성 (Jitter Determinism)
   - 지연 변동이 허용 범위 이내
   - 예: "지연 변동(Jitter) < ±10 µs"

3. 가용성 결정성 (Availability Determinism)
   - 지정된 시간 내에 네트워크가 항상 사용 가능
   - 예: "제어 사이클 1ms 중 네트워크 사용 불가 구간 < 5 µs"
```

---

## 2. Best-Effort Ethernet의 한계

### 2.1 일반 Ethernet의 비결정성 원인

```
원인 1: 큐잉 지연 (Queuing Delay) - 비예측
  트래픽이 몰리면 스위치 출력 버퍼에서 대기
  최솟값: 0 (혼잡 없을 때)
  최댓값: 버퍼 크기 / 링크 속도 (예: 1MB / 1Gbps = 8 ms)
  → 동일 조건에서 0 µs ~ 8 ms 범위 가능

원인 2: 충돌 재전송 - CSMA/CD (레거시, Full Duplex에서는 해결)
  Half Duplex 환경에서 충돌 시 랜덤 지연 후 재전송
  → 현대 Full Duplex 스위치에서는 해결됨, 하지만 이 한계는 중요한 역사적 배경

원인 3: HOL Blocking
  낮은 우선순위 대형 프레임이 긴급 소형 프레임 차단
  → 최대 1518 Byte @ 100Mbps = 121 µs 블로킹

원인 4: Broadcast/Multicast 폭풍
  네트워크 장애 시 Broadcast Storm → 정상 트래픽 전달 불가
  → 예측 불가능한 네트워크 마비

원인 5: 클록 비동기
  각 노드가 독립 클록 사용 → 수십 ~ 수백 µs 편차
  → 시간 기반 스케줄링(TAS) 적용 불가
```

### 2.2 실측 예시: Best-Effort vs TSN

```
실험 조건: 1Gbps 스위치, 배경 트래픽 70% 부하

Best-Effort Ethernet:
  최소 지연:   12 µs
  최대 지연: 8,200 µs  (8.2 ms!)
  평균 지연:  350 µs
  Jitter:   8,188 µs  → 제어 루프 불안정

TSN (TAS 적용):
  최소 지연:   14 µs
  최대 지연:   18 µs  (±2 µs 범위!)
  평균 지연:   15 µs
  Jitter:      4 µs  → 제어 루프 안정

→ 동일 네트워크 인프라에서 TSN이 결정성을 달성
```

---

## 3. 의료 로봇에서 비결정성의 안전 영향

### 3.1 실시간 제어 루프 구조

```
의료 로봇 제어 루프 (예: 수술 로봇):

마스터 콘솔 ──── Ethernet ──── 슬레이브 로봇 팔
     │                              │
  T=0: 위치 명령 전송               │
     │                              │
     │ ← 네트워크 지연 →            │
     │                              │
  T=δ: 명령 수신, 모터 제어         │
     │                              │
  T=δ+ε: 상태 피드백 전송 ─────────►│
     │                              │
  T=δ+ε+δ': 피드백 수신, 다음 명령 계산
     │
     └── 총 사이클: δ + ε + δ' < T_cycle (예: 1ms)

요구사항 예시 (로봇 수술 시스템):
  T_cycle        = 1 ms (1kHz 제어 루프)
  허용 네트워크 지연 = 200 µs (편도)
  허용 Jitter   = ±20 µs
  허용 패킷 손실 = 0 (또는 < 10^-6 BER)
```

### 3.2 네트워크 지연과 제어 안전성

```
시나리오: 관절 Force Torque 제어 (임피던스 제어)

정상 (지연 < 200 µs):
  시간 → ─────────────────────────────────────────────────
           명령 ──► 수신 ──► 모터 응답 ──► 피드백 ──► 보정
  결과: 부드럽고 안전한 힘 제어

지연 폭증 (지연 1000 µs, Jitter 심함):
  시간 → ─────────────────────────────────────────────────
           명령 ──────────────────────────► 수신 (늦음)
                 ↑ 이 사이에 모터는 잘못된 힘 인가
  결과:
    - 위치 오차 누적 → 조직 손상 위험
    - 힘 제어 불안정 → 진동, 경련
    - 비상 정지 오작동 (응답 없음으로 오인)
    - IEC 62304 SIL 레벨 위반
```

### 3.3 패킷 손실과 안전 시스템

```
제어 패킷 손실 시나리오:

방법 1: 마지막 명령 유지 (Hold-Last-Command)
  └─ 위험: 로봇이 잘못된 방향으로 계속 이동
  └─ 허용 시간: 일반적으로 < 2~3 사이클 (2~3ms)

방법 2: 안전 상태로 전환 (Fail-Safe)
  └─ 즉시 모터 토크 0 (또는 안전 위치 복귀)
  └─ IEC 62304 Class C 요구사항: 장애 감지 시간 정의 필수

방법 3: 중복 경로 (Redundant Path)
  └─ 두 개 이상의 독립 네트워크 경로
  └─ IEEE 802.1CB (FRER)로 구현 (3장 TSN에서 상세 설명)

네트워크 워치독 (Network Watchdog):
  const int TIMEOUT_CYCLES = 3;  // 허용 손실 패킷 수
  uint32_t last_received_seq = 0;
  uint32_t expected_seq = 0;

  void on_packet_timeout() {
      if (++missed_count >= TIMEOUT_CYCLES) {
          trigger_fail_safe();  // 즉시 안전 상태로
          log_safety_event("NETWORK_TIMEOUT");
      }
  }
```

---

## 4. 네트워크 장애 유형 분류

### 4.1 물리/링크 계층 장애

```
유형 1: 링크 단선 (Link Down)
  원인: 케이블 물리적 단절, 커넥터 불량
  탐지: PHY에서 즉시 감지 → OS에 interrupt → 수 ms 이내 감지
  영향: 해당 경로 전체 불통
  대응: STP/RSTP 경로 전환 (수백 ms), FRER 중복 경로 (<1ms)

유형 2: 케이블 열화 (Degraded Link)
  원인: 장기 굴곡, 커넥터 산화, 전자기 간섭(EMI)
  탐지: CRC 오류 증가, 링크 속도 강등, 간헐적 링크 다운
  영향: 산발적 패킷 손실, 지연 폭증 → 제어 불안정
  대응: ethtool -S 주기적 모니터링, 예방 교체 스케줄

유형 3: Duplex Mismatch
  원인: 한쪽 Auto-Negotiation 비활성화
  탐지: 충돌 카운터 증가, 처리량 대폭 감소
  영향: 비결정적 지연 (충돌 재전송)
  대응: 양단 Auto-Negotiation 통일

유형 4: EMI 간섭 (전자기 방해)
  원인: 수술 로봇 내 고전압 모터, 전기소작기, MRI 장비
  증상: 간헐적 CRC 오류, 링크 불안정
  대응: 차폐 케이블(STP/FTP), 광섬유, 물리적 경로 분리
```

### 4.2 L2/스위치 계층 장애

```
유형 5: Broadcast Storm
  원인: 스위치 루프 + STP 미설정 또는 오작동
  증상: CPU 부하 100%, 네트워크 마비, 모든 통신 두절
  탐지: tx/rx 패킷 수 폭증 (ethtool -S | grep packets)
  대응: STP/RSTP 활성화, Storm Control 임계값 설정

유형 6: MAC Table Overflow (MAC Flooding)
  원인: 공격자가 허위 MAC 주소로 스위치 테이블 포화
  결과: 스위치가 모든 프레임을 Flood (허브처럼 동작)
       → 트래픽 도청 + 성능 저하
  대응: Port Security (포트당 MAC 주소 수 제한)

유형 7: HOL Blocking (앞서 설명)
  원인: 대형 프레임이 긴급 소형 프레임 차단
  대응: QoS 우선순위 큐, Frame Preemption

유형 8: Babbling Idiot (떠드는 바보)
  정의: 오작동 노드가 멈추지 않고 프레임을 연속 전송
  원인: 소프트웨어 결함, EMI에 의한 PHY 오작동
  결과: 링크 포화, 정상 노드 통신 불가 → 안전 시스템 마비
  탐지: 특정 MAC 주소의 tx 카운터 비정상 증가
  대응: PSFP (IEEE 802.1Qci) 스트림 필터링으로 자동 차단

  TSN Babbling Idiot 방어 예시 (802.1Qci):
    스트림 ID: 로봇 팔 ECU (MAC: AA:BB:CC:DD:EE:01)
    허용 대역폭: 200 Mbps (설계 값)
    초과 시: 즉시 Drop + 경보 발생
```

### 4.3 시간 동기화 장애

```
유형 9: PTP 마스터 장애 (gPTP Clock Loss)
  증상: 네트워크 내 시간 동기화 해제
  영향:
    - TAS(802.1Qbv) 스케줄 비동기 → 타임 슬롯 충돌
    - 제어 루프 타이밍 오류
    - 다중 ECU 간 동작 비동기
  대응: PTP Grandmaster 이중화 (Redundant Grandmaster)
        BMCA(Best Master Clock Algorithm)로 자동 전환

유형 10: 클록 편차 누적 (Clock Drift)
  원인: 온도 변화에 의한 수정 발진기 주파수 편차
  증상: PTP offset 서서히 증가
  허용 범위: ±500 ns (TAS 활용 시 권장)
  대응: linuxptp 주기적 동기화, GPS 기반 Grandmaster
```

### 4.4 애플리케이션 계층 장애

```
유형 11: 소프트웨어 Babbling Idiot
  원인: 애플리케이션 루프 오류 → 소켓 flood
  대응: 소켓 전송 속도 제한 (SO_SNDBUF 제한, Token Bucket)

유형 12: 잘못된 멀티캐스트 그룹 참여
  원인: 설정 오류로 불필요한 트래픽 수신
  결과: CPU 부하 증가, 필요 트래픽 처리 지연

유형 13: ARP Storm
  원인: 대규모 IP 주소 충돌, 잘못된 ARP 설정
  대응: Dynamic ARP Inspection, ARP 속도 제한
```

---

## 5. Fail-Safe 네트워크 설계 기초

### 5.1 Fail-Safe 설계 원칙

```
핵심 원칙: "장애 발생 시 시스템이 더 안전한 상태로 전환"

네트워크 관점 Fail-Safe:
  1. 통신 두절 → 즉각적인 안전 동작 전환 (안전 정지)
  2. 부분 장애 → 격리 후 나머지 기능 유지
  3. 장애 감지 시간 최소화 (watchdog 타임아웃 설계)
  4. 단일 실패 지점(SPOF) 제거
```

### 5.2 Watchdog 타임아웃 설계

```
네트워크 워치독 계층:

계층 1: PHY 레벨 (하드웨어)
  링크 다운 → PHY가 즉시 (<1ms) ECU에 인터럽트
  ECU → 즉시 출력 비활성화 또는 안전 상태

계층 2: L2 하트비트 (소형 주기 프레임)
  주기: 제어 사이클과 동일 (예: 1ms)
  내용: 시퀀스 번호 + 상태 플래그
  감지: 3회 연속 미수신 → 장애 판정

계층 3: 애플리케이션 워치독
  소프트웨어 내 타이머 (예: 5ms)
  정상 수신 시마다 타이머 리셋
  타임아웃 시: Fail-Safe 루틴 호출

워치독 타임아웃 계산:
  T_timeout = T_detect + T_reaction
  T_detect  = N_missed × T_cycle + T_jitter_max
            = 3 × 1ms + 0.05ms = 3.05ms
  T_reaction = 안전 정지 실행 시간 (예: 20ms)
  T_timeout  ≤ T_hazard (위험 발생 전 정지 시간)
```

### 5.3 경로 이중화 (Redundancy)

```
단일 경로 (SPOF):
  ECU A ────── 스위치 ────── ECU B
          ↑
    이 링크 장애 시 전체 통신 두절

이중 경로 (FRER - IEEE 802.1CB):
  ECU A ──────── 스위치 1 ────────── ECU B
        │                           │
        └─────── 스위치 2 ───────────┘

  동작:
    - ECU A가 두 경로로 동일 프레임 복제 전송
    - ECU B가 먼저 도착한 프레임 사용, 중복 폐기
    - 한 경로 장애 → 다른 경로 자동 계속 사용
    - 장애 전환 시간: 0ms (무중단)

  (상세: 3장 IEEE 802.1CB 문서 참조)
```

### 5.4 트래픽 격리에 의한 장애 격리

```
VLAN을 이용한 장애 도메인 분리:

VLAN 10 (안전 제어):
  ┌─────────────────────────────────────────────┐
  │  안전 ECU ──── TSN 스위치 ──── 안전 ECU     │
  │  (장애 도메인 1: 완전 격리)                 │
  └─────────────────────────────────────────────┘

VLAN 30 (진단/관리):
  ┌─────────────────────────────────────────────┐
  │  진단 PC ──── 동일 스위치 ──── OTA 서버     │
  │  (장애 도메인 2: VLAN 10에 영향 없음)       │
  └─────────────────────────────────────────────┘

→ 진단 PC가 Broadcast Storm 발생해도 VLAN 10의 안전 제어는 영향 없음
→ 단, 스위치 내부 처리 용량은 공유됨 → 완전 분리 필요 시 물리적 분리
```

---

## 6. IEC 62304 관점의 네트워크 요구사항

### 6.1 소프트웨어 안전 분류와 네트워크

```
IEC 62304 소프트웨어 안전 등급:

Class A: 부상 위험 없음
  네트워크 요구: 일반 Best-Effort 허용
  예: 로그 수집, 원격 모니터링

Class B: 심각하지 않은 부상 가능
  네트워크 요구: QoS 우선순위, 패킷 손실 모니터링
  예: 알림 시스템, 비실시간 진단

Class C: 사망 또는 심각한 부상 위험
  네트워크 요구 (엄격):
    ✓ 결정론적 지연 보장 (TSN 또는 동등 기술)
    ✓ 패킷 손실 허용치 정의 및 검증
    ✓ 단일 실패 지점 없음 (이중화)
    ✓ 장애 감지 시간 사전 계산 및 검증
    ✓ 모든 네트워크 장애 유형에 대한 FMEA
    ✓ 워스트케이스 지연 분석 (WCRT 분석)
  예: 수술 로봇 제어, 방사선 치료 빔 제어
```

### 6.2 네트워크 관련 FMEA (Failure Mode and Effects Analysis)

```
의료 로봇 네트워크 FMEA 예시:

┌──────────────────┬──────────────┬──────────┬─────────────┬──────────────┐
│ 장애 모드        │ 장애 영향    │ 심각도   │ 탐지 방법   │ 완화 조치    │
├──────────────────┼──────────────┼──────────┼─────────────┼──────────────┤
│ 링크 단선        │ 제어 두절    │ 치명     │ PHY interrupt│ FRER 이중화  │
│ 패킷 손실 > 0.1% │ 제어 불안정  │ 심각     │ Seq# 확인   │ Watchdog     │
│ 지연 > 500µs    │ 제어 루프 실패│ 심각    │ Timestamp   │ TSN TAS      │
│ Babbling Idiot   │ 네트워크 마비 │ 치명    │ PSFP 카운터 │ 802.1Qci     │
│ 시간 동기 손실   │ TAS 비동기   │ 중간    │ PTP offset  │ GM 이중화    │
│ Broadcast Storm  │ 전체 마비    │ 치명    │ tx 카운터   │ STP + VLAN   │
└──────────────────┴──────────────┴──────────┴─────────────┴──────────────┘
```

### 6.3 네트워크 성능 검증 요구사항

```
IEC 62304 + IEC 61508 기반 검증 항목:

1. WCRT (Worst-Case Response Time) 분석
   - 수학적 분석 또는 실측으로 최악 지연 계산/측정
   - 도구: iperf3 (대역폭), cyclictest (지연), Wireshark (패킷 분석)
   - 허용 기준: WCRT < 제어 사이클 × 허용 비율 (예: 200µs < 1ms × 0.2)

2. 패킷 손실 테스트
   - 정상 부하: 0% 손실 목표
   - 최대 부하 (80%): 안전 트래픽 0% 손실 목표
   - 측정 기간: ≥ 24시간 연속 (신뢰도 확보)

3. Jitter 측정
   - linuxptp phc2sys 정밀도 측정
   - cyclictest로 OS 레이턴시 분포 확인
   - 허용 Jitter: 제어 루프 요구사항에서 역산

4. Fault Injection 테스트
   - tc netem으로 지연/손실 인위 주입
   - Fail-Safe 동작 확인
   - 복구 시간 측정

tc netem 예시 (Fault Injection):
  # 10ms 지연 + 1% 패킷 손실 주입
  tc qdisc add dev eth0 root netem delay 10ms loss 1%
  # Fail-Safe 동작 확인 후
  tc qdisc del dev eth0 root
```

---

## 7. 결정성 네트워크로의 전환 로드맵

### 7.1 단계별 전환 경로

```
Phase 1: Best-Effort Ethernet (현재 레거시)
  - 일반 스위치 + QoS 우선순위 큐
  - 패킷 손실 허용 (재전송 의존)
  - Jitter: 수 ms
  - 적합: 비실시간 데이터 수집, 설정 전송

Phase 2: QoS 강화 Ethernet
  - SP + DWRR 큐 스케줄링
  - VLAN 격리
  - Jitter: 수백 µs
  - 적합: 실시간성 요구 낮은 피드백 제어

Phase 3: TSN (Time-Sensitive Networking)
  - IEEE 802.1AS: µs 수준 시간 동기화
  - IEEE 802.1Qbv: 시간 기반 게이트 스케줄링 (TAS)
  - IEEE 802.1Qci: 스트림 필터링 (Babbling Idiot 방어)
  - IEEE 802.1CB: 프레임 이중화 (무중단 복구)
  - Jitter: < 수십 µs
  - 적합: IEC 62304 Class C 의료 로봇 제어

Phase 4: TSN + 검증 완료
  - WCRT 분석 완료
  - FMEA 완료
  - IEC 62304 규제 문서 완료
  - 적합: 규제 승인 의료기기 출시
```

### 7.2 1장에서 3장으로의 학습 연결

```
1장 학습 완료 후 이해해야 할 핵심 개념:

✓ OSI L2에서 모든 TSN 표준이 동작함을 이해
  → 3장: IEEE 802.1AS (L2 멀티캐스트 PTP)
  → 3장: IEEE 802.1Qbv (L2 스케줄러)

✓ Ethernet 프레임 구조를 이해
  → 3장: TAS Guard Band = max_frame_size / link_speed
  → 3장: Frame Preemption의 mPacket 개념

✓ VLAN 격리를 이해
  → 3장: TSN 스케줄링은 VLAN 별로 독립 적용

✓ 흐름 제어와 큐를 이해
  → 3장: TAS가 혼잡 자체를 방지하는 원리

✓ 비결정성의 원인을 이해
  → 3장: TSN이 각 원인을 어떻게 해결하는지
  → 5장: 결정성 부재가 기능 안전 위반이 되는 이유
```

---

## 8. 실습: 네트워크 결정성 측정

### 8.1 기본 Jitter 측정 (cyclictest)

```bash
# RT-Preempt 커널에서 네트워크 관련 지연 측정
cyclictest --mlockall --smp --priority=80 --interval=1000 --distance=0 \
  --nsecs --histfile=/tmp/latency_hist.txt --duration=60

# UDP 소켓 왕복 지연 측정 (Python)
import socket, time, statistics

def measure_rtt(host, port, count=10000):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(0.01)  # 10ms timeout
    latencies = []
    for i in range(count):
        t0 = time.perf_counter_ns()
        sock.sendto(b'\x00' * 64, (host, port))
        try:
            sock.recv(64)
            t1 = time.perf_counter_ns()
            latencies.append((t1 - t0) / 1000)  # µs
        except socket.timeout:
            print(f"Packet {i} lost!")
    print(f"Min: {min(latencies):.1f} µs")
    print(f"Max: {max(latencies):.1f} µs")
    print(f"Avg: {statistics.mean(latencies):.1f} µs")
    print(f"Jitter(std): {statistics.stdev(latencies):.1f} µs")

measure_rtt('192.168.10.2', 5000, 10000)
```

### 8.2 tc netem으로 장애 시뮬레이션

```bash
# 배경 트래픽 50% 부하 생성
iperf3 -c 192.168.10.2 -u -b 500M -t 60 &

# 1% 패킷 손실 주입
tc qdisc add dev eth0 root netem loss 1%

# 제어 소프트웨어에서 Fail-Safe 동작 확인 후
tc qdisc del dev eth0 root

# 10ms 지연 + 5ms 지연 변동 주입
tc qdisc add dev eth0 root netem delay 10ms 5ms distribution normal

# 결과: 제어 루프가 Watchdog 타임아웃으로 Fail-Safe 진입 확인
```

---

## Reference
- [IEC 62304:2006+AMD1:2015 - Medical Device Software: Software Life Cycle Processes](https://www.iso.org/standard/64686.html)
- [IEC 61508 - Functional Safety of E/E/PE Safety-Related Systems](https://www.iec.ch/functionalsafety/)
- [IEEE 802.1Q-2022 TSN Overview](https://standards.ieee.org/ieee/802.1Q/6844/)
- [IEEE 802.1CB-2017 - Frame Replication and Elimination for Reliability (FRER)](https://standards.ieee.org/ieee/802.1CB/6421/)
- [IEEE 802.1Qci-2017 - Per-Stream Filtering and Policing (PSFP)](https://standards.ieee.org/ieee/802.1Qci/6070/)
- [IEC 62443 - Industrial Automation and Control Systems Security](https://www.iec.ch/iec62443/)
- [cyclictest - Linux real-time latency testing](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start)
- [Linux tc-netem - Network Emulator](https://man7.org/linux/man-pages/man8/tc-netem.8.html)
