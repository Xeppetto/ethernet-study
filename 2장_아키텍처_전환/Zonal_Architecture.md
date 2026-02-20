# Zonal Architecture (영역 기반 아키텍처)

## 개요
전통적인 차량 및 로봇 아키텍처는 기능별로 제어기(ECU)를 구분하는 '도메인(Domain) 아키텍처'였습니다. 하지만 시스템의 복잡도가 증가하고 연결되는 장치가 많아지면서, 물리적인 위치를 기반으로 제어기를 통합하는 'Zonal Architecture'가 대두되었습니다. 의료 로봇과 같이 다양한 센서와 액추에이터가 로봇 전체에 분산된 시스템에서 배선 복잡도를 획기적으로 줄일 수 있는 핵심 개념입니다.

### Zonal Architecture의 개념
Zonal Architecture는 로봇이나 차량을 물리적인 구역(Zone)으로 나누고, 각 구역마다 'Zone Controller'를 배치하는 방식입니다. 이 Zone Controller는 해당 구역 내의 센서, 액추에이터, 하위 ECU들을 연결하고 관리하는 허브 역할을 수행하며, 중앙 컴퓨터(HPC)와 고속 Ethernet 백본으로 연결됩니다.

#### 특징
- **물리적 위치 기반**: 기능이 아닌, 장치의 위치에 따라 ECU를 그룹화합니다. (예: 로봇 팔의 손목 부분, 이동 플랫폼의 바퀴 부분 등)
- **배선 최적화**: 각 센서가 중앙 제어기까지 개별적으로 연결될 필요 없이, 가까운 Zone Controller까지만 연결되면 되므로 케이블 길이와 무게가 감소합니다.
- **확장성**: 새로운 센서를 추가할 때 전체 배선을 수정할 필요 없이 해당 Zone Controller에만 연결하면 됩니다.
- **데이터 통합**: Zone Controller가 구역 내의 다양한 데이터를 수집하고 1차적으로 처리(Aggregation)하여 중앙으로 전송하므로, 백본 네트워크의 부하를 효율적으로 관리할 수 있습니다.

### 의료 로봇 적용 및 이점
복잡한 관절 구조를 가진 수술 로봇이나 대형 의료 장비에서 Zonal Architecture의 이점은 극대화됩니다.

- **설계 유연성**: 로봇의 모듈화 설계가 용이해집니다. 특정 기능을 담당하는 Zone 모듈을 개발하여 다양한 로봇에 재사용할 수 있습니다.
- **유지보수**: 특정 구역에 문제가 발생했을 때 해당 Zone Controller와 하위 장치들만 점검하면 되므로 디버깅과 수리가 간편해집니다.
- **제조 효율성**: 케이블 하네스(Harness)가 단순화되어 로봇 조립 공정이 단축되고 비용이 절감됩니다.

### 기술적 요구사항
Zonal Architecture를 구현하기 위해서는 Zone Controller와 중앙 컴퓨터 간의 고속 통신을 위한 1Gbps 이상의 Ethernet, 그리고 실시간 제어를 위한 TSN(Time Sensitive Networking) 기술이 필수적입니다. 또한, 전원 분배의 효율성을 높이기 위해 Power over Ethernet (PoE)이나 단일 케이블로 전력과 데이터를 동시에 전송하는 PoDL(Power over Data Line) 기술도 함께 고려되어야 합니다.

## Reference
- [Aptiv - Smart Vehicle Architecture](https://www.aptiv.com/en/insights/article/smart-vehicle-architecture)
- [NXP - Zonal Architecture](https://www.nxp.com/applications/automotive/zonal-architecture:ZONAL-ARCHITECTURE)
