# Lab 03: ROS 2 + DDS 기반 로봇 통신 실습

## 개요
이 실습에서는 ROS 2(Robot Operating System 2)와 DDS(Data Distribution Service) 미들웨어를 사용하여 의료 로봇 통신 구조를 구현합니다. Publisher/Subscriber 패턴으로 관절 상태를 공유하고, QoS 정책을 설정하여 실시간 통신을 보장합니다.

**필요 환경:**
- Ubuntu 22.04 LTS
- ROS 2 Humble (또는 Iron)
- Python 3.10+

---

## ROS 2 설치

```bash
# ROS 2 Humble 설치
sudo apt-get install software-properties-common
sudo add-apt-repository universe

# ROS 2 키 추가
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
    http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | \
    sudo tee /etc/apt/sources.list.d/ros2.list

sudo apt-get update
sudo apt-get install ros-humble-desktop python3-colcon-common-extensions

# 환경 설정
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc

# DDS 미들웨어 설치
sudo apt-get install ros-humble-rmw-fastrtps-cpp
sudo apt-get install ros-humble-rmw-cyclonedds-cpp

# 설치 확인
ros2 run demo_nodes_cpp talker &
ros2 run demo_nodes_cpp listener
```

---

## Step 1: 워크스페이스 생성

```bash
# ROS 2 워크스페이스 생성
mkdir -p ~/robot_ws/src
cd ~/robot_ws

# 패키지 생성
cd src
ros2 pkg create --build-type ament_python medical_robot_comm \
    --dependencies rclpy std_msgs geometry_msgs sensor_msgs

ls medical_robot_comm/
# medical_robot_comm/  package.xml  setup.cfg  setup.py
```

---

## Step 2: 메시지 및 노드 구현

```python
# ~/robot_ws/src/medical_robot_comm/medical_robot_comm/joint_publisher.py
"""
관절 상태 Publisher 노드
- 1kHz 주기로 6개 관절 위치/속도/토크 발행
- QoS: 실시간 제어 (BestEffort, Volatile)
"""
import math
import time
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy, QoSDurabilityPolicy
from sensor_msgs.msg import JointState


class JointStatePublisher(Node):
    """
    로봇 관절 상태 발행자
    실시간 제어를 위한 최소 지연 QoS 설정
    """

    def __init__(self):
        super().__init__('joint_state_publisher')

        # QoS 설정: 실시간 제어 (지연 최소화 우선)
        control_qos = QoSProfile(
            reliability=QoSReliabilityPolicy.BEST_EFFORT,  # 손실 허용, 지연 최소
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=1,
            durability=QoSDurabilityPolicy.VOLATILE,
        )

        self.publisher = self.create_publisher(
            JointState,
            '/robot/joint_states',
            qos_profile=control_qos
        )

        # 1kHz 타이머 (1ms 주기)
        self.timer = self.create_timer(0.001, self.publish_joint_state)

        # 관절 정보
        self.joint_names = [
            'shoulder_pan', 'shoulder_lift', 'elbow',
            'wrist_1', 'wrist_2', 'wrist_3'
        ]
        self.t = 0.0
        self.get_logger().info('JointState Publisher 시작 (1kHz)')

    def publish_joint_state(self):
        """
        시뮬레이션 관절 상태 발행 (사인파 궤적)
        실제 로봇에서는 EtherCAT/CAN에서 읽은 값 사용
        """
        msg = JointState()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.name = self.joint_names

        # 관절별 사인파 위치 (-π ~ π)
        msg.position = [
            math.sin(self.t + i * math.pi / 3) * math.pi / 4
            for i in range(6)
        ]

        # 속도 (미분값)
        msg.velocity = [
            math.cos(self.t + i * math.pi / 3) * 0.5
            for i in range(6)
        ]

        # 토크 (가상: N·m)
        msg.effort = [
            abs(math.sin(self.t + i * math.pi / 3)) * 10.0
            for i in range(6)
        ]

        self.publisher.publish(msg)
        self.t += 0.001  # 1ms 증가


def main():
    rclpy.init()
    node = JointStatePublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

```python
# ~/robot_ws/src/medical_robot_comm/medical_robot_comm/safety_monitor.py
"""
안전 모니터링 Subscriber 노드
- 관절 속도/토크 한계 초과 시 E-Stop 발행
- QoS: 신뢰성 있는 안전 명령 (Reliable)
"""
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy, QoSDurabilityPolicy
from sensor_msgs.msg import JointState
from std_msgs.msg import Bool


class SafetyMonitorNode(Node):
    """
    안전 모니터링 노드
    관절 상태 구독 → 임계치 검사 → E-Stop 발행
    """

    def __init__(self):
        super().__init__('safety_monitor')

        # 파라미터 (launch 파일에서 재설정 가능)
        self.declare_parameter('max_velocity_rad_s', 2.0)     # 2 rad/s
        self.declare_parameter('max_effort_nm', 50.0)         # 50 N·m
        self.declare_parameter('comm_timeout_ms', 100.0)      # 100ms 타임아웃

        self.max_vel = self.get_parameter('max_velocity_rad_s').value
        self.max_effort = self.get_parameter('max_effort_nm').value
        self.comm_timeout = self.get_parameter('comm_timeout_ms').value / 1000

        # 구독 (BestEffort - 최신 데이터 우선)
        control_qos = QoSProfile(
            reliability=QoSReliabilityPolicy.BEST_EFFORT,
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=1,
        )
        self.joint_sub = self.create_subscription(
            JointState,
            '/robot/joint_states',
            self.on_joint_state,
            qos_profile=control_qos
        )

        # E-Stop 발행 (Reliable - 반드시 전달)
        safety_qos = QoSProfile(
            reliability=QoSReliabilityPolicy.RELIABLE,
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=10,
            durability=QoSDurabilityPolicy.TRANSIENT_LOCAL,
        )
        self.estop_pub = self.create_publisher(
            Bool,
            '/robot/emergency_stop',
            qos_profile=safety_qos
        )

        self.last_msg_time = self.get_clock().now()
        self.estop_active = False

        # 통신 타임아웃 감시 타이머 (10ms 주기)
        self.watchdog = self.create_timer(0.01, self.check_timeout)
        self.get_logger().info('Safety Monitor 시작')

    def on_joint_state(self, msg: JointState):
        """관절 상태 수신 및 안전 검사"""
        self.last_msg_time = self.get_clock().now()

        for i, name in enumerate(msg.name):
            # 속도 검사
            if i < len(msg.velocity) and abs(msg.velocity[i]) > self.max_vel:
                self.trigger_estop(
                    f"속도 초과 [{name}]: {msg.velocity[i]:.2f} rad/s "
                    f"(최대: {self.max_vel})"
                )

            # 토크 검사
            if i < len(msg.effort) and abs(msg.effort[i]) > self.max_effort:
                self.trigger_estop(
                    f"토크 초과 [{name}]: {msg.effort[i]:.2f} N·m "
                    f"(최대: {self.max_effort})"
                )

        # 정상이면 E-Stop 해제 (이전에 활성화된 경우)
        if self.estop_active:
            self.release_estop()

    def check_timeout(self):
        """통신 타임아웃 감시"""
        elapsed = (self.get_clock().now() - self.last_msg_time).nanoseconds / 1e9
        if elapsed > self.comm_timeout:
            self.trigger_estop(f"통신 타임아웃: {elapsed*1000:.0f}ms")

    def trigger_estop(self, reason: str):
        if not self.estop_active:
            self.get_logger().error(f"[SAFETY] E-STOP 발동: {reason}")
            self.estop_active = True
        msg = Bool()
        msg.data = True
        self.estop_pub.publish(msg)

    def release_estop(self):
        self.get_logger().info("[SAFETY] E-STOP 해제")
        self.estop_active = False
        msg = Bool()
        msg.data = False
        self.estop_pub.publish(msg)


def main():
    rclpy.init()
    node = SafetyMonitorNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## Step 3: 빌드 및 실행

```bash
# setup.py 수정 (entry_points 추가)
# ~/robot_ws/src/medical_robot_comm/setup.py
entry_points={
    'console_scripts': [
        'joint_publisher = medical_robot_comm.joint_publisher:main',
        'safety_monitor = medical_robot_comm.safety_monitor:main',
    ],
},

# 빌드
cd ~/robot_ws
colcon build --symlink-install
source install/setup.bash

# 터미널 1: Publisher
ros2 run medical_robot_comm joint_publisher

# 터미널 2: Safety Monitor
ros2 run medical_robot_comm safety_monitor

# 터미널 3: 토픽 모니터링
ros2 topic hz /robot/joint_states
# → 1000.000 Hz 확인

ros2 topic echo /robot/joint_states --once
# → 관절 이름, 위치, 속도, 토크 확인

# 지연 측정
ros2 topic delay /robot/joint_states
# → message delay: 0.5ms 수준

# E-Stop 토픽 확인
ros2 topic echo /robot/emergency_stop
```

---

## Step 4: QoS 비교 실험

```python
# lab03_qos_experiment.py
# 네트워크 부하 하에서 QoS 정책 비교 실험

import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, QoSReliabilityPolicy
from std_msgs.msg import Float64MultiArray
import time
import statistics

class QoSExperiment(Node):
    """QoS 정책별 지연 및 손실 측정"""

    def __init__(self, reliability: QoSReliabilityPolicy):
        super().__init__(f'qos_test_{reliability.name.lower()}')

        self.reliability = reliability
        self.received_count = 0
        self.latencies = []
        self.send_timestamps = {}

        qos = QoSProfile(
            reliability=reliability,
            depth=10
        )

        topic = f'/qos_test/{reliability.name.lower()}'

        self.pub = self.create_publisher(Float64MultiArray, topic, qos)
        self.sub = self.create_subscription(
            Float64MultiArray, topic, self.on_receive, qos
        )

        self.seq = 0
        self.timer = self.create_timer(0.001, self.send)  # 1kHz

    def send(self):
        msg = Float64MultiArray()
        t = time.perf_counter()
        msg.data = [float(self.seq), t]
        self.send_timestamps[self.seq] = t
        self.pub.publish(msg)
        self.seq += 1

    def on_receive(self, msg: Float64MultiArray):
        seq = int(msg.data[0])
        send_t = msg.data[1]
        recv_t = time.perf_counter()
        latency_ms = (recv_t - send_t) * 1000
        self.latencies.append(latency_ms)
        self.received_count += 1

    def report(self, sent: int) -> dict:
        if not self.latencies:
            return {}
        return {
            'reliability': self.reliability.name,
            'sent': sent,
            'received': self.received_count,
            'loss_rate': (sent - self.received_count) / sent * 100,
            'avg_latency_ms': statistics.mean(self.latencies),
            'max_latency_ms': max(self.latencies),
            'jitter_ms': statistics.stdev(self.latencies) if len(self.latencies) > 1 else 0,
        }
```

---

## Step 5: DDS Discovery 분석

```bash
# DDS 디스커버리 트래픽 확인 (Wireshark)
sudo tshark -i lo \
    -Y "rtps" \
    -T fields \
    -e frame.time_relative \
    -e rtps.submessageId \
    -e rtps.domain_id

# ROS 2 토픽 그래프 확인
ros2 topic list
ros2 node list
ros2 node info /joint_state_publisher

# DDS 미들웨어 변경 (FastDDS vs CycloneDDS)
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
ros2 run medical_robot_comm joint_publisher

export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ros2 run medical_robot_comm joint_publisher

# 성능 비교 (ros2 topic delay)
for rmw in rmw_fastrtps_cpp rmw_cyclonedds_cpp; do
    export RMW_IMPLEMENTATION=$rmw
    ros2 run medical_robot_comm joint_publisher &
    sleep 5
    ros2 topic delay /robot/joint_states --window 1000
    kill %1
done
```

---

## 실습 과제

```
Lab 03 과제:

기본 과제:
  ☐ joint_publisher + safety_monitor 실행 및 토픽 확인
  ☐ ros2 topic hz로 1kHz 주기 확인
  ☐ ros2 topic delay로 지연 측정

심화 과제:
  ☐ Safety Monitor 임계치를 낮춰서 E-Stop 발동 확인
  ☐ BEST_EFFORT vs RELIABLE QoS 지연/손실 비교
  ☐ FastDDS vs CycloneDDS 성능 비교
  ☐ ROS 2 + TSN VLAN 분리 설정

도전 과제:
  ☐ SOME/IP ↔ ROS 2 브리지 구현 (ros2-someipmesssageinterface)
  ☐ Wireshark로 DDS RTPS 패킷 분석
  ☐ 멀티 노드 ROS 2 네트워크 (PC 2대 간 토픽 공유)

토론 주제:
  Q1. ROS 2의 BEST_EFFORT QoS와 RELIABLE QoS의 의료 로봇 활용 차이는?
  Q2. DDS의 멀티캐스트 디스커버리가 의료 로봇 격리 네트워크에서 문제가 되는 이유는?
  Q3. IEC 62304 관점에서 ROS 2는 SOUP인가요? 어떻게 관리해야 하나요?
```
