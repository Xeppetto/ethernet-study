# Service Oriented Architecture (SOA)

## 신호 기반에서 서비스 기반으로: 왜 바꿔야 하는가

CAN 기반 시스템에서 관절 각도는 버스에 주기적으로 브로드캐스트된다. 누가 듣든 듣지 않든, 필요하든 필요하지 않든, 10밀리초마다 메시지가 나간다. 이것이 신호 기반(signal-based) 아키텍처다. 단순하고 예측 가능하지만, 확장성이 없다. 새로운 기능을 추가하려면 DBC 파일을 열고 메시지 ID를 할당하고 모든 관련 ECU의 소프트웨어를 다시 컴파일해야 한다.

Ethernet 기반 SOA는 다르게 동작한다. 관절 각도를 제공하는 서비스가 네트워크에 자신을 광고한다. 필요한 컴포넌트가 그 서비스를 찾아서 구독한다. 데이터는 변경이 있을 때만, 또는 구독자가 원하는 주기로 전달된다. 새로운 기능은 그냥 서비스를 찾아서 구독하면 된다—기존 코드를 건드리지 않고.

이것이 소프트웨어 정의 아키텍처의 통신 기반이다.

---

## SOA의 세 가지 통신 패턴

### Request/Response: 명확한 질문과 답변

클라이언트가 서버에 요청을 보내고 응답을 기다리는 패턴이다. 결과가 필요한 경우, 즉 "지금 관절 1번의 각도가 얼마인가?"처럼 즉각적인 답이 필요할 때 사용한다.

```
Motion DC (Client)                    Joint Service (Server)
      │                                       │
      │── GetJointAngle(joint_id=1) ─────────►│
      │   SOME/IP: Svc=0x0101, Method=0x0001  │
      │   Request ID: 0x00000042              │ (처리: 200µs)
      │                                       │
      │◄── Response: angle=1.5708 rad ────────│
      │    Return Code: E_OK (0x00)           │
      │                                       │
      │── SetGripForce(force=2.5N) ──────────►│
      │◄── Response: E_OK ────────────────────│
```

Request/Response는 양방향이고 동기적이다. 서버가 응답하지 않으면 클라이언트는 기다린다(또는 타임아웃). 의료 로봇에서 이 패턴을 실시간 제어 루프(1kHz)에 직접 사용하면 안 된다—서버 응답 지연이 제어 루프를 블로킹하기 때문이다.

### Publish/Subscribe: 데이터 흐름의 주류

가장 많이 사용되는 패턴이다. Publisher는 토픽에 데이터를 발행하고, Subscriber는 원하는 토픽을 구독한다. 1:N, N:1, N:N 모두 가능하다.

```
Joint State Publisher                    Subscribers
(Motion DC, 1kHz)
      │                                  HMI DC
      │── /joint_states ────────────────►│ (화면 표시)
      │   [pos, vel, torque × 12]       │
      │                                  Safety DC
      │   ──────────────────────────────►│ (안전 범위 감시)
      │                                  │
      │                                  Planner Node
      │   ──────────────────────────────►│ (경로 계획)

DDS QoS 설정 (관절 상태):
  Reliability:  BEST_EFFORT (손실 1~2개 허용, 다음 값이 곧 옴)
  History:      KEEP_LAST(1) (최신 값만 유지)
  Durability:   VOLATILE (과거 값 불필요)
  Deadline:     1ms (이 주기로 안 오면 이상 감지)
```

**DDS DEADLINE QoS의 실용적 의미:** `Deadline(1ms)`를 설정하면 DDS 미들웨어가 1ms 이내에 데이터가 도착하지 않을 때 `on_requested_deadline_missed` 콜백을 호출한다. Safety DC는 이 콜백에서 Hold-Last-Command 또는 E-Stop을 트리거한다. 애플리케이션 레벨에서 네트워크 타임아웃을 직접 구현하지 않아도 된다.

### Field: 상태 값 관리

Field는 유지되어야 하는 상태 값을 관리한다. Getter(현재 값 조회), Setter(값 변경), Notifier(변경 시 알림)의 세 가지 오퍼레이션을 묶은 패턴이다.

```
MaxSpeedService (Field 패턴):

  Getter:   현재 최대 속도 = 0.8 m/s
            → 시스템 시작 시 각 컴포넌트가 조회
  Setter:   최대 속도를 1.2 m/s로 변경
            → 수술 단계 전환 시 (더 섬세한 동작 필요 시 낮춤)
  Notifier: 구독자들에게 변경 사실 통보
            → 관절 제어, 속도 제한 로직 모두 즉시 업데이트

구현: SOME/IP Field 또는 DDS Parameter 서버
```

---

## SOME/IP: 차량용 SOA의 표준

SOME/IP(Scalable service-Oriented MiddlEware over IP)는 AUTOSAR가 차량용으로 설계한 미들웨어 프로토콜이다. 의료 로봇이 AUTOSAR Adaptive를 채택하면 SOME/IP가 기본 통신 계층이 된다.

### 헤더 구조 상세

```
SOME/IP 메시지 구조 (최소 16 byte 헤더 + 페이로드):

┌─────────────────────────────────────────────────────────────────┐
│ Byte  0- 1: Service ID (서비스 식별자, 2 byte)                  │
│             예: 0x0101 = JointControlService                    │
│ Byte  2- 3: Method/Event ID (2 byte)                           │
│             0x0001~0x7FFF: Method (Request/Response)           │
│             0x8000~0xFFFE: Event (Notification)                │
│ Byte  4- 7: Length (페이로드 + 나머지 헤더 길이, 4 byte)        │
│ Byte  8- 9: Client ID (요청자 식별, 2 byte)                    │
│ Byte 10-11: Session ID (Request-Response 매칭, 2 byte)         │
│ Byte 12:    Protocol Version (현재 0x01)                        │
│ Byte 13:    Interface Version (서비스 버전)                     │
│ Byte 14:    Message Type                                        │
│             0x00: Request                                       │
│             0x01: Request No Return (fire-and-forget)          │
│             0x02: Notification (Event)                         │
│             0x80: Response                                      │
│             0x81: Error                                         │
│ Byte 15:    Return Code (0x00=E_OK, 0x01=E_NOT_OK, ...)        │
└─────────────────────────────────────────────────────────────────┘
그 뒤: Payload (직렬화된 데이터, big-endian)
```

### Service Discovery (SD)

SOME/IP SD는 서비스 제공자와 소비자가 동적으로 서로를 찾는 메커니즘이다. UDP 멀티캐스트(기본 포트 30490)를 통해 동작한다.

```
SD 흐름 (JointControlService 등록 및 발견):

Joint Service (Provider)              Motion DC (Consumer)
  시작 → Offer Service 전송           시작 → Find Service 전송
  ──────────────────────────────────►
  SOME/IP SD: OfferService
  Service ID: 0x0101, Port: 40000
                                      ◄──────────────────────
                                      FindService: 0x0101 요청
  ──────────────────────────────────►
  Unicast OfferService → Motion DC

  ◄──────────────────────────────────
                                      SubscribeEventgroup
                                      Event: 0x8001 (Position)

  ──────────────────────────────────►
  SubscribeEventgroupAck

  이후 이벤트 전송 시작 (1kHz)...
```

```bash
# Wireshark로 SOME/IP SD 분석
# 필터: someipsd
# 또는: udp.port == 30490

# vsomeip 설정 예시 (의료 로봇 Joint Service)
{
  "unicast": "192.168.10.10",
  "logging": { "level": "warning" },
  "applications": [
    { "name": "JointService", "id": "0x1100" }
  ],
  "services": [
    {
      "service": "0x0101",
      "instance": "0x0001",
      "reliable": { "port": "40000" },
      "unreliable": "40001"
    }
  ],
  "routing": "JointService"
}
```

---

## DDS: 로봇공학의 미들웨어 표준

OMG(Object Management Group) DDS 표준은 ROS 2의 통신 백본이다. SOME/IP가 차량 업계의 선택이라면, DDS는 로봇·국방·항공 업계의 선택이다. 의료 로봇은 두 세계의 교차점에 있어 둘 다 만날 가능성이 높다.

### DDS vs SOME/IP: 핵심 차이

| 관점 | SOME/IP | DDS |
|---|---|---|
| 서비스 발견 | 명시적 SD 메시지 (offer/find) | 자동 SDP (같은 도메인 내) |
| QoS 정책 | 제한적 (신뢰성, 이벤트 주기) | 22개 정책 (풍부) |
| 데이터 모델 | 바이트 스트림 (직렬화 별도) | 타입드 데이터 (IDL 정의) |
| 브로커 | 없음 (P2P) | 없음 (P2P) |
| ROS 2 지원 | rmw_cyclonedds_someip (실험적) | 기본 (rmw_cyclonedds, rmw_fastrtps) |
| 표준 기관 | AUTOSAR | OMG |
| 의료 로봇 적합성 | AUTOSAR Adaptive 환경 | ROS 2 환경 |

### DDS QoS와 TSN의 연계

DDS QoS 정책이 Ethernet 계층의 TSN 설정과 어떻게 매핑되는지 이해하면 전체 시스템의 타이밍 보장을 설계할 수 있다.

```
DDS QoS ↔ TSN ↔ DSCP 매핑 (의료 로봇 설계):

DDS Topic          DDS QoS               TSN       VLAN  DSCP
─────────────────────────────────────────────────────────────────
/e_stop            RELIABLE              PCP=7     V10   CS7
                   DEADLINE(1ms)         TAS 최우선
/joint_commands    RELIABLE              PCP=6     V20   CS6
                   DEADLINE(1ms)         TAS 스케줄링
/joint_states      BEST_EFFORT           PCP=5     V20   CS5
                   KEEP_LAST(1)          CBS 보장
/camera/image      BEST_EFFORT           PCP=3     V30   AF31
                   HISTORY KEEP_LAST(3)  CBS 대역폭
/diagnostics       BEST_EFFORT           PCP=0     V50   DF
                   KEEP_ALL              Best Effort
```

```python
# ROS 2에서 DEADLINE QoS 설정 예시
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy
from rclpy.duration import Duration

class SafetyMonitorNode(Node):
    def __init__(self):
        super().__init__('safety_monitor')
        qos = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            depth=1,
            deadline=Duration(nanoseconds=1_000_000),  # 1ms DEADLINE
        )
        self.sub = self.create_subscription(
            JointState, '/joint_states', self.joint_callback, qos)
        # DEADLINE miss 시 on_requested_deadline_missed 콜백 자동 호출

    def joint_callback(self, msg):
        # 정상 수신 처리
        for i, torque in enumerate(msg.effort):
            if abs(torque) > SAFE_TORQUE_LIMIT:
                self.trigger_safety_stop()

    # DDS가 자동으로 호출: DEADLINE 초과 시
    def on_requested_deadline_missed(self, event):
        self.get_logger().error('Joint state DEADLINE missed! Triggering safe stop.')
        self.trigger_safety_stop()
```

### 의료 로봇 서비스 인터페이스 설계

ROS 2 Action Server가 수술 로봇의 복잡한 태스크에 적합한 이유: Action은 long-running 작업(수초~수분)에서 진행 상황을 중간에 피드백하고 취소할 수 있는 패턴이다.

```python
# ROS 2 Action: 수술 도구 이동 (목표 위치까지 이동하며 피드백)
# Action 정의 (SurgicalMove.action)
# Goal: target_pose (목표 자세)
# Feedback: current_pose, distance_remaining, force
# Result: final_pose, success

import rclpy
from rclpy.action import ActionServer
from medical_robot_interfaces.action import SurgicalMove

class SurgicalMoveActionServer(Node):
    def __init__(self):
        super().__init__('surgical_move_server')
        self._action_server = ActionServer(
            self, SurgicalMove, 'surgical_move',
            self.execute_callback)

    async def execute_callback(self, goal_handle):
        target = goal_handle.request.target_pose
        feedback_msg = SurgicalMove.Feedback()

        # 이동 루프
        while not self._reached(target):
            if goal_handle.is_cancel_requested:
                # 의사가 취소 → 즉시 정지
                goal_handle.canceled()
                return SurgicalMove.Result(success=False)

            current = self._get_current_pose()
            feedback_msg.current_pose = current
            feedback_msg.distance_remaining = self._distance(current, target)
            feedback_msg.force = self._get_tip_force()
            goal_handle.publish_feedback(feedback_msg)

            # 힘이 안전 한계 초과 시 즉시 중단
            if feedback_msg.force > SAFE_FORCE_LIMIT:
                goal_handle.abort()
                return SurgicalMove.Result(success=False)

            await asyncio.sleep(0.001)  # 1ms 주기

        goal_handle.succeed()
        return SurgicalMove.Result(success=True, final_pose=target)
```

---

## 미들웨어 비교: 의료 로봇 적합성

| 특성 | SOME/IP | DDS (ROS 2) | MQTT | OPC-UA |
|---|---|---|---|---|
| 표준 기관 | AUTOSAR | OMG | OASIS | OPC Foundation |
| 실시간 보장 | 높음 (UDP) | 매우 높음 | 낮음 (TCP) | 중간 |
| QoS 풍부도 | 제한적 | 22개 정책 | 3단계 | 구독 기반 |
| ROS 2 통합 | 실험적 | 기본 | 별도 패키지 | 별도 |
| 의료 인증 레퍼런스 | AUTOSAR 생태계 | NASA, 방산, 연구 | IoT/클라우드 | 공장 자동화 |
| 브로커 필요 | 아니오 | 아니오 | 예 (MQTT Broker) | 아니오 |
| 격리된 네트워크 지원 | 예 | 예 (도메인 ID) | 브로커 필요 | 예 |
| 권장 용도 | AUTOSAR 환경 | ROS 2 제어 | 클라우드 텔레메트리 | 진단/HMI |

**격리된 수술실 네트워크에서 MQTT가 부적합한 이유:** MQTT는 브로커(서버)를 중심으로 동작한다. 수술실 네트워크가 외부와 단절되어 있으면 클라우드 MQTT 브로커에 접근할 수 없다. 로컬 브로커를 두면 해결되지만, 브로커가 단일 실패 지점이 된다. DDS와 SOME/IP는 P2P이므로 이 문제가 없다.

---

## 서비스 발견과 보안: 격리된 의료 네트워크에서

수술 로봇 제어 네트워크는 병원 일반 IT 네트워크와 격리되어 있다. 이 환경에서 서비스 발견이 올바르게 동작하도록 설정해야 한다.

```bash
# DDS 도메인 ID로 격리 (다른 도메인은 서로 볼 수 없음)
# 수술 제어 도메인: ID 10
# 진단/모니터링 도메인: ID 20
export ROS_DOMAIN_ID=10  # 모든 제어 노드에 동일 설정

# DDS 멀티캐스트 주소 제한 (서브넷 내에서만 SD 동작)
# CycloneDDS 설정 (cyclone_config.xml):
# <General>
#   <NetworkInterfaceAddress>192.168.10.0/24</NetworkInterfaceAddress>
#   <AllowMulticast>LocalSubnet</AllowMulticast>
# </General>

# SOME/IP 멀티캐스트 주소 설정 (vsomeip.json)
# "multicast": { "address": "239.192.255.251", "port": "30490" }
# → 서브넷 내에서만 SD 가능
```

**802.1X 포트 인증:** 의료 로봇 네트워크에 허가되지 않은 장치가 연결되는 것을 막기 위해, TSN 스위치에서 802.1X를 활성화한다. RADIUS 서버가 장치 인증서를 검증하고, 인증된 장치만 네트워크에 포트를 할당한다.

---

## 참고 문헌

- AUTOSAR SOME/IP Protocol Specification R22-11
- OMG DDS Specification v1.4
- Eclipse CycloneDDS Documentation: https://cyclonedds.io/
- ROS 2 QoS Policies: https://docs.ros.org/en/rolling/Concepts/About-Quality-of-Service-Settings.html
- Maruyama, Y. et al. "Exploring the Performance of ROS2" ISORC (2016)
- IEEE 802.1X-2020: Port-Based Network Access Control

---

*관련: [2장 AUTOSAR Classic vs Adaptive](./AUTOSAR_Classic_vs_Adaptive.md) | [3장 IEEE 802.1Qbv TAS](../3장_TSN_및_결정성_네트워크/IEEE_802.1Qbv.md)*
