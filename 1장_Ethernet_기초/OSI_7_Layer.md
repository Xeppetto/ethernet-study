# OSI 7 Layer (OSI 7 계층 모델)

## 개요
OSI 7 계층 모델(Open Systems Interconnection Reference Model)은 국제 표준화 기구(ISO)에서 개발한 모델로, 컴퓨터 네트워크 프로토콜 디자인과 통신을 계층별로 나누어 설명합니다. 이기종 시스템 간의 통신 호환성을 보장하고, 네트워크 통신 과정을 단계별로 파악하여 문제 해결을 용이하게 하는 데 목적이 있습니다.

산업용 Ethernet(Ethernet APL, TSN, SPE 등)으로 전환되는 현재 흐름에서, OSI 모델은 어느 계층에서 어떤 기술이 동작하는지 이해하는 핵심 틀입니다.

---

## 계층 구조 전체 그림

```
송신 측                                   수신 측
┌──────────────────────┐         ┌──────────────────────┐
│ L7 - 응용 계층       │◄───────►│ L7 - 응용 계층       │  HTTP, SOME/IP, DDS
├──────────────────────┤         ├──────────────────────┤
│ L6 - 표현 계층       │         │ L6 - 표현 계층       │  암호화/복호화, 직렬화
├──────────────────────┤         ├──────────────────────┤
│ L5 - 세션 계층       │         │ L5 - 세션 계층       │  세션 관리
├──────────────────────┤         ├──────────────────────┤
│ L4 - 전송 계층       │         │ L4 - 전송 계층       │  TCP/UDP, 포트
├──────────────────────┤         ├──────────────────────┤
│ L3 - 네트워크 계층   │         │ L3 - 네트워크 계층   │  IP, 라우팅
├──────────────────────┤         ├──────────────────────┤
│ L2 - 데이터 링크 계층│         │ L2 - 데이터 링크 계층│  MAC, Ethernet 프레임
├──────────────────────┤         ├──────────────────────┤
│ L1 - 물리 계층       │◄───────►│ L1 - 물리 계층       │  케이블, PHY, 신호
└──────────────────────┘  (물리)  └──────────────────────┘
```

**데이터 캡슐화(Encapsulation)**: 송신 시 L7→L1 방향으로 각 계층이 헤더를 추가하고, 수신 시 L1→L7 방향으로 헤더를 제거합니다.

---

## 계층별 역할과 기능

### L1 - 물리 계층 (Physical Layer)
물리적인 연결과 전기/광학 신호를 담당합니다.

| 항목 | 내용 |
|------|------|
| 전송 단위 | 비트(Bit) |
| 주요 장비 | 케이블, 커넥터, 리피터, 허브, PHY 칩 |
| 핵심 기능 | 비트 스트림 → 전기/광학/무선 신호 변환 |

**주요 사양 예시:**
- **100BASE-TX**: 100Mbps, CAT5 케이블, 최대 100m
- **100BASE-T1**: 100Mbps, 단일 쌍(STP), 최대 15m (차량/로봇용)
- **1000BASE-T**: 1Gbps, CAT5e 케이블, 최대 100m
- **10GBASE-SR**: 10Gbps, 멀티모드 광섬유, 최대 300m

**디버깅 포인트**: 링크 LED 꺼짐, 자동 협상(Auto-Negotiation) 실패, CRC 오류 급증 → L1 문제 의심

---

### L2 - 데이터 링크 계층 (Data Link Layer)
인접한 두 장치 간의 신뢰성 있는 프레임 전송을 담당합니다.

| 항목 | 내용 |
|------|------|
| 전송 단위 | 프레임(Frame) |
| 주소 체계 | MAC 주소 (48비트, OUI + NIC) |
| 주요 장비 | Ethernet 스위치, 브리지 |
| 핵심 기능 | 오류 검출(CRC), 흐름 제어, MAC 학습 |

**Ethernet 프레임 구조:**
```
┌────────┬────────┬────────┬───────┬───────────────┬─────┐
│ Dst MAC│ Src MAC│VLAN Tag│  Type │    Payload    │ FCS │
│ 6 Byte │ 6 Byte │ 4 Byte │ 2 Byte│  46~1500 Byte │4 Byte│
└────────┴────────┴───────┴───────┴───────────────┴─────┘
```
- **VLAN Tag(802.1Q)**: 선택적. 트래픽 논리적 분리에 사용
- **EtherType**: 0x0800(IPv4), 0x0806(ARP), 0x86DD(IPv6), 0x8100(VLAN), 0x8892(PROFINET)
- **FCS (Frame Check Sequence)**: CRC-32로 오류 검출

**TSN과의 관계**: IEEE 802.1Q, 802.1Qbv, 802.1Qav 등 TSN 표준이 모두 L2에서 동작합니다.

---

### L3 - 네트워크 계층 (Network Layer)
여러 네트워크를 거쳐 목적지까지 경로를 찾아 패킷을 전달합니다.

| 항목 | 내용 |
|------|------|
| 전송 단위 | 패킷(Packet) |
| 주소 체계 | IPv4(32비트), IPv6(128비트) |
| 주요 장비 | 라우터, L3 스위치 |
| 핵심 프로토콜 | IP, ICMP, ARP, OSPF, BGP |

**IPv4 헤더 주요 필드:**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─────────┬─────────┬──────────────────────────────────────────────┤
│ Version │   IHL   │    DSCP/ECN    │         Total Length        │
├─────────┴─────────┴──────────────────────────────────────────────┤
│          TTL        │    Protocol    │       Header Checksum      │
├──────────────────────────────────────────────────────────────────┤
│                       Source IP Address                          │
├──────────────────────────────────────────────────────────────────┤
│                    Destination IP Address                        │
└──────────────────────────────────────────────────────────────────┘
```
- **DSCP (Differentiated Services Code Point)**: QoS 우선순위 표시. TSN 환경에서 트래픽 분류에 활용

---

### L4 - 전송 계층 (Transport Layer)
종단 간(End-to-End) 신뢰성 있는 데이터 전송을 담당합니다.

| 항목 | 내용 |
|------|------|
| 전송 단위 | 세그먼트(TCP) / 데이터그램(UDP) |
| 식별자 | 포트 번호 (0~65535) |

**TCP vs UDP 비교:**

| 특성 | TCP | UDP |
|------|-----|-----|
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | ACK/재전송 보장 | 미보장 |
| 순서 보장 | 보장 | 미보장 |
| 오버헤드 | 높음 | 낮음 |
| 지연 | 상대적으로 높음 | 낮음 |
| 활용 사례 | DoIP 진단, OTA, 파일 전송 | SOME/IP 이벤트, DDS, 실시간 제어 |

**자주 사용되는 포트:**
- `13400`: DoIP (UDP/TCP)
- `30490`: SOME/IP Service Discovery (UDP)
- `30501`: SOME/IP (기본값, 설정 가능)
- `1883`: MQTT
- `7400~7500`: DDS (RTPS)

---

### L5~L7 - 세션/표현/응용 계층

실제 구현에서 TCP/IP 모델에서는 이 3개 계층이 하나의 "응용 계층"으로 통합되어 사용됩니다.

| 계층 | 역할 | 관련 기술 |
|------|------|-----------|
| L5 세션 | 통신 세션 수립/유지/종료 | TLS 세션, SOME/IP SD, DoIP 논리 연결 |
| L6 표현 | 데이터 인코딩/암호화/압축 | TLS/DTLS 암호화, SOME/IP 직렬화, Protobuf |
| L7 응용 | 사용자 애플리케이션 | SOME/IP, DDS, MQTT, OPC-UA, HTTP/REST, DoIP |

---

## TCP/IP 4계층 모델과 비교

실무에서는 TCP/IP 4계층 모델이 더 많이 사용됩니다.

```
OSI 7계층          TCP/IP 4계층       주요 프로토콜
──────────────     ──────────────     ──────────────────────────────
L7 응용 계층   ┐
L6 표현 계층   ├──→ 응용 계층        SOME/IP, DDS, MQTT, HTTP, DoIP
L5 세션 계층   ┘
──────────────     ──────────────
L4 전송 계층   ───→ 전송 계층        TCP, UDP
──────────────     ──────────────
L3 네트워크    ───→ 인터넷 계층      IP, ICMP, ARP
──────────────     ──────────────
L2 데이터링크  ┐
L1 물리 계층   ├──→ 네트워크 접근    Ethernet, Wi-Fi, 100BASE-T1
               ┘
```

---

## 계층별 문제 진단 방법

```
문제 현상                       확인 계층    확인 명령어/도구
───────────────────────────     ─────────    ──────────────────────────
링크 자체가 안 올라옴           L1           ethtool eth0 | grep Link
MAC 주소 학습 안 됨             L2           ip neigh show / arp -a
Ping 안 됨 (같은 서브넷)        L2/L3        arp -n, ip route show
Ping 안 됨 (다른 서브넷)        L3           ip route show, traceroute
TCP 연결 안 됨                  L4           netstat -an, ss -tlnp
애플리케이션 오류               L7           Wireshark SOME/IP 필터
```

**Wireshark 계층별 필터:**
```
eth.dst == ff:ff:ff:ff:ff:ff    # L2: 브로드캐스트 프레임
ip.addr == 192.168.1.1          # L3: 특정 IP 패킷
tcp.port == 13400               # L4: DoIP TCP 트래픽
udp.port == 30490               # L4: SOME/IP SD 트래픽
someip                          # L7: SOME/IP 애플리케이션 트래픽
```

---

## 산업용 Ethernet 프로토콜의 OSI 매핑

| 프로토콜 | L1 | L2 | L3 | L4 | L7 | 용도 |
|----------|----|----|----|----|-----|------|
| SOME/IP | - | Ethernet | IP | UDP/TCP | SOME/IP | 서비스 지향 통신 |
| DDS/RTPS | - | Ethernet | IP | UDP | DDS | 실시간 pub/sub |
| DoIP | - | Ethernet | IP | UDP/TCP | DoIP | 차량 진단 |
| OPC-UA | - | Ethernet | IP | TCP | OPC-UA | 산업 자동화 |
| PROFINET | - | Ethernet | (IP) | - | PROFINET | 공장 자동화 |
| EtherCAT | - | Ethernet | - | - | EtherCAT | 모션 제어 |

---

## Reference
- [ISO/IEC 7498-1:1994 - OSI Basic Reference Model](https://www.iso.org/standard/20269.html)
- [RFC 1122 - Requirements for Internet Hosts](https://datatracker.ietf.org/doc/html/rfc1122)
- [IEEE 802.3 Ethernet Standard](https://standards.ieee.org/ieee/802.3/7222/)
- [Wikipedia - OSI model](https://en.wikipedia.org/wiki/OSI_model)
