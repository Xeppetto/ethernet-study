# IEEE 802.1Qci (Per-Stream Filtering and Policing)

## 개요
IEEE 802.1Qci는 Ethernet 스위치의 **수신 포트(Ingress)**에서 각 트래픽 스트림을 감시하고, 약속된 범위를 벗어난 트래픽을 차단하거나 마킹하는 표준입니다. "Ingress Policing" 또는 "PSFP(Per-Stream Filtering and Policing)"라고도 불립니다.

특히 **Babbling Idiot 방지** (오작동 노드가 네트워크를 독점하는 현상 차단)에 핵심적인 역할을 합니다.

---

## 핵심 개념

### Stream Identification (스트림 식별)
들어오는 프레임을 특정 규칙에 따라 스트림으로 분류합니다.

```
분류 기준:
  - MAC 주소 (Src/Dst)
  - VLAN ID
  - IPv4/IPv6 주소
  - TCP/UDP 포트
  - DSCP 값

예: Stream 1 = (VLAN 10, Src MAC = AA:BB:CC:DD:EE:01, Dst = 브로드캐스트)
    Stream 2 = (VLAN 20, TCP Port 30490, Dst = 192.168.1.1)
```

### PSFP 파이프라인

```
수신 프레임
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Stage 1: Stream Identification                      │
│  (어떤 스트림인지 식별)                               │
├─────────────────────────────────────────────────────┤
│  Stage 2: Stream Filter                              │
│  (스트림 별 필터 규칙 적용)                           │
│  • Max SDU 크기 검사 (너무 큰 프레임 폐기)            │
│  • 우선순위 매핑                                     │
├─────────────────────────────────────────────────────┤
│  Stage 3: Flow Meter (토큰 버킷)                     │
│  (약속된 대역폭 초과 여부 판단)                       │
│  • CIR (Committed Information Rate): 보장 대역폭    │
│  • CBS (Committed Burst Size): 허용 최대 버스트      │
├─────────────────────────────────────────────────────┤
│  Stage 4: Gate Control                               │
│  (스트림별 게이트 열림/닫힘 제어)                     │
│  → 802.1Qbv TAS와 연동 가능                         │
└─────────────────────────────────────────────────────┘
    │
    ▼
정상 프레임 → 포워딩
초과 프레임 → 폐기 또는 DEI 마킹 (Drop Eligible)
```

---

## Flow Meter (토큰 버킷 알고리즘)

```
토큰 버킷 개념:
  ┌─────────────────────────────────────────────────┐
  │  버킷 용량 = CBS (Committed Burst Size)          │
  │                                                 │
  │  토큰 생성률 = CIR (Committed Information Rate) │
  │                                                 │
  │  프레임 도착 → 프레임 크기만큼 토큰 소비          │
  │  토큰 부족 → 프레임 폐기 또는 마킹               │
  └─────────────────────────────────────────────────┘

설정 예시:
  Stream 1 (관절 제어): CIR = 10Mbps, CBS = 1500 Byte
  → 10Mbps를 초과하는 순간 초과 프레임 폐기
  → 짧은 버스트(1500B)는 허용

  Stream 2 (영상): CIR = 500Mbps, CBS = 64KB
  → 카메라가 폭발적으로 전송 시 64KB까지 허용
  → 이후 500Mbps로 제한
```

---

## Babbling Idiot 방어

```
정상 동작:
  Joint ECU #1 ── 제어 패킷 10Mbps ──► TSN Switch ──► Main ECU

Babbling Idiot 발생:
  Joint ECU #1 (고장!) ── 패킷 1000Mbps (최대 링크 속도) ──► TSN Switch

802.1Qci 없는 경우:
  전체 네트워크 포화 → 다른 ECU 통신 불가 → 수술 중 시스템 마비

802.1Qci 적용:
  TSN Switch Ingress Policing:
    Stream: ECU #1 (MAC 기준 식별)
    CIR: 10Mbps (설정값)
    조치: 10Mbps 초과 즉시 폐기
  결과: ECU #1 고장에도 다른 ECU 정상 통신 유지
```

---

## Linux에서 PSFP 설정 (tc 사용)

```bash
# 스트림 필터 설정 (최신 Linux 커널 5.14+)
# Stream 1: MAC 기반 필터, 최대 대역폭 10Mbps

# 1. Stream 필터 생성
tc filter add dev eth0 ingress \
    protocol 802.1Q \
    flower \
    vlan_id 10 \
    src_mac aa:bb:cc:dd:ee:01 \
    action gate \
        base-time 1600000000000000000 \
        sched-entry OPEN 100000 10485760 9000 \
        sched-entry CLOSE 900000 -1 -1

# 2. 대역폭 폴리싱 (토큰 버킷)
tc filter add dev eth0 ingress \
    protocol ip \
    flower \
    ip_proto tcp \
    dst_port 30490 \
    action police \
        rate 10mbit \
        burst 15k \
        drop

# 현재 필터 확인
tc filter show dev eth0 ingress

# 폐기 통계 확인
tc -s filter show dev eth0 ingress | grep dropped
```

---

## 스트림 Gate Control (802.1Qbv와 연동)

```
802.1Qci Gate를 802.1Qbv TAS와 함께 사용:

TAS GCL:
  T=0~100µs: TC7 OPEN (제어 트래픽)
  T=100~1000µs: TC0 OPEN (BE 트래픽)

PSFP Gate (Stream 1 = 관절 제어 스트림):
  T=0~100µs: Gate OPEN
  T=100µs~: Gate CLOSED

결과: 관절 제어 스트림은 TAS + PSFP 이중 보호
  - TAS: 큐 수준에서 TC7 전용 슬롯
  - PSFP: 스트림 수준에서 특정 스트림만 통과
```

---

## 모니터링: 이상 감지

```bash
# Wireshark로 폐기된 패킷 분석
# (실제로는 스위치 관리 인터페이스 사용)

# 스위치 PSFP 통계 조회 (NETCONF/YANG 또는 관리 콘솔)
# Stream 1 폐기 카운터:
# ingress_frames_received: 100000
# ingress_frames_passed:   99850
# ingress_frames_dropped:  150  ← 150개 폐기
# 이상 발생 시 알람 트리거

# tc 통계 (Linux 브리지)
tc -s qdisc show dev eth0 | grep -A5 "ingress"
```

---

## 의료 로봇 PSFP 정책 예시

```
스트림 분류 및 정책:

Stream 1 (안전 제어):
  식별: VLAN=10, Src=Safety ECU MAC
  CIR: 5 Mbps, CBS: 9000 Byte
  Gate: TAS TC7 슬롯에서만 OPEN
  초과 시: 즉시 폐기 + 알람

Stream 2 (관절 제어):
  식별: VLAN=20, Src=Joint ECU MAC 그룹
  CIR: 50 Mbps, CBS: 64 KB
  Gate: TAS TC5 슬롯에서만 OPEN
  초과 시: DEI 마킹 (폐기 후보)

Stream 3 (영상):
  식별: VLAN=30, Src=Camera MAC
  CIR: 400 Mbps, CBS: 1 MB
  Gate: 항상 OPEN (Best Effort 큐 사용)
  초과 시: 폐기 (영상 일부 손실 허용)

Stream 4 (진단 DoIP):
  식별: VLAN=40, TCP Port 13400
  CIR: 100 Mbps, CBS: 1 MB
  Gate: 항상 OPEN
  초과 시: 폐기
```

---

## Reference
- [IEEE 802.1Qci-2017 - Per-Stream Filtering and Policing](https://standards.ieee.org/ieee/802.1Qci/6045/)
- [TSN Task Group - 802.1Qci](https://1.ieee802.org/tsn/802-1qci/)
- [Linux tc-flower 필터](https://man7.org/linux/man-pages/man8/tc-flower.8.html)
- [Linux tc-police 폴리서](https://man7.org/linux/man-pages/man8/tc-police.8.html)
