# OPC UA over TSN

## 개요
OPC UA(OPC Unified Architecture)는 OPC Foundation이 개발한 산업 자동화용 통신 표준으로, 강력한 데이터 모델링과 보안 기능이 특징입니다. TSN(Time Sensitive Networking)과 결합하여 **OPC UA over TSN**은 실시간 제어와 정보 관리를 단일 네트워크에서 통합하는 차세대 산업 통신 표준입니다.

---

## OPC UA 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│  OPC UA Information Model                                    │
│  (객체, 변수, 메서드, 뷰를 계층적으로 모델링)                  │
├─────────────────────────────────────────────────────────────┤
│  OPC UA Services                                            │
│  • Attribute Service:  Read/Write                           │
│  • MonitoredItem:      변경 감시 (Subscription)             │
│  • Method Service:     메서드 호출                          │
│  • Discovery Service:  서버 탐색                            │
├─────────────────────────────────────────────────────────────┤
│  OPC UA Session Layer                                       │
│  (세션 관리, 보안, 인코딩)                                   │
├─────────────────────────────────────────────────────────────┤
│  Transport Layer                                            │
│  ┌─────────────────┬──────────────────────────────────────┐ │
│  │  Client/Server  │  Pub/Sub (UDP Multicast / TSN VLAN)  │ │
│  │  (TCP 4840)     │  (UDP 4840 / Ethernet Level)         │ │
│  └─────────────────┴──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Client/Server vs Pub/Sub

### Client/Server (기존)
```
OPC UA Client (SCADA, HMI)  ←─── TCP 4840 ───► OPC UA Server (PLC, Robot)

특징:
  - 폴링 기반 또는 모니터링 (Subscription)
  - 단방향 연결 (Client가 연결 시작)
  - 지연 허용 (수ms ~ 수백ms)
  - 인터넷 경유 가능 (방화벽 친화적)
```

### Pub/Sub (TSN용)
```
Publisher (로봇 ECU)          네트워크          Subscriber (제어 PC)
      │                          │                    │
      │─── UDP Multicast ────────►│───────────────────►│
      │    또는 TSN VLAN L2      │                    │
      │    (브로커 불필요)        │                    │

특징:
  - UDP 멀티캐스트 (브로커 없음)
  - TSN과 통합 시 결정론적 지연
  - 실시간 제어 루프에 적합
  - UADP (UA Binary Datagram Protocol) 인코딩
```

---

## OPC UA 정보 모델 예시

```
로봇 OPC UA 정보 모델:

Objects
└─ Robot (ObjectType: RobotType)
   ├─ Joints (FolderType)
   │  ├─ Joint1 (JointType)
   │  │  ├─ Position (Variable, Double, rad)
   │  │  ├─ Velocity (Variable, Double, rad/s)
   │  │  ├─ Temperature (Variable, Float, °C)
   │  │  └─ SetPosition (Method, input: Double → output: Boolean)
   │  ├─ Joint2 ...
   │  └─ Joint6 ...
   ├─ Status (ObjectType)
   │  ├─ IsReady (Variable, Boolean)
   │  ├─ ErrorCode (Variable, UInt32)
   │  └─ Mode (Variable, Enum: {IDLE, MOVING, ERROR})
   └─ Diagnostics
      ├─ OperatingHours (Variable, UInt64, hours)
      └─ DTCList (Variable, Array<UInt32>)
```

---

## Python OPC UA 예시 (asyncua)

### OPC UA 서버 (로봇 측)
```python
import asyncio
from asyncua import Server, ua

async def main():
    server = Server()
    await server.init()
    server.set_endpoint("opc.tcp://0.0.0.0:4840/robot/")
    server.set_server_name("Medical Robot OPC UA Server")

    # 보안 설정
    await server.set_security_IDs(["Anonymous", "Username"])

    # 네임스페이스 등록
    ns = await server.register_namespace("http://hospital.com/robot")

    # 정보 모델 구성
    robot = await server.nodes.objects.add_object(ns, "Robot")
    joint1 = await robot.add_object(ns, "Joint1")
    position = await joint1.add_variable(ns, "Position", 0.0)
    await position.set_writable()

    temperature = await joint1.add_variable(ns, "Temperature", 25.0)

    # 메서드 추가
    async def set_position_method(parent, *args):
        target_pos = args[0].Value
        await position.write_value(target_pos)
        return [ua.Variant(True, ua.VariantType.Boolean)]

    await joint1.add_method(
        ns, "SetPosition",
        set_position_method,
        [ua.VariantType.Double],    # 입력 타입
        [ua.VariantType.Boolean]    # 출력 타입
    )

    async with server:
        # 실시간 데이터 업데이트
        while True:
            # 하드웨어에서 실제 값 읽어오기
            current_pos = read_encoder()
            await position.write_value(current_pos)
            await asyncio.sleep(0.001)  # 1kHz

asyncio.run(main())
```

### OPC UA 클라이언트 (모니터링 PC)
```python
import asyncio
from asyncua import Client
from asyncua.common.subscription import SubHandler

class RobotDataHandler(SubHandler):
    def datachange_notification(self, node, val, data):
        print(f"Joint1/Position 변경: {val:.4f} rad")

async def main():
    async with Client("opc.tcp://192.168.1.100:4840/") as client:
        # 네임스페이스 인덱스 확인
        ns = await client.get_namespace_index("http://hospital.com/robot")

        # 노드 탐색
        robot = await client.nodes.objects.get_child([f"{ns}:Robot"])
        joint1 = await robot.get_child([f"{ns}:Joint1"])
        position = await joint1.get_child([f"{ns}:Position"])

        # 현재 값 읽기
        current_pos = await position.read_value()
        print(f"현재 위치: {current_pos:.4f} rad")

        # Subscription (변경 감시)
        handler = RobotDataHandler()
        subscription = await client.create_subscription(
            period=100,       # 100ms 폴링 주기
            handler=handler
        )
        await subscription.subscribe_data_change([position])

        # 메서드 호출 (관절 이동)
        await joint1.call_method(f"{ns}:SetPosition", 1.57)
        print("관절 1을 1.57 rad로 이동 명령")

        await asyncio.sleep(10)

asyncio.run(main())
```

---

## OPC UA over TSN 구성

```
TSN 네트워크에서 OPC UA Pub/Sub:

Publisher (로봇 ECU)
  │
  │ OPC UA UADP 메시지 (UDP Multicast)
  │ + TSN 우선순위 (PCP=4, CBS Class A)
  │
TSN 스위치 (대역폭 보장)
  │
  ├─► Subscriber A (HMI 대시보드)
  └─► Subscriber B (SCADA 시스템)

UADP 메시지 구조:
  [UADPHeader] [GroupHeader] [DataSetMessage]
  DataSetMessage = 로봇 상태 (JSON 또는 Binary)

지연 목표: < 10ms (Pub/Sub + TSN CBS)
```

---

## OPC UA 보안

```
보안 채널 (SecurityMode):
  None:          보안 없음 (개발/테스트용)
  Sign:          메시지 서명만 (무결성)
  SignAndEncrypt: 서명 + 암호화 (기밀성)

보안 정책:
  Basic256Sha256:   AES-256-CBC + SHA-256 (권장)
  Aes128_Sha256_RsaOaep: AES-128-GCM + RSA-OAEP

인증 방식:
  Anonymous:         인증 없음
  Username/Password: 사용자 ID/PW
  Certificate:       X.509 인증서 (의료/산업 권장)

인가 (Authorization):
  Role-Based Access Control (RBAC):
    Operator: 읽기 + 기본 명령
    Engineer: 읽기 + 쓰기 + 진단
    Admin:    전체 권한 + 설정 변경
```

---

## 의료 기기 상호운용성 (IEEE 11073 SDC)

```
수술실 의료기기 통합:

OPC UA + IEEE 11073 SDC (Service-oriented Device Connectivity):
  수술 로봇 ─── OPC UA ──► 의료기기 게이트웨이
  마취기    ─── SDC ───►  (상호 변환)
  환자 모니터 ─ SDC ───►
  영상 장비 ─── DICOM ──►
  (모두 동일한 정보 모델 공유)

→ 수술 중 ECG 패턴 이상 감지 시 로봇 자동 일시 정지
→ 마취 심도 데이터와 로봇 움직임 동기 기록
```

---

## Reference
- [OPC Foundation - OPC UA Specification](https://opcfoundation.org/developer-tools/specifications-unified-architecture)
- [OPC UA over TSN](https://opcfoundation.org/developer-tools/specifications-opc-ua-for-specific-domains)
- [asyncua Python Library](https://github.com/FreeOpcUa/opcua-asyncio)
- [IEEE 11073 SDC](https://standards.ieee.org/ieee/11073-20701/6979/)
- [open62541 C Library](https://www.open62541.org/)
