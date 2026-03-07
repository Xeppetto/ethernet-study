# SOME/IP (Scalable service-Oriented MiddlewarE over IP)

## 개요
SOME/IP는 AUTOSAR에서 정의한 차량용 서비스 지향 미들웨어 프로토콜입니다. Ethernet 위에서 동작하며 Service Discovery, RPC(Remote Procedure Call), 이벤트 통지(Event Notification)를 제공합니다. AUTOSAR Adaptive Platform의 핵심 통신 레이어로, 기존 CAN/LIN 기반 신호 지향(Signal-oriented) 통신에서 서비스 지향(Service-oriented) 통신으로의 패러다임 전환을 구현합니다.

의료 로봇에서는 SOME/IP가 **시스템 내부 서비스 통신** 역할을 담당합니다. 예를 들어 관절 제어 서비스, 영상 스트리밍 서비스, 진단 서비스가 모두 SOME/IP를 통해 상호 통신하며, DDS가 외부 시스템과의 통신을 담당하고 MQTT가 클라우드 레이어를 담당하는 3계층 구조로 운용됩니다.

---

## 프로토콜 비교 (미들웨어 선택 가이드)

| 특성 | SOME/IP | DDS / ROS 2 | MQTT | OPC UA |
|------|---------|-------------|------|--------|
| 표준 기관 | AUTOSAR | OMG | OASIS | OPC Foundation |
| 주요 도메인 | 자동차, 의료 | 로봇, 방산, 의료 | IoT, 클라우드 | 산업 자동화 |
| 통신 패턴 | RPC + Pub/Sub | Pub/Sub 중심 | Pub/Sub | Client/Server + Pub/Sub |
| 전송 계층 | UDP / TCP | UDP (RTPS) | TCP | TCP / UDP |
| 실시간성 | 중간 (ms급) | 높음 (< 1ms) | 낮음 (초급) | 중간 |
| 페이로드 효율 | 우수 (바이너리) | 우수 (IDL 직렬화) | 보통 (JSON) | 보통 |
| 서비스 디스커버리 | 내장 (SD) | 자동 (SPDP) | 브로커 의존 | 내장 |
| TSN 통합 | 가능 | 우수 (QoS 매핑) | 미흡 | 우수 (Pub/Sub) |
| AUTOSAR 지원 | 공식 | 비공식 | 비공식 | 비공식 |

---

## SOME/IP 메시지 구조

### 기본 헤더 (16 Byte)

```
SOME/IP 메시지 = Header (16 Byte) + Payload (가변)

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────────────────────────────┬───────────────────────────────┐
│         Service ID (2B)       │         Method ID (2B)        │
├───────────────────────────────┴───────────────────────────────┤
│                        Length (4B)                            │
│          (나머지 헤더 8B + Payload 길이, 단위: Byte)            │
├───────────────────────────────┬───────────────────────────────┤
│         Client ID (2B)        │         Session ID (2B)       │
├───────────┬───────────┬───────┴───────────────────────────────┤
│ Proto Ver │ Iface Ver │ Message Type (1B) │ Return Code (1B) │
│  (1B)     │  (1B)     │                   │                  │
└───────────┴───────────┴───────────────────┴──────────────────┘

Service ID (0x0000~0xFFFE): 서비스 식별자. 0xFFFF는 SD용 예약
Method ID  (0x0001~0x7FFF): RPC 메서드 (Request/Response 가능)
           (0x8000~0xFFFF): Event 알림 (단방향)
Length     : Request ID(4B) + Protocol/Interface/MsgType/RC(4B) + Payload
Client ID  : 클라이언트 애플리케이션 식별자
Session ID : 요청-응답 매칭용 시퀀스 번호 (0x0001~0xFFFF)
```

### Message Type 및 Return Code

```
Message Type (MsgType) 값:
  0x00: REQUEST               ← 응답 필요한 RPC 요청
  0x01: REQUEST_NO_RETURN     ← Fire-and-forget (응답 불필요)
  0x02: NOTIFICATION          ← 이벤트 알림 (단방향)
  0x80: RESPONSE              ← 요청에 대한 정상 응답
  0x81: ERROR                 ← 요청에 대한 오류 응답
  0x20: TP_REQUEST            ← SOME/IP-TP 분할 요청
  0x21: TP_REQUEST_NO_RETURN  ← SOME/IP-TP Fire-and-forget
  0x22: TP_NOTIFICATION       ← SOME/IP-TP 이벤트 알림
  0xa0: TP_RESPONSE           ← SOME/IP-TP 응답

Return Code (RC) 값:
  0x00: E_OK                         ← 정상
  0x01: E_NOT_OK                     ← 일반 오류
  0x02: E_UNKNOWN_SERVICE            ← 서비스 없음
  0x03: E_UNKNOWN_METHOD             ← 메서드 없음
  0x04: E_NOT_READY                  ← 서비스 준비 안 됨
  0x05: E_NOT_REACHABLE              ← 도달 불가
  0x06: E_TIMEOUT                    ← 타임아웃
  0x07: E_WRONG_PROTOCOL_VERSION     ← 프로토콜 버전 불일치
  0x08: E_WRONG_INTERFACE_VERSION    ← 인터페이스 버전 불일치
  0x09: E_MALFORMED_MESSAGE          ← 메시지 형식 오류
  0x0A: E_WRONG_MESSAGE_TYPE         ← 메시지 타입 오류
```

---

## SOME/IP Transport 계층

```
포트 및 프로토콜 구성:

  SOME/IP SD (Service Discovery):
    UDP 30490 (Multicast: 239.132.1.1 또는 설정값)

  SOME/IP Data 전송:
    UDP: 이벤트 알림, 짧은 Fire-and-forget (오버헤드 최소화)
    TCP: 큰 페이로드 RPC, 신뢰성 필요 요청/응답

  SOME/IP-TP (Transport Protocol):
    UDP로 65507 Byte 이상의 대용량 페이로드 분할 전송
    예: AI 추론 모델 부분 업데이트, 고해상도 이미지 전송

SOME/IP-TP 분할 헤더 (TP 확장 4 Byte):
  ┌─────────────────────────────────────────────────────────┐
  │  Offset (28bit, 단위 16 Byte) │ Reserved (3bit) │ More │
  │  현재 세그먼트 시작 오프셋     │                  │Segm. │
  └─────────────────────────────────────────────────────────┘
  More Segment Flag = 1: 뒤에 세그먼트 더 있음
  More Segment Flag = 0: 마지막 세그먼트
```

---

## Service Discovery (SD)

### SD 프로토콜 동작 흐름

```
State Machine (Provider 측):
  ┌─────────────────────────────────────────────────────────────┐
  │           Down                                              │
  │  (서비스 미제공, SD 미전송)                                   │
  │           │ 서비스 준비 완료                                  │
  │           ▼                                                 │
  │        Initial Wait Phase                                   │
  │  (SD 전송 전 랜덤 대기, 충돌 방지)                            │
  │  Initial Delay: random(0, INITIAL_DELAY_MAX)                │
  │           │ 대기 완료                                         │
  │           ▼                                                 │
  │        Repetition Phase                                     │
  │  (빠른 반복 Offer 전송, 2배씩 증가)                           │
  │  Repetition Max: 3~5회, Base Delay: 100ms                   │
  │           │ 반복 완료                                         │
  │           ▼                                                 │
  │         Main Phase                                          │
  │  (주기적 Offer 전송, CycleTime마다)                          │
  │  Cycle Offer Delay: 1000ms (기본값)                         │
  └─────────────────────────────────────────────────────────────┘

SD 메시지 교환 순서:

Provider                            Consumer
    │                                   │
    │─── [OfferService] UDP Multicast ──►│  (Main Phase, 1초 주기)
    │                                   │
    │◄── [FindService]  UDP Multicast ───│  (Consumer 시작 시)
    │                                   │
    │─── [OfferService] UDP Unicast ────►│  (Find에 대한 응답)
    │                                   │
    │◄── [SubscribeEventgroup] Unicast ──│  (이벤트 구독 요청)
    │    EventGroup ID, TTL, Endpoint   │
    │                                   │
    │─── [SubscribeEventgroupAck] ──────►│  (구독 수락)
    │                                   │
    │═══ [Event Notification] Multicast ►│  (이벤트 발생 시)
```

### SD Entry 구조 (Entry Type별)

```
SD Entry (12 Byte per Entry):
┌──────┬──────┬──────────────┬──────────────┬───────┬───────────┐
│ Type │ Idx1 │ Idx2 │#Opt1│#Opt2│ Service ID │ Inst. │ Major Ver │
│ (1B) │ (1B) │ (1B) │ (4b)│(4b) │    (2B)    │  (2B) │   (1B)    │
├──────┴──────┴──────┴─────┴─────┴────────────┴───────┴───────────┤
│                    TTL (3B)         │     Minor Version (4B)     │
└────────────────────────────────────┴────────────────────────────┘

Entry Type:
  0x00: FindService          (Consumer → Provider)
  0x01: OfferService         (Provider → Consumer)
  0x06: SubscribeEventgroup  (Consumer → Provider)
  0x07: SubscribeEventgroupAck (Provider → Consumer)

TTL = 0: StopOffer 또는 StopSubscribe
TTL = 0xFFFFFF: 무제한 (서비스가 존재하는 동안)

Option Types (SD Options):
  0x01: Configuration Option (키-값 설정)
  0x04: IPv4 Endpoint Option (IP + Port + Protocol)
  0x06: IPv4 Multicast Option (Multicast 주소)
  0x14: IPv6 Endpoint Option
```

---

## SOME/IP 직렬화 (Serialization)

### 기본 타입 직렬화

```
데이터 타입별 직렬화 규칙 (AUTOSAR SOME/IP TR-00029):

  uint8:    1 Byte, Big Endian
  uint16:   2 Byte, Big Endian
  uint32:   4 Byte, Big Endian
  uint64:   8 Byte, Big Endian
  sint8:    1 Byte (2의 보수)
  float32:  4 Byte, IEEE 754
  float64:  8 Byte, IEEE 754
  bool:     1 Byte (0x00=false, 0x01=true)
  string:   [Length(4B, Big Endian)] + UTF-8 데이터 + NULL(0x00)
  struct:   멤버 순서대로 연속 직렬화 (패딩 없음)
  array:    [Length(4B)] + 요소들 연속 직렬화
  optional: [Tag(2B)][Length(4B)][Value] (TLV 방식)

직렬화 효율 비교 (JointState 예시):
  JointState { id: uint8, angle: float32, velocity: float32 }

  SOME/IP 바이너리: 01 | 3F C9 0F DB | 3F 00 00 00
    → 9 Byte
  JSON 문자열: {"id":1,"angle":1.57,"velocity":0.5}
    → 38 Byte
  SOME/IP가 JSON 대비 76% 경량화
```

### TLV (Type-Length-Value) 직렬화 (v1.4+)

```
하위 호환성 유지하며 선택적 필드를 추가하는 방식:

TLV Wire Format:
  ┌──────────────────────────────────────────────────┐
  │  Tag (2B)  │  Length (4B, 선택)  │  Value (가변) │
  └──────────────────────────────────────────────────┘

Tag 구조 (16 bit):
  [Wire Type (3b)] [Data ID (13b)]
  Wire Type:
    0: boolean / uint8 / sint8
    1: uint16 / sint16
    2: uint32 / sint32 / float32
    3: uint64 / sint64 / float64
    4: 가변 길이 (string, array 등 - Length 필드 포함)
    5: struct (복합 타입 - Length 필드 포함)

사용 목적:
  → 수신 측이 모르는 Tag는 Length 만큼 건너뜀 (버전 호환성)
  → 새 필드 추가 시 기존 클라이언트 영향 없음
```

---

## 실습 구현 (vsomeip)

### 서버 측 C++ 예시 (관절 제어 서비스)

```cpp
// joint_service.cpp - vsomeip 기반 관절 제어 서비스
#include <vsomeip/vsomeip.hpp>
#include <thread>
#include <condition_variable>

#define JOINT_SERVICE_ID    0x0101
#define JOINT_INSTANCE_ID   0x0001
#define GET_POSITION_METHOD 0x0010
#define SET_POSITION_METHOD 0x0011
#define JOINT_EVENT_ID      0x8001
#define JOINT_EVENTGROUP_ID 0x0001

class JointControlService {
public:
    JointControlService() {
        app_ = vsomeip::runtime::get()->create_application("JointService");
    }

    bool init() {
        if (!app_->init()) return false;

        // 요청 핸들러 등록
        app_->register_message_handler(
            JOINT_SERVICE_ID, JOINT_INSTANCE_ID, GET_POSITION_METHOD,
            std::bind(&JointControlService::on_get_position, this,
                      std::placeholders::_1));

        app_->register_message_handler(
            JOINT_SERVICE_ID, JOINT_INSTANCE_ID, SET_POSITION_METHOD,
            std::bind(&JointControlService::on_set_position, this,
                      std::placeholders::_1));

        // 이벤트 그룹 등록
        std::set<vsomeip::eventgroup_t> groups;
        groups.insert(JOINT_EVENTGROUP_ID);
        app_->offer_event(JOINT_SERVICE_ID, JOINT_INSTANCE_ID,
                          JOINT_EVENT_ID, groups,
                          vsomeip::event_type_e::ET_FIELD);

        app_->offer_service(JOINT_SERVICE_ID, JOINT_INSTANCE_ID);
        return true;
    }

    void on_get_position(const std::shared_ptr<vsomeip::message>& request) {
        auto response = vsomeip::runtime::get()->create_response(request);
        auto payload = vsomeip::runtime::get()->create_payload();

        // float32 관절 각도 직렬화 (Big Endian)
        float angle = joint_angle_;
        uint8_t buf[4];
        memcpy(buf, &angle, 4);
        std::reverse(buf, buf + 4);  // Little → Big Endian
        payload->set_data(buf, 4);
        response->set_payload(payload);
        app_->send(response);
    }

    void on_set_position(const std::shared_ptr<vsomeip::message>& request) {
        auto& data = request->get_payload()->get_data();
        if (data.size() >= 4) {
            uint8_t buf[4] = {data[0], data[1], data[2], data[3]};
            std::reverse(buf, buf + 4);
            memcpy(&joint_angle_, buf, 4);
        }
        // 이벤트 알림 발행
        notify_joint_change();
        auto response = vsomeip::runtime::get()->create_response(request);
        app_->send(response);
    }

    void notify_joint_change() {
        auto payload = vsomeip::runtime::get()->create_payload();
        uint8_t buf[4];
        memcpy(buf, &joint_angle_, 4);
        std::reverse(buf, buf + 4);
        payload->set_data(buf, 4);
        app_->notify(JOINT_SERVICE_ID, JOINT_INSTANCE_ID,
                     JOINT_EVENT_ID, payload);
    }

    void start() { app_->start(); }

private:
    std::shared_ptr<vsomeip::application> app_;
    float joint_angle_ = 0.0f;
};
```

### vsomeip JSON 설정 파일

```json
{
    "unicast": "192.168.1.10",
    "netmask": "255.255.255.0",
    "logging": {
        "level": "info",
        "console": "true"
    },
    "applications": [
        {
            "name": "JointService",
            "id": "0x1234"
        }
    ],
    "services": [
        {
            "service": "0x0101",
            "instance": "0x0001",
            "unreliable": "30501",
            "reliable": {
                "port": "30502",
                "enable-magic-cookies": "false"
            }
        }
    ],
    "events": [
        {
            "service": "0x0101",
            "instance": "0x0001",
            "event": "0x8001",
            "is_field": "true",
            "update-cycle": "0"
        }
    ],
    "eventgroups": [
        {
            "service": "0x0101",
            "instance": "0x0001",
            "eventgroup": "0x0001",
            "events": ["0x8001"],
            "multicast": {
                "address": "239.132.1.1",
                "port": "30503"
            }
        }
    ],
    "service-discovery": {
        "enable": "true",
        "multicast": "239.132.1.1",
        "port": "30490",
        "protocol": "udp",
        "initial-delay-min": "10",
        "initial-delay-max": "100",
        "repetitions-base-delay": "200",
        "repetitions-max": "3",
        "ttl": "3",
        "cyclic-offer-delay": "1000",
        "request-response-delay": "150"
    }
}
```

### Wireshark 분석

```bash
# SOME/IP 전체 트래픽
someip

# 특정 Service ID 필터
someip.serviceid == 0x0101

# 특정 Method 필터 (GET_POSITION: 0x0010)
someip.methodid == 0x0010

# SOME/IP SD만 보기
someipsd

# SD OfferService 메시지
someipsd.entry.type == 0x01

# SD SubscribeEventgroup
someipsd.entry.type == 0x06

# Request / Response 쌍
someip.msgtype == 0x00  # REQUEST
someip.msgtype == 0x80  # RESPONSE

# 오류 응답만
someip.msgtype == 0x81

# 특정 세션 추적
someip.sessionid == 42

# SOME/IP-TP 세그먼트
someip.msgtype == 0x20
```

---

## SOME/IP 보안 (SOME/IP-Sec)

```
SOME/IP over TLS (TCP 기반):
  TLS 1.3 + 클라이언트 인증서 (mTLS)
  → RPC Request/Response 암호화
  → 서비스 소비자 인증

SOME/IP over DTLS (UDP 기반):
  DTLS 1.3 (RFC 9147)
  → Event Notification 암호화
  → SD 메시지 보호

IAM (Identity & Access Management) 통합:
  서비스별 접근 제어 정책:
    0x0101 GET_POSITION: 모든 클라이언트 허용
    0x0101 SET_POSITION: 인증된 제어 클라이언트만
    0x0300 DiagnosticsService: 진단 도구만
```

---

## 의료 로봇 SOME/IP 서비스 설계

### 서비스 ID 할당 체계

```
서비스 ID 할당 (예시, 0x0100~0x03FF 범위):

  0x0100: RobotBaseService     (이동 플랫폼)
  0x0101: JointControlService  (관절 제어, 6-DOF)
  0x0102: SurgicalToolService  (수술 도구 제어)
  0x0200: VisionService        (카메라/영상 처리)
  0x0201: LightingService      (수술 조명 제어)
  0x0300: DiagnosticsService   (UDS 진단 브릿지)
  0x0301: SystemHealthService  (헬스 모니터링)
  0x0302: SafetyMonitorService (안전 감시)
```

### Method / Event ID 할당 관례

```
Method ID (0x0001~0x7FFF: Request/Response):
  0x0001: Initialize     (초기화, 응답 필요)
  0x0002: Start          (동작 시작)
  0x0003: Stop           (동작 정지)
  0x0004: Reset          (리셋)
  0x0010: GetStatus      (상태 조회)
  0x0011: GetParameter   (파라미터 읽기)
  0x0012: SetParameter   (파라미터 쓰기)
  0x0020: ExecuteAction  (동작 명령 실행)
  0x0021: AbortAction    (동작 중단)

Event ID (0x8000~0xFFFF: 단방향 Notification):
  0x8001: StatusChanged      (상태 변경 알림)
  0x8002: ErrorOccurred      (오류 발생)
  0x8003: DataAvailable      (새 데이터 준비)
  0x8004: WarningOccurred    (경고 발생)
  0x8005: CalibrationDone    (캘리브레이션 완료)

Field 설계 (Getter + Setter + Notifier 3종 세트):
  Getter:  GetJointAngle  → Method ID: 0x0010
  Setter:  SetJointAngle  → Method ID: 0x0011
  Notifier: JointAngleChanged → Event ID: 0x8001
```

### 시스템 통합 계층 다이어그램

```
의료 로봇 통신 계층:

  클라우드 모니터링 레이어
  ┌─────────────────────────────────────────────────┐
  │  MQTT over TLS (1Hz, 비실시간 상태 보고)          │
  │  └── Edge Gateway → MQTT Broker → Dashboard     │
  └─────────────────────────────────────────────────┘
         ↕ JSON 변환
  시스템 통합 레이어
  ┌─────────────────────────────────────────────────┐
  │  SOME/IP over Ethernet (내부 서비스 통신)         │
  │  ├── JointControlService 0x0101 (TCP, 1kHz)     │
  │  ├── VisionService 0x0200 (UDP-TP, 30fps)        │
  │  └── DiagnosticsService 0x0300 (TCP, 온디맨드)  │
  └─────────────────────────────────────────────────┘
         ↕ TSN 이더넷 스위칭
  실시간 제어 레이어
  ┌─────────────────────────────────────────────────┐
  │  DDS / ROS 2 (< 1ms, 결정론적 제어)              │
  │  ├── /joint_states (1kHz, QoS: RELIABLE)        │
  │  └── /cmd_vel (100Hz, QoS: BEST_EFFORT)         │
  └─────────────────────────────────────────────────┘
```

---

## Reference
- [AUTOSAR SOME/IP Protocol Specification R22-11](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [AUTOSAR SOME/IP SD Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPServiceDiscovery.pdf)
- [AUTOSAR SOME/IP-TP Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/20-11/AUTOSAR_SWS_SOMEIPTransformer.pdf)
- [vsomeip - COVESA Open Source SOME/IP](https://github.com/COVESA/vsomeip)
- [Vector SOME/IP Introduction](https://www.vector.com/kr/ko/know-how/protocols/some-ip/)
- [GENIVI/COVESA SOME/IP Test Suite](https://github.com/COVESA/vsomeip/tree/master/test)
