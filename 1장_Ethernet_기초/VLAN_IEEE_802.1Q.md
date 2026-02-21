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

## Reference
- [IEEE 802.1Q-2022 - Bridges and Bridged Networks](https://standards.ieee.org/ieee/802.1Q/6844/)
- [IEEE 802.1ad - Provider Bridges (QinQ)](https://standards.ieee.org/ieee/802.1ad/3979/)
- [Linux VLAN Howto](https://www.kernel.org/doc/html/latest/networking/vlan.html)
- [RFC 5517 - Cisco Systems' Private VLANs: Scalable Security in a Multi-Client Environment](https://datatracker.ietf.org/doc/html/rfc5517)
