# Service Oriented Architecture (SOA)

## 개요
Service Oriented Architecture(SOA)는 소프트웨어 구성 요소를 '서비스(Service)'라는 독립적인 단위로 나누고, 네트워크를 통해 서로 호출하여 시스템을 구축하는 아키텍처 스타일입니다. 기존의 신호 중심(Signal-based) 통신 방식에서 벗어나, 서비스 중심의 유연하고 확장 가능한 통신 방식으로 전환하는 핵심 개념입니다.

### SOA의 핵심 개념
SOA에서는 기능을 제공하는 'Service Provider'와 기능을 사용하는 'Service Consumer'로 나뉩니다. Provider는 자신의 서비스(인터페이스)를 네트워크 상에 등록(Publish)하고, Consumer는 필요한 서비스를 찾아(Discover) 사용(Subscribe/Request)합니다.

#### 주요 특징
- **느슨한 결합 (Loose Coupling)**: 서비스 제공자와 사용자는 서로의 내부 구현을 알 필요 없이 인터페이스만 알면 통신할 수 있습니다. 이는 시스템의 유연성을 높이고 유지보수를 용이하게 합니다.
- **재사용성 (Reusability)**: 한 번 개발된 서비스는 다른 시스템이나 애플리케이션에서 재사용될 수 있습니다.
- **표준화된 인터페이스**: IDL(Interface Description Language)을 사용하여 서비스 인터페이스를 정의하므로, 서로 다른 언어나 플랫폼 간에도 통신이 가능합니다. (예: SOME/IP, DDS, REST API 등)

### 신호 중심 vs 서비스 중심
- **Signal-based (CAN)**: 센서 데이터와 같은 신호(Signal)를 주기적으로 브로드캐스팅합니다. 수신 측은 필요한 데이터를 필터링하여 사용합니다. 정적이고 대역폭 낭비가 심할 수 있습니다.
- **Service-based (Ethernet)**: 필요한 시점에 필요한 서비스만 호출(Method Call)하거나, 특정 이벤트가 발생했을 때만 데이터를 전송(Event)합니다. 동적이고 효율적인 통신이 가능합니다.

### 의료 로봇 분야 적용 (SOME/IP, DDS)
의료 로봇 시스템은 복잡한 기능을 수행하며, 다양한 센서와 액추에이터가 연결됩니다. SOA를 적용하면 다음과 같은 이점이 있습니다.
- **유연한 시스템 구성**: 수술 도구를 교체하거나 새로운 센서를 추가할 때, 해당 서비스만 등록하면 즉시 사용할 수 있습니다. (Plug & Play)
- **원격 제어 및 모니터링**: 외부 클라우드나 원격지 의사가 로봇의 특정 서비스(카메라 영상, 햅틱 피드백 등)에 접근하기 쉽습니다.
- **데이터 효율성**: 불필요한 데이터를 주기적으로 전송하지 않고, 필요한 시점에만 데이터를 요청하거나 변경 사항만 전송하므로 네트워크 대역폭을 절약할 수 있습니다.

### 대표적인 미들웨어
- **SOME/IP (Scalable service-Oriented MiddlewarE over IP)**: 차량용 Ethernet에서 주로 사용되는 미들웨어로, 서비스 검색(Service Discovery)과 RPC(Remote Procedure Call), 이벤트(Event) 전송 기능을 제공합니다.
- **DDS (Data Distribution Service)**: 국방, 항공, 로봇 분야에서 널리 사용되는 데이터 중심의 미들웨어로, QoS(Quality of Service) 설정이 강력하여 실시간성과 신뢰성이 중요한 시스템에 적합합니다. ROS 2의 통신 백본으로 채택되었습니다.

## Reference
- [AUTOSAR - SOME/IP Protocol Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [OMG - Data Distribution Service (DDS)](https://www.omg.org/spec/DDS/)
