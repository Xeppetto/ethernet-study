# OSI 7 Layer (OSI 7 계층 모델)

## 개요
OSI 7 계층 모델(Open Systems Interconnection Reference Model)은 국제 표준화 기구(ISO)에서 개발한 모델로, 컴퓨터 네트워크 프로토콜 디자인과 통신을 계층별로 나누어 설명한 것입니다. 이 모델은 이기종 시스템 간의 통신 호환성을 보장하고, 네트워크 통신 과정을 단계별로 파악하여 문제 해결을 용이하게 하는 데 목적이 있습니다. 의료 로봇과 같은 복잡한 시스템에서도 데이터의 흐름을 이해하고 디버깅하는 데 필수적인 기초 지식입니다.

### 계층별 역할과 기능

#### 물리 계층 (Physical Layer, Layer 1)
물리 계층은 시스템 간의 물리적인 연결과 전기적 신호를 담당합니다. 0과 1로 이루어진 비트(Bit) 스트림을 전기적, 광학적 신호로 변환하여 매체를 통해 전송합니다. Ethernet 케이블, 커넥터, 리피터, 허브 등이 이 계층에 해당하며, 의료 로봇에서는 센서와 액추에이터가 제어기와 물리적으로 연결되는 인터페이스 레벨에서 중요하게 다루어집니다.

#### 데이터 링크 계층 (Data Link Layer, Layer 2)
데이터 링크 계층은 인접한 두 장치 간의 신뢰성 있는 정보 전송을 보장합니다. 물리적인 주소인 MAC(Media Access Control) 주소를 사용하여 통신하며, 전송 단위는 프레임(Frame)입니다. 오류 제어와 흐름 제어 기능이 수행되며, Ethernet 스위치가 이 계층에서 동작합니다. 로봇 내부 네트워크에서 각 모듈이 서로를 식별하고 데이터를 주고받는 기본 단위가 됩니다.

#### 네트워크 계층 (Network Layer, Layer 3)
네트워크 계층은 여러 노드를 거칠 때마다 경로를 찾아주는 라우팅 기능을 수행합니다. 논리적인 주소인 IP(Internet Protocol) 주소를 사용하여 목적지까지 가장 안전하고 빠른 경로를 결정합니다. 전송 단위는 패킷(Packet)이며, 라우터가 이 계층의 대표적인 장비입니다. 로봇이 외부 서버나 클라우드와 통신할 때 필수적인 계층입니다.

#### 전송 계층 (Transport Layer, Layer 4)
전송 계층은 양 끝단(End-to-End)의 사용자들이 신뢰성 있는 데이터를 주고받을 수 있도록 합니다. TCP(Transmission Control Protocol)와 UDP(User Datagram Protocol)가 대표적이며, 포트 번호를 통해 상위 애플리케이션을 구분합니다. 실시간 제어가 중요한 로봇 시스템에서는 속도가 빠른 UDP가 선호되기도 하지만, 데이터의 정확성이 중요한 경우 TCP가 사용됩니다.

#### 세션 계층 (Session Layer, Layer 5), 표현 계층 (Presentation Layer, Layer 6), 응용 계층 (Application Layer, Layer 7)
상위 3개 계층은 데이터의 표현 방식, 세션 관리, 그리고 최종 사용자와의 인터페이스를 담당합니다. 의료 로봇 애플리케이션 소프트웨어는 주로 이 계층들에서 동작하며, SOME/IP나 DDS와 같은 미들웨어 프로토콜들이 이 영역과 밀접하게 연관되어 데이터를 처리하고 교환합니다.

### 의료 로봇 분야에서의 적용
의료 로봇 개발 시 OSI 7 계층을 이해하는 것은 통신 장애가 발생했을 때 문제의 원인이 물리적인 연결 문제인지(L1), 주소 설정 오류인지(L2/L3), 혹은 애플리케이션 로직의 문제인지(L7)를 빠르게 파악하는 데 도움을 줍니다. 특히 규제 기반의 검증 과정에서 각 계층별 요구사항을 충족하는지 확인하는 것은 시스템의 신뢰성을 입증하는 중요한 근거가 됩니다.

## Reference
- [ISO/IEC 7498-1:1994 - Information technology — Open Systems Interconnection — Basic Reference Model](https://www.iso.org/standard/20269.html)
- [Wikipedia - OSI model](https://en.wikipedia.org/wiki/OSI_model)
