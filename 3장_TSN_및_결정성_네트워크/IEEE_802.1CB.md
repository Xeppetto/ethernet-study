# IEEE 802.1CB (Frame Replication and Elimination for Reliability)

## 개요
IEEE 802.1CB는 동일한 데이터 프레임을 복제하여 **서로 다른 경로(Multi-path)**로 전송하고, 수신 측에서 중복 프레임을 제거하는 표준입니다. 링크 장애 발생 시에도 **무중단(Zero Recovery Time)** 통신을 제공하여 고가용성(High Availability)을 보장합니다.

FRER(Frame Replication and Elimination for Reliability)라고도 합니다.

---

## 동작 원리

### 기본 구성 (병렬 경로)

```
                    ┌── 경로 A ──►│
송신자 → [복제기] ──┤              │── [제거기] → 수신자
                    └── 경로 B ──►│

경로 A: 스위치 S1 → 스위치 S3
경로 B: 스위치 S2 → 스위치 S3 (물리적으로 다른 경로)

장애 시나리오:
  경로 A 단선 → 경로 B로만 수신 → 수신자: 차이 없이 정상 수신
  Recovery Time = 0 (Zero Recovery)
```

### R-Tag (Redundancy Tag) 구조

중복 프레임을 식별하기 위해 각 프레임에 R-Tag를 추가합니다.

```
R-Tag 위치: Ethernet 프레임의 Src MAC 뒤, EtherType 앞에 삽입

일반 Ethernet 프레임:
  [7B Preamble][1B SFD][6B Dst MAC][6B Src MAC][2B EtherType][Payload][4B FCS]

R-Tag 삽입 후:
  [7B Preamble][1B SFD][6B Dst MAC][6B Src MAC]
  [2B R-TAG EtherType (0xF1C1)][2B Reserved][2B SequenceNumber]
  [2B 원본 EtherType][Payload][4B FCS]

R-Tag 상세:
  ┌──────────────────────────────────────────────────────┐
  │  R-TAG (6 Byte)                                      │
  │  ┌────────────┬─────────────┬──────────────────────┐ │
  │  │ EtherType  │  Reserved   │  SequenceNumber      │ │
  │  │  0xF1C1    │   2 Byte    │  2 Byte (0~65535)    │ │
  │  └────────────┴─────────────┴──────────────────────┘ │
  └──────────────────────────────────────────────────────┘

SequenceNumber: 0~65535 순환 (모듈로 65536)
  송신 측이 각 프레임마다 증가
  수신 측이 이미 수신한 번호면 중복 → 폐기
```

### 복제 및 제거 과정

```
송신 ECU
    │
    │ 원본 프레임 (SeqNo=42)
    ▼
┌─────────────────┐
│  Sequence Encode │  SeqNo 부여 + R-Tag 삽입
└────────┬────────┘
         │ 복제
   ┌─────┴──────┐
   │            │
경로 A         경로 B
(스위치 S1)   (스위치 S2)
   │            │
   │ SeqNo=42  │ SeqNo=42
   ▼            ▼
┌─────────────────┐
│  Sequence Decode │  두 프레임 수신
│  (제거기)        │  첫 번째: 수신자에게 전달
│                 │  두 번째: SeqNo=42 이미 봄 → 폐기
└────────┬────────┘
         │
    수신 ECU (정상 수신)
```

---

## Sequence Recovery 알고리즘

수신 측이 중복 프레임을 식별하는 두 가지 알고리즘입니다.

### Vector Recovery (비트맵 방식)

```
최근 수신한 SequenceNumber를 비트맵으로 관리합니다.

HistoryLen = 10 (최근 10개 번호 추적)
수신 SeqNo=42:
  비트맵 [현재~현재-9] 검사
  → 42가 없으면: 정상 수신, 비트맵 업데이트
  → 42가 있으면: 중복, 폐기

장점: 순서가 바뀌어 도착해도 처리 가능 (네트워크 재정렬)
단점: 비트맵 크기(HistoryLen) 이상 지연 차이가 있으면 중복 감지 실패
```

### Match Recovery (단순 비교)

```
마지막으로 수신한 SeqNo와 비교합니다.

LastRxSeqNo = 41
수신 SeqNo=42:
  → 42 > 41: 정상, LastRxSeqNo = 42

수신 SeqNo=42 (중복, 경로 B 도착):
  → 42 == LastRxSeqNo: 중복, 폐기

장점: 구현 단순
단점: 두 경로 간 도착 순서 역전 시 중복 미감지 가능
```

### HistoryLen (역사 길이) 설정

```
frerSeqRcvyHistLen: 추적할 최근 SeqNo 수 (권장: 2~128)

설정 기준:
  두 경로 간 최대 지연 차이 = Δt
  최소 필요 HistLen = Δt × 프레임 전송률

예:
  경로 A: 1ms 지연, 경로 B: 3ms 지연
  Δt = 2ms, 전송률 = 1000fps
  HistLen ≥ 2ms × 1000fps = 2 → HistLen = 8 (여유분 포함)
```

---

## 경로 구성 방식

### 노드 이중화 (Node Replication)

```
      ┌──── 스위치 A ────┐
ECU A ─┤                  ├──── ECU B
      └──── 스위치 B ────┘

두 스위치 중 하나 고장해도 통신 유지
```

### 링크 이중화 (Link Replication)

```
ECU A ──── 스위치 ──── 링크 1 ────┐
                │                ├──── ECU B
                └──── 링크 2 ────┘

링크 하나 단선해도 통신 유지
```

### 조합 구성 (의료 로봇 권장)

```
Main ECU                    Zone ECU
    │                           │
    ├──링크1──► 스위치 A ──링크3─►│
    │                           │
    └──링크2──► 스위치 B ──링크4─►│

완전 경로 이중화:
  링크 1개 또는 스위치 1개 고장해도 무중단
  (링크1 단선: 링크2→스위치B→링크4 경로 계속)
```

---

## STP vs FRER 비교

```
STP(Spanning Tree Protocol) 방식:
  링크 A (활성) ──► 사용 중
  링크 B (대기) ──► 차단됨 (비용 낭비)
  링크 A 단선 → STP Reconverge (최대 수십 초)
  → 수술 중 수십 초 통신 중단 = 심각한 위험

FRER 방식:
  링크 A ──► 사용 중 (복제 전송)
  링크 B ──► 동시에 사용 중 (복제 전송)
  링크 A 단선 → 링크 B 계속 사용 중
  → Recovery Time = 0 (무중단)

추가 장점:
  두 경로 동시 사용 → 대역폭 소비 증가 (복제 전송)
  대신 MTTR = 0, 가용성 99.9999%
```

---

## FRER과 TSN 통합

```
TSN 스택에서 FRER 위치:

IEEE 802.1AS  ────── 시간 동기화 (기반)
IEEE 802.1Qbv ────── 스케줄링 (지연 보장)
IEEE 802.1Qci ────── 폴리싱 (보호)
IEEE 802.1CB  ────── 이중화 (신뢰성)

FRER + TAS 조합:
  FRER: 안전 프레임을 경로 A, B로 복제
  TAS: 두 경로 모두 TC7 큐 → 동일 슬롯에서 전송
  결과: 결정론적 지연(TAS) + 무중단(FRER) = 안전-결정론적 통신
```

---

## Linux에서 FRER 설정

Linux 커널 5.16+에서 FRER 부분 지원 (하드웨어 의존적).

```bash
# FRER 지원 확인 (하드웨어 및 드라이버)
ip -d link show eth0 | grep frer

# 브리지 기반 VLAN 필터링 (FRER 기반 경로 분리)
bridge vlan add vid 10 dev eth0 tagged
bridge vlan add vid 10 dev eth1 tagged

# bonding 드라이버로 링크 이중화 (FRER 유사 효과)
# 참고: 실제 FRER R-Tag와 다르지만 링크 이중화 효과
ip link add bond0 type bond mode active-backup
ip link set eth0 master bond0
ip link set eth1 master bond0
ip link set bond0 up

# 완전한 FRER 구현: 하드웨어 전용 API 또는 NETCONF/YANG 사용
# (NXP S32G, Renesas R-Car, Intel I210/I225 드라이버 지원)

# YANG 모델로 FRER 설정 (NETCONF)
# ietf-frer.yang 또는 ieee802-dot1cb-frer.yang 사용
```

---

## 신뢰성 지표

### 가용성 계산

```python
# 단일 경로 vs FRER 이중 경로 가용성 비교
def calc_availability(link_avail=0.999, switch_avail=0.999):
    """
    link_avail: 단일 링크 가용성
    switch_avail: 단일 스위치 가용성
    """
    # 단일 경로 (FRER 없음)
    single_path = link_avail * switch_avail * link_avail
    print(f"단일 경로 가용성: {single_path:.6f} ({(1-single_path)*100:.4f}% downtime)")

    # FRER 이중 경로
    # 두 경로 모두 실패할 확률
    path_a_fail = 1 - single_path
    path_b_fail = 1 - single_path
    both_fail = path_a_fail * path_b_fail
    dual_path = 1 - both_fail
    print(f"FRER 이중 경로 가용성: {dual_path:.8f} ({both_fail*1e6:.4f} ppm downtime)")
    print(f"개선 비율: {dual_path/single_path:.2f}x")

calc_availability(0.999, 0.999)
# 단일 경로 가용성: 0.997003 (0.2997% downtime)
# FRER 이중 경로 가용성: 0.99999991 (0.0000 ppm downtime)
```

---

## 의료 로봇 FRER 적용 시나리오

```
수술 로봇 이중화 네트워크:

Main ECU (HPC)
  │          │
  │ 경로 A   │ 경로 B
  │          │
Switch A   Switch B
  │    ╲  ╱  │
  │     ╲╱   │ (교차 연결)
  │     ╱╲   │
  │    ╱  ╲  │
  │          │
Joint ECU #1, #2, #3
Safety ECU

FRER 보호 대상 스트림:
  ① 안전 E-Stop 신호 (ASIL-D 요구)
    → VLAN 10, SeqEnc/Dec 모두 활성화
  ② 조인트 제어 명령 (500Hz, ASIL-B)
    → VLAN 20, 이중 경로
  ③ 엔코더 피드백 (안전 감시용)
    → VLAN 30, 이중 경로

FRER 불필요 스트림 (비용 절감):
  ④ 내시경 영상 → 일부 손실 허용
  ⑤ 진단 DoIP → TCP 재전송으로 보완

비용 대비 효과:
  대역폭 2배 소비 (복제 전송)
  하드웨어 2배 (스위치 이중화)
  대신: 안전 규제(IEC 61508) ASIL-D 달성 가능
```

---

## Reference
- [IEEE 802.1CB-2017 - Frame Replication and Elimination for Reliability](https://standards.ieee.org/ieee/802.1CB/6055/)
- [TSN Task Group - 802.1CB](https://1.ieee802.org/tsn/802-1cb/)
- [IEC 62439-3 - PRP/HSR (유사 기술 비교)](https://webstore.iec.ch/publication/7018)
- [Avnu Alliance - Network Redundancy for TSN](https://avnu.org/)
- [IEEE 802.1CB YANG Model - ietf-frer.yang](https://datatracker.ietf.org/doc/html/rfc9023)
