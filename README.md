# 의료 로봇 Ethernet 전환 학습 가이드

## 개요 (Overview)

본 리포지토리는 **의료 로봇 분야의 Software Test Engineer**를 위한 Ethernet 기술 학습 자료입니다. 전통적인 제어 시스템에서 차세대 **Ethernet 기반 아키텍처**로 전환하는 과정에서 필요한 핵심 기술과 검증 방법론을 다룹니다.

주요 내용은 다음과 같습니다:
- **기초 개념**: OSI 7 Layer, Switching, Single Pair Ethernet
- **아키텍처**: Zonal Architecture, SDV, SOA
- **실시간 통신 (TSN)**: IEEE 802.1AS, Qbv, Qci 등 결정성 네트워크 기술
- **미들웨어**: SOME/IP, DoIP, DDS, ROS 2
- **안전 및 보안**: ISO 26262(기능 안전), ISO/SAE 21434(사이버 보안)
- **검증 도구**: Wireshark, linuxptp, HIL 시뮬레이션

---

## 목차 (Table of Contents)

### 1장. Ethernet 기초 (Network Basics)
네트워크의 기본 원리와 Ethernet의 핵심 개념을 다룹니다.
- [OSI 7 Layer](./1장_Ethernet_기초/OSI_7_Layer.md)
- [MAC와 IP 차이](./1장_Ethernet_기초/MAC_IP_Difference.md)
- [Unicast, Multicast, Broadcast](./1장_Ethernet_기초/Unicast_Multicast_Broadcast.md)
- [Full Duplex Switching](./1장_Ethernet_기초/Full_Duplex_Switching.md)
- [Single Pair Ethernet (10BASE-T1S, 100BASE-T1)](./1장_Ethernet_기초/Single_Pair_Ethernet.md)

### 2장. 아키텍처 전환 (Architecture Transition)
전통적인 아키텍처에서 Ethernet 기반의 차세대 아키텍처로의 전환을 설명합니다.
- [Zonal Architecture](./2장_아키텍처_전환/Zonal_Architecture.md)
- [중앙집중형 컴퓨팅 (HPC/HPVC)](./2장_아키텍처_전환/Centralized_Computing.md)
- [Software Defined Vehicle (SDV)](./2장_아키텍처_전환/SDV.md)
- [Domain Controller 통합](./2장_아키텍처_전환/Domain_Controller_Integration.md)
- [Service Oriented Architecture (SOA)](./2장_아키텍처_전환/SOA.md)

### 3장. TSN 및 결정성 네트워크 (TSN & Deterministic Network)
의료 로봇의 제어에 필수적인 실시간성과 결정성을 보장하는 TSN 기술을 다룹니다.
- [IEEE 802.1AS Time Synchronization](./3장_TSN_및_결정성_네트워크/IEEE_802.1AS.md)
- [IEEE 802.1Qbv Time Aware Shaper](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qbv.md)
- [IEEE 802.1Qci Ingress Policing](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qci.md)
- [IEEE 802.1CB Frame Replication](./3장_TSN_및_결정성_네트워크/IEEE_802.1CB.md)
- [IEEE 802.1DG Automotive Profile](./3장_TSN_및_결정성_네트워크/IEEE_802.1DG.md)

### 4장. 상위 프로토콜 및 미들웨어 (Upper Protocols & Middleware)
Ethernet 위에서 동작하는 응용 계층 프로토콜과 미들웨어를 학습합니다.
- [SOME/IP](./4장_상위_프로토콜_및_미들웨어/SOME_IP.md)
- [DoIP](./4장_상위_프로토콜_및_미들웨어/DoIP.md)
- [DDS & ROS2](./4장_상위_프로토콜_및_미들웨어/DDS_ROS2.md)
- [MQTT](./4장_상위_프로토콜_및_미들웨어/MQTT.md)
- [OPC UA over TSN](./4장_상위_프로토콜_및_미들웨어/OPC_UA_over_TSN.md)

### 5장. 기능 안전 및 사이버 보안 (Functional Safety & Cybersecurity)
의료 기기 규제 준수를 위한 기능 안전 및 보안 표준을 다룹니다.
- [ISO 26262](./5장_기능_안전_및_사이버_보안/ISO_26262.md)
- [ISO/SAE 21434](./5장_기능_안전_및_사이버_보안/ISO_SAE_21434.md)
- [UNECE R155/R156](./5장_기능_안전_및_사이버_보안/UNECE_R155_R156.md)
- [Secure Boot](./5장_기능_안전_및_사이버_보안/Secure_Boot.md)
- [TLS & PKI 인증서 관리](./5장_기능_안전_및_사이버_보안/TLS_PKI.md)

### 6장. 검증 및 분석 도구 (Verification & Tools)
실제 개발 및 테스트 환경에서 활용되는 분석 도구와 방법론을 소개합니다.
- [Wireshark PCAP 분석](./6장_검증_및_분석_도구/Wireshark_PCAP.md)
- [linuxptp PTP 설정](./6장_검증_및_분석_도구/linuxptp_PTP.md)
- [tc taprio](./6장_검증_및_분석_도구/tc_taprio.md)
- [Latency와 Jitter 분석](./6장_검증_및_분석_도구/Latency_Jitter.md)
- [HIL (Hardware in the Loop)](./6장_검증_및_분석_도구/HIL.md)

### 7장. 산업 구조 변화 (Industry Trends)
Ethernet 도입으로 인한 산업 전반의 구조적 변화와 미래 트렌드를 전망합니다.
- [ECU 통합 감소](./7장_산업_구조_변화/ECU_Integration.md)
- [플랫폼화 전략](./7장_산업_구조_변화/Platform_Strategy.md)
- [클라우드 연동](./7장_산업_구조_변화/Cloud_Connectivity.md)
- [OTA 인프라](./7장_산업_구조_변화/OTA_Infrastructure.md)
- [데이터 기반 비즈니스 모델](./7장_산업_구조_변화/Data_Driven_Business.md)

## License
MIT License
