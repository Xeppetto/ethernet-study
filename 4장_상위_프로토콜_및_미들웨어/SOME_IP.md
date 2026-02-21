# SOME/IP (Scalable service-Oriented MiddlewarE over IP)

## 개요
SOME/IP는 AUTOSAR에서 정의한 차량용 서비스 지향 미들웨어 프로토콜입니다. Ethernet 위에서 동작하며, Service Discovery, RPC(Remote Procedure Call), 이벤트 통지를 제공합니다. AUTOSAR Adaptive Platform의 핵심 통신 레이어입니다.

---

## 메시지 구조 (Header)

```
SOME/IP 메시지 = SOME/IP Header (16 Byte) + Payload

┌──────────────────────────────────────────────────────────┐
│  Service ID (2B)  │  Method ID (2B)                      │
├──────────────────────────────────────────────────────────┤
│  Length (4B) - Payload 길이 포함한 나머지 헤더 + 페이로드│
├──────────────────────────────────────────────────────────┤
│  Client ID (2B)   │  Session ID (2B)                     │
├──────────────────────────────────────────────────────────┤
│  Protocol Version(1B)│Iface Version(1B)│MsgType(1B)│RC(1B)│
└──────────────────────────────────────────────────────────┘

Message Type (MsgType):
  0x00: Request (응답 필요)
  0x01: Request No Return (응답 불필요)
  0x02: Notification (Event 알림)
  0x80: Response (요청에 대한 응답)
  0x81: Error (오류 응답)

Return Code (RC):
  0x00: E_OK
  0x01: E_NOT_OK
  0x02: E_UNKNOWN_SERVICE
  0x03: E_UNKNOWN_METHOD
  0x05: E_NOT_REACHABLE
  0x08: E_WRONG_PROTOCOL_VERSION
```

---

## SOME/IP Transport

```
포트 설정 (기본값):
  SOME/IP SD: UDP 30490 (Multicast: 239.132.1.1 또는 설정에 따름)
  SOME/IP Data: 임의 UDP/TCP 포트 (SD에서 협상)

UDP 사용: 이벤트 알림, 짧은 요청/응답 (오버헤드 최소화)
TCP 사용: 큰 페이로드 전송, 신뢰성이 중요한 RPC

SOME/IP-TP (Transport Protocol): UDP로 큰 페이로드 분할 전송
  최대 UDP 페이로드 = 65507 Byte 제한 우회
  예: AI 모델 업데이트, 영상 프레임 전송 시 활용
```

---

## Service Discovery (SD)

### SD 메시지 흐름
```
Offer Phase:
  Provider → [SD Offer] → 네트워크 (Multicast 또는 Unicast)
  "나는 Service 0x0101을 제공합니다. IP:Port로 연결하세요"

Find Phase:
  Consumer → [SD Find] → 네트워크 (Multicast)
  "Service 0x0101 어디 있나요?"

Subscribe Phase:
  Consumer → [SD Subscribe] → Provider (Unicast)
  "Service 0x0101의 Event Group 0x0001을 구독하겠습니다"

  Provider → [SD SubscribeAck] → Consumer (Unicast)
  "구독 수락. 이후 이벤트는 UDP로 보내드립니다"
```

### SD Entry 구조
```
SD Entry Type:
  0x00: FindService
  0x01: OfferService
  0x06: SubscribeEventgroup
  0x07: SubscribeEventgroupAck

주요 필드:
  Service ID: 어떤 서비스인지
  Instance ID: 서비스 인스턴스 (여러 인스턴스 지원)
  Major Version / Minor Version
  TTL: 유효 시간 (0 = 스톱 오퍼/구독)
  Endpoint Option: IP 주소 + 포트 정보
```

---

## SOME/IP 직렬화 (Serialization)

```
데이터 타입별 직렬화:
  uint8:  1 Byte (Big Endian)
  uint16: 2 Byte (Big Endian)
  uint32: 4 Byte (Big Endian)
  float:  4 Byte (IEEE 754)
  string: 4B(Length) + UTF-8 데이터
  struct: 멤버 순서대로 직렬화 (패딩 없음)
  array:  4B(Length) + 요소들 직렬화

예: JointState {uint8 id=1, float32 angle=1.57, float32 velocity=0.5}
  직렬화: 01 | 3F C9 0F DB | 3F 00 00 00
  (총 9 Byte, JSON "{"id":1,"angle":1.57,"velocity":0.5}" = 38 Byte보다 76% 경량)
```

### TLV (Type-Length-Value) 직렬화 (SOME/IP v1.4+)
```
하위 호환성을 유지하면서 선택적 필드를 추가하는 방식:
  [Tag(2B)][Length(4B)][Value]
  → 수신 측이 모르는 Tag는 무시 (버전 호환성)
```

---

## Wireshark로 SOME/IP 분석

```bash
# SOME/IP 트래픽 필터
someip

# 특정 서비스 ID
someip.serviceid == 0x0101

# SOME/IP SD만 보기
someipsd

# Service Discovery Offer 메시지
someipsd.entry.type == 0x01

# 특정 Method의 Request/Response
someip.methodid == 0x0010 and someip.msgtype == 0x00  # Request
someip.methodid == 0x8010 and someip.msgtype == 0x80  # Response
```

### SOME/IP 플러그인 설치 (Wireshark)
```
Wireshark → Help → About → Plugins → 설치 경로 확인
SOME/IP dissector: 기본 내장 (Wireshark 3.x 이상)

설정: Edit → Preferences → Protocols → SOMEIP
  → Service 포트 설정
  → SD 멀티캐스트 주소 설정
```

---

## 실용적인 SOME/IP 테스트

### vsomeip (오픈소스 구현체)
```bash
# vsomeip 빌드 및 설치
apt-get install libboost-all-dev
git clone https://github.com/COVESA/vsomeip
cd vsomeip && mkdir build && cd build
cmake .. -DENABLE_SIGNAL_HANDLING=1
make -j4 && sudo make install

# 설정 파일 예시 (service_example.json)
cat > /etc/vsomeip/service.json << 'EOF'
{
    "unicast": "192.168.1.1",
    "logging": {"level": "info"},
    "applications": [{"name": "joint_service", "id": "0x1234"}],
    "services": [
        {
            "service": "0x0101",
            "instance": "0x0001",
            "unreliable": "30501"
        }
    ],
    "service-discovery": {
        "enable": "true",
        "multicast": "239.132.1.1",
        "port": "30490",
        "protocol": "udp"
    }
}
EOF
```

---

## 의료 로봇 SOME/IP 서비스 설계

```
서비스 ID 할당 (예시):
  0x0100: RobotBaseService     (이동 플랫폼)
  0x0101: JointControlService  (관절 제어)
  0x0102: SurgicalToolService  (수술 도구)
  0x0200: VisionService        (영상)
  0x0201: LightingService      (조명)
  0x0300: DiagnosticsService   (진단)
  0x0301: SystemHealthService  (헬스 모니터링)

Method ID 할당 (0x0001~0x7FFF: Request/Response):
  0x0001: Initialize
  0x0002: Start
  0x0003: Stop
  0x0010: GetStatus
  0x0011: SetParameter
  0x0020: ExecuteAction

Event ID (0x8000~0xFFFF: Notification):
  0x8001: StatusChanged
  0x8002: ErrorOccurred
  0x8003: DataAvailable

Field 구분:
  Getter Method ID: 0x0010 (예: GetJointAngle)
  Setter Method ID: 0x0011 (예: SetJointAngle)
  Notifier Event ID: 0x8001 (예: JointAngleChanged)
```

---

## Reference
- [AUTOSAR SOME/IP Protocol Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPProtocol.pdf)
- [AUTOSAR SOME/IP SD Specification](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_SOMEIPServiceDiscovery.pdf)
- [vsomeip - COVESA Open Source SOME/IP](https://github.com/COVESA/vsomeip)
- [Vector SOME/IP Introduction](https://www.vector.com/kr/ko/know-how/protocols/some-ip/)
