# DDS & ROS 2

## 개요
ROS(Robot Operating System) 2는 로봇 애플리케이션 개발을 위한 표준 프레임워크로, 통신 미들웨어 계층에 DDS(Data Distribution Service) 표준을 채택하여 실시간성과 신뢰성을 대폭 향상시켰습니다. DDS는 OMG(Object Management Group)에서 제정한 데이터 중심의 통신 미들웨어로, 국방, 항공, 에너지 분야 등 미션 크리티컬 시스템에서 널리 검증된 기술입니다.

### DDS의 핵심 특징
1.  **Data-Centric Pub/Sub**: 데이터 자체를 중심으로 통신하며, Topic(주제) 기반으로 발행(Publish)과 구독(Subscribe)이 이루어집니다.
2.  **QoS (Quality of Service)**: 데이터 전송의 품질(신뢰성, 내구성, 이력 관리 등)을 세밀하게 제어할 수 있는 정책(Policy)을 제공합니다. (예: Reliability, Durability, History)
3.  **Discovery**: 네트워크 상의 참여자(Participant)를 자동으로 탐색하고 연결합니다. (Plug-and-Play)
4.  **Real-time Support**: 실시간 데이터 전송을 위한 우선순위 제어 및 지연 시간 최소화를 지원합니다.

### ROS 2와 DDS의 통합
ROS 2는 RMW(ROS Middleware) 계층을 두어 다양한 DDS 벤더(RTI Connext, eProsima Fast DDS, Eclipse Cyclone DDS 등)의 구현체를 플러그인 형태로 사용할 수 있도록 설계되었습니다. 개발자는 ROS 2 API를 사용하면서 하부의 DDS가 제공하는 강력한 통신 기능을 활용할 수 있습니다.

### 의료 로봇 분야 활용
의료 로봇은 수술 중 환자의 안전을 보장하기 위해 높은 신뢰성과 실시간성을 요구합니다. ROS 2와 DDS를 활용하면 다음과 같은 이점이 있습니다.
-   **실시간 제어**: 로봇 팔의 관절 제어 주기(예: 1ms)에 맞춰 정확한 시간에 명령을 전달하고 피드백을 받을 수 있습니다. (QoS: Reliability=RELIABLE, Durability=VOLATILE)
-   **데이터 무결성**: 수술 영상이나 센서 데이터가 손실되지 않고 안전하게 전송되도록 보장합니다.
-   **시스템 확장성**: 로봇 시스템을 모듈화하여 개발하고, 필요에 따라 센서나 기능을 추가/제거하기 쉽습니다.
-   **오픈 소스 생태계**: 전 세계 수많은 로봇 개발자들이 사용하는 ROS 2 커뮤니티의 다양한 패키지와 도구(Rviz, Gazebo 등)를 활용할 수 있습니다.

## Reference
- [ROS 2 Documentation - DDS](https://docs.ros.org/en/humble/Concepts/About-Domain-ID.html)
- [OMG - Data Distribution Service (DDS)](https://www.omg.org/spec/DDS/)
