# IEEE 802.1DG (Automotive Profile)

## 개요
TSN(Time Sensitive Networking)은 산업용 이더넷 표준의 집합체(Set of Standards)로, 다양한 요구사항을 가진 산업 분야에 적용될 수 있도록 수많은 표준(IEEE 802.1Qbv, 802.1AS, 802.1CB 등)이 존재합니다. 하지만 모든 산업 분야가 모든 TSN 기능을 필요로 하는 것은 아닙니다. 따라서 각 산업(Automotive, Industrial, Aerospace)에 맞게 필요한 기능과 설정을 정의한 것을 'Profile(프로파일)'이라고 합니다. IEEE 802.1DG는 차량용(Automotive) 이더넷을 위한 프로파일을 정의하는 표준입니다.

### 프로파일의 필요성
TSN 표준들은 매우 유연하고 방대하여, 제조사마다 구현 방식이나 설정이 다를 수 있습니다. 이는 상호 호환성(Interoperability) 문제를 야기할 수 있습니다. 예를 들어, A사의 스위치는 802.1Qbv의 특정 기능을 지원하지만, B사의 스위치는 지원하지 않을 수 있습니다. 이를 해결하기 위해, IEEE 802.1DG는 차량 내 네트워크에서 필수적으로 지원해야 하는 기능, 성능, 설정값 등을 명확하게 규정합니다.

### 주요 내용
- **Mandatory Features**: 차량용 이더넷 스위치 및 종단 장치(End Station)가 반드시 지원해야 하는 TSN 기능을 명시합니다. (예: 802.1AS-2020 필수, 802.1Qbv 필수 등)
- **Configuration**: 각 기능에 대한 기본 설정값(Default Value)과 설정 범위(Range)를 정의합니다.
- **Performance Requirements**: 네트워크 지연(Latency), 지터(Jitter), 패킷 손실률(Packet Loss Rate) 등에 대한 최소 요구사항을 제시합니다.

### 의료 로봇 분야의 적용 (Automotive Profile의 재활용)
의료 로봇 분야는 아직 자체적인 TSN 프로파일 표준이 확립되지 않았습니다. 하지만 의료 로봇의 요구사항(실시간성, 안전성, 신뢰성)은 자율주행차의 요구사항과 매우 유사합니다. 따라서 많은 의료 로봇 개발자들은 이미 검증되고 성숙한 IEEE 802.1DG Automotive Profile을 참조하여 네트워크를 설계하고 있습니다.

- **안전성**: ISO 26262(기능 안전)를 만족하기 위한 네트워크 요구사항이 802.1DG에 반영되어 있어, 의료 기기 안전 규격(IEC 60601-1)을 준수하는 데 유리합니다.
- **실시간성**: 차량 제어에 필요한 1ms 이하의 지연 시간을 보장하는 기술들이 포함되어 있어, 정밀 수술 로봇 제어에 적합합니다.
- **생태계 활용**: 차량용 이더넷 칩셋과 스위치를 그대로 활용할 수 있어 비용 절감 및 부품 수급 안정성을 확보할 수 있습니다.

## Reference
- [IEEE P802.1DG - Automotive Profile](https://1.ieee802.org/tsn/802-1dg/)
- [Avnu Alliance - Automotive Profile for TSN](https://avnu.org/automotive/)
