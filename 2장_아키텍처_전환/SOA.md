# Service Oriented Architecture (SOA)

## 개요
SOA(Service Oriented Architecture)는 소프트웨어 기능을 독립적인 '서비스(Service)' 단위로 분리하고 네트워크를 통해 동적으로 연결하는 아키텍처 스타일입니다. 기존 CAN 기반의 정적 신호 브로드캐스팅에서 벗어나, 필요한 서비스만 동적으로 탐색하고 사용하는 방식으로 전환합니다.

---

## 신호 중심 vs 서비스 중심 비교

| 특성 | Signal-Based (CAN) | Service-Based (Ethernet SOA) |
|------|-------------------|------------------------------|
| 통신 모델 | 브로드캐스트 (항상 전송) | 요청-응답 / 구독 (필요 시 전송) |
| 구성 방식 | 정적 (DBC 파일 컴파일 타임 확정) | 동적 (런타임 Service Discovery) |
| 새 소비자 추가 | 전체 DBC 수정 필요 | SD로 자동 탐색, 재설정 불필요 |
| 대역폭 효율 | 낮음 (불필요 데이터 포함) | 높음 (변경 시만 전송) |
| 확장성 | 낮음 (버스 포화 위험) | 높음 (Ethernet 1Gbps+) |
| 미들웨어 | 없음 (Raw CAN 신호) | SOME/IP, DDS, MQTT, OPC UA |
| 오류 처리 | 없음 (신호 손실 감지 불가) | Return Code, Timeout, NRC |
| 보안 | 제한적 (SecOC 추가 필요) | TLS, mTLS, SecOC 내장 |

---

## Signal-Based 통신의 한계
```
ECU A                    CAN 버스 (2Mbps)            ECU B, C, D
  │                            │                          │
  │──── joint_angle (10ms) ────►│─────────────────────────►│ (항상 수신)
  │──── tool_force  (5ms)  ────►│─────────────────────────►│
  │──── temperature (100ms)────►│─────────────────────────►│

문제점:
  ① ECU D가 joint_angle을 필요로 하지 않아도 수신 → CPU 낭비
  ② ECU E 추가 시 DBC 파일 재컴파일, 전체 ECU 업데이트 필요
  ③ CAN 2Mbps 포화 → 메시지 지연 발생
  ④ 새로운 데이터 타입 추가 시 Signal 정의 재협의 필요
```

## Service-Based 통신 흐름
```
Provider (ECU A)        SOME/IP SD (UDP 30490)       Consumer (ECU B)
      │                          │                         │
      │─── ①OfferService ────────►│                         │
      │    Service 0x0101         │◄──── ②FindService ──────│
      │    (Multicast)            │                         │
      │◄───────────────────────────── ③SubscribeEvent ──────│
      │    Event 0x8001 구독       │                         │
      │────────────────────────────── ④EventNotify ─────────►│
      │    (변경 시 또는 주기적으로)  │                        │
      │────────────────────────────── ④EventNotify ─────────►│

장점: 동적 탐색, 필요 시에만 전송, ECU 추가/제거 시 재설정 불필요
```

---

## SOA 핵심 패턴

### 패턴 1: Request/Response (Method Call)
```
Client (HMI)                               Server (JointControl)
  │                                               │
  │──── SOME/IP Request ─────────────────────────►│
  │     [Service ID: 0x0101 | Method ID: 0x0010]  │
  │     [Request ID: 0x0001 | MsgType: 0x00]      │ ← 조인트 각도 읽기
  │     Payload: joint_id=1 (1 byte)              │
  │                                               │ (처리 중, ~500µs)
  │◄─── SOME/IP Response ──────────────────────────│
  │     [Method ID: 0x8010 | Return Code: 0x00]   │
  │     Payload: 45.3° = 0x4235999A (float32 BE)  │

활용: 명령 전송, 단일 값 조회 (동기적 특성 필요 시)
```

### 패턴 2: Publish/Subscribe (Event 통지)
```
Publisher (센서 ECU)     SOME/IP Event              Subscriber (제어 ECU)
      │                        │                          │
      │ 센서 값 갱신            │                          │
      │──── Event 0x8001 ───────────────────────────────►│
      │     [Service 0x0101 | Event 0x8001 | Notify]     │
      │     Payload: {joint_id=1, angle=1.57, ts=...}    │
      │──── Event 0x8001 ───────────────────────────────►│

활용: 고빈도 센서 데이터, 상태 변경 알림 (1kHz 제어 루프)
```

### 패턴 3: Field (Getter / Setter / Notifier)
```
Field: MaxVelocity (최대 속도 속성)

Getter (현재 값 조회):
  Client ──── Get MaxVelocity ────► Server
  Client ◄─── 1.5 m/s ─────────────

Setter (값 변경 요청):
  Client ──── Set MaxVelocity = 2.0 m/s ──► Server
  Client ◄─── OK ─────────────────────────

Notifier (변경 알림 구독):
  Client ◄─── MaxVelocityChanged = 2.0 m/s ── Server (설정 완료 후)

SOME/IP Method ID 구조:
  Getter: Method 0x0001
  Setter: Method 0x0002
  Notifier: Event 0x8001
```

---

## SOME/IP 메시지 구조 상세

```
SOME/IP 헤더 (16 Byte 고정):
 0        1        2        3
┌────────┬────────┬────────┬────────┐
│ Service ID (2B) │ Method ID (2B)  │  ← 서비스/메서드 식별
├────────┴────────┴────────┴────────┤
│            Length (4B)           │  ← 헤더 이후 데이터 길이
├────────┬────────┬────────┬────────┤
│ Client ID (2B)  │ Session ID (2B) │  ← 요청 추적 (Request ID)
├────────┬────────┬────────┬────────┤
│ProtoV  │IfaceV  │MsgType │RetCode │  ← 각 1 Byte
└────────┴────────┴────────┴────────┘
[Payload: 가변 길이]

Message Type 값:
  0x00: Request           0x01: Request No Return
  0x02: Notification      0x40: Request ACK
  0x80: Response          0x81: Error

Return Code 값:
  0x00: E_OK              0x01: E_NOT_OK
  0x02: E_UNKNOWN_SERVICE 0x03: E_UNKNOWN_METHOD
  0x05: E_NOT_REACHABLE   0x08: E_TIMEOUT
  0x15: E_WRONG_INTERFACE_VERSION
```

---

## SOME/IP Service Discovery (SD) 상세

```
SD 포트: UDP 30490 (Multicast 239.192.x.x 또는 Unicast)
SD 헤더: SOME/IP 헤더 + Flags(1) + Reserved(3) + Length(4) + Entries + Options

SD 상태 기계:
  ┌─────────┐  ECU 부팅    ┌──────────────┐
  │  Down   │────────────►│  Initial Wait│ (랜덤 지연 100~200ms)
  └─────────┘             └──────┬───────┘
                                 │ 타이머 만료
                          ┌──────▼───────┐
                          │  Repetition  │ OfferService × 3회
                          │  Phase       │ 간격 200ms, 400ms, 800ms
                          └──────┬───────┘
                                 │ 반복 완료
                          ┌──────▼───────┐
                          │  Main Phase  │ OfferService 주기적 반복
                          │              │ (기본 10초 간격)
                          └──────────────┘

SD Entry 구조 (16 Byte):
  [Type(1)] [Index1(1)] [Index2(1)] [#Opt1/#Opt2(1)]
  [Service ID(2)] [Instance ID(2)]
  [Major Ver(1)] [TTL(3)]
  [Minor Ver(4) or Counter+EventID]

Entry Type:
  0x00: FindService      0x01: OfferService
  0x06: Subscribe        0x07: SubscribeAck
```

```bash
# Wireshark 필터
someipsd                               # SD 전체 트래픽
someipsd.entry.serviceid == 0x0101    # 특정 서비스 SD
someipsd.entry.type == 0x01           # OfferService 메시지만
someip                                 # SOME/IP 전체 트래픽
someip.service_id == 0x0101           # 특정 서비스 트래픽
someip.method_id == 0x0010            # 특정 메서드
udp.port == 30490                      # SD 포트
```

---

## DDS (Data Distribution Service)

OMG 표준 기반의 데이터 중심 미들웨어로, ROS 2의 통신 백본입니다.

### DDS 아키텍처
```
DDS 도메인 (Domain ID 기반 격리, 같은 도메인만 통신)
┌──────────────────────────────────────────────────────────┐
│  DomainParticipant (프로세스 단위)                        │
│  ┌─────────────────────┐  ┌────────────────────────────┐ │
│  │ Publisher           │  │ Subscriber                 │ │
│  │  DataWriter         │  │  DataReader                │ │
│  │  Topic: /joint_states│  │  Topic: /joint_states     │ │
│  │  QoS: RELIABLE,     │  │  QoS: RELIABLE,            │ │
│  │  KEEP_LAST(10)      │  │  KEEP_LAST(10)             │ │
│  └──────────┬──────────┘  └────────────┬───────────────┘ │
│             │    RTPS 프로토콜 (UDP/TCP) │                 │
│             └─────────────────────────► │                 │
└──────────────────────────────────────────────────────────┘

자동 탐색: SPDP (Simple Participant Discovery Protocol)
→ 멀티캐스트 Heartbeat로 Participant 발견
→ SEDP (Simple Endpoint Discovery Protocol)로 DataWriter/Reader 매칭
```

### DDS QoS 주요 정책 (22개 중 핵심 6개)

| QoS 정책 | 옵션 | 설명 | 의료 로봇 활용 |
|---------|------|------|----------------|
| Reliability | BEST_EFFORT / RELIABLE | 메시지 보장 전송 | 안전 신호: RELIABLE |
| Durability | VOLATILE / TRANSIENT_LOCAL | 신규 구독자 이전 데이터 전달 | 설정값: TRANSIENT_LOCAL |
| Deadline | 주기 설정 | 최대 데이터 도착 기한 | 1ms 제어 루프 |
| History | KEEP_LAST(N) / KEEP_ALL | 히스토리 보관 | 센서 버퍼링 |
| Liveliness | AUTOMATIC / MANUAL | 노드 생존 확인 | 장애 감지 |
| Latency Budget | 지연 예산 | 허용 처리 지연 | 실시간성 조정 |

```python
# ROS 2 Python: 실시간 제어 Publisher (DDS 기반)
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy, DurabilityPolicy

class JointStatePublisher(Node):
    def __init__(self):
        super().__init__('joint_state_publisher')

        # 실시간 제어용 QoS: RELIABLE + KEEP_LAST(1)
        qos_realtime = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=1,
            durability=DurabilityPolicy.VOLATILE
        )

        self.pub = self.create_publisher(
            JointState, '/joint_states', qos_realtime)
        self.timer = self.create_timer(0.001, self.publish_cb)  # 1kHz
        self.joint_names = [f'joint_{i}' for i in range(1, 8)]

    def publish_cb(self):
        msg = JointState()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.name = self.joint_names
        msg.position = self._read_encoders()     # 하드웨어 읽기
        msg.velocity = self._read_velocities()
        msg.effort = self._read_torques()
        self.pub.publish(msg)

    def _read_encoders(self):
        return [0.0] * 7  # 실제: EtherCAT 또는 CAN FD 읽기

rclpy.init()
rclpy.spin(JointStatePublisher())
```

---

## SOA 미들웨어 비교

| 특성 | SOME/IP | DDS (RTPS) | MQTT v5.0 | OPC UA |
|------|---------|------------|-----------|--------|
| 표준 기관 | AUTOSAR | OMG | OASIS | OPC Foundation |
| 주요 도메인 | 차량 내부 | 로봇/국방/산업 | IoT/클라우드 | 공장 자동화 |
| 전송 레이어 | UDP/TCP | UDP (RTPS) | TCP/TLS | TCP/TLS/UADP |
| QoS 정책 | 제한적 (3~4개) | 풍부 (22개) | 3단계 (0/1/2) | 구독 모델 |
| 서비스 탐색 | SD (명시적 단계) | SPDP/SEDP (자동) | 브로커 중심 | OPC UA SD |
| 실시간성 | 높음 (µs) | 매우 높음 (µs) | 낮음 (ms~s) | 중간 (ms) |
| 브로커 필요 | 불필요 | 불필요 | 필요 | 불필요 |
| TSN 통합 | SOME/IP + TSN | DDS+TSN 표준 | 간접 | OPC UA + TSN |
| 보안 | SOME/IP-Sec, TLS | DDS Security | TLS, mTLS | TLS, X.509 |
| ROS 2 바인딩 | rmw_someip | rmw_fastrtps (기본) | 별도 패키지 | 별도 패키지 |

---

## 의료 로봇 SOA 서비스 설계

### 서비스 인터페이스 정의
```
수술 로봇 SOME/IP 서비스 카탈로그

Service 0x0101: JointControlService (Motion Domain)
  ├─ Method 0x0001: SetPosition(joint_id:u8, angle_rad:f32) → {error:u8}
  ├─ Method 0x0002: GetPosition(joint_id:u8) → {angle_rad:f32}
  ├─ Method 0x0003: SetVelocityLimit(joint_id:u8, vel:f32) → {error:u8}
  ├─ Event  0x8001: PositionChanged   {joint_id:u8, angle:f32, ts:u64}
  └─ Field  0x0004: MaxVelocity       Getter/Setter/Notifier

Service 0x0102: SurgicalToolService (Instrument Domain)
  ├─ Method 0x0001: ActivateTool(tool_id:u8) → {error:u8, state:u8}
  ├─ Method 0x0002: SetGripForce(force_N:f32) → {actual_N:f32}
  └─ Event  0x8001: ToolStatus        {tool_id:u8, state:u8, force_N:f32}

Service 0x0201: CameraService (Vision Domain)
  ├─ Method 0x0001: Configure(w:u16, h:u16, fps:u8) → {error:u8}
  ├─ Event  0x8001: FrameReady        {frame_id:u32, ts:u64, data:bytes}
  └─ Event  0x8002: AIAnnotation      {frame_id:u32, detections:[]}

Service 0x0301: DiagnosticsService (Safety Domain)
  ├─ Method 0x0001: ReadActiveDTC() → {dtc_list:[{code:u32, status:u8}]}
  ├─ Method 0x0002: ClearDTC(dtc:u32) → {error:u8}
  └─ Method 0x0003: ReadECUInfo() → {sw_ver:str, hw_rev:u8, ...}

Service 0x0401: SafetyService (Safety Domain - ASIL-D)
  ├─ Method 0x0001: RequestEmergencyStop() → {accepted:bool}
  ├─ Method 0x0002: GetSafetyStatus() → {asil_mode:u8, faults:u32}
  └─ Event  0x8001: SafetyAlert       {fault_id:u8, severity:u8, ts:u64}
```

### C++ SOME/IP 서비스 구현 (vsomeip)
```cpp
#include <vsomeip/vsomeip.hpp>
#include <thread>

constexpr uint16_t SERVICE_ID = 0x0101;
constexpr uint16_t METHOD_SET_POSITION = 0x0001;
constexpr uint16_t EVENT_POSITION_CHANGED = 0x8001;

class JointControlServer {
    std::shared_ptr<vsomeip::application> app_;
    std::shared_ptr<vsomeip::payload> event_payload_;

public:
    JointControlServer() {
        app_ = vsomeip::runtime::get()->create_application("JointControlSvc");
        app_->init();

        app_->register_message_handler(
            SERVICE_ID, vsomeip::ANY_INSTANCE, METHOD_SET_POSITION,
            [this](const std::shared_ptr<vsomeip::message>& req) {
                handle_set_position(req);
            });

        app_->offer_service(SERVICE_ID, 0x0001);
        app_->offer_event(SERVICE_ID, 0x0001,
            EVENT_POSITION_CHANGED, {0x8001},
            vsomeip::event_type_e::ET_FIELD);
    }

    void handle_set_position(const std::shared_ptr<vsomeip::message>& req) {
        auto& data = req->get_payload()->get_data();
        uint8_t joint_id = data[0];
        float angle = *reinterpret_cast<const float*>(&data[1]);
        /* Big Endian 변환 필요: ntohl 적용 */

        bool ok = move_joint(joint_id, angle);

        /* 응답 전송 */
        auto resp = vsomeip::runtime::get()->create_response(req);
        auto payload = vsomeip::runtime::get()->create_payload({ok ? 0x00u : 0x01u});
        resp->set_payload(payload);
        app_->send(resp);

        /* 이벤트 알림 */
        if (ok) notify_position_change(joint_id, angle);
    }

    void run() {
        std::thread t([this]{ app_->start(); });
        t.join();
    }
};
```

---

## Reference
- [AUTOSAR SOME/IP Protocol Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [AUTOSAR SOME/IP Service Discovery Spec](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPServiceDiscovery.pdf)
- [OMG DDS Specification v1.4](https://www.omg.org/spec/DDS/1.4/PDF)
- [ROS 2 DDS 미들웨어 비교](https://docs.ros.org/en/rolling/Concepts/About-Different-Middleware-Vendors.html)
- [vsomeip - COVESA 오픈소스 SOME/IP](https://github.com/COVESA/vsomeip)
- [Eclipse Cyclone DDS](https://github.com/eclipse-cyclonedds/cyclonedds)
