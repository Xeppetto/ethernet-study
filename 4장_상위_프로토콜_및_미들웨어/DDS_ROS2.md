# DDS & ROS 2

## 개요
DDS(Data Distribution Service)는 OMG 표준 기반의 실시간 pub/sub 미들웨어입니다. ROS 2는 DDS를 통신 백본으로 채택하여 로봇 시스템의 실시간성, 확장성, 분산 처리를 지원합니다. 의료 로봇 분야에서 ROS 2 + DDS는 사실상의 표준 미들웨어로 자리 잡고 있습니다.

---

## DDS 아키텍처

### RTPS (Real-Time Publish-Subscribe Protocol)
DDS의 물리적 통신은 RTPS 프로토콜(UDP 기반)로 구현됩니다.

```
Application Layer:
┌─────────────────────────────────────────────────────────┐
│  DDS API (DataWriter / DataReader)                      │
├─────────────────────────────────────────────────────────┤
│  DCPS (Data-Centric Publish-Subscribe)                  │
│  • Domain: 격리된 통신 공간 (Domain ID로 구분)           │
│  • Participant: DDS 참여자                               │
│  • Topic: 공유할 데이터 주제 (타입 정의 포함)             │
│  • Publisher/Subscriber: 그룹 관리                      │
│  • DataWriter/DataReader: 실제 데이터 입출력             │
├─────────────────────────────────────────────────────────┤
│  RTPS Wire Protocol (UDP Multicast/Unicast)             │
└─────────────────────────────────────────────────────────┘
```

### DDS Discovery (자동 탐색)
```
Simple Discovery Protocol (SDP):
  1. Participant 시작 → SPDP (Simple Participant Discovery Protocol)
     → UDP Multicast로 존재 알림 (224.0.0.1 또는 239.255.0.1)
  2. 동일 Domain ID의 참여자가 서로 발견
  3. SEDP (Simple Endpoint Discovery Protocol)
     → DataWriter/DataReader 매칭
  4. QoS 정책 호환성 검사 → 호환 시 자동 연결

ROS 2 도메인 ID 기반 격리:
  Domain 0: 로봇 내부 (실시간 제어)
  Domain 1: 원격 모니터링
  Domain 2: 시뮬레이션
  각 도메인 포트: 7400 + (Domain ID * 250)
```

---

## DDS QoS (Quality of Service)

22개의 QoS 정책 중 주요 정책을 설명합니다.

```
QoS 정책 계층 구조:

실시간 제어 (1kHz, 신뢰성):
  Reliability: RELIABLE
  History: KEEP_LAST(10)
  Deadline: 1ms (이 시간 내 데이터 없으면 알림)
  Liveliness: AUTOMATIC, lease_duration=500ms

센서 스트리밍 (고속, 손실 허용):
  Reliability: BEST_EFFORT
  History: KEEP_LAST(1) (최신만 유지)
  Durability: VOLATILE
  Latency Budget: 10ms

안전 신호 (신뢰성 + 내구성):
  Reliability: RELIABLE
  Durability: TRANSIENT_LOCAL (신규 구독자에게 마지막 값 전송)
  Deadline: 2ms
  Liveliness: AUTOMATIC, lease_duration=100ms
```

---

## ROS 2 실습 예시

### 관절 상태 Publisher (Python)
```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy, DurabilityPolicy
from rclpy.duration import Duration

class JointStatePublisher(Node):
    def __init__(self):
        super().__init__('joint_state_publisher')

        # 실시간 제어 QoS
        qos = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=10,
            durability=DurabilityPolicy.VOLATILE
        )

        self.pub = self.create_publisher(JointState, '/joint_states', qos)
        self.timer = self.create_timer(0.001, self.publish_joint_state)  # 1kHz
        self.joint_positions = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]

    def publish_joint_state(self):
        msg = JointState()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.name = [f'joint_{i}' for i in range(6)]
        msg.position = self.joint_positions
        msg.velocity = [0.0] * 6
        msg.effort = [0.0] * 6
        self.pub.publish(msg)

def main():
    rclpy.init()
    node = JointStatePublisher()
    rclpy.spin(node)
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### 관절 상태 Subscriber (Python)
```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

class JointStateSubscriber(Node):
    def __init__(self):
        super().__init__('joint_state_subscriber')

        qos = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=10
        )

        self.sub = self.create_subscription(
            JointState, '/joint_states',
            self.callback, qos
        )

    def callback(self, msg):
        self.get_logger().info(
            f'Joint 0 position: {msg.position[0]:.4f} rad'
        )

def main():
    rclpy.init()
    node = JointStateSubscriber()
    rclpy.spin(node)
    rclpy.shutdown()
```

### ROS 2 CLI 도구
```bash
# 토픽 목록 확인
ros2 topic list

# 토픽 데이터 실시간 확인
ros2 topic echo /joint_states

# 토픽 주파수 확인
ros2 topic hz /joint_states
# average rate: 999.8 Hz (목표: 1000 Hz)

# 토픽 지연 측정
ros2 topic delay /joint_states
# average delay: 0.000234 s (234µs)

# 노드 정보 확인
ros2 node info /joint_state_publisher

# DDS 도메인 변경
export ROS_DOMAIN_ID=5
ros2 topic list  # Domain 5의 토픽만 보임
```

---

## DDS RMW 구현체 비교

| 구현체 | 라이브러리 | 특징 | ROS 2 지원 |
|--------|-----------|------|-----------|
| eProsima Fast DDS | rmw_fastrtps | 가장 많이 사용, 오픈소스 | 기본 |
| Eclipse Cyclone DDS | rmw_cyclonedds | 가볍고 빠름, 오픈소스 | 지원 |
| RTI Connext DDS | rmw_connextdds | 상용, 최고 성능, ASIL 인증 | 지원 |
| GurumDDS | rmw_gurumdds | 한국 구룸네트웍스, 고성능 | 지원 |

```bash
# RMW 구현체 변경
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ros2 run demo_nodes_cpp talker

# 현재 RMW 확인
ros2 doctor --report | grep middleware
```

---

## DDS + TSN 통합

```
ROS 2 DDS over TSN 네트워크:

TSN 스위치 설정:
  VLAN 10 + PCP=5: DDS 실시간 토픽 (/joint_states, /cmd_vel)
  VLAN 20 + PCP=3: DDS 센서 토픽 (/scan, /image_raw)
  VLAN 30 + PCP=0: DDS 진단 토픽 (/diagnostics, /rosout)

DDS QoS + TSN 매핑:
  Deadline < 1ms → PCP=5 (TAS TC5 슬롯)
  Deadline < 10ms → PCP=3 (CBS TC3 Class A)
  Deadline > 100ms → PCP=0 (Best Effort)

설정 방법 (Fast DDS QoS Profile XML):
  <profiles>
    <transport_descriptors>
      <transport_descriptor>
        <transport_id>UDPv4TransportDescriptor</transport_id>
        <type>UDPv4</type>
        <TTL>1</TTL>
        <socket_buffer_size>1048576</socket_buffer_size>
      </transport_descriptor>
    </transport_descriptors>
  </profiles>
```

---

## ROS 2 + 의료 로봇 패키지

```bash
# MoveIt 2: 모션 계획 (관절 경로 계획)
sudo apt install ros-humble-moveit

# ros2_control: 하드웨어 인터페이스 (모터, 엔코더)
sudo apt install ros-humble-ros2-control

# Nav2: 자율 주행 (모바일 로봇)
sudo apt install ros-humble-navigation2

# robot_localization: 센서 퓨전 (IMU + 엔코더)
sudo apt install ros-humble-robot-localization

# image_transport: 효율적 영상 전송
sudo apt install ros-humble-image-transport
```

---

## Reference
- [OMG DDS Specification v1.4](https://www.omg.org/spec/DDS/1.4/PDF)
- [OMG RTPS Specification v2.3](https://www.omg.org/spec/DDSI-RTPS/2.3/PDF)
- [ROS 2 Documentation (Humble)](https://docs.ros.org/en/humble/)
- [eProsima Fast DDS](https://fast-dds.docs.eprosima.com/)
- [Eclipse Cyclone DDS](https://cyclonedds.io/)
