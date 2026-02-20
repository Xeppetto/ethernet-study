# IEEE 802.1CB (Frame Replication)

## 개요
Ethernet 네트워크에서 데이터 손실(Packet Loss)은 치명적일 수 있습니다. 특히 의료 로봇과 같이 실시간 제어가 필요한 시스템에서는 단 하나의 패킷 손실도 동작의 오차를 발생시키거나 위험한 상황을 초래할 수 있습니다. IEEE 802.1CB는 'Frame Replication and Elimination for Reliability'라고 불리는 기술로, 동일한 데이터를 복제하여 서로 다른 경로로 전송함으로써 네트워크의 신뢰성(Reliability)을 극대화하는 표준입니다.

### 동작 원리
802.1CB는 네트워크 스위치에서 들어오는 프레임을 복제하여 여러 경로(Multi-path)로 전송하고, 수신 측 스위치에서 중복된 프레임을 제거하는 방식입니다.

1.  **Frame Replication (복제)**: 송신 측 스위치는 중요한 데이터 프레임을 복제하여, 서로 다른 경로(Path)를 통해 목적지로 보냅니다.
2.  **Path Diversity**: 만약 A 경로에 장애가 발생하거나 혼잡하여 프레임이 손실되더라도, B 경로를 통해 안전하게 데이터가 전달될 수 있습니다.
3.  **Frame Elimination (제거)**: 수신 측 스위치는 먼저 도착한 프레임을 정상적으로 전달하고, 나중에 도착한 동일한 프레임(중복)은 폐기합니다.
4.  **Sequence Number**: 각 프레임에는 순서 번호(Sequence Number)가 부여되어 있어, 수신 측에서 중복 여부를 판단하는 데 사용됩니다.

### 주요 기능
- **Seamless Redundancy**: 링크 장애가 발생하더라도 데이터 전송이 끊기지 않고 즉시 복구되는(Zero Switchover Time) 효과를 제공합니다. 기존의 STP(Spanning Tree Protocol)와 같은 방식은 경로를 재설정하는 데 시간이 걸리지만(수 초 ~ 수 분), 802.1CB는 실시간 무중단 통신을 보장합니다.
- **오류 감지**: 만약 두 경로 모두에서 데이터가 손실되었다면, 이는 심각한 네트워크 문제임을 알리는 지표가 됩니다.

### 의료 로봇에서의 활용
수술 로봇이나 환자 모니터링 시스템에서는 어떤 상황에서도 통신이 끊겨서는 안 됩니다. 802.1CB를 적용하면 네트워크 케이블 하나가 단선되거나 스위치 하나가 고장 나더라도, 복제된 프레임이 다른 경로를 통해 안전하게 전달되므로 수술을 중단 없이 계속 진행할 수 있습니다. 이는 IEC 60601-1과 같은 의료 기기 안전 규격을 만족시키는 데 중요한 요소가 됩니다.

## Reference
- [IEEE 802.1CB-2017 - Frame Replication and Elimination for Reliability](https://standards.ieee.org/ieee/802.1CB/6055/)
- [TSN Task Group - Frame Replication](https://1.ieee802.org/tsn/802-1cb/)
