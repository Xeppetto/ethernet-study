# 플랫폼화 전략 (Platform Strategy)

## 개요
플랫폼화 전략은 다양한 제품 라인업을 **하나의 공통 기반(플랫폼)** 위에서 개발하여 비용을 절감하고 개발 기간을 단축하는 핵심 전략입니다. 자동차 산업(VW MEB, Hyundai E-GMP)처럼 의료 로봇 분야에서도 공통 HW/SW 플랫폼을 구축하고 이를 기반으로 다양한 파생 로봇을 개발하는 방식이 표준화되고 있습니다.

---

## 플랫폼화 효과 분석

```
플랫폼 기반 개발 효과:

개발 비용:
  비플랫폼: 각 제품 독립 개발
    제품 A: $10M, 제품 B: $9M, 제품 C: $8M = $27M
  플랫폼 전략:
    플랫폼: $15M, 제품 A: $3M, B: $2M, C: $1.5M = $21.5M
    절감: 20%~40%

개발 기간:
  비플랫폼: 36개월/제품
  플랫폼 기반: 첫 제품 42개월, 이후 18개월/제품
  Time-to-Market 단축: 50%+

품질:
  검증된 플랫폼 재사용 → 초기 결함 60% 감소
  공통 테스트 환경 → 회귀 테스트 효율화
```

---

## 하드웨어 플랫폼 설계

```
수술 로봇 HW 플랫폼 구조:

공통 베이스 플랫폼:
┌─────────────────────────────────────────────────────────────┐
│               로봇 기본 플랫폼 (RP-100)                      │
│                                                             │
│  ┌──────────────────┐     ┌─────────────────────────────┐  │
│  │  로봇 컴퓨터 모듈  │     │  표준화된 관절 모듈          │  │
│  │  (RCM-200)       │     │  (JM-100, 직렬 연결)         │  │
│  │  - SoC NXP S32G  │     │  - 모터 + 인코더             │  │
│  │  - TSN Ethernet  │     │  - 관절 제어 MCU             │  │
│  │  - 안전 ECU      │     │  - 토크 센서                 │  │
│  │  - OTA 클라이언트 │     │  - EtherCAT 인터페이스       │  │
│  └──────────────────┘     └─────────────────────────────┘  │
│                                                             │
│  파생 제품:                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │  수술 로봇 A │ │  수술 로봇 B │ │   재활 로봇       │   │
│  │  (6 DOF)    │ │  (7 DOF)    │ │   (4 DOF, CE)    │   │
│  │  RP-100 기반│ │  RP-100 기반│ │   RP-100 기반     │   │
│  │  JM-100 × 6 │ │  JM-100 × 7 │ │   JM-100 × 4     │   │
│  └──────────────┘ └──────────────┘ └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘

관절 모듈 표준화 효과:
  - 크기: S(±30Nm), M(±100Nm), L(±300Nm) 3가지
  - 동일 SW 스택, 파라미터만 변경
  - 재고 공유 (JM-100M이 6DOF/7DOF 공통)
```

---

## 소프트웨어 플랫폼 (ROS 2 기반)

```
로봇 SW 플랫폼 계층:

┌──────────────────────────────────────────────────────────────┐
│                Application Layer (제품별 커스텀)               │
│  수술 로봇 A 앱   │   수술 로봇 B 앱   │   재활 로봇 앱       │
├──────────────────────────────────────────────────────────────┤
│               Platform Services (공통)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ 안전 모니터│ │ OTA 클라이언│ │ 진단 서비스 │ │ 클라우드 연동 │  │
│  │ (ASIL D) │ │ (UPTANE)  │ │ (UDS/DoIP) │ │ (MQTT/REST) │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
├──────────────────────────────────────────────────────────────┤
│               Middleware (ROS 2 + DDS)                        │
│  - FastDDS (실시간 통신)                                      │
│  - TSN 네트워크 추상화                                        │
│  - SOME/IP 게이트웨이                                         │
├──────────────────────────────────────────────────────────────┤
│               OS / RTOS                                       │
│  - Linux 5.15 + PREEMPT_RT                                   │
│  - Hypervisor (QNX) : Safety Partition                       │
├──────────────────────────────────────────────────────────────┤
│               Hardware Abstraction Layer (HAL)                │
│  - 관절 모듈 드라이버 (EtherCAT)                              │
│  - 센서 드라이버 (I2C, SPI, Ethernet)                        │
│  - 전원 관리                                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## ROS 2 플랫폼 패키지 구조

```bash
# 의료 로봇 ROS 2 플랫폼 패키지 구조
robot_platform/
├── platform_core/           # 공통 플랫폼 패키지
│   ├── robot_base/          # 기본 로봇 인터페이스
│   │   ├── joint_interface.py
│   │   ├── safety_monitor.py
│   │   └── e2e_protection.py
│   ├── diagnostics/         # 진단 서비스 (UDS)
│   │   ├── dtc_manager.py
│   │   └── doip_server.py
│   ├── cloud_connector/     # 클라우드 연동
│   │   ├── mqtt_client.py
│   │   └── ota_client.py
│   └── common_msgs/         # 공통 ROS 2 메시지
│       ├── msg/JointState.msg
│       ├── msg/SafetyStatus.msg
│       └── srv/ExecuteCommand.srv
│
├── product_surgical_a/      # 수술 로봇 A (플랫폼 사용)
│   ├── trajectory_planner.py
│   ├── haptic_controller.py
│   └── config/surgical_a.yaml
│
├── product_surgical_b/      # 수술 로봇 B
│   ├── 7dof_kinematics.py
│   └── config/surgical_b.yaml
│
└── product_rehab/           # 재활 로봇
    ├── therapy_mode.py
    └── config/rehab.yaml
```

```python
# 공통 플랫폼 Safety Monitor (재사용)
import rclpy
from rclpy.node import Node
from std_msgs.msg import Bool
from platform_core.common_msgs.msg import SafetyStatus

class PlatformSafetyMonitor(Node):
    """
    공통 안전 모니터링 노드
    - 모든 제품에서 재사용
    - 제품별 임계치만 파라미터로 설정
    """

    def __init__(self):
        super().__init__('safety_monitor')

        # 파라미터 (제품별 YAML에서 설정)
        self.declare_parameter('max_joint_speed_mm_s', 500.0)
        self.declare_parameter('max_force_n', 10.0)
        self.declare_parameter('comm_timeout_ms', 100.0)

        self.max_speed = self.get_parameter('max_joint_speed_mm_s').value
        self.max_force = self.get_parameter('max_force_n').value
        self.comm_timeout = self.get_parameter('comm_timeout_ms').value / 1000

        # 구독/발행
        self.joint_sub = self.create_subscription(
            SafetyStatus, '/robot/joint_status',
            self.on_joint_status, 10
        )
        self.estop_pub = self.create_publisher(Bool, '/robot/emergency_stop', 10)

        self.last_msg_time = self.get_clock().now()

        # 타임아웃 타이머
        self.timer = self.create_timer(
            0.01,  # 10ms 주기 확인
            self.check_comm_timeout
        )

    def on_joint_status(self, msg: SafetyStatus):
        self.last_msg_time = self.get_clock().now()

        # 속도 제한 확인
        if abs(msg.velocity) > self.max_speed:
            self.trigger_estop(f"속도 초과: {msg.velocity:.1f} mm/s")

        # 힘 제한 확인
        if abs(msg.force) > self.max_force:
            self.trigger_estop(f"힘 초과: {msg.force:.1f} N")

    def check_comm_timeout(self):
        elapsed = (self.get_clock().now() - self.last_msg_time).nanoseconds / 1e9
        if elapsed > self.comm_timeout:
            self.trigger_estop(f"통신 타임아웃: {elapsed*1000:.0f}ms")

    def trigger_estop(self, reason: str):
        self.get_logger().error(f"[SAFETY] E-STOP: {reason}")
        msg = Bool()
        msg.data = True
        self.estop_pub.publish(msg)
```

---

## 개방형 플랫폼 (SDK/API)

```
외부 개발자를 위한 개방형 인터페이스:

SDK 레이어:
  Robot API (Python/C++):
    robot.move_to(target_pose, speed=100)
    robot.get_joint_state() → JointState
    robot.set_force_limit(max_n=10.0)
    robot.subscribe_sensor(callback)

  Simulation API:
    sim.load_anatomy(ct_file)
    sim.run_trajectory(trajectory)
    sim.check_collision() → bool

  데이터 API:
    analytics.get_surgery_report(session_id)
    analytics.export_dataset(format='FHIR')

개방형 플랫폼 사례:
  Intuitive da Vinci Research Kit (dVRK):
    - ROS 인터페이스 제공
    - 연구용 API 공개
    - 학술 협력 50+ 기관

  KUKA Sunrise OS:
    - Java 기반 SDK
    - 외부 애플리케이션 연동
    - 산업 로봇 표준

표준 인터페이스:
  ROS 2: 로봇 미들웨어 표준
  OPC UA: 산업 장비 인터페이스
  HL7 FHIR: 의료 데이터 교환
  DICOM: 의료 영상 표준
```

---

## 플랫폼화 성공 사례

```
의료 로봇 플랫폼 사례:

Intuitive Surgical (da Vinci):
  플랫폼: 마스터 콘솔 + 비전 시스템
  제품: Si, Xi, X, SP (단일 플랫폼, 다양한 설정)
  효과: SP에서 Xi 기술 자산 70% 재활용

Medtronic Hugo:
  설계: 모듈형 암 구조
  장점: 수술실 배치 유연성
  (여러 암을 독립적으로 위치 조정 가능)

Stryker Mako:
  플랫폼: Mako 로봇 팔
  제품: 전체 무릎, 부분 무릎, 엉덩이
  → 동일 HW에 수술별 SW 패키지 변경

자동차 벤치마크:
  VW MEB 플랫폼 (전기차):
    ID.3, ID.4, ID.5, Audi Q4, Skoda Enyaq...
    개발비 분담으로 제품당 50% 절감
```

---

## Reference
- [HBR - Pipelines, Platforms, and the New Rules of Strategy](https://hbr.org/2016/04/pipelines-platforms-and-the-new-rules-of-strategy)
- [ROS 2 for Medical Devices](https://ros.org/blog/getting-started/)
- [Intuitive Surgical da Vinci Research Kit](https://research.intusurg.com/index.php/Main_Page)
- [KUKA Sunrise OS SDK](https://www.kuka.com/en-us/products/robotics-systems/software/system-software/kuka_sunrise_os)
- [OPC UA for Robotics](https://opcfoundation.org/developer-tools/specifications-opc-ua-information-models/opc-unified-architecture-for-robotics/)
