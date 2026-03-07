# DDS & ROS 2 (Data Distribution Service / Robot Operating System 2)

## 개요
DDS(Data Distribution Service)는 OMG(Object Management Group) 표준 기반의 실시간 발행/구독(Pub/Sub) 미들웨어입니다. 브로커 없이 피어-투-피어(P2P) 방식으로 동작하며 네트워크 참여자들이 자동으로 서로를 발견하고 연결됩니다. ROS 2(Robot Operating System 2)는 DDS를 통신 백본(RMW: ROS Middleware Interface)으로 채택하여 로봇 시스템의 실시간성, 확장성, 분산 처리를 지원합니다.

의료 로봇 분야에서 ROS 2 + DDS는 **관절 제어, 센서 퓨전, 모션 계획, 안전 감시** 등 핵심 기능의 사실상의 표준 미들웨어로 자리 잡았습니다. RTI Connext DDS는 IEC 62304 Class C 인증을 획득하여 의료기기 소프트웨어로도 사용 가능합니다.

---

## DDS 아키텍처 계층

```
Application Layer
┌─────────────────────────────────────────────────────────────┐
│  DDS API (DataWriter / DataReader)                           │
│  사용자 코드는 이 API를 통해 데이터 쓰기/읽기                   │
├─────────────────────────────────────────────────────────────┤
│  DCPS (Data-Centric Publish-Subscribe) Layer                │
│  • Domain     : 격리된 통신 공간 (DomainID로 구분)            │
│  • DomainParticipant: DDS 참여자 (프로세스당 보통 1개)         │
│  • Topic      : 공유 데이터 주제 (이름 + IDL 타입 정의)        │
│  • Publisher  : DataWriter들의 그룹 (QoS 상속)               │
│  • Subscriber : DataReader들의 그룹                          │
│  • DataWriter : 특정 Topic에 데이터 쓰기 (발행)               │
│  • DataReader : 특정 Topic에서 데이터 읽기 (구독)             │
├─────────────────────────────────────────────────────────────┤
│  RTPS (Real-Time Publish-Subscribe) Wire Protocol           │
│  (OMG DDSI-RTPS Specification v2.3, UDP 기반)               │
├─────────────────────────────────────────────────────────────┤
│  UDP/IP + TSN Ethernet (물리/링크 계층)                      │
└─────────────────────────────────────────────────────────────┘
```

---

## RTPS 프로토콜 구조

### RTPS 메시지 형식

```
RTPS 메시지 구조:
┌─────────────────────────────────────────────────────────────────┐
│  RTPS Header (20 Byte)                                          │
│  ┌────────┬───────────────────────────────────────────────────┐ │
│  │ "RTPS" │ Protocol Version │ Vendor ID │ GUID Prefix (12B) │ │
│  │ (4B)   │     (2B)         │   (2B)    │                   │ │
│  └────────┴───────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  SubMessage 1                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ SubMessage ID (1B) │ Flags (1B) │ OctetsToNextHeader (2B)  ││
│  │ SubMessage Content (가변)                                   ││
│  └─────────────────────────────────────────────────────────────┘│
│  SubMessage 2, 3, ... (가변 개수)                               │
└─────────────────────────────────────────────────────────────────┘

GUID (Globally Unique Identifier, 16 Byte):
  GUID Prefix (12B): 호스트 IP + 프로세스 PID + 랜덤
  Entity ID (4B): 00 00 01 C1 (Participant)
                  00 00 02 07 (Writer)
                  00 00 07 04 (Reader)

주요 SubMessage ID:
  0x15: DATA            ← 실제 데이터 (직렬화된 페이로드)
  0x06: ACKNACK         ← RELIABLE 모드에서 수신 확인
  0x0C: HEARTBEAT       ← DataWriter의 버퍼 상태 알림
  0x09: INFO_TS         ← 타임스탬프 정보
  0x0E: INFO_DEST       ← 대상 참여자 지정
```

### DDS Discovery 메커니즘 (자동 탐색)

```
2단계 Discovery Protocol:

Phase 1: SPDP (Simple Participant Discovery Protocol)
  → 새 Participant 시작 시 UDP 멀티캐스트로 자신 알림
  → 멀티캐스트 주소: 239.255.0.1 (기본)
  → 포트: PB + DomainID × DG + BUILTIN_MULTICAST_PORT_GAIN
         PB=7400, DG=250: Domain 0 → 7400

  Participant 1 ──► 239.255.0.1:7400 (SPDP Announce: "나 있음")
  Participant 2 ──► 239.255.0.1:7400 (SPDP Announce: "나도 있음")
  → 서로를 발견, Liveliness 교환

Phase 2: SEDP (Simple Endpoint Discovery Protocol)
  → Participant 간 유니캐스트로 DataWriter/DataReader 정보 교환
  → Topic 이름 + 타입명 + QoS 정책 공유
  → QoS 호환성 검사 후 자동 매칭

ROS 2 Domain ID 기반 격리:
  Domain 0:  로봇 실시간 제어 (기본)
  Domain 42: 시뮬레이션 (Gazebo)
  Domain 1:  원격 모니터링
  포트 계산: 7400 + DomainID × 250
```

---

## DDS QoS 정책 (Quality of Service)

### 핵심 QoS 정책 상세

```
주요 QoS 정책 22개 중 의료 로봇에서 자주 사용하는 7가지:

1. RELIABILITY (신뢰성):
   BEST_EFFORT : 손실 허용, 오버헤드 최소 (센서 스트리밍)
   RELIABLE    : 손실 없음, ACKNACK 재전송 (제어 명령)

2. HISTORY (이력 관리):
   KEEP_LAST(N): 최근 N개만 유지 (N=1~수백)
   KEEP_ALL    : 모든 이력 유지 (메모리 주의)

3. DURABILITY (내구성):
   VOLATILE        : 구독 전 데이터 무시 (센서 실시간값)
   TRANSIENT_LOCAL : 신규 구독자에게 마지막 값 전송 (상태 정보)
   TRANSIENT       : 영속 서비스를 통해 유지
   PERSISTENT      : 디스크에 저장

4. DEADLINE (데드라인):
   period: 이 시간 내 데이터 없으면 DEADLINE_MISSED 콜백 발생
   → 1ms: 실시간 제어 감시
   → 100ms: 센서 단절 감지

5. LIVELINESS (생존 여부):
   AUTOMATIC : DDS가 자동으로 생존 갱신
   MANUAL_BY_PARTICIPANT / MANUAL_BY_TOPIC
   lease_duration: 이 시간 내 신호 없으면 LIVELINESS_LOST

6. LATENCY_BUDGET (지연 예산):
   duration: DDS가 이 시간 내 전달 보장 (최선 노력)

7. OWNERSHIP (소유권):
   SHARED    : 여러 Writer가 같은 Topic에 동시 발행
   EXCLUSIVE : strength가 높은 Writer만 독점 발행
               → 1번 제어기 장애 시 2번 자동 전환
```

### QoS 프리셋 (의료 로봇 상황별)

```
시나리오별 QoS 조합:

A. 실시간 제어 루프 (1kHz, 관절 명령):
   Reliability: RELIABLE
   History:     KEEP_LAST(1)
   Deadline:    1ms
   Liveliness:  AUTOMATIC, lease_duration=100ms
   Durability:  VOLATILE

B. 안전 상태 신호 (비상정지, Safety-Critical):
   Reliability: RELIABLE
   History:     KEEP_LAST(10)
   Deadline:    2ms
   Liveliness:  AUTOMATIC, lease_duration=50ms
   Durability:  TRANSIENT_LOCAL  ← 신규 구독자도 즉시 수신

C. 영상 스트리밍 (30fps, 손실 허용):
   Reliability: BEST_EFFORT
   History:     KEEP_LAST(1)
   Durability:  VOLATILE
   Latency Budget: 33ms

D. 이벤트 로그 (진단 데이터):
   Reliability: RELIABLE
   History:     KEEP_ALL
   Durability:  TRANSIENT_LOCAL
   Deadline:    무제한
```

---

## IDL 타입 정의 및 C++ 예시

### IDL 타입 정의

```idl
// surgical_robot.idl
module medical_robot {
    // 관절 상태
    struct JointState {
        uint64 timestamp_ns;           // 나노초 타임스탬프
        sequence<string> name;         // 관절 이름
        sequence<double> position;     // rad
        sequence<double> velocity;     // rad/s
        sequence<double> effort;       // Nm
    };

    // 비상정지 커맨드
    enum SafetyCommand {
        NORMAL,
        WARNING,
        STOP,
        EMERGENCY_STOP
    };

    struct SafetyStatus {
        uint64 timestamp_ns;
        SafetyCommand command;
        string reason;
        boolean acknowledged;
    };
};
```

### ROS 2 C++ Publisher (실시간 1kHz)

```cpp
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/joint_state.hpp>
#include <rclcpp/qos.hpp>

class JointStatePublisher : public rclcpp::Node {
public:
    JointStatePublisher() : Node("joint_state_publisher") {
        // 실시간 제어 QoS
        rclcpp::QoS realtime_qos(1);
        realtime_qos.reliability(rclcpp::ReliabilityPolicy::Reliable);
        realtime_qos.durability(rclcpp::DurabilityPolicy::Volatile);
        realtime_qos.deadline(rclcpp::Duration(0, 1'000'000));  // 1ms

        publisher_ = this->create_publisher<sensor_msgs::msg::JointState>(
            "/joint_states", realtime_qos);

        // 1kHz 타이머 (1,000,000 ns = 1ms)
        timer_ = this->create_wall_timer(
            std::chrono::microseconds(1000),
            std::bind(&JointStatePublisher::publish_cb, this));
    }

private:
    void publish_cb() {
        auto msg = sensor_msgs::msg::JointState();
        msg.header.stamp = this->now();
        msg.name = {"joint_1", "joint_2", "joint_3",
                    "joint_4", "joint_5", "joint_6"};
        msg.position = read_encoders();   // 실제 엔코더 읽기
        msg.velocity = compute_velocity();
        msg.effort   = read_torque_sensors();
        publisher_->publish(msg);
    }

    rclcpp::Publisher<sensor_msgs::msg::JointState>::SharedPtr publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    // 실시간 스케줄러 설정 (SCHED_FIFO, 우선순위 80)
    struct sched_param param = {.sched_priority = 80};
    sched_setscheduler(0, SCHED_FIFO, &param);
    rclcpp::spin(std::make_shared<JointStatePublisher>());
    rclcpp::shutdown();
}
```

---

## DDS RMW 구현체 비교

| 구현체 | 라이브러리 | 라이선스 | 의료/기능안전 인증 | 성능 | 추천 용도 |
|--------|-----------|---------|-----------------|------|---------|
| eProsima Fast DDS | rmw_fastrtps | Apache 2.0 | IEC 62304 진행 중 | 높음 | ROS 2 기본, 오픈소스 |
| Eclipse Cyclone DDS | rmw_cyclonedds | Eclipse 2.0 | - | 매우 높음 | 저지연 요구 |
| RTI Connext DDS | rmw_connextdds | 상용 | **IEC 62304, DO-178C** | 최고 | 의료기기, 방산 |
| GurumDDS | rmw_gurumdds | 상용 | - | 높음 | 국내 프로젝트 |
| Eclipse iceoryx | rmw_iceoryx | Apache 2.0 | - | 극고성능 | Zero-copy IPC |

```bash
# RMW 구현체 변경
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ros2 run demo_nodes_cpp talker

# 현재 RMW 확인
ros2 doctor --report | grep middleware

# Fast DDS 로 명시 변경
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

---

## ROS 2 CLI 진단 도구

```bash
# 토픽 목록 및 타입 확인
ros2 topic list -t

# 토픽 데이터 실시간 출력
ros2 topic echo /joint_states

# 발행 주파수 측정
ros2 topic hz /joint_states
# expected: average rate: 999.8 Hz (목표: 1000 Hz)

# 엔드-투-엔드 지연 측정
ros2 topic delay /joint_states
# expected: average delay: 0.000234 s (234µs)

# QoS 정책 확인
ros2 topic info /joint_states --verbose

# 노드 상세 정보
ros2 node info /joint_state_publisher

# 토픽 그래프 시각화
ros2 run rqt_graph rqt_graph

# DDS 레이어 진단
ros2 run rmw_fastrtps_cpp fastdds_discovery_server --help

# 도메인 격리
export ROS_DOMAIN_ID=5
ros2 topic list   # Domain 5의 토픽만 표시
```

---

## DDS + TSN 통합 설정

```
ROS 2 / DDS over TSN 네트워크 구성:

TSN VLAN 우선순위 매핑:
  PCP 7 (VLAN 10): 비상정지, 안전 신호 (/safety_status)
  PCP 5 (VLAN 20): 실시간 제어 (/joint_states, /cmd_vel)
  PCP 3 (VLAN 30): 센서 스트리밍 (/scan, /image_raw)
  PCP 0 (VLAN 40): 진단/로깅 (/diagnostics, /rosout)

Linux IP DSCP → PCP 매핑 (tc qdisc):
  sudo tc qdisc add dev eth0 root handle 1: prio
  sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 \
    u32 match ip dscp 46 0xfc flowid 1:1  # EF → PCP7

Fast DDS QoS 프로파일 (XML, DDS+TSN):
  <profiles>
    <participant profile_name="rt_control">
      <rtps>
        <userTransports>
          <transport_id>UDPv4Transport</transport_id>
        </userTransports>
        <useBuiltinTransports>false</useBuiltinTransports>
      </rtps>
    </participant>
    <transport_descriptors>
      <transport_descriptor>
        <transport_id>UDPv4Transport</transport_id>
        <type>UDPv4</type>
        <socket_buffer_size>1048576</socket_buffer_size>
        <TTL>2</TTL>
      </transport_descriptor>
    </transport_descriptors>
  </profiles>

Cyclone DDS 설정 파일 (cyclone_rt.xml):
  <CycloneDDS>
    <Domain>
      <General>
        <Interfaces><NetworkInterface name="eth0" priority="default"/></Interfaces>
      </General>
      <Tracing>
        <Verbosity>warning</Verbosity>
      </Tracing>
    </Domain>
  </CycloneDDS>

  export CYCLONEDDS_URI=file://cyclone_rt.xml
```

---

## ROS 2 주요 패키지 (의료 로봇)

```bash
# MoveIt 2: 6-DOF 로봇 모션 계획
sudo apt install ros-humble-moveit
# → OMPL, STOMP 기반 경로 계획
# → 충돌 감지 (FCL, Bullet Physics)

# ros2_control: 하드웨어 추상화 레이어
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers
# → JointTrajectoryController: 궤적 추종
# → JointStatebroadcaster: 엔코더 → /joint_states

# Nav2: 모바일 플랫폼 자율 이동
sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup

# robot_localization: IMU + 엔코더 + GPS 센서 퓨전 (EKF/UKF)
sudo apt install ros-humble-robot-localization

# image_transport: 압축 영상 전송 (JPEG, H.264)
sudo apt install ros-humble-image-transport-plugins

# diagnostic_updater: 시스템 진단 표준화
sudo apt install ros-humble-diagnostic-updater

# 실시간 커널 패치 (PREEMPT_RT):
# 의료 로봇용 제어 루프는 PREEMPT_RT 패치 커널 필수
uname -r  # 예: 5.15.0-1032-realtime
chrt -f 80 ros2 run my_robot joint_controller
```

---

## Reference
- [OMG DDS Specification v1.4](https://www.omg.org/spec/DDS/1.4/PDF)
- [OMG DDSI-RTPS Specification v2.3](https://www.omg.org/spec/DDSI-RTPS/2.3/PDF)
- [ROS 2 Documentation (Humble Hawksbill)](https://docs.ros.org/en/humble/)
- [eProsima Fast DDS Documentation](https://fast-dds.docs.eprosima.com/)
- [Eclipse Cyclone DDS](https://cyclonedds.io/)
- [RTI Connext DDS Professional](https://www.rti.com/products/connext-dds-professional)
- [ros2_control Framework](https://control.ros.org/master/index.html)
- [MoveIt 2 Documentation](https://moveit.picknik.ai/)
