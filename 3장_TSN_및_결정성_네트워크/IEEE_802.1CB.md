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
```

### 시퀀스 번호 (Sequence Number)
중복 프레임을 식별하기 위해 각 프레임에 시퀀스 번호를 추가합니다.

```
┌──────────────────────────────────────────────────────────┐
│  R-TAG (FRER Tag, 6 Byte)                                │
│  ┌─────────────┬─────────────┬────────────────────────┐  │
│  │  EtherType  │  Reserved   │  SequenceNumber        │  │
│  │   0xF1C1    │   2 Byte    │  2 Byte (0~65535)      │  │
│  └─────────────┴─────────────┴────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
R-TAG는 Ethernet 프레임의 MAC 주소 뒤, Payload 앞에 삽입
```

### 복제 및 제거 과정
```
송신 ECU
    │
    │ 원본 프레임 (SeqNo=42)
    ▼
┌─────────────────┐
│  Sequence Encode │  ← SeqNo 부여 + R-TAG 추가
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
│  Sequence Decode │  ← 두 프레임 수신
│  (제거 기)       │  첫 번째 수신: 수신자에게 전달
│                 │  두 번째 수신: SeqNo=42 이미 봄 → 폐기
└────────┬────────┘
         │
    수신 ECU

장애 시나리오:
  경로 A 단선 → 경로 B로만 수신
  수신자: 차이 없이 정상 수신 (Recovery Time = 0)
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

완전 경로 이중화: 링크 1개 또는 스위치 1개 고장해도 무중단
```

---

## STP와 FRER 비교

```
STP(Spanning Tree Protocol) 방식:
  링크 A (활성) ──► 사용 중
  링크 B (대기) ──► 차단됨
  링크 A 단선 → STP Reconverge 실행 (최대 수십 초)
  → 수술 중 수십 초 통신 중단 = 심각한 위험

FRER 방식:
  링크 A ──► 사용 중 (복제 전송)
  링크 B ──► 동시에 사용 중 (복제 전송)
  링크 A 단선 → 링크 B 계속 사용 중
  → Recovery Time = 0 (무중단)
```

---

## FRER과 TSN 통합

```
TSN 스택에서 FRER 위치:

IEEE 802.1AS  ────── 시간 동기화 (기반)
IEEE 802.1Qbv ────── 스케줄링 (지연 보장)
IEEE 802.1Qci ────── 폴리싱 (보호)
IEEE 802.1CB  ────── 이중화 (신뢰성)
                              ↑
                     안전 제어 트래픽에 필수
```

### FRER + TAS 조합
```
FRER로 안전 프레임 복제:
  경로 A: TC7 큐 → TAS 슬롯 0~50µs 전송
  경로 B: TC7 큐 → TAS 슬롯 0~50µs 전송 (동일 슬롯)

결과:
  결정론적 지연(TAS) + 무중단(FRER) = 안전-결정론적 통신
```

---

## Linux에서 FRER 설정

Linux 커널 5.16+에서 FRER 부분 지원 (하드웨어 의존적).

```bash
# 기본 FRER 설정 (추상적 예시)
# 실제 설정은 스위치 제조사 API/SDK 또는 NETCONF 사용

# 브리지 VLAN 필터링 (FRER 기반 경로 분리)
bridge vlan add vid 10 dev eth0 tagged
bridge vlan add vid 10 dev eth1 tagged

# bonding 드라이버로 링크 이중화 (FRER 유사 효과)
# (실제 FRER과 다르지만 링크 이중화 효과)
ip link add bond0 type bond
ip link set eth0 master bond0
ip link set eth1 master bond0
echo "active-backup" > /sys/class/net/bond0/bonding/mode

# FRER 지원 하드웨어: NXP S32G, Renesas R-Car, Intel i210/i225
```

---

## 신뢰성 지표

### 가용성 계산
```
단일 경로 (FRER 없음):
  링크 가용성 = 99.9% (3-nine)
  스위치 가용성 = 99.9%
  전체 경로 = 99.9% × 99.9% ≈ 99.8%

FRER (이중 경로):
  경로 A 가용성 = 99.9%
  경로 B 가용성 = 99.9%
  두 경로 모두 실패 확률 = 0.1% × 0.1% = 0.0001%
  전체 가용성 = 99.9999% (6-nine)
  → MTTR(평균 복구 시간) = 0초
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
  │     ╲╱   │
  │     ╱╲   │
  │    ╱  ╲  │
  │          │
Joint ECU #1, #2, #3
Safety ECU

FRER 보호 대상 스트림:
  ① 안전 E-Stop 신호 (ASIL-D 요구)
  ② 조인트 제어 명령 (500Hz, ASIL-B)
  ③ 엔코더 피드백 (안전 감시용)

FRER 불필요 스트림 (비용 절감):
  ④ 내시경 영상 (일부 손실 허용)
  ⑤ 진단 DoIP 트래픽 (TCP 재전송으로 보완)
```

---

## Reference
- [IEEE 802.1CB-2017 - Frame Replication and Elimination for Reliability](https://standards.ieee.org/ieee/802.1CB/6055/)
- [TSN Task Group - 802.1CB](https://1.ieee802.org/tsn/802-1cb/)
- [IEC 62439-3 - PRP/HSR (유사 기술 비교)](https://webstore.iec.ch/publication/7018)
- [Avnu Alliance - Network Redundancy for TSN](https://avnu.org/)
