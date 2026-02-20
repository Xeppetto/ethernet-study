# Single Pair Ethernet (10BASE-T1S, 100BASE-T1)

## 개요
전통적인 Ethernet(RJ45)은 4쌍의 구리선을 사용하지만, 산업용 및 차량용 네트워크에서는 경량화와 비용 절감을 위해 단일 쌍(Single Pair)으로 통신하는 기술이 도입되었습니다. Single Pair Ethernet(SPE)은 특히 의료 로봇과 같이 공간 제약이 있는 환경에서 중요한 기술로 자리 잡고 있습니다.

### Single Pair Ethernet (SPE)
SPE는 기존 Ethernet과 동일한 프로토콜 스택을 사용하면서, 물리 계층(Physical Layer)에서만 한 쌍의 전선(Twisted Pair)을 사용하는 기술입니다. 이를 통해 케이블 무게를 줄이고 설치 공간을 최소화할 수 있습니다. 또한, 기존 CAN, LIN과 같은 차량용 통신 방식을 대체하거나 보완하여 더 높은 대역폭을 제공합니다.

#### 주요 기술 표준
1.  **10BASE-T1S (Short Reach)**: IEEE 802.3cg 표준으로 정의된 기술입니다. 최대 25m 거리까지 10Mbps 속도를 지원하며, 멀티드롭(Multi-drop) 버스 토폴로지를 지원하여 여러 노드를 하나의 라인에 연결할 수 있습니다. 센서나 액추에이터와 같은 저속 장치 연결에 적합합니다.
2.  **100BASE-T1 (Automotive Ethernet)**: IEEE 802.3bw 표준으로 정의된 기술입니다. 최대 15m 거리까지 100Mbps 속도를 지원하며, 포인트-투-포인트(Point-to-Point) 연결 방식을 사용합니다. 카메라, 레이더, LIDAR 등 고대역폭 센서 데이터 전송에 주로 활용됩니다.
3.  **1000BASE-T1**: IEEE 802.3bp 표준으로, 기가비트 속도를 지원하는 SPE 기술입니다. 자율 주행, 고해상도 영상 처리 등 대용량 데이터 처리에 필요합니다.

### 의료 로봇 적용 및 이점
의료 로봇은 복잡한 구조와 제한된 공간 내에서 수많은 센서와 모터를 연결해야 합니다. SPE를 적용하면 다음과 같은 이점을 얻을 수 있습니다.

-   **경량화**: 케이블 무게 감소는 로봇 팔의 가반 하중(Payload)을 늘리거나 에너지 효율을 높이는 데 기여합니다.
-   **공간 효율성**: 얇고 유연한 케이블은 복잡한 관절 부위 배선을 용이하게 합니다.
-   **호환성**: 기존 IP 기반 네트워크 프로토콜을 그대로 사용할 수 있어 소프트웨어 개발 및 유지보수가 쉽습니다.
-   **비용 절감**: 케이블 및 커넥터 비용을 절감할 수 있습니다.

특히 10BASE-T1S의 멀티드롭 기능은 기존 CAN 통신을 대체하면서도 이더넷의 장점을 활용할 수 있어, 차세대 의료 로봇 아키텍처의 핵심 기술로 주목받고 있습니다.

## Reference
- [IEEE 802.3cg - 10 Mb/s Operation and Associated Power Delivery over a Single Balanced Pair of Conductors](https://standards.ieee.org/ieee/802.3cg/7438/)
- [OPEN Alliance - 100BASE-T1 Specification](https://www.opensig.org/about/specifications/)
