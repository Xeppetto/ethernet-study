# Unicast, Multicast, Broadcast, Anycast

## 개요
네트워크에서 데이터를 전송하는 방식은 수신 대상의 범위에 따라 Unicast, Multicast, Broadcast, Anycast로 구분됩니다. 각 방식의 특성을 이해하고 적절히 활용하는 것은 네트워크 설계의 효율성과 실시간성에 직접적인 영향을 미칩니다.

---

## 4가지 전송 방식 비교

```
Unicast (1:1)          Multicast (1:그룹)
  A ──────► B            A ──────► B
                         A ──────► C
                         A ──────► D (그룹 구성원만)

Broadcast (1:전체)     Anycast (1:가장 가까운 1개)
  A ──────► B            A ──────► B (B, C, D 중
  A ──────► C            A ──────► 가장 가까운 한 곳)
  A ──────► D
  A ──────► E ... (전체)
```

| 특성 | Unicast | Multicast | Broadcast | Anycast |
|------|---------|-----------|-----------|---------|
| 수신 대상 | 1개 | 그룹 구성원 | 전체 | 가장 가까운 1개 |
| L2 MAC | 특정 MAC | 01:00:5E:xx:xx:xx | FF:FF:FF:FF:FF:FF | 특정 MAC (라우팅 기반) |
| L3 IP | 특정 IP | 224.0.0.0/4 | 서브넷 브로드캐스트 | 여러 노드가 동일 IP 공유 |
| 네트워크 부하 | 낮음 (1:1) | 중간 (그룹) | 높음 (전체) | 낮음 |
| 라우터 통과 | 가능 | 가능 (PIM 필요) | 불가 | 가능 |
| 프로토콜 활용 | TCP, UDP | DDS, PTP, IGMP | ARP, DHCP, SOME/IP SD | DNS, NTP |

---

## Unicast (유니캐스트)

### 동작 원리
```
호스트 A (192.168.1.1)                    호스트 B (192.168.1.2)
     │                                           │
     │  Ethernet 프레임                          │
     │  Dst MAC: BB:BB:BB:BB:BB:BB ────────────► │
     │  Src MAC: AA:AA:AA:AA:AA:AA               │
     │  IP Dst: 192.168.1.2                      │
     │  IP Src: 192.168.1.1                      │
```

### 주요 특징
- 스위치는 MAC 테이블로 특정 포트로만 전달 → 다른 포트에는 영향 없음
- TCP는 Unicast 전용 (신뢰성 보장)
- UDP도 Unicast 가능

### 활용 사례
- 제어기 ↔ 센서 간 1:1 명령/응답 (SOME/IP RPC)
- DoIP 진단 세션 (UDP/TCP Unicast)
- OTA 패키지 다운로드

---

## Multicast (멀티캐스트)

### IPv4 멀티캐스트 주소 범위 (224.0.0.0/4)
| 범위 | 용도 |
|------|------|
| 224.0.0.0 ~ 224.0.0.255 | 링크 로컬 (라우터 통과 안 함) |
| 224.0.0.1 | 모든 호스트 |
| 224.0.0.2 | 모든 라우터 |
| 224.0.1.0 ~ 238.255.255.255 | 글로벌 멀티캐스트 |
| 239.0.0.0 ~ 239.255.255.255 | 사이트 로컬 멀티캐스트 (내부망) |

### L2 멀티캐스트 MAC 주소 매핑
```
IPv4 멀티캐스트 IP → L2 MAC 변환 규칙:
  224.x.y.z → 01:00:5E:0x:y:z  (하위 23비트만 사용)

예: 239.255.0.1 → 01:00:5E:7F:00:01
    224.0.0.251 (mDNS) → 01:00:5E:00:00:FB

※ 상위 9비트 버림 → 32개 IP가 같은 MAC 공유 가능 (모호성 존재)
```

### IGMP (Internet Group Management Protocol)
멀티캐스트 그룹 참가/탈퇴를 관리하는 프로토콜입니다.

```bash
# Linux에서 멀티캐스트 그룹 가입
ip maddr add 239.255.0.1 dev eth0

# 현재 가입된 멀티캐스트 그룹 확인
ip maddr show dev eth0

# 출력 예시
3:	eth0
	link  01:00:5e:7f:00:01
	inet  239.255.0.1
	inet  224.0.0.1
```

### IGMP Snooping
스위치가 IGMP 메시지를 감시하여 멀티캐스트 트래픽을 필요한 포트로만 전달합니다.

```
IGMP Snooping 없이:
  멀티캐스트 → 브로드캐스트처럼 모든 포트로 전달 (낭비)

IGMP Snooping 활성화:
  멀티캐스트 → 해당 그룹 가입 포트로만 전달 (효율적)
```

### DDS에서의 Multicast 활용
```
DDS Publisher (Topic: /robot/joint_state)
      │
      │ UDP Multicast 239.255.0.1:7400
      │
      ├──► DDS Subscriber A (Motion Controller)
      ├──► DDS Subscriber B (Safety Monitor)
      └──► DDS Subscriber C (HMI Display)

장점: Publisher 1개로 복수 Subscriber에게 동일 데이터 효율 전달
```

---

## Broadcast (브로드캐스트)

### 종류
| 종류 | 주소 | 범위 |
|------|------|------|
| 제한 브로드캐스트 | 255.255.255.255 | 동일 호스트 내 |
| 서브넷 브로드캐스트 | 네트워크 주소 + 모두 1 (예: 192.168.1.255) | 동일 서브넷 |
| L2 브로드캐스트 | FF:FF:FF:FF:FF:FF | 동일 L2 도메인 |

### 브로드캐스트를 사용하는 주요 프로토콜
```
ARP Request   → 누가 IP X.X.X.X를 가지고 있나? (FF:FF:FF:FF:FF:FF)
DHCP Discover → IP 주소 주세요! (255.255.255.255)
SOME/IP SD    → 서비스 Offer/Find 메시지 (Multicast 또는 Broadcast)
```

### Broadcast Storm 주의
```
루프가 있는 네트워크에서 브로드캐스트가 무한 순환:

스위치 A ─────────────────────────► 스위치 B
          ◄─────────────────────────
          ◄─ 브로드캐스트 무한 순환 ─

결과: CPU 100%, 네트워크 마비
예방: STP/RSTP, 포트 보안, Storm Control 설정

# 스톰 컨트롤 설정 (Linux)
tc qdisc add dev eth0 root handle 1: prio
# 특정 임계값 이상의 브로드캐스트 제한
```

---

## Anycast (애니캐스트)

주로 IPv6 및 라우팅 프로토콜 기반으로 동작합니다. 여러 노드가 동일한 IP 주소를 공유하고, 라우팅 메트릭으로 가장 가까운 노드에 연결됩니다.

**활용 사례:**
- DNS 서버 (1.1.1.1, 8.8.8.8는 실제로 전 세계 다수 서버가 Anycast로 공유)
- NTP 서버 클러스터
- 로봇 분산 서비스에서 가장 응답 빠른 모듈 선택

---

## 실무: 트래픽 유형 선택 기준

```
제어 명령 (1:1, 신뢰성 필요)
  → TCP Unicast 또는 UDP Unicast + 애플리케이션 ACK

센서 데이터 브로드캐스트 (1:다수, 실시간)
  → UDP Multicast (DDS pub/sub 모델)

네트워크 발견 (초기화 단계)
  → Broadcast/Multicast (ARP, SOME/IP SD)

진단 (1:1, 신뢰성)
  → TCP Unicast (DoIP)

시간 동기화 (정밀, 낮은 지연)
  → L2 Multicast (PTP: 224.0.1.129, 01:1B:19:00:00:00)
```

---

## Wireshark 필터 예시

```bash
# Broadcast만 보기
eth.dst == ff:ff:ff:ff:ff:ff

# Multicast만 보기 (IPv4)
eth.dst[0] & 1

# ARP만 보기
arp

# DDS RTPS 트래픽
udp.port == 7400 or udp.port == 7401

# SOME/IP Service Discovery (Multicast)
udp.port == 30490

# PTP 멀티캐스트 (IEEE 1588)
eth.dst == 01:1b:19:00:00:00
```

---

## Reference
- [RFC 1112 - Host Extensions for IP Multicasting (IGMP)](https://datatracker.ietf.org/doc/html/rfc1112)
- [RFC 3171 - IANA Guidelines for IPv4 Multicast Address Assignments](https://datatracker.ietf.org/doc/html/rfc3171)
- [IEEE 802.1D - MAC Bridges (STP)](https://standards.ieee.org/ieee/802.1D/6744/)
- [OMG DDS Specification](https://www.omg.org/spec/DDS/1.4/PDF)
