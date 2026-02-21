# MAC 주소와 IP 주소의 차이 (MAC vs IP Address)

## 개요
네트워크 통신에서 장치를 식별하는 데 MAC 주소와 IP 주소 두 가지가 사용됩니다. 이 두 주소는 역할, 사용 계층, 범위가 서로 달라 함께 사용됩니다. Ethernet 기반 시스템을 디버깅할 때 어떤 계층의 주소 문제인지 파악하는 것이 문제 해결의 첫걸음입니다.

---

## 비교 요약

| 특성 | MAC 주소 | IP 주소 |
|------|----------|---------|
| OSI 계층 | L2 (데이터 링크) | L3 (네트워크) |
| 전송 단위 | 프레임(Frame) | 패킷(Packet) |
| 주소 길이 | 48비트 (6바이트) | IPv4: 32비트 / IPv6: 128비트 |
| 표기법 | `AA:BB:CC:DD:EE:FF` (16진수) | `192.168.1.1` / `fe80::1` |
| 부여 방식 | 제조사 하드웨어 할당 | 관리자 설정 또는 DHCP |
| 유효 범위 | 동일 서브넷(L2 도메인) 내 | 글로벌 인터넷 포함 |
| 변경 가능 여부 | 소프트웨어로 변경 가능 (MAC Spoofing) | 언제든지 변경 가능 |

---

## MAC 주소 (Media Access Control Address)

### 구조
```
┌─────────────────┬─────────────────────────┐
│  OUI (3 Byte)   │  NIC Specific (3 Byte)  │
│  제조사 식별자  │  장치 고유 번호          │
└─────────────────┴─────────────────────────┘
예시: AA:BB:CC:DD:EE:FF
      └────────┘ └────────┘
         OUI       NIC 고유값
```

- **OUI (Organizationally Unique Identifier)**: IEEE에서 제조사에 할당. `AA:BB:CC` 부분
- **비트 의미**: 첫 바이트의 LSB가 `0` → Unicast, `1` → Multicast
- **브로드캐스트 MAC**: `FF:FF:FF:FF:FF:FF`

### 특수 MAC 주소
| 주소 | 의미 |
|------|------|
| `FF:FF:FF:FF:FF:FF` | 브로드캐스트 (모든 장치) |
| `01:00:5E:xx:xx:xx` | IPv4 멀티캐스트 |
| `33:33:xx:xx:xx:xx` | IPv6 멀티캐스트 |
| `01:80:C2:00:00:0E` | LLDP (Link Layer Discovery Protocol) |
| `01:1B:19:00:00:00` | IEEE 1588 PTP 이벤트 메시지 |

### MAC 주소 확인 명령어
```bash
# Linux
ip link show
# 또는
cat /sys/class/net/eth0/address

# 출력 예시
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
```

---

## IP 주소 (Internet Protocol Address)

### IPv4 주소 구조
```
192  .  168  .   1   .  100
 8비트  8비트   8비트   8비트  = 32비트 총
└────────────────┘ └─────────┘
   네트워크 주소    호스트 주소
  (서브넷에 따라 다름)
```

### 서브넷 마스크 (Subnet Mask)
```
IP 주소:     192.168.1.100  = 11000000.10101000.00000001.01100100
서브넷 마스크: 255.255.255.0  = 11111111.11111111.11111111.00000000
                                └─────────────────────────┘ └──────┘
                                        네트워크 부분        호스트 부분

네트워크 주소: 192.168.1.0
브로드캐스트:  192.168.1.255
사용 가능 호스트: 192.168.1.1 ~ 192.168.1.254 (총 254개)
```

### CIDR 표기법
```
192.168.1.0/24   → 서브넷 마스크 255.255.255.0  → 호스트 254개
192.168.1.0/16   → 서브넷 마스크 255.255.0.0    → 호스트 65534개
10.0.0.0/8       → 서브넷 마스크 255.0.0.0      → 호스트 16,777,214개
```

### IPv4 사설 주소 범위
| 클래스 | 범위 | CIDR | 용도 |
|--------|------|------|------|
| A | 10.0.0.0 ~ 10.255.255.255 | /8 | 대규모 내부망 |
| B | 172.16.0.0 ~ 172.31.255.255 | /12 | 중규모 내부망 |
| C | 192.168.0.0 ~ 192.168.255.255 | /16 | 소규모 내부망 |

### IPv6 주소 구조
```
fe80::1a2b:3c4d:5e6f:7g8h/64
└──┘   └──────────────────┘
링크로컬  인터페이스 ID (64비트)
```
- **링크 로컬**: `fe80::/10` — 동일 링크 내에서만 유효 (라우팅 불가)
- **글로벌 유니캐스트**: `2000::/3` — 인터넷 라우팅 가능

---

## ARP (Address Resolution Protocol): MAC ↔ IP 연결고리

동일 서브넷 내에서 IP 주소로부터 MAC 주소를 알아내는 프로토콜입니다.

```
호스트 A (192.168.1.1)          스위치          호스트 B (192.168.1.2)
      │                            │                    │
      │── ARP Request (Broadcast) ──────────────────────►
      │   "192.168.1.2의 MAC 주소는?" (FF:FF:FF:FF:FF:FF로 전송)
      │                            │                    │
      │◄── ARP Reply (Unicast) ────────────────────────│
      │    "나는 192.168.1.2, MAC은 BB:BB:BB:BB:BB:BB"  │
      │                            │                    │
      │ (이후 ARP 캐시에 저장하여 재사용)               │
```

### ARP 캐시 확인
```bash
# Linux
ip neigh show
# 또는
arp -n

# 출력 예시
192.168.1.2 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
192.168.1.1 dev eth0 lladdr 11:22:33:44:55:66 STALE
```

### ARP 관련 문제
```
문제: IP는 맞는데 통신 안 됨
원인: ARP 캐시에 잘못된 MAC 주소 저장 (ARP Cache Poisoning 또는 IP 충돌)
해결: ip neigh flush all (ARP 캐시 초기화)

문제: 동일 IP에 두 장치가 존재
원인: IP 주소 중복 할당
증상: Wireshark에서 "Duplicate IP address detected" 경고
해결: ip addr show 로 모든 인터페이스 IP 확인 후 수정
```

---

## IP 구성 및 확인 명령어

```bash
# 현재 IP 설정 확인
ip addr show

# 정적 IP 설정 (임시, 재부팅 시 사라짐)
ip addr add 192.168.1.100/24 dev eth0
ip route add default via 192.168.1.1

# 기본 게이트웨이 확인
ip route show
# 예시 출력:
# default via 192.168.1.1 dev eth0
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100

# Ping 테스트 (네트워크 연결 확인)
ping -c 4 192.168.1.2

# 경로 추적
traceroute 8.8.8.8
```

---

## 실무 트러블슈팅: 계층별 접근

```
증상: 디바이스 A에서 B로 통신 불가

Step 1: 물리 연결 확인 (L1)
  → ip link show eth0  (UP/DOWN 확인)
  → ethtool eth0  (링크 속도/이중화 협상 확인)

Step 2: 동일 서브넷 내 MAC 통신 확인 (L2)
  → ping -I eth0 192.168.1.2  (ARP → ICMP 순서로 동작)
  → ip neigh show  (ARP 테이블 확인)

Step 3: IP 라우팅 확인 (L3)
  → ip route show  (라우팅 테이블)
  → traceroute 192.168.1.2  (홉별 경로 확인)

Step 4: 포트/서비스 확인 (L4)
  → ss -tlnp  (리스닝 포트 확인)
  → nc -zv 192.168.1.2 30490  (특정 포트 연결 테스트)
```

---

## 임베디드/로봇 시스템에서의 IP 설계 예시

```
로봇 내부 네트워크 설계 (192.168.10.0/24)

192.168.10.1   → 메인 제어기 (Main ECU)
192.168.10.2   → 조인트 제어기 #1
192.168.10.3   → 조인트 제어기 #2
192.168.10.4   → 비전 시스템
192.168.10.5   → 안전 모니터링 ECU
192.168.10.10  → TSN 스위치 관리 포트
192.168.10.254 → 게이트웨이 (병원 네트워크로 연결)

VLAN 10: 안전 제어 트래픽 (고우선순위)
VLAN 20: 센서/영상 트래픽
VLAN 30: 관리/진단 트래픽 (DoIP)
```

---

## Reference
- [RFC 826 - An Ethernet Address Resolution Protocol](https://datatracker.ietf.org/doc/html/rfc826)
- [RFC 791 - Internet Protocol (IPv4)](https://datatracker.ietf.org/doc/html/rfc791)
- [RFC 4291 - IP Version 6 Addressing Architecture](https://datatracker.ietf.org/doc/html/rfc4291)
- [IEEE 802.3 Ethernet Standard](https://standards.ieee.org/ieee/802.3/7222/)
- [IEEE OUI Registry](https://regauth.standards.ieee.org/standards-ra-web/pub/view.html#registries)
