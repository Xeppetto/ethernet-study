# OPC UA over TSN (OPC Unified Architecture over Time-Sensitive Networking)

## 개요
OPC UA(OPC Unified Architecture)는 OPC Foundation이 개발한 플랫폼 독립적 산업 통신 표준입니다(IEC 62541). 강력한 데이터 모델링(정보 모델), 내장 보안(인증·인가·암호화), 서비스 지향 아키텍처가 특징입니다. TSN(Time Sensitive Networking)과 결합한 **OPC UA over TSN**은 공장 자동화, 의료 기기 네트워크, 로봇 시스템에서 실시간 제어와 정보 관리를 단일 네트워크로 통합하는 차세대 산업 통신 표준입니다.

의료 분야에서는 **IEEE 11073 SDC(Service-oriented Device Connectivity)**와 OPC UA가 결합하여 수술실 내 의료기기 상호운용성 표준으로 부상하고 있습니다. 수술 로봇, 환자 모니터, 마취기, 영상 장비가 동일한 OPC UA 정보 모델을 공유하면 통합 수술실(IOR: Integrated Operating Room)이 실현됩니다.

---

## OPC UA vs 경쟁 프로토콜 비교

| 특성 | OPC UA | SOME/IP | DDS | MQTT |
|------|--------|---------|-----|------|
| 표준 기관 | OPC Foundation (IEC 62541) | AUTOSAR | OMG | OASIS |
| 정보 모델링 | 우수 (계층적 객체 모델) | 없음 | IDL | 없음 |
| 보안 (내장) | 우수 (채널+세션+RBAC) | 외부 의존 | 외부 의존 | 기본 |
| TSN 통합 | 우수 (Pub/Sub) | 가능 | 우수 | 미흡 |
| 실시간성 | 중간 (C/S) → 높음 (Pub/Sub+TSN) | 높음 | 높음 | 낮음 |
| 산업 지원 | IIoT, 로봇, 의료 | 차량, 의료 | 로봇, 방산 | IoT, 클라우드 |
| IEC 62443 | 공식 지원 | 비공식 | 비공식 | 비공식 |

---

## OPC UA 아키텍처 계층

```
┌────────────────────────────────────────────────────────────────┐
│  Application Layer                                             │
│  OPC UA Information Model (AddressSpace)                       │
│  Objects / Variables / Methods / Views / EventTypes            │
├────────────────────────────────────────────────────────────────┤
│  Service Layer                                                 │
│  ┌──────────┬──────────┬────────────┬───────────────────────┐  │
│  │ Attribute│ Monitored│   Method   │     Discovery         │  │
│  │ Read/Write│  Items   │   Call     │     Service           │  │
│  └──────────┴──────────┴────────────┴───────────────────────┘  │
├────────────────────────────────────────────────────────────────┤
│  Session / Security Layer                                      │
│  SecureChannel (암호화 채널) + Session (논리 세션)               │
│  SecurityMode: None / Sign / SignAndEncrypt                    │
├────────────────────────────────────────────────────────────────┤
│  Transport Layer                                               │
│  ┌──────────────────────┬─────────────────────────────────────┐ │
│  │    Client/Server     │         Pub/Sub                     │ │
│  │   OPC UA Binary      │   UADP (UDP Multicast / L2 ETH)    │ │
│  │   TCP 4840           │   AMQP / MQTT over OPC UA          │ │
│  └──────────────────────┴─────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

## Client/Server vs Pub/Sub 모드

### Client/Server (요청-응답, 폴링/구독)

```
OPC UA Client (SCADA, HMI, Digital Twin)
         │
         │ OPC UA Binary over TCP 4840
         │ SecureChannel → Session → Service
         │
OPC UA Server (로봇, PLC, 의료기기)
  │
  └── AddressSpace (정보 모델)
      └── Subscription (MonitoredItem 변경 감시)
          PublishingInterval: 100ms~수초

특징:
  - 방화벽 친화적 (TCP Out-bound 연결)
  - 인터넷 경유 원격 접근 가능
  - Subscription으로 변경 감지 가능 (폴링 아님)
  - 지연 허용 (수ms ~ 수백ms)
  - 적합: SCADA, 원격 모니터링, 데이터 히스토리
```

### Pub/Sub + TSN (브로커리스 실시간)

```
Publisher (로봇 ECU)          TSN 스위치         Subscriber (제어/감시 PC)
      │                          │                      │
      │─── UADP UDP Multicast ──►│──────────────────────►│
      │    PCP=4, VLAN 20        │  대역폭 보장           │
      │    10ms 주기             │  결정론적 전달          │
      │                         │                      │

UADP (UA Datagram Protocol) 메시지 구조:
┌──────────────────────────────────────────────────────────┐
│  UADP Network Message Header                             │
│  ┌────────┬────────────┬───────────────┬───────────────┐ │
│  │ Flags1 │ Version(1B)│ PublisherID   │ DataSetClassID│ │
│  │ (1B)   │            │ (가변)        │ (선택)         │ │
│  └────────┴────────────┴───────────────┴───────────────┘ │
├──────────────────────────────────────────────────────────┤
│  Group Header                                            │
│  WriterGroupID (2B) + GroupVersion (4B) + NetworkMsg#  │
├──────────────────────────────────────────────────────────┤
│  DataSetMessage(s)                                       │
│  DataSetWriterID (2B) + Sequence# + Payload (UA Binary) │
└──────────────────────────────────────────────────────────┘

지연 목표: < 10ms (Pub/Sub + TSN CBS Class A)
```

---

## OPC UA 정보 모델 (AddressSpace)

### NodeClass 및 NodeID 체계

```
OPC UA NodeClass 8종:
  Object         : 객체 (Robot, Joint, Sensor)
  Variable       : 변수 값 보유 (Position, Temperature)
  Method         : 호출 가능 메서드 (SetPosition, Start)
  ObjectType     : Object의 타입 정의 (클래스 역할)
  VariableType   : Variable의 타입 정의
  ReferenceType  : 노드 간 관계 정의
  DataType       : 데이터 타입 정의 (Enum 등)
  View           : AddressSpace의 서브셋

NodeID 형식:
  i=숫자     (Numeric):  ns=2;i=1001
  s=문자열   (String):   ns=2;s=Robot.Joint1.Position
  g=GUID     :           ns=1;g=09087e75-8a5d-11d3-...
  b=바이트배열(Opaque):   ns=1;b=M/RjKoRxWn...

Namespace 0: OPC UA 표준 (예약)
Namespace 1+: 애플리케이션 정의
```

### 의료 로봇 정보 모델

```
Objects (Root)
└── Robot (ObjectType: RobotType) ns=2;i=1001
    ├── Joints (FolderType)
    │   ├── Joint1 (JointType) ns=2;i=1010
    │   │   ├── Position     (Variable, Double, Unit: rad)
    │   │   ├── Velocity     (Variable, Double, Unit: rad/s)
    │   │   ├── Torque       (Variable, Double, Unit: Nm)
    │   │   ├── Temperature  (Variable, Float, Unit: °C)
    │   │   ├── SetPosition  (Method)
    │   │   │   Input: [target: Double, speed: Double]
    │   │   │   Output: [success: Boolean, error_code: UInt32]
    │   │   └── LimitMin/Max (Variable, Double, Writable=Admin)
    │   ├── Joint2~Joint6 (동일 구조)
    │   └── JointsGroup   (Method: MoveAll)
    ├── Status (ObjectType)
    │   ├── Mode        (Variable, Enum: IDLE/MOVING/ERROR/E-STOP)
    │   ├── IsReady     (Variable, Boolean)
    │   ├── ErrorCode   (Variable, UInt32)
    │   └── LastError   (Variable, String)
    ├── Diagnostics
    │   ├── OperatingHours (Variable, UInt64)
    │   ├── ProcedureCount (Variable, UInt32)
    │   └── DTCList        (Variable, Array<UInt32>)
    └── Events
        ├── EmergencyStop  (Event: EmergencyStopEventType)
        └── MaintenanceDue (Event: MaintenanceEventType)
```

---

## Python OPC UA 구현 (asyncua)

### OPC UA 서버 (로봇 측)

```python
import asyncio
from asyncua import Server, ua
from asyncua.common.methods import uamethod

async def main():
    server = Server()
    await server.init()
    server.set_endpoint("opc.tcp://0.0.0.0:4840/medical_robot/")
    server.set_server_name("Medical Surgical Robot OPC UA Server v1.0")

    # 보안 정책 설정
    await server.set_security_policy([
        ua.SecurityPolicyType.NoSecurity,               # 개발용
        ua.SecurityPolicyType.Basic256Sha256_SignAndEncrypt  # 운영용
    ])

    # 네임스페이스
    ns = await server.register_namespace("http://hospital.com/robot/v1")

    # 정보 모델 구성
    robot = await server.nodes.objects.add_object(ns, "Robot")

    # 관절 객체
    joints = await robot.add_object(ns, "Joints")
    joint_nodes = []
    for i in range(1, 7):
        jt = await joints.add_object(ns, f"Joint{i}")
        pos = await jt.add_variable(ns, "Position",    0.0, ua.VariantType.Double)
        vel = await jt.add_variable(ns, "Velocity",    0.0, ua.VariantType.Double)
        tmp = await jt.add_variable(ns, "Temperature", 25.0, ua.VariantType.Float)
        await pos.set_writable()
        joint_nodes.append((jt, pos, vel, tmp))

    # 상태 객체
    status = await robot.add_object(ns, "Status")
    mode_var    = await status.add_variable(ns, "Mode",     "IDLE", ua.VariantType.String)
    ready_var   = await status.add_variable(ns, "IsReady",  False,  ua.VariantType.Boolean)
    error_var   = await status.add_variable(ns, "ErrorCode",0,      ua.VariantType.UInt32)

    async with server:
        print("OPC UA Server started at opc.tcp://0.0.0.0:4840/")
        while True:
            # 하드웨어에서 실제 데이터 읽어 정보 모델 업데이트
            for i, (jt, pos, vel, tmp) in enumerate(joint_nodes):
                await pos.write_value(read_encoder(i))
                await vel.write_value(compute_velocity(i))
                await tmp.write_value(read_temperature(i))
            await asyncio.sleep(0.01)  # 100Hz 업데이트

asyncio.run(main())
```

### OPC UA 클라이언트 (SCADA/모니터링)

```python
import asyncio
from asyncua import Client
from asyncua.common.subscription import SubHandler

class RobotDataHandler(SubHandler):
    """MonitoredItem 변경 콜백"""
    def datachange_notification(self, node, val, data):
        node_id = node.nodeid.to_string()
        print(f"[변경] {node_id} = {val}")

async def monitor_robot():
    async with Client("opc.tcp://192.168.1.100:4840/") as client:
        # 보안 연결 (운영 환경)
        await client.set_security_string(
            "Basic256Sha256,SignAndEncrypt,"
            "/certs/client.der,/certs/client.pem,/certs/server.der"
        )

        ns = await client.get_namespace_index("http://hospital.com/robot/v1")

        # 노드 탐색
        robot  = await client.nodes.objects.get_child([f"{ns}:Robot"])
        joints = await robot.get_child([f"{ns}:Joints"])
        j1     = await joints.get_child([f"{ns}:Joint1"])
        pos    = await j1.get_child([f"{ns}:Position"])

        # 현재값 읽기
        current = await pos.read_value()
        print(f"Joint1 Position: {current:.4f} rad")

        # Subscription 설정 (100ms 주기 감시)
        handler = RobotDataHandler()
        sub = await client.create_subscription(100, handler)
        await sub.subscribe_data_change([pos])

        # 메서드 호출 (관절 이동 명령)
        result = await j1.call_method(f"{ns}:SetPosition", 1.5708, 0.5)
        print(f"SetPosition 결과: {result}")

        await asyncio.sleep(30)

asyncio.run(monitor_robot())
```

---

## OPC UA over TSN 네트워크 설정

```
WriterGroup / ReaderGroup 구성:

Publisher (로봇 ECU):
  PublishedDataSet:
    - Robot/Joints/*/Position  (6개 변수)
    - Robot/Status/Mode
  WriterGroup:
    GroupVersion:      1
    WriterGroupId:     0x0001
    PublishingInterval: 10ms  ← 100Hz 발행
    Priority:          4      ← TSN PCP=4

Subscriber (제어 PC):
  ReaderGroup (WriterGroupId=0x0001 구독)
  SubscribedDataSet → Robot 상태 업데이트

TSN 네트워크 설정:
  VLAN 20, PCP=4: OPC UA Pub/Sub (10ms, CBS Class A)
  VLAN 30, PCP=2: OPC UA Client/Server (100ms)
  Multicast 주소: 239.0.0.1 (OPC UA PubSub 기본)
```

---

## OPC UA 보안

```
보안 채널 (SecureChannel) 설정:
  SecurityMode:
    None:          개발/테스트 환경 전용 (운영 금지)
    Sign:          서명만 (무결성 검증)
    SignAndEncrypt : 서명 + 암호화 (의료 환경 필수)

보안 정책 (SecurityPolicy):
  Basic256Sha256:
    대칭: AES-256-CBC + HMAC-SHA256
    비대칭: RSA-2048 이상 (세션 키 교환)
    권장: 범용 산업 환경
  Aes128_Sha256_RsaOaep:
    대칭: AES-128-GCM + HMAC-SHA256
    비대칭: RSA-OAEP (더 안전한 패딩)
  Aes256_Sha256_RsaPss:
    비대칭: RSA-PSS (가장 강력)
    권장: 의료기기, 고보안 환경

인증 방식:
  Anonymous:          인증 없음 (개발용)
  UserName/Password:  사용자 ID + 비밀번호
  Certificate:        X.509 인증서 (의료/산업 권장)
  Kerberos:           도메인 통합

RBAC (Role-Based Access Control):
  Observer:  읽기 전용
  Operator:  읽기 + 기본 명령 (Start/Stop)
  Engineer:  읽기/쓰기 + 진단 + 파라미터
  Admin:     전체 권한 + 설정 변경 + 인증서 관리
```

---

## 의료 기기 상호운용성 (IEEE 11073 SDC)

```
통합 수술실(IOR) 아키텍처:

수술 로봇 ────── OPC UA ──────► 의료기기 게이트웨이
마취기    ────── SDC (IEEE 11073-20701) ──►  (상호 변환)
환자 모니터 ─── SDC ──────────────────────►
영상 장비  ────── DICOM ───────────────────►
                                           │
                                    통합 환자 데이터 뷰
                                    (동일 OPC UA 정보 모델)

연동 시나리오:
  ECG 이상(ST 상승) 감지
    └► 환자 모니터 → OPC UA → 수술 로봇
    └► 로봇: 현재 동작 일시정지, 안전 대기 상태
    └► 알림: 외과 의사 HMD에 경고 표시

  마취 심도 데이터 기록
    └► 마취기 → SDC → OPC UA → 히스토리안
    └► 로봇 동작과 타임스탬프 동기 기록
    └► 수술 후 분석 데이터 자동 생성

IHE (Integrating Healthcare Enterprise) 프로파일:
  IHE PCD-01: 환자 케어 기기 데이터 관찰
  IHE PCD-02: 기기 경보 통신
```

---

## Reference
- [OPC UA Specification Part 1-22 (IEC 62541)](https://opcfoundation.org/developer-tools/specifications-unified-architecture)
- [OPC UA Part 14: PubSub](https://opcfoundation.org/developer-tools/specifications-unified-architecture/part-14-pubsub/)
- [OPC UA over TSN - OPC Foundation](https://opcfoundation.org/developer-tools/specifications-opc-ua-for-specific-domains)
- [asyncua Python Library (GitHub)](https://github.com/FreeOpcUa/opcua-asyncio)
- [open62541 - C/C++ OPC UA Stack](https://www.open62541.org/)
- [IEEE 11073-20701 SDC](https://standards.ieee.org/ieee/11073-20701/6979/)
- [IEC 62541 OPC UA (ISO)](https://www.iso.org/standard/71460.html)
