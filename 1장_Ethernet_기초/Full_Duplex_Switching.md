# Full Duplex Switching (전이중 스위칭)

## 개요
초기 Ethernet은 Half Duplex(반이중) 방식을 사용하여 데이터 충돌(Collision)이 빈번하게 발생했고, CSMA/CD(Carrier Sense Multiple Access with Collision Detection) 알고리즘으로 이를 해결했습니다. 하지만 현대의 Ethernet, 특히 의료 로봇과 같은 고성능 시스템에서는 Full Duplex(전이중) 방식과 스위칭 기술이 기본이 되었습니다. 이 장에서는 Full Duplex 통신과 스위칭의 원리, 그리고 이것이 네트워크 성능에 미치는 영향을 알아봅니다.

### Half Duplex vs Full Duplex
- **Half Duplex (반이중)**: 무전기 통신처럼 송신과 수신을 번갈아 가며 수행합니다. 송신 중에는 수신이 불가능하며, 동시에 데이터를 보내면 충돌이 발생합니다. 대역폭 효율이 낮고 지연 시간이 길어질 수 있습니다.
- **Full Duplex (전이중)**: 전화 통화처럼 송신과 수신을 동시에 수행할 수 있습니다. 송신용 채널과 수신용 채널이 물리적으로 분리되어 있어 충돌이 발생하지 않습니다. 이론적으로 대역폭이 2배로 늘어나며, 지연 시간을 줄일 수 있습니다.

### Switching (스위칭)
스위치(Switch)는 OSI 2계층 장비로, 네트워크에 연결된 장비들의 MAC 주소를 학습하여 데이터를 목적지 포트로만 전달하는 역할을 합니다. 이를 통해 불필요한 트래픽을 줄이고 보안성을 높이며, 충돌 도메인(Collision Domain)을 분리하여 네트워크 성능을 극대화합니다.

#### Store-and-Forward vs Cut-Through
스위칭 방식에는 크게 두 가지가 있습니다.
1.  **Store-and-Forward**: 프레임 전체를 수신하여 버퍼에 저장한 후, 오류 검사(CRC)를 거쳐 목적지로 전송합니다. 안정적이지만 지연 시간(Latency)이 발생할 수 있습니다.
2.  **Cut-Through**: 프레임의 목적지 주소만 확인하고 즉시 전송을 시작합니다. 지연 시간을 최소화할 수 있어 실시간성이 중요한 로봇 제어 시스템에 적합하지만, 오류가 있는 프레임도 전송될 수 있다는 단점이 있습니다.

### 의료 로봇 네트워크에서의 중요성
의료 로봇은 수많은 센서 데이터와 제어 명령을 실시간으로 처리해야 합니다. Full Duplex 방식은 데이터 전송과 수신이 동시에 가능하게 하여 빠른 반응 속도를 보장합니다. 또한, 스위칭 기술을 통해 각 제어기(ECU) 간의 독립적인 통신 경로를 확보함으로써 네트워크 혼잡을 줄이고 안정적인 통신 환경을 구축할 수 있습니다. 특히 Cut-Through 방식의 스위치는 모터 제어와 같은 초저지연(Low Latency) 통신에 필수적인 요소로 고려됩니다.

## Reference
- [IEEE 802.3x - Full Duplex Operation](https://standards.ieee.org/ieee/802.3x/)
- [Cisco - LAN Switching Technologies](https://www.cisco.com/c/en/us/products/switches/what-is-a-network-switch.html)
