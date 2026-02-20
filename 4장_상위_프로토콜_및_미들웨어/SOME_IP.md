# SOME/IP (Scalable service-Oriented MiddlewarE over IP)

## 개요
SOME/IP는 자동차 산업의 요구사항을 반영하여 개발된 서비스 지향(Service-Oriented) 미들웨어 프로토콜입니다. 기존의 CAN 통신과 달리, Ethernet의 높은 대역폭을 효율적으로 사용하며, 서비스 발견(Service Discovery)과 원격 프로시저 호출(RPC), 이벤트(Event) 전송 기능을 제공합니다. AUTOSAR Adaptive Platform의 핵심 통신 프로토콜로 채택되었으며, 의료 로봇과 같은 고성능 임베디드 시스템에서도 널리 사용되고 있습니다.

### 주요 기능
1.  **Service Discovery (SD)**: 네트워크에 어떤 서비스가 존재하는지 동적으로 탐색하고, 필요한 서비스를 구독(Subscribe)하거나 제공(Offer)합니다. 이를 통해 시스템의 유연성과 확장성을 높일 수 있습니다.
2.  **RPC (Remote Procedure Call)**: 클라이언트가 서버의 함수를 마치 로컬 함수처럼 호출할 수 있게 해줍니다. (Request/Response 패턴)
3.  **Event Notification**: 특정 조건이 만족되거나 데이터가 변경되었을 때, 이를 구독한 클라이언트들에게 자동으로 알림을 보냅니다. (Publish/Subscribe 패턴)
4.  **Field**: 속성(Attribute) 값을 읽거나(Getter), 쓰거나(Setter), 변경 시 알림(Notifier)을 받을 수 있는 기능입니다.

### 데이터 직렬화 (Serialization)
SOME/IP는 데이터를 네트워크로 전송하기 위해 바이너리 형태로 직렬화합니다. 이때 페이로드(Payload) 앞에 헤더(Header)를 붙여 메시지의 종류, 길이, 서비스 ID 등을 명시합니다. XML이나 JSON과 같은 텍스트 기반 포맷보다 훨씬 가볍고 처리 속도가 빠릅니다.

### 의료 로봇 분야 활용
의료 로봇은 수술 도구 제어, 영상 전송, 환자 감시 등 다양한 기능을 수행합니다. SOME/IP를 적용하면 다음과 같은 이점이 있습니다.
- **모듈화**: 각 기능을 독립적인 서비스로 개발하여, 로봇의 구성을 쉽게 변경하거나 확장할 수 있습니다.
- **실시간성**: 가벼운 헤더와 바이너리 직렬화를 통해 낮은 지연 시간(Low Latency)으로 데이터를 주고받을 수 있어, 실시간 제어에 적합합니다.
- **표준 준수**: AUTOSAR 표준을 따르므로, 검증된 소프트웨어 스택을 활용하여 개발 기간을 단축하고 신뢰성을 높일 수 있습니다.

## Reference
- [AUTOSAR - SOME/IP Protocol Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [Vector - SOME/IP Introduction](https://www.vector.com/kr/ko/know-how/protocols/some-ip/)
