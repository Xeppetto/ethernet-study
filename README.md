# 산업 전반에 걸친 Ethernet 전환 : 학습 가이드

v0.23

## 작업자 소회

1996년 HTML을 배워 개인 홈페이지를 만든 이후 현재까지 개인 블로그를 개발하거나 거대 플랫폼이 제공하는 Social Media에 이런 저런 생각들을 정리하여 게시하였지만, 2023년 AI가 본격적으로 사용된 이후 큰 충격에 빠져 Publish했던 블로그 글들 대다수를 local PC로 내리고 배포를 중단했습니다. 2025년의 AI Agent 대격변 시대를 겪는 동안 '인간은 앞으로 어떻게 글을 쓰고 어떻게 지식 체계를 만들어가야 하는가'에 대한 질문을 던지고 고민을 하였습니다. 

2026년 현재, 이 repository가 제가 찾은 답입니다. AI는 인간이 문제라고 입력하는 것들을 인간이 입력해야만 그게 문제라고 알 수 있을 뿐입니다. AI는 인간과 같은 형태로 진화하지 않았습니다. AI는 인간의 '도마뱀의 뇌'도 이해할 수 없을 거고, 인간 행동의 90% 이상이 본능에서 발현되는 비-이성적인 행동임을 몸소 체험할 수 없을 것입니다. 그러므로, 현실에서 발생하는 문제의 대부분은 비-이성적이고 비-합리적인 형태의 문제들이므로, 그 현상은 인간만이 알 수 있고 볼 수 있습니다.

각 문제 상황에 맞는 지식 체계는 그 문제 상황을 겪는 인간 본인만이 해결할 수 있는 거 같습니다. 그렇게 지식 체계를 다듬고, AI의 도움을 받아 지식을 축적하고, 인간은 그것을 다시 정리하여 문제를 해결하는 방식... 그것이 2026년 현재 제가 찾은 답입니다. 물론 앞으로 제가 찾아갈 해답들은 변해갈 수 있습니다. AI는 계속 발전할테니 AI와 함께 답을 찾아가는 좋은 인간 친구가 되어줘어야 겠습니다.

---

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

### 0장. Ethernet 전환 배경 (Why Ethernet?)
기존 제어 네트워크의 한계를 분석하고, 왜 Ethernet으로 전환해야 하는지, 왜 Standard Ethernet만으로 부족한지를 의료 로봇 맥락에서 설명합니다.

- [기존 제어 네트워크의 한계](./0장_Ethernet_전환_배경/Legacy_Control_Network_Limitations.md) - CAN/LIN/Fieldbus 대역폭·토폴로지·IP 미지원 한계, 의료 로봇 요구 대역폭 계산
- [왜 Ethernet인가](./0장_Ethernet_전환_배경/Why_Ethernet.md) - 속도·생태계·IP 확장성·SW 정의 적합성, CAN 대비 5가지 우위
- [Ethernet은 제어 네트워크가 아니다](./0장_Ethernet_전환_배경/Ethernet_Control_Network_Gap.md) - Best-Effort 한계, 지터·지연 분석, TSN 해법 매핑
- [의료 로봇 맥락](./0장_Ethernet_전환_배경/Medical_Robot_Context.md) - Safety-Critical 요구사항, IEC 62304, 지연이 수술 품질에 미치는 영향
- [이 문서의 학습 전략](./0장_Ethernet_전환_배경/Learning_Strategy.md) - 1장→2장→3장→5장→7장 로드맵, 학습자 유형별 경로, 핵심 개념 30개

### 1장. Ethernet 기초 (Network Basics)
네트워크의 기본 원리와 Ethernet의 핵심 개념을 다룹니다. 3장 TSN 및 5장 기능 안전으로 이어지는 기반을 제공합니다.

- [OSI 7 Layer](./1장_Ethernet_기초/OSI_7_Layer.md) - OSI 모델, LLC/MAC 하위 계층, Collision/Broadcast Domain, TCP/IP 비교, DSCP↔PCP 매핑
- [Ethernet 프레임 구조 심화](./1장_Ethernet_기초/Ethernet_Frame_Structure.md) - Preamble/SFD/IFG, 프레임 크기, EtherType, 직렬화 지연, PTP 프레임, 안전 무결성
- [MAC와 IP 차이](./1장_Ethernet_기초/MAC_IP_Difference.md) - MAC 주소 구조, ARP, IPv4/IPv6, 서브네팅
- [Unicast, Multicast, Broadcast](./1장_Ethernet_기초/Unicast_Multicast_Broadcast.md) - 전송 방식 비교, IGMP, DDS 멀티캐스트
- [Full Duplex Switching](./1장_Ethernet_기초/Full_Duplex_Switching.md) - 스위칭, QoS, 직렬화 지연, WRR/DWRR, STP/RSTP, TSN Guard Band
- [흐름 제어와 버퍼 관리](./1장_Ethernet_기초/Flow_Control_and_Buffer.md) - PAUSE/PFC, HOL Blocking, CBS, 큐 스케줄링, 안전 설계 지침
- [VLAN / IEEE 802.1Q](./1장_Ethernet_기초/VLAN_IEEE_802.1Q.md) - VLAN 태깅, QinQ, VLAN Hopping 방어, PVST vs MSTP, TSN 연동
- [Ethernet 속도 표준](./1장_Ethernet_기초/Ethernet_Speed_Standards.md) - 10M~400G 표준, 지연 예산 계산, EEE 위험, TSN PHY 요구사항
- [Single Pair Ethernet](./1장_Ethernet_기초/Single_Pair_Ethernet.md) - 10BASE-T1S, 100BASE-T1, PLCA, PoDL
- [결정성 네트워크와 의료 안전 기초](./1장_Ethernet_기초/Determinism_and_Safety_Basics.md) - 비결정성 원인, 네트워크 장애 분류, Fail-safe 설계, IEC 62304 요구사항, 3장 TSN 연결

### 2장. 아키텍처 전환 (Architecture Transition)
전통적인 아키텍처에서 Ethernet 기반 차세대 아키텍처로의 전환을 설명합니다. 의료 로봇 설계 시나리오를 중심으로 Zone 설계, HPC 가상화, OTA 보안, 미들웨어 선택, RTOS 비교, 하드웨어 플랫폼 선정까지 아키텍처 전환의 전 과정을 다룹니다.

- [Zonal Architecture](./2장_아키텍처_전환/Zonal_Architecture.md) - Zone Controller, 배선 절감 원리, TAS GCL 설정 예제, 의료 로봇 3-Zone 설계, IEC 62304 요구사항
- [중앙집중형 컴퓨팅 (HPC)](./2장_아키텍처_전환/Centralized_Computing.md) - HPC 구조, Type-1 하이퍼바이저, CPU isolcpus 격리, Safety MCU 워치독, 이중화·열 관리
- [Software Defined Vehicle (SDV)](./2장_아키텍처_전환/SDV.md) - OTA 보안, IEC 62304 OTA 요구사항, 의료기기 CI/CD 파이프라인, Yocto 빌드
- [Domain Controller 통합](./2장_아키텍처_전환/Domain_Controller_Integration.md) - 4-도메인 아키텍처, 도메인 간 지연 예산, IVSHMEM IPC, FMEA, 3단계 마이그레이션
- [Service Oriented Architecture (SOA)](./2장_아키텍처_전환/SOA.md) - SOME/IP 16-byte 헤더 구조, DDS 매핑표, ROS 2 Action Server, 미들웨어 비교
- [AUTOSAR Classic vs Adaptive](./2장_아키텍처_전환/AUTOSAR_Classic_vs_Adaptive.md) - 모듈 비교, 워치독, IEC 62304 플랫폼 적합성, 3단계 마이그레이션 전략
- [RTOS와 실시간 OS](./2장_아키텍처_전환/RTOS_and_Realtime_OS.md) - Hard/Soft/Firm 실시간 분류, OS 비교, cyclictest WCRT 측정 방법
- [하드웨어 플랫폼 선정](./2장_아키텍처_전환/Hardware_Platform_Selection.md) - SoC 비교, TSN NIC 선정, TSN 스위치 체크리스트, 메모리·스토리지 가이드

### 3장. TSN 및 결정성 네트워크 (TSN & Deterministic Network)
의료 로봇 제어에 필수적인 실시간성을 보장하는 TSN 기술을 다룹니다.

- [IEEE 802.1AS](./3장_TSN_및_결정성_네트워크/IEEE_802.1AS.md) - gPTP, BMCA, linuxptp, 정밀도 비교
- [IEEE 802.1Qbv (TAS)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qbv.md) - GCL, Guard Band, tc taprio, Python 계산
- [IEEE 802.1Qav (CBS)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qav.md) - Credit-Based Shaper, SRP, tc cbs
- [IEEE 802.1Qbu (Frame Preemption)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qbu.md) - 프레임 선점, mPacket, ethtool
- [IEEE 802.1Qci (PSFP)](./3장_TSN_및_결정성_네트워크/IEEE_802.1Qci.md) - 스트림 필터링, Babbling Idiot 방어
- [IEEE 802.1CB (FRER)](./3장_TSN_및_결정성_네트워크/IEEE_802.1CB.md) - 프레임 복제, 6-nine 가용성
- [IEEE 802.1DG (Automotive TSN)](./3장_TSN_및_결정성_네트워크/IEEE_802.1DG.md) - 자동차 TSN 프로파일, YANG

### 4장. 상위 프로토콜 및 미들웨어 (Upper Protocols & Middleware)
Ethernet 위에서 동작하는 응용 계층 프로토콜과 미들웨어를 학습합니다.

- [SOME/IP](./4장_상위_프로토콜_및_미들웨어/SOME_IP.md) - 헤더 구조, SD, vsomeip 설정
- [DoIP](./4장_상위_프로토콜_및_미들웨어/DoIP.md) - ISO 13400, 연결 흐름, Python 코드
- [UDS Protocol](./4장_상위_프로토콜_및_미들웨어/UDS_Protocol.md) - ISO 14229, Flash 절차, Python
- [DDS & ROS 2](./4장_상위_프로토콜_및_미들웨어/DDS_ROS2.md) - RTPS, QoS, Publisher/Subscriber
- [MQTT](./4장_상위_프로토콜_및_미들웨어/MQTT.md) - QoS 0/1/2, TLS, paho-mqtt
- [OPC UA over TSN](./4장_상위_프로토콜_및_미들웨어/OPC_UA_over_TSN.md) - asyncua, 보안, IEEE 11073 SDC

### 5장. 기능 안전 및 사이버 보안 (Functional Safety & Cybersecurity)
의료 기기 규제 준수를 위한 기능 안전 및 보안 표준을 다룹니다.

- [ISO 26262](./5장_기능_안전_및_사이버_보안/ISO_26262.md) - ASIL, HARA, E2E, FFI, FMEDA
- [ISO/SAE 21434](./5장_기능_안전_및_사이버_보안/ISO_SAE_21434.md) - TARA, CAL, IDS, SBOM, 취약점 관리
- [UNECE R155/R156](./5장_기능_안전_및_사이버_보안/UNECE_R155_R156.md) - CSMS, SUMS, UPTANE, FDA 비교
- [Secure Boot](./5장_기능_안전_및_사이버_보안/Secure_Boot.md) - Chain of Trust, dm-verity, IMA, Anti-Rollback
- [TLS & PKI 인증서 관리](./5장_기능_안전_및_사이버_보안/TLS_PKI.md) - TLS 1.3, mTLS, MACsec, EST
- [IEC 62304](./5장_기능_안전_및_사이버_보안/IEC_62304.md) - 의료기기 SW 수명 주기, SOUP, MC/DC

### 6장. 검증 및 분석 도구 (Verification & Tools)
실제 개발 및 테스트 환경에서 활용되는 분석 도구와 방법론을 소개합니다.

- [Wireshark PCAP 분석](./6장_검증_및_분석_도구/Wireshark_PCAP.md) - BPF/디스플레이 필터, tshark, 프로토콜 분석
- [linuxptp PTP 설정](./6장_검증_및_분석_도구/linuxptp_PTP.md) - ptp4l, phc2sys, pmc, 성능 측정
- [tc taprio / CBS](./6장_검증_및_분석_도구/tc_taprio.md) - taprio, CBS, flower 필터 통합
- [Latency와 Jitter 분석](./6장_검증_및_분석_도구/Latency_Jitter.md) - cyclictest, Python 분석, 시스템 튜닝
- [HIL (Hardware in the Loop)](./6장_검증_및_분석_도구/HIL.md) - HIL 구성, Fault Injection, CI/CD 통합
- [iperf3 대역폭 측정](./6장_검증_및_분석_도구/iperf3_Bandwidth.md) - TCP/UDP 측정, Jitter 비교, VLAN 검증
- [Scapy 패킷 생성/분석](./6장_검증_및_분석_도구/Scapy_Packet.md) - 커스텀 패킷, Fault Injection, PCAP 분석

### 7장. 산업 구조 변화 (Industry Trends)
Ethernet 도입으로 인한 산업 전반의 구조적 변화와 미래 트렌드를 전망합니다.

- [ECU 통합 감소](./7장_산업_구조_변화/ECU_Integration.md) - 통합 단계, 하이퍼바이저, 비용 계산
- [플랫폼화 전략](./7장_산업_구조_변화/Platform_Strategy.md) - HW/SW 플랫폼, ROS 2 아키텍처, SDK
- [클라우드 연동](./7장_산업_구조_변화/Cloud_Connectivity.md) - Edge-Fog-Cloud, MQTT, HIPAA, 디지털 트윈
- [OTA 인프라](./7장_산업_구조_변화/OTA_Infrastructure.md) - UPTANE, A/B 파티션, Delta 업데이트
- [데이터 기반 비즈니스 모델](./7장_산업_구조_변화/Data_Driven_Business.md) - RaaS, 데이터 모네타이제이션, AI 서비스

### 8장. 실습 가이드 (Practical Lab Guide)
이론을 실습으로 확인하는 단계별 가이드입니다.

- [Lab 01: TSN 환경 구축](./8장_실습_가이드/Lab01_TSN_환경_구축.md) - PTP 동기화, VLAN, taprio 설정
- [Lab 02: Wireshark 프로토콜 분석](./8장_실습_가이드/Lab02_Wireshark_프로토콜_분석.md) - 트래픽 생성, SOME/IP/PTP/MQTT 분석
- [Lab 03: ROS 2 DDS 통신](./8장_실습_가이드/Lab03_ROS2_DDS_통신.md) - Publisher/Subscriber, QoS, 안전 모니터

---

## 학습 로드맵

본 문서에 있는 내용은 Ethernet을 학습하기 위한 가이드를 제공합니다. 본 문서의 내용을 중심으로 추가 자료들을 검색하고 학습을 확장해 나아갈 수 있는 형태로 자료들을 생성하려 하였습니다. 본 문서에 있는 키워드들을 검색하거나 AI에게 문의하여 보다 전문적인 지식 체계를 구축하시기를 바랍니다.
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
MIT License - 이 문서는 AI와 협업하여 작성하였습니다 : 인간 + Claude + GPT + Gemini
