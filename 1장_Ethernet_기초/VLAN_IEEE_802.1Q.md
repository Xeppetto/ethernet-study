# VLAN과 IEEE 802.1Q

## 개요
VLAN(Virtual LAN)은 물리적인 네트워크 구성에 관계없이 소프트웨어적으로 네트워크를 논리적으로 분리하는 기술입니다. 하나의 물리적 스위치에 연결된 장치들을 마치 별도의 스위치에 연결된 것처럼 격리할 수 있습니다.

Ethernet 기반 산업 시스템에서 VLAN은 **안전 제어 트래픽과 일반 트래픽의 격리**, **보안 강화**, **대역폭 효율화**를 위한 필수 기술입니다.

---

## VLAN이 필요한 이유

```
VLAN 없는 경우 (문제):
  스위치에 안전 제어 ECU, 카메라, 진단 PC가 모두 연결
  → 브로드캐스트 범람: 진단 PC의 대량 트래픽이 안전 제어 ECU에 영향
  → 보안 위협: 카메라 해킹 시 안전 제어 트래픽에 접근 가능

VLAN 적용 후 (해결):
  VLAN 10 (안전 제어): 안전 관련 ECU만 참여 → 격리됨
  VLAN 20 (센서/영상): 카메라, 라이다 → 대용량 트래픽 분리
  VLAN 30 (진단/관리): 진단 PC, OTA 서버 → Best Effort
```

---

## IEEE 802.1Q VLAN Tag 구조

Ethernet 프레임에 4바이트의 VLAN 태그가 추가됩니다.

```
기본 Ethernet 프레임:
┌──────────┬──────────┬────────┬──────────────┬─────┐
│ Dst MAC  │ Src MAC  │  Type  │   Payload    │ FCS │
│  6 Byte  │  6 Byte  │ 2 Byte │ 46~1500 Byte │4 Byte│
└──────────┴──────────┴────────┴──────────────┴─────┘

802.1Q Tagged 프레임:
┌──────────┬──────────┬──────────────┬────────┬──────────────┬─────┐
│ Dst MAC  │ Src MAC  │  802.1Q Tag  │  Type  │   Payload    │ FCS │
│  6 Byte  │  6 Byte  │   4 Byte     │ 2 Byte │ 46~1496 Byte │4 Byte│
└──────────┴──────────┴──────┬───────┴────────┴──────────────┴─────┘
                              │
                    ┌─────────▼──────────────────────────┐
                    │  TPID (2 Byte)  │  TCI (2 Byte)   │
                    │   0x8100        │ PCP│DEI│  VID    │
                    │  (VLAN 식별)    │ 3b │1b │  12b    │
                    └────────────────┴────┴───┴─────────┘
                                        │    │     │
                                        │    │   VLAN ID (0~4094)
                                        │  Drop Eligible Indicator
                                      Priority Code Point (0~7)
```

### 필드 설명
| 필드 | 크기 | 설명 |
|------|------|------|
| TPID | 2 Byte | `0x8100` (VLAN 프레임 식별자) |
| PCP | 3 bit | Priority Code Point (IEEE 802.1p), 0~7 우선순위 |
| DEI | 1 bit | Drop Eligible Indicator (혼잡 시 우선 폐기 여부) |
| VID | 12 bit | VLAN ID (0~4094, 0과 4095는 예약) |

---

## VLAN 유형

### Access VLAN (Untagged)
- 엔드 포인트(ECU, PC 등)가 연결되는 포트
- 포트에 하나의 VLAN만 설정
- 프레임은 VLAN 태그 없이 송수신 (스위치가 태그 추가/제거)

```
ECU (태그 없는 프레임) ──► [Access 포트, VLAN 10] ──► 스위치 내부 (VLAN 10 태그 추가)
```

### Trunk VLAN (Tagged)
- 스위치 간 연결 또는 복수 VLAN을 처리하는 서버/라우터 연결
- 여러 VLAN의 태그된 프레임이 동시에 전달됨

```
스위치 A ──[Trunk 포트]──► 스위치 B
             VLAN 10, 20, 30 모두 전달 (각각 태그 포함)
```

### Native VLAN
- Trunk 포트에서 태그 없이 전달되는 VLAN (기본값: VLAN 1)
- 보안상 Native VLAN을 사용하지 않는 VLAN으로 변경 권장

---

## Linux에서 VLAN 설정

### VLAN 인터페이스 생성
```bash
# VLAN 모듈 로드 (대부분 자동)
modprobe 8021q

# VLAN 인터페이스 생성 (eth0 기반, VLAN ID 10)
ip link add link eth0 name eth0.10 type vlan id 10

# VLAN 인터페이스 활성화 및 IP 할당
ip link set eth0.10 up
ip addr add 192.168.10.1/24 dev eth0.10

# VLAN 인터페이스 목록 확인
ip link show type vlan

# VLAN 설정 확인
cat /proc/net/vlan/eth0.10
```

### 영구적 VLAN 설정 (systemd-networkd)
```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
VLAN=vlan10
VLAN=vlan20
VLAN=vlan30

# /etc/systemd/network/20-vlan10.netdev
[NetDev]
Name=vlan10
Kind=vlan

[VLAN]
Id=10

# /etc/systemd/network/20-vlan10.network
[Match]
Name=vlan10

[Network]
Address=192.168.10.1/24
```

### QoS와 VLAN 결합 (PCP 설정)
```bash
# 소켓에서 VLAN PCP 설정 (SO_PRIORITY → PCP 매핑)
# 프로그램 내에서 소켓 옵션 설정
setsockopt(sock, SOL_SOCKET, SO_PRIORITY, &priority, sizeof(priority));

# 이 우선순위가 VLAN PCP로 매핑됨
# /proc/net/vlan/eth0.10 에서 egress map 확인
```

---

## VLAN 설계 원칙

### 산업/의료 로봇 VLAN 설계 예시
```
물리 스위치 (TSN 지원)
├── VLAN 10: 안전 제어 (Safety-Critical)
│   ├── 안전 ECU #1 (192.168.10.1)
│   ├── 안전 ECU #2 (192.168.10.2)
│   └── TSN TAS: PCP=5, 전용 타임슬롯 할당
│
├── VLAN 20: 실시간 제어 (Real-time Control)
│   ├── 조인트 제어 ECU (192.168.20.x)
│   ├── 센서 허브 (192.168.20.x)
│   └── TSN CBS: PCP=4, Credit-Based Shaping
│
├── VLAN 30: 영상/센서 (Video & Sensor)
│   ├── 수술 카메라 (192.168.30.x)
│   ├── 라이다 (192.168.30.x)
│   └── PCP=3, 대역폭 제한 설정
│
└── VLAN 99: 관리/진단 (Management)
    ├── 진단 PC (192.168.99.x)
    ├── OTA 서버 게이트웨이
    └── PCP=0, Best Effort
```

### 라우터 없이 VLAN 간 통신 방지
```
VLAN 10 ──────────────────────────────── (안전 제어망)
                  (물리 스위치)
VLAN 30 ──────────────────────────────── (진단망)

→ VLAN 10과 VLAN 30 간 직접 통신 불가
→ 방화벽/라우터 거쳐야 가능 (접근 제어 적용)
→ 카메라 보안 취약점이 안전 제어망에 영향 불가
```

---

## QinQ (IEEE 802.1ad) - VLAN 스태킹

S-VLAN(서비스 제공자)과 C-VLAN(고객)을 중첩하는 기술로, 대규모 클라우드 연결 또는 다중 로봇 관리 플랫폼에서 활용됩니다.

```
┌──────────┬──────────┬────────────────┬────────────────┬──────────┐
│ Dst MAC  │ Src MAC  │ S-Tag (0x88A8) │ C-Tag (0x8100) │ Payload  │
│          │          │ S-VLAN ID: 100 │ C-VLAN ID: 10  │          │
└──────────┴──────────┴────────────────┴────────────────┴──────────┘
  병원 A의 로봇 팜 = S-VLAN 100
  병원 A 내 수술실 A = C-VLAN 10
```

---

## Wireshark로 VLAN 분석

```bash
# VLAN 태그된 트래픽 필터
vlan.id == 10

# VLAN 10 내의 SOME/IP 트래픽
vlan.id == 10 and someip

# PCP 우선순위 필터 (PCP=5인 제어 트래픽)
vlan.priority == 5

# 특정 VLAN의 ARP 트래픽
vlan.id == 20 and arp
```

---

---

## VLAN 보안 취약점 (의료기기 사이버보안 관련)

IEC 62304, ISO/SAE 21434, UNECE R155를 준수하는 의료 로봇 설계에서 VLAN 보안 취약점을 이해하는 것은 필수입니다.

### VLAN Hopping 공격 1: Double Tagging

```
공격 원리:
  공격자 PC가 S-Tag(Native VLAN) + C-Tag(목표 VLAN) 이중 태그 전송
  첫 번째 스위치: S-Tag 제거 → C-Tag만 남음 → 목표 VLAN으로 전달
  두 번째 스위치: 목표 VLAN 프레임으로 인식 → 안전망 침투!

조건: 공격자 포트의 Native VLAN = Outer Tag VLAN ID

시나리오:
  공격자 (VLAN 30) → [VLAN 30 outer][VLAN 10 inner] 이중 태그 전송
  스위치 1: VLAN 30 outer 제거
  스위치 2: VLAN 10으로 포워딩 → 안전 ECU 망 침투!

방어:
  1. Native VLAN을 사용하지 않는 전용 ID로 변경 (예: VLAN 999)
  2. Trunk 포트에서 Native VLAN 태깅 강제
  3. 사용하지 않는 전용 VLAN ID를 Native VLAN으로 설정
```

### VLAN Hopping 공격 2: Switch Spoofing

```
공격 원리:
  공격자가 DTP(Dynamic Trunking Protocol) 메시지 전송
  스위치가 공격자 포트를 Trunk 포트로 오인
  → 모든 VLAN 트래픽에 접근 가능

방어:
  모든 Access 포트를 명시적으로 고정 설정:
  # Linux bridge 환경에서 포트를 명시적 Access로 설정
  bridge vlan add dev eth1 vid 10 pvid untagged  # VLAN 10 Access 포트
```

### 의료 로봇 VLAN 보안 설계 체크리스트

```
✓ Native VLAN 변경 (기본 VLAN 1 사용 금지)
✓ 사용하지 않는 포트: 비활성화 또는 격리 VLAN 할당
✓ 안전 제어 VLAN에 ACL(Access Control List) 적용
✓ IGMP Snooping 활성화 (멀티캐스트 범람 방지)
✓ Port Security: 포트당 허용 MAC 주소 제한
✓ Storm Control: Broadcast/Multicast/Unknown Unicast 임계값 설정
✓ VLAN 간 통신: 방화벽 통과 후에만 허용
✓ 관리 VLAN을 안전 VLAN과 완전 분리
✓ 주기적 VLAN 설정 감사(Audit)
```

---

## PVST vs MSTP 비교

다수 VLAN 환경에서 Spanning Tree 방식을 적절히 선택해야 합니다.

### PVST (Per-VLAN Spanning Tree) - Cisco 독점

```
특성:
  - VLAN마다 독립 STP 인스턴스 실행 → VLAN별 경로 최적화 가능
  - CPU/메모리 부하: VLAN 수에 비례 증가
  - 비표준 (Cisco 전용, 멀티벤더 호환성 없음)
```

### MSTP (Multiple Spanning Tree Protocol) - IEEE 802.1s (표준)

```
특성:
  - 다수 VLAN을 몇 개의 MST 인스턴스로 그룹화
  - 표준 기반 멀티벤더 호환
  - CPU/메모리 효율적

설계 예 (의료 로봇):
  MST 인스턴스 0 (CIST): VLAN 1, 99 (관리)
  MST 인스턴스 1: VLAN 10, 20 (안전/제어) → Root = 주 스위치
  MST 인스턴스 2: VLAN 30, 40 (영상/Best Effort) → Root = 부 스위치

TSN 환경 권장:
  루프 없는 토폴로지 설계 → MSTP는 비상 Fallback으로만 사용
  FRER(802.1CB)으로 이중화 → STP 전환 없이 무중단 경로 전환
```

---

## TSN과 VLAN 연동

TSN 스케줄링(TAS, CBS)은 VLAN PCP 값을 기반으로 트래픽을 분류합니다.

```
VLAN PCP ─► TSN 큐 매핑 예시 (1Gbps 링크):

PCP 7: 네트워크 제어(STP BPDU, LLDP, PTP)  → Strict Priority
PCP 5: 안전 제어 (ST Traffic)               → TAS Gate 0 (100µs 슬롯)
PCP 4: 실시간 제어 (AVB Class A)            → CBS idleSlope=400Mbps
PCP 3: 영상 (AVB Class B)                   → CBS idleSlope=200Mbps
PCP 0: Best Effort (진단, OTA)              → FIFO

TAS Gate Control List 예시 (400µs 사이클):
  T=0   ~ T=100µs: PCP 5 게이트 OPEN만      (ST 전용 슬롯)
  T=100 ~ T=300µs: PCP 4, 3 게이트 OPEN     (CBS + BE 허용)
  T=300 ~ T=400µs: PCP 0, 3 게이트 OPEN     (BE + 영상)
```

---

## Reference
- [IEEE 802.1Q-2022 - Bridges and Bridged Networks](https://standards.ieee.org/ieee/802.1Q/6844/)
- [IEEE 802.1ad - Provider Bridges (QinQ)](https://standards.ieee.org/ieee/802.1ad/3979/)
- [IEEE 802.1s - Multiple Spanning Tree Protocol (MSTP)](https://standards.ieee.org/ieee/802.1s/3968/)
- [IEEE 802.1CB-2017 - Frame Replication and Elimination for Reliability](https://standards.ieee.org/ieee/802.1CB/6421/)
- [Linux VLAN Howto](https://www.kernel.org/doc/html/latest/networking/vlan.html)
- [RFC 5517 - Cisco Systems' Private VLANs](https://datatracker.ietf.org/doc/html/rfc5517)
- [ENISA - Network Security for Medical Devices](https://www.enisa.europa.eu/)
