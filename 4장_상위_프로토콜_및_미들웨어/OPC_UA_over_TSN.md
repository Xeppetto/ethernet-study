# OPC UA over TSN

## 개요
산업용 자동화 분야의 표준 통신 프로토콜인 OPC UA(Open Platform Communications Unified Architecture)와 실시간 이더넷 표준인 TSN(Time Sensitive Networking)이 결합된 기술입니다. 기존의 필드버스(Fieldbus)들이 가진 폐쇄성과 상호운용성(Interoperability) 문제를 해결하고, 제조 현장(OT)과 정보 시스템(IT)을 잇는 단일 네트워크 인프라를 구축하기 위해 탄생했습니다.

### 기술적 배경
- **OPC UA**: 플랫폼 독립적인 서비스 지향 아키텍처(SOA)로, 데이터 모델링과 보안 기능이 강력합니다. 하지만 TCP/IP 위에서 동작하여 실시간성을 보장하기 어려웠습니다.
- **TSN**: 이더넷 네트워크 상에서 확정적인 지연 시간(Deterministic Latency)과 시간 동기화를 제공하지만, 상위 계층의 데이터 표현 방식은 정의하지 않습니다.
- **OPC UA over TSN**: 이 두 기술을 결합하여, 실시간 제어 데이터(Controller-to-Controller)와 정보 데이터(Controller-to-Cloud)를 하나의 네트워크에서 동시에 처리할 수 있는 차세대 산업용 통신 표준이 되었습니다.

### 주요 특징
1.  **Pub/Sub 모델**: 기존의 Client/Server 방식 외에, 실시간 데이터 전송에 적합한 Publish/Subscribe 모델을 추가했습니다. UDP 기반의 효율적인 멀티캐스트 전송을 지원합니다.
2.  **TSN 설정 자동화**: OPC UA의 설정 정보(Information Model)를 통해 TSN 스위치의 큐(Queue)와 스케줄(Schedule)을 자동으로 설정할 수 있는 CUC(Centralized User Configuration)와 CNC(Centralized Network Configuration) 개념이 도입되었습니다.
3.  **벤더 독립성**: 특정 제조사에 종속되지 않고, 서로 다른 회사의 로봇, PLC, 센서들이 표준화된 방식으로 데이터를 교환할 수 있습니다.

### 의료 로봇 분야의 적용
의료 로봇 시스템은 수술실 내의 다른 의료 기기(마취기, 환자 모니터, 영상 장비 등)와 연동되어야 합니다. 현재는 각 기기마다 제조사가 달라 통합이 어렵지만, OPC UA over TSN을 도입하면 다음과 같은 혁신이 가능합니다.
- **SDi(Service-Oriented Device Connectivity)**: IEEE 11073 SDC(Service-oriented Device Connectivity) 표준과 연계하여, 수술실 내 모든 장비가 플러그 앤 플레이(Plug & Play) 방식으로 연결되고, 로봇이 수술 단계에 맞춰 주변 기기를 자동으로 제어할 수 있습니다.
- **실시간 데이터 동기화**: 로봇 팔의 움직임과 환자 생체 신호(ECG, SpO2)를 마이크로초 단위로 동기화하여 저장하고 분석할 수 있어, 수술 정밀도와 안전성을 높일 수 있습니다.

## Reference
- [OPC Foundation - OPC UA over TSN](https://opcfoundation.org/flc/)
- [IEEE 11073 SDC - Service-oriented Device Connectivity](https://standards.ieee.org/ieee/11073-20701/6979/)
