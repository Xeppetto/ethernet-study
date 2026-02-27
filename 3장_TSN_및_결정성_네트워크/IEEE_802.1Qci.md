# IEEE 802.1Qci (Per-Stream Filtering and Policing)

## 개요
IEEE 802.1Qci는 Ethernet 스위치의 **수신 포트(Ingress)**에서 각 트래픽 스트림을 감시하고, 약속된 범위를 벗어난 트래픽을 차단하거나 마킹하는 표준입니다. "Ingress Policing" 또는 "PSFP(Per-Stream Filtering and Policing)"라고도 불립니다.

특히 **Babbling Idiot 방지** (오작동 노드가 네트워크를 독점하는 현상 차단)에 핵심적인 역할을 합니다.

---

## 핵심 개념

### Stream Identification (스트림 식별)
들어오는 프레임을 특정 규칙에 따라 스트림으로 분류합니다.

```
분류 기준 (IEEE 802.1CB와 공유):
  - MAC 주소 (Src/Dst)
  - VLAN ID (VID)
  - EtherType
  - IPv4/IPv6 주소
  - TCP/UDP 포트
  - DSCP 값

예: Stream 1 = (VLAN 10, Src MAC = AA:BB:CC:DD:EE:01)
    Stream 2 = (VLAN 20, TCP Port 30490, Dst IP = 192.168.1.1)
```

### PSFP 파이프라인

```
수신 프레임
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Stage 1: Stream Identification                      │
│  어떤 스트림인지 식별                                │
│  → Stream Filter Instance (SFI) 매핑                 │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  Stage 2: Stream Filter Instance (SFI)               │
│  스트림 별 필터 규칙 적용                             │
│  • MaxSDUSize 검사: 큰 프레임 즉시 폐기              │
│  • IPV (Internal Priority Value) 오버라이드          │
│  • Gate/Flow Meter Instance 연결                     │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────┴──────────┐
              │                   │
              ▼                   ▼
┌─────────────────┐   ┌──────────────────────────────┐
│ Stage 3: SGI    │   │ Stage 4: Flow Meter Instance  │
│ (Stream Gate)   │   │ (토큰 버킷 폴리서)             │
│ OPEN/CLOSE 제어 │   │ CIR / CBS 기반 폐기/마킹      │
└────────┬────────┘   └──────────────┬───────────────┘
         │                           │
         └─────────────┬─────────────┘
                       │
                       ▼
              정상 프레임 → 포워딩
              초과 프레임 → 폐기 또는 DEI 마킹
```

---

## Stream Gate Instance (SGI) 상세

SGI는 각 스트림에 대한 시간 기반 게이트 제어입니다. TAS의 포트 수준 게이트와 달리 **스트림 수준**에서 동작합니다.

```
SGI 파라미터:
  AdminGateStates:      초기 게이트 상태 (OPEN/CLOSED)
  AdminIPV:             게이트 열릴 때 허용하는 Internal Priority Value
  AdminControlListLength: GCL 항목 수
  AdminCycleTime:       주기 (분자/분모로 표현)
  AdminBaseTime:        GCL 시작 기준 시각 (TAI)

GCL 항목:
  GateState:   OPEN (통과 허용) / CLOSED (즉시 폐기)
  IPV:         이 슬롯에서 허용되는 최소 우선순위
               -1 = 모든 우선순위 허용
               7  = PCP 7만 허용
  TimeInterval: 슬롯 지속 시간 (ns)
  OctetMaxSDU: 이 슬롯의 최대 프레임 크기 (0=제한 없음)

동작 예시:
  Stream 1 (관절 제어):
    GCL Entry 0: OPEN, IPV=5, Interval=100µs  ← TAS TC5 슬롯과 일치
    GCL Entry 1: CLOSED, IPV=-1, Interval=900µs

  결과: TAS TC5 슬롯(0~100µs)에서만 관절 제어 스트림 통과
        다른 시간에 도착한 제어 프레임은 즉시 폐기
        (Babbling ECU가 잘못된 시간에 전송해도 차단)
```

### IPV (Internal Priority Value)

```
IPV: 프레임이 스위치 내부에서 사용할 우선순위 재지정

일반 흐름: VLAN PCP → TC 매핑 (802.1Q 테이블)
PSFP 흐름: SGI에서 IPV 오버라이드

예: PCP=2로 도착한 프레임을 PSFP에서 IPV=5로 올림
  → TC5 큐로 전달 (원본 PCP 무시)
  → 관리자 실수로 낮은 PCP로 전송된 제어 트래픽도 올바르게 처리
```

---

## Flow Meter (토큰 버킷 알고리즘)

```
토큰 버킷 개념:
  ┌─────────────────────────────────────────────────┐
  │  버킷 용량 = CBS (Committed Burst Size, Byte)    │
  │                                                 │
  │  토큰 생성률 = CIR (Committed Information Rate) │
  │               (bps 단위)                        │
  │                                                 │
  │  프레임 도착 → 프레임 크기만큼 토큰 소비          │
  │  토큰 부족 → MarkRed (폐기 또는 DEI 마킹)        │
  │  토큰 충분 → MarkGreen (정상 통과)               │
  └─────────────────────────────────────────────────┘

Flow Meter 파라미터:
  CIR (Committed Information Rate): 보장 대역폭 (bps)
  CBS (Committed Burst Size):       허용 최대 버스트 (Byte)
  EIR (Excess Information Rate):    초과 허용 대역폭 (선택)
  EBS (Excess Burst Size):          초과 버스트 크기 (선택)
  DropOnYellow:                     Yellow 패킷 폐기 여부

설정 예시:
  Stream 1 (관절 제어): CIR=10Mbps, CBS=9000Byte
  → 10Mbps 초과 즉시 폐기 (Babbling Idiot 차단)

  Stream 2 (영상):      CIR=500Mbps, CBS=65536Byte
  → 카메라 버스트(64KB)는 허용, 이후 500Mbps 제한
```

---

## Babbling Idiot 방어

```
정상 동작:
  Joint ECU #1 ── 제어 패킷 10Mbps ──► TSN Switch ──► Main ECU

Babbling Idiot 발생:
  Joint ECU #1 (고장!) ── 패킷 1000Mbps ──► TSN Switch

802.1Qci 없는 경우:
  전체 네트워크 포화 → 다른 ECU 통신 불가 → 수술 중 시스템 마비

802.1Qci 적용:
  TSN Switch Ingress:
    Stream: ECU #1 (Src MAC 기준 식별)
    CIR: 10Mbps
    조치: 10Mbps 초과 즉시 폐기
  결과: ECU #1 고장에도 다른 ECU 정상 통신 유지

이중 보호 (SGI + Flow Meter 조합):
  SGI:         잘못된 시간에 전송된 프레임 차단
  Flow Meter:  과도한 대역폭 사용 제한
```

---

## Linux에서 PSFP 설정 (tc 사용)

```bash
# 스트림 필터 설정 (Linux 커널 5.14+)

# 1. 스트림 게이트 (SGI) 설정 - VLAN 10의 관절 제어 스트림
tc filter add dev eth0 ingress \
    protocol 802.1Q \
    flower \
    vlan_id 10 \
    src_mac aa:bb:cc:dd:ee:01 \
    action gate \
        base-time 1600000000000000000 \
        sched-entry OPEN  100000 -1 9000 \
        sched-entry CLOSE 900000 -1 -1
# sched-entry <OPEN|CLOSE> <interval_ns> <ipv> <max_octets>
# ipv=-1: 모든 우선순위, max_octets=-1: 제한 없음

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
tc -s filter show dev eth0 ingress | grep -A3 "dropped"

# 필터 삭제
tc filter del dev eth0 ingress
```

---

## SGI + TAS 연동 (이중 보호)

```
TAS (포트 수준):
  T=0~100µs: TC7 OPEN (안전 트래픽)
  T=100~1000µs: TC0 OPEN (BE 트래픽)

SGI (스트림 수준 - Safety ECU 스트림):
  T=0~100µs: Gate OPEN, IPV=7, maxOctets=9000
  T=100µs~: Gate CLOSED

결과:
  TAS: TC7 큐 전체 슬롯 보장
  SGI: Safety ECU 스트림만 TC7 슬롯에 통과
  → TAS + SGI 이중 보호 = 완전한 스트림 격리

허용 여부 매트릭스:
              TAS TC7 슬롯  TAS TC0 슬롯
Safety 스트림     O             X (SGI 닫힘)
제어 스트림       O (IPV≥5)     X
BE 트래픽         X (TAS 닫힘)  O
```

---

## 모니터링: 이상 감지

```bash
# PSFP 통계 조회 (스위치 관리 인터페이스 또는 NETCONF/YANG)
# Stream Filter 폐기 카운터 예시:
# ingress_frames_received: 100000
# ingress_frames_passed:   99850
# ingress_frames_dropped:   150  ← 이상 발생 지표

# Linux tc 통계
tc -s filter show dev eth0 ingress
# filter ... (statistics):
#   Sent 99850 bytes 850 pkt (dropped 150, ...)

# 임계값 초과 시 알람 생성 (예시 스크립트)
DROPPED=$(tc -s filter show dev eth0 ingress | grep dropped | awk '{print $2}')
if [ "$DROPPED" -gt 100 ]; then
    logger -p local0.warning "PSFP: Stream filter dropped $DROPPED frames!"
fi
```

---

## 의료 로봇 PSFP 정책 예시

```
스트림 분류 및 정책:

Stream 1 (안전 제어):
  식별: VLAN=10, Src=Safety ECU MAC
  CIR: 5 Mbps, CBS: 9000 Byte
  SGI: TAS TC7 슬롯에서만 OPEN
  초과 시: 즉시 폐기 + 시스템 알람 트리거

Stream 2 (관절 제어):
  식별: VLAN=20, Src=Joint ECU MAC 그룹
  CIR: 50 Mbps, CBS: 64 KB
  SGI: TAS TC5 슬롯에서만 OPEN
  초과 시: DEI 마킹 (폐기 후보)

Stream 3 (영상):
  식별: VLAN=30, Src=Camera MAC
  CIR: 400 Mbps, CBS: 1 MB
  SGI: 항상 OPEN (Best Effort 큐 사용)
  초과 시: 폐기 (영상 일부 손실 허용)

Stream 4 (진단 DoIP):
  식별: VLAN=40, TCP Port 13400
  CIR: 100 Mbps, CBS: 1 MB
  SGI: 항상 OPEN
  초과 시: 폐기
```

---

## Reference
- [IEEE 802.1Qci-2017 - Per-Stream Filtering and Policing](https://standards.ieee.org/ieee/802.1Qci/6045/)
- [TSN Task Group - 802.1Qci](https://1.ieee802.org/tsn/802-1qci/)
- [Linux tc-flower 필터](https://man7.org/linux/man-pages/man8/tc-flower.8.html)
- [Linux tc-police 폴리서](https://man7.org/linux/man-pages/man8/tc-police.8.html)
- [Linux tc-gate (TAPRIO/SGI)](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
