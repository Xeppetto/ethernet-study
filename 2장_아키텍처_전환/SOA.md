# Service Oriented Architecture (SOA)

## 개요
SOA(Service Oriented Architecture)는 소프트웨어 기능을 독립적인 '서비스(Service)' 단위로 분리하고 네트워크를 통해 동적으로 연결하는 아키텍처 스타일입니다. 기존 CAN 기반의 정적 신호 브로드캐스팅에서 벗어나, 필요한 서비스만 동적으로 탐색하고 사용하는 방식으로 전환합니다.

---

## 신호 중심 vs 서비스 중심

### Signal-Based (CAN 방식)
```
ECU A                    CAN 버스                    ECU B, C, D
  │                         │                            │
  │──── 조인트 속도 신호 ────►│──────────────────────────►│ (항상 전송)
  │    (10ms 주기)           │                            │
  │──── 온도 신호 ───────────►│──────────────────────────►│
  │    (100ms 주기)          │                            │

문제점:
  - 수신자가 필요 없어도 항상 전송 → 대역폭 낭비
  - 새 ECU 추가 시 DBC 파일 전체 수정 필요
  - 정적 구성, 유연성 부족
```

### Service-Based (Ethernet SOA)
```
Provider (ECU A)          네트워크          Consumer (ECU B)
     │                       │                    │
     │ ①SD Offer ────────────►│                    │
     │                       │◄── ②SD Find ────────│
     │◄───────────────────────── ③Subscribe ────────│
     │────── ④Event (변경 시만) ────────────────────►│

장점: 동적 탐색, 필요 시에만 전송, Plug & Play
```

---

## SOA 핵심 패턴

### 1. Request/Response (Method Call)
```
Client                            Server
  │──── SOME/IP Request ─────────►│
  │     Service ID: 0x0101        │
  │     Method ID:  0x0010        │  ← 조인트 각도 읽기
  │     Request ID: 0x0001        │
  │                               │  (처리 중)
  │◄─── SOME/IP Response ──────────│
  │     Method ID:  0x8010        │  ← 응답 (MSB=1)
  │     Return Code: 0x00 (OK)    │
  │     Payload: 45.3°            │
```

### 2. Publish/Subscribe (이벤트 통지)
```
Publisher              Middleware              Subscriber
(센서 ECU)                                  (제어 ECU)
  │                                              │
  │─── 이벤트 발행 (Topic: joint_state) ──────────►│
  │    (데이터 변경 시 또는 주기적으로)              │
  │─── 이벤트 발행 ──────────────────────────────►│
```

### 3. Field (Getter / Setter / Notifier)
```
속성 값 조회:  Getter → Request/Response
속성 값 설정:  Setter → Request/Response
변경 알림:    Notifier → Event

예: 로봇 최대 속도 관리
  Getter:   현재 최대 속도 = 1.5 m/s
  Setter:   최대 속도를 2.0 m/s로 변경 요청
  Notifier: 변경 완료 후 구독자에게 알림
```

---

## SOME/IP 메시지 구조

```
SOME/IP 헤더 (8 Byte 고정):
┌─────────┬─────────┬──────────┬────────────┐
│ Svc ID  │ Mth ID  │  Length  │ Request ID │
│ 2 Byte  │ 2 Byte  │  4 Byte  │  4 Byte    │
├─────────┴─────────┴──────────┴────────────┤
│ Proto V  │ Iface V  │ Msg Type │ Ret. Code │
│  1 Byte  │  1 Byte  │  1 Byte  │  1 Byte   │
└──────────┴──────────┴──────────┴───────────┘
뒤에 Payload (데이터) 추가

Message Type:
  0x00: Request
  0x01: Request No Return
  0x02: Notification (Event)
  0x80: Response
  0x81: Error

Return Code:
  0x00: E_OK (성공)
  0x01: E_NOT_OK (일반 오류)
  0x02: E_UNKNOWN_SERVICE
  0x05: E_NOT_REACHABLE
```

---

## SOME/IP Service Discovery (SD)

```
SD 포트: UDP 30490 (Multicast 또는 Unicast)

메시지 흐름:
1. Provider 시작 → Offer Service (Multicast)
2. Consumer 시작 → Find Service (Multicast)
3. Provider → Consumer Unicast Offer 응답
4. Consumer → Subscribe 요청 (Unicast)
5. Provider → SubscribeAck
6. 이벤트 전송 시작
```

```bash
# Wireshark 필터
someipsd                          # SD 전체
someipsd.entry.serviceid == 0x0101 # 특정 서비스 SD
someip                            # SOME/IP 전체 트래픽
udp.port == 30490                 # SD 포트
```

---

## DDS (Data Distribution Service)

OMG 표준 기반의 데이터 중심 미들웨어로, ROS 2의 통신 백본입니다.

```
DDS 도메인 (Domain ID 기반 격리)
┌─────────────────────────────────────────────────────────┐
│  DataWriter (Publisher)         DataReader (Subscriber) │
│  Topic: /joint_states           Topic: /joint_states    │
│  QoS: RELIABLE                  QoS: RELIABLE           │
│         └──── RTPS 프로토콜 (UDP) ────────────────►       │
└─────────────────────────────────────────────────────────┘

자동 탐색: Simple Discovery Protocol → 동일 도메인에서 자동 매칭
```

### DDS QoS 주요 정책

| QoS 정책 | 옵션 | 설명 |
|---------|------|------|
| Reliability | BEST_EFFORT / RELIABLE | 신뢰성 보장 여부 |
| Durability | VOLATILE / TRANSIENT_LOCAL | 신규 구독자에게 이전 데이터 전달 |
| Deadline | 주기 설정 | 데이터 최대 도착 기한 |
| History | KEEP_LAST(N) / KEEP_ALL | 히스토리 보관 |
| Liveliness | 타임아웃 설정 | 노드 생존 확인 |

```python
# ROS 2 Python: DDS 기반 Publisher 예시
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

class JointPublisher(Node):
    def __init__(self):
        super().__init__('joint_publisher')
        qos = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=10
        )
        self.pub = self.create_publisher(JointState, '/joint_states', qos)
        self.timer = self.create_timer(0.001, self.publish)  # 1kHz

    def publish(self):
        msg = JointState()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.name = ['joint_1', 'joint_2']
        msg.position = [1.57, 0.78]
        self.pub.publish(msg)

rclpy.init()
node = JointPublisher()
rclpy.spin(node)
```

---

## SOA 미들웨어 비교

| 특성 | SOME/IP | DDS | MQTT | OPC-UA |
|------|---------|-----|------|--------|
| 표준 기관 | AUTOSAR | OMG | OASIS | OPC Foundation |
| 주요 도메인 | 차량 | 로봇/국방 | IoT/클라우드 | 공장 자동화 |
| 전송 | UDP/TCP | UDP | TCP | TCP |
| QoS | 제한적 | 풍부 (22개) | 3단계 | 구독 모델 |
| 서비스 탐색 | SD (명시적) | SDP (자동) | 브로커 중심 | OPC-UA SD |
| 실시간성 | 높음 | 매우 높음 | 낮음 | 중간 |
| 브로커 | 불필요 | 불필요 | 필요 | 불필요 |
| ROS 2 | rmw_someip | rmw_fastrtps | 별도 | 별도 |

---

## 의료 로봇 SOA 서비스 설계 예시

```
수술 로봇 서비스 인터페이스 정의

Service 0x0101: JointControlService
  Method 0x0001: SetPosition(joint_id:u8, angle:f32) → Result
  Method 0x0002: GetPosition(joint_id:u8) → angle:f32
  Event  0x8001: PositionChanged {joint_id, angle, timestamp}

Service 0x0102: SurgicalToolService
  Method 0x0001: ActivateTool(tool_id:u8) → Result
  Method 0x0002: SetGripForce(force_N:f32) → Result
  Event  0x8001: ToolStatus {tool_id, state, force}

Service 0x0201: CameraService
  Method 0x0001: Configure(width:u16, height:u16, fps:u8) → Result
  Event  0x8001: FrameReady {frame_id, timestamp, data}

Service 0x0301: DiagnosticsService
  Method 0x0001: ReadDTC() → dtc_list:[]
  Method 0x0002: ClearDTC() → Result
  Method 0x0003: ReadECUInfo() → info:{}
```

---

## Reference
- [AUTOSAR - SOME/IP Protocol Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [OMG DDS Specification v1.4](https://www.omg.org/spec/DDS/1.4/PDF)
- [ROS 2 DDS 미들웨어](https://docs.ros.org/en/rolling/Concepts/About-Different-Middleware-Vendors.html)
- [MQTT v5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
