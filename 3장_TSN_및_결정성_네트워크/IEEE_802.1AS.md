# IEEE 802.1AS (gPTP: Generalized Precision Time Protocol)

## 개요
TSN(Time Sensitive Networking)의 가장 기초가 되는 표준은 바로 '시간 동기화'입니다. 네트워크 상의 모든 장치들이 동일한 시간을 공유해야만 정밀한 제어 명령과 데이터 수집이 가능하기 때문입니다. IEEE 802.1AS는 이더넷 네트워크에서 마이크로초(µs) 이하의 정밀한 시간 동기화를 제공하는 프로토콜로, gPTP(Generalized Precision Time Protocol)라고도 불립니다.

### 동작 원리
IEEE 802.1AS는 Master-Slave 구조를 따릅니다. 네트워크 내에서 가장 정확한 시계(Grandmaster Clock, GM)를 선출하고, 이 GM의 시간을 네트워크 전체로 전파합니다.

1.  **Best Master Clock Algorithm (BMCA)**: 네트워크에 연결된 장치 중 가장 정밀하고 안정적인 클럭 소스를 가진 장치를 GM으로 자동 선출합니다.
2.  **Path Delay Measurement**: GM에서 보낸 시간 정보가 각 스위치와 케이블을 통과하며 발생하는 지연 시간(Propagation Delay)을 측정하여 보정합니다. (Pdelay Request/Response 메시지 사용)
3.  **Sync & Follow_Up**: GM은 주기적으로 Sync 메시지를 보내고, 정확한 발송 시각을 담은 Follow_Up 메시지를 연이어 보냅니다. 슬레이브는 이 정보를 바탕으로 자신의 시계를 GM 시계에 맞춥니다.

### 기존 PTP (IEEE 1588)와의 차이점
- **Layer 2 기반**: IEEE 1588은 IP(Layer 3) 위에서도 동작할 수 있지만, 802.1AS는 Ethernet(Layer 2) 레벨에서만 동작하도록 최적화되어 오버헤드가 적습니다.
- **프로파일**: 802.1AS는 IEEE 1588의 특정 프로파일(Profile)로 정의되어 있으며, 불필요한 옵션을 제거하고 필수 기능만 간소화했습니다.

### 의료 로봇에서의 중요성
수술 로봇과 같이 여러 개의 관절이 동시에 협조 제어(Coordinated Motion)를 수행해야 하는 시스템에서, 각 관절 모터 제어기(Motor Controller)들이 서로 다른 시간을 가지고 있다면 어떻게 될까요?
- **동작 불일치**: 1번 관절은 0.001초에 움직이고, 2번 관절은 0.002초에 움직인다면 로봇 팔 끝단(End-effector)의 궤적은 의도한 경로를 벗어나게 됩니다.
- **데이터 분석 오류**: 센서 데이터의 타임스탬프가 서로 다르면, 센서 퓨전(Sensor Fusion) 알고리즘이 엉뚱한 결과를 내놓을 수 있습니다.

따라서 802.1AS를 통해 모든 제어기와 센서의 시간을 완벽하게 동기화하는 것은 정밀 제어의 필수 전제 조건입니다.

## Reference
- [IEEE 802.1AS-2020 - Timing and Synchronization for Time-Sensitive Applications](https://standards.ieee.org/ieee/802.1AS/7123/)
- [Avnu Alliance - Time Synchronization](https://avnu.org/knowledgebase/time-synchronization/)
