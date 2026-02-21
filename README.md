# 의료 로봇 Ethernet 전환 학습 가이드

## 개요 (Overview)

본 리포지토리는 **Software Engineer / Test Engineer**를 위한 Ethernet 기술 학습 자료입니다. 전통적인 제어 시스템에서 차세대 **Ethernet 기반 아키텍처**로 전환하는 과정에서 필요한 핵심 기술과 검증 방법론을 다루며, 의료 로봇(수술 로봇, 재활 로봇)을 중심 적용 사례로 활용합니다.

주요 내용:
- **기초 개념**: OSI 7 Layer, VLAN, Ethernet 속도 표준, SPE
- **아키텍처**: Zonal/Domain Controller, SDV, SOA, AUTOSAR
- **실시간 통신 (TSN)**: IEEE 802.1AS/Qbv/Qav/Qbu/Qci/CB/DG
- **미들웨어**: SOME/IP, DoIP, UDS, DDS/ROS 2, MQTT, OPC UA
- **기능 안전**: ISO 26262, IEC 62304, ISO/SAE 21434, UNECE R155/R156
- **보안**: TLS/PKI, Secure Boot, MACsec
- **검증 도구**: Wireshark, linuxptp, tc taprio, iperf3, Scapy, HIL
- **실습 가이드**: TSN 환경 구축, 프로토콜 분석, ROS 2 통신

---

## 목차 (Table of Contents)

### 1장. Ethernet 기초 (Network Basics)
네트워크의 기본 원리와 Ethernet의 핵심 개념을 다룹니다.

| 파일 | 내용 |
|------|------|
| [OSI 7 Layer](./1장_Ethernet_기초/OSI_7_Layer.md) | OSI 모델, TCP/IP 비교, 산업 프로토콜 매핑 |
| [MAC와 IP 차이](./1장_Ethernet_기초/MAC_IP_Difference.md) | MAC 주소 구조, ARP, IPv4/IPv6, 서브네팅 |
| [Unicast, Multicast, Broadcast](./1장_Ethernet_기초/Unicast_Multicast_Broadcast.md) | 전송 방식 비교, IGMP, DDS 멀티캐스트 |
| [Full Duplex Switching](./1장_Ethernet_기초/Full_Duplex_Switching.md) | 스위칭, QoS, 802.1p, 오류 통계 |
| [Single Pair Ethernet](./1장_Ethernet_기초/Single_Pair_Ethernet.md) | 10BASE-T1S, 100BASE-T1, PLCA, PoDL |
| [VLAN / IEEE 802.1Q](./1장_Ethernet_기초/VLAN_IEEE_802.1Q.md) | **[신규]** VLAN 태깅, QinQ, Linux VLAN 설정 |
| [Ethernet 속도 표준](./1장_Ethernet_기초/Ethernet_Speed_Standards.md) | **[신규]** 10M~400G 표준, PHY, 케이블 |

### 2장. 아키텍처 전환 (Architecture Transition)
전통적인 아키텍처에서 Ethernet 기반 차세대 아키텍처로의 전환을 설명합니다.

| 파일 | 내용 |
|------|------|
| [Zonal Architecture](./2장_아키텍처_전환/Zonal_Architecture.md) | Zone Controller, 배선 절감, TSN 통합 |
| [중앙집중형 컴퓨팅 (HPC)](./2장_아키텍처_전환/Centralized_Computing.md) | HPC 구조, 하이퍼바이저, AI 통합 |
| [Software Defined Vehicle (SDV)](./2장_아키텍처_전환/SDV.md) | SW 중심 설계, OTA, CI/CD |
| [Domain Controller 통합](./2장_아키텍처_전환/Domain_Controller_Integration.md) | DC 진화, CPU 격리, IOMMU |
| [Service Oriented Architecture (SOA)](./2장_아키텍처_전환/SOA.md) | SOME/IP SD, DDS QoS, ROS 2 |
| [AUTOSAR Classic vs Adaptive](./2장_아키텍처_전환/AUTOSAR_Classic_vs_Adaptive.md) | **[신규]** 비교, ara::com, 공존 아키텍처 |

### 3장. TSN 및 결정성 네트워크 (TSN & Deterministic Network)
의료 로봇 제어에 필수적인 실시간성을 보장하는 TSN 기술을 다룹니다.

| 파일 | 내용 |
|------|------|
| [IEEE 802.1AS](./3장_TSN_및_결정성_네트워크/IEEE_802.1AS.md) | gPTP, BMCA, linuxptp, 정밀도 비교 |
| [IEEE 802.1Qbv (TAS)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qbv.md) | GCL, Guard Band, tc taprio, Python 계산 |
| [IEEE 802.1Qav (CBS)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qav.md) | **[신규]** Credit-Based Shaper, SRP, tc cbs |
| [IEEE 802.1Qbu (Frame Preemption)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qbu.md) | **[신규]** 프레임 선점, mPacket, ethtool |
| [IEEE 802.1Qci (PSFP)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qci.md) | 스트림 필터링, Babbling Idiot 방어 |
| [IEEE 802.1CB (FRER)](./3장_TSN_및_결정성_네트워크/IEEE_802.1CB.md) | 프레임 복제, 6-nine 가용성 |
| [IEEE 802.1DG (Automotive TSN)](./3장_TSN_및_결정성_네트워크/IEEE_802.1DG.md) | 자동차 TSN 프로파일, YANG |

### 4장. 상위 프로토콜 및 미들웨어 (Upper Protocols & Middleware)
Ethernet 위에서 동작하는 응용 계층 프로토콜과 미들웨어를 학습합니다.

| 파일 | 내용 |
|------|------|
| [SOME/IP](./4장_상위_프로토콜_및_미들웨어/SOME_IP.md) | 헤더 구조, SD, vsomeip 설정 |
| [DoIP](./4장_상위_프로토콜_및_미들웨어/DoIP.md) | ISO 13400, 연결 흐름, Python 코드 |
| [UDS Protocol](./4장_상위_프로토콜_및_미들웨어/UDS_Protocol.md) | **[신규]** ISO 14229, Flash 절차, Python |
| [DDS & ROS 2](./4장_상위_프로토콜_및_미들웨어/DDS_ROS2.md) | RTPS, QoS, Publisher/Subscriber |
| [MQTT](./4장_상위_프로토콜_및_미들웨어/MQTT.md) | QoS 0/1/2, TLS, paho-mqtt |
| [OPC UA over TSN](./4장_상위_프로토콜_및_미들웨어/OPC_UA_over_TSN.md) | asyncua, 보안, IEEE 11073 SDC |

### 5장. 기능 안전 및 사이버 보안 (Functional Safety & Cybersecurity)
의료 기기 규제 준수를 위한 기능 안전 및 보안 표준을 다룹니다.

| 파일 | 내용 |
|------|------|
| [ISO 26262](./5장_기능_안전_및_사이버_보안/ISO_26262.md) | ASIL, HARA, E2E, FFI, FMEDA |
| [ISO/SAE 21434](./5장_기능_안전_및_사이버_보안/ISO_SAE_21434.md) | TARA, CAL, IDS, SBOM, 취약점 관리 |
| [UNECE R155/R156](./5장_기능_안전_및_사이버_보안/UNECE_R155_R156.md) | CSMS, SUMS, UPTANE, FDA 비교 |
| [Secure Boot](./5장_기능_안전_및_사이버_보안/Secure_Boot.md) | Chain of Trust, dm-verity, IMA, Anti-Rollback |
| [TLS & PKI 인증서 관리](./5장_기능_안전_및_사이버_보안/TLS_PKI.md) | TLS 1.3, mTLS, MACsec, EST |
| [IEC 62304](./5장_기능_안전_및_사이버_보안/IEC_62304.md) | **[신규]** 의료기기 SW 수명 주기, SOUP, MC/DC |

### 6장. 검증 및 분석 도구 (Verification & Tools)
실제 개발 및 테스트 환경에서 활용되는 분석 도구와 방법론을 소개합니다.

| 파일 | 내용 |
|------|------|
| [Wireshark PCAP 분석](./6장_검증_및_분석_도구/Wireshark_PCAP.md) | BPF/디스플레이 필터, tshark, 프로토콜 분석 |
| [linuxptp PTP 설정](./6장_검증_및_분석_도구/linuxptp_PTP.md) | ptp4l, phc2sys, pmc, 성능 측정 |
| [tc taprio / CBS](./6장_검증_및_분석_도구/tc_taprio.md) | taprio, CBS, flower 필터 통합 |
| [Latency와 Jitter 분석](./6장_검증_및_분석_도구/Latency_Jitter.md) | cyclictest, Python 분석, 시스템 튜닝 |
| [HIL (Hardware in the Loop)](./6장_검증_및_분석_도구/HIL.md) | HIL 구성, Fault Injection, CI/CD 통합 |
| [iperf3 대역폭 측정](./6장_검증_및_분석_도구/iperf3_Bandwidth.md) | **[신규]** TCP/UDP 측정, Jitter 비교, VLAN 검증 |
| [Scapy 패킷 생성/분석](./6장_검증_및_분석_도구/Scapy_Packet.md) | **[신규]** 커스텀 패킷, Fault Injection, PCAP 분석 |

### 7장. 산업 구조 변화 (Industry Trends)
Ethernet 도입으로 인한 산업 전반의 구조적 변화와 미래 트렌드를 전망합니다.

| 파일 | 내용 |
|------|------|
| [ECU 통합 감소](./7장_산업_구조_변화/ECU_Integration.md) | 통합 단계, 하이퍼바이저, 비용 계산 |
| [플랫폼화 전략](./7장_산업_구조_변화/Platform_Strategy.md) | HW/SW 플랫폼, ROS 2 아키텍처, SDK |
| [클라우드 연동](./7장_산업_구조_변화/Cloud_Connectivity.md) | Edge-Fog-Cloud, MQTT, HIPAA, 디지털 트윈 |
| [OTA 인프라](./7장_산업_구조_변화/OTA_Infrastructure.md) | UPTANE, A/B 파티션, Delta 업데이트 |
| [데이터 기반 비즈니스 모델](./7장_산업_구조_변화/Data_Driven_Business.md) | RaaS, 데이터 모네타이제이션, AI 서비스 |

### 8장. 실습 가이드 (Practical Lab Guide)
이론을 실습으로 확인하는 단계별 가이드입니다.

| 파일 | 내용 |
|------|------|
| [Lab 01: TSN 환경 구축](./8장_실습_가이드/Lab01_TSN_환경_구축.md) | **[신규]** PTP 동기화, VLAN, taprio 설정 |
| [Lab 02: Wireshark 프로토콜 분석](./8장_실습_가이드/Lab02_Wireshark_프로토콜_분석.md) | **[신규]** 트래픽 생성, SOME/IP/PTP/MQTT 분석 |
| [Lab 03: ROS 2 DDS 통신](./8장_실습_가이드/Lab03_ROS2_DDS_통신.md) | **[신규]** Publisher/Subscriber, QoS, 안전 모니터 |

---

## 학습 로드맵

```
입문 (1~2주):
  1장 전체 → OSI, VLAN, SPE 기초

기초 (3~4주):
  2장 전체 → 아키텍처 변화 이해
  3장 802.1AS + 802.1Qbv → TSN 핵심

중급 (5~8주):
  4장 전체 → 프로토콜 실습
  6장 Wireshark + linuxptp → 도구 실습
  8장 Lab 01~03 → 실습 완료

고급 (9~12주):
  5장 전체 → 안전/보안 표준
  3장 나머지 (Qav, Qbu, Qci, CB, DG)
  7장 전체 → 산업 동향

전문가:
  규제 문서 직접 읽기 (ISO, IEEE 표준)
  오픈소스 기여 (linuxptp, OpenAvnu)
  실제 TSN 스위치/NIC 실습
```

---

## 참고 도구 설치

```bash
# Ubuntu 22.04 기준 필수 도구 설치

# 네트워크 도구
sudo apt-get install -y \
    wireshark tshark \
    linuxptp \
    ethtool \
    iproute2 \
    iperf3 \
    net-tools

# Python 라이브러리
pip3 install scapy paho-mqtt asyncua

# ROS 2 Humble (선택)
# https://docs.ros.org/en/humble/Installation.html

# Docker (HIL 시뮬레이션 환경)
sudo apt-get install docker.io docker-compose
```

---

## License
MIT License
