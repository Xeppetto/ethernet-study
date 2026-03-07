# MQTT (Message Queuing Telemetry Transport)

## 개요
MQTT는 IoT 환경을 위해 설계된 경량 발행/구독(Pub/Sub) 메시징 프로토콜입니다(OASIS 표준, ISO/IEC 20922). 최소 2 Byte 고정 헤더, TCP 기반 중앙 브로커 아키텍처가 특징입니다. 저대역폭·고지연 네트워크(2G/3G, 위성)에서도 안정적으로 동작하도록 설계되었습니다.

의료 로봇 시스템에서 MQTT는 **클라우드 원격 모니터링, 알림 시스템, 예측 유지보수** 레이어에 활용됩니다. 실시간 관절 제어에는 DDS/SOME/IP를 사용하고, 클라우드로 1초 주기 상태 보고에 MQTT를 사용하는 **2중 통신 계층**이 표준 아키텍처입니다. Edge Gateway가 DDS 메시지를 JSON으로 변환하여 MQTT Broker로 발행합니다.

---

## MQTT 아키텍처

```
┌────────────────────────────────────────────────────────────────┐
│  MQTT Broker (메시지 라우팅 허브)                               │
│  - 토픽 기반 메시지 라우팅                                       │
│  - QoS 레벨 보장 (0/1/2)                                        │
│  - Retained 메시지 저장 (토픽당 최신 1개)                        │
│  - Will Message 처리 (비정상 종료 감지)                          │
│  - 세션 관리 (CleanSession / PersistSession)                    │
│  - TLS/mTLS 인증, ACL 인가                                      │
└────────┬────────────────────────────────────┬──────────────────┘
         │                                    │
    Publisher                           Subscriber
  (Robot Edge Gateway)              (Cloud Dashboard, 알림 서버)
  Topic: robot/R001/status          Topic: robot/+/status
  QoS: 1, Retain: false             QoS: 1

브로커 포트:
  1883: TCP (평문, 개발용)
  8883: TCP + TLS (운영 필수)
  9001: WebSocket (브라우저 대시보드)
  9443: WebSocket + TLS
```

---

## MQTT 패킷 구조

### 고정 헤더 (Fixed Header)

```
MQTT Fixed Header (최소 2 Byte):
 7       4  3       0
┌──────────┬──────────┐
│ Pkt Type │  Flags   │  Byte 1
│  (4 bit) │  (4 bit) │
├──────────────────────┤
│  Remaining Length    │  Byte 2~5 (가변 길이 인코딩)
│  (1~4 Byte VLE)      │
└──────────────────────┘

Remaining Length 인코딩 (Variable Length Encoding):
  0~127 Byte     : 1 Byte (MSB=0)
  128~16383 Byte : 2 Byte (MSB=1, 연속)
  최대: 268,435,455 Byte (약 256MB)

주요 패킷 타입:
  0x10: CONNECT      (클라이언트 → 브로커, 연결 요청)
  0x20: CONNACK      (브로커 → 클라이언트, 연결 응답)
  0x30: PUBLISH      (데이터 발행, Flags: DUP|QoS|RETAIN)
  0x40: PUBACK       (QoS 1 수신 확인)
  0x50: PUBREC       (QoS 2 수신 확인 1단계)
  0x60: PUBREL       (QoS 2 릴리즈)
  0x70: PUBCOMP      (QoS 2 완료)
  0x80: SUBSCRIBE    (구독 요청)
  0x90: SUBACK       (구독 응답)
  0xA0: UNSUBSCRIBE  (구독 해제)
  0xB0: UNSUBACK     (구독 해제 응답)
  0xC0: PINGREQ      (연결 유지 Ping)
  0xD0: PINGRESP     (Ping 응답)
  0xE0: DISCONNECT   (연결 종료)
```

### CONNECT 패킷 주요 필드

```
CONNECT Payload:
  Protocol Name:   "MQTT" (MQTT v3.1.1 이상)
  Protocol Level:  0x04 (v3.1.1) / 0x05 (v5.0)
  Connect Flags:
    Bit 7: UserName Flag
    Bit 6: Password Flag
    Bit 5: Will Retain
    Bit 4~3: Will QoS
    Bit 2: Will Flag        ← LWT 설정 여부
    Bit 1: CleanSession     ← 1=세션 초기화, 0=이전 세션 복원
    Bit 0: Reserved
  KeepAlive: 60초 (PINGREQ 주기)
  ClientID: "robot_R001" (고유 식별자)
  Will Topic, Will Message (선택)
  Username, Password (선택)
```

---

## QoS (Quality of Service) 레벨

```
QoS 0: At Most Once (최대 한 번)
────────────────────────────────────────────────────
Publisher ──► Broker: PUBLISH (한 번만)
Broker    ──► Subscriber: PUBLISH (한 번만)
ACK 없음, 재전송 없음, 손실 가능
오버헤드 최소 (헤더 2B)
적합: 실시간 센서 스트리밍, 빈번한 업데이트

QoS 1: At Least Once (최소 한 번)
────────────────────────────────────────────────────
Publisher ──► Broker: PUBLISH (PacketID=5)
Broker    ──► Publisher: PUBACK (PacketID=5)  ← ACK
Broker    ──► Subscriber: PUBLISH (PacketID=5)
Subscriber ──► Broker: PUBACK (PacketID=5)
ACK 없으면 재전송 (DUP 플래그 세팅)
중복 수신 가능 (구독자가 멱등성 처리 필요)
적합: 알림 메시지, 상태 업데이트

QoS 2: Exactly Once (정확히 한 번)
────────────────────────────────────────────────────
4-way Handshake:
  Publisher ──► Broker: PUBLISH (PacketID=5)
  Broker    ──► Publisher: PUBREC (5)   ← 수신 확인
  Publisher ──► Broker: PUBREL (5)      ← 릴리즈 요청
  Broker    ──► Publisher: PUBCOMP (5)  ← 완료
  (브로커 → 구독자도 동일 4-way)
중복 없음, 손실 없음, 가장 높은 오버헤드
적합: 중요 명령, 설정 변경, 과금 데이터
```

---

## Will Message (LWT: Last Will and Testament)

```
비정상 종료 감지 메커니즘:

1. CONNECT 시 Will 등록:
   Will Topic:   robot/R001/status
   Will Payload: {"online": false, "reason": "unexpected_disconnect"}
   Will QoS:     1
   Will Retain:  true

2. 클라이언트가 DISCONNECT 없이 연결 끊김 감지:
   Broker KeepAlive 타이머 만료 (KeepAlive × 1.5)
   예: KeepAlive=60s → 90초 후 LWT 발행

3. Broker가 Will Message 자동 발행:
   모든 robot/+/status 구독자에게 전달
   → 오퍼레이터 알림, 자동 재시작 트리거

정상 종료 (DISCONNECT) 시: Will Message 발행 안 됨
```

---

## Retained Message (보존 메시지)

```
Retain Flag = 1로 발행된 메시지:
  브로커가 토픽당 최신 1개 보관
  → 새로운 구독자 연결 시 즉시 최신값 수신
  → 로봇 상태 대시보드에 필수

사용 예:
  robot/R001/status (retain=true): 로봇 온라인 상태
  robot/R001/firmware/version (retain=true): 현재 FW 버전
  hospital/A/robot_count (retain=true): 활성 로봇 수

Retained Message 삭제:
  동일 토픽에 빈 페이로드(0 Byte)로 Retain=true 발행
```

---

## Python MQTT 구현 (paho-mqtt)

### Publisher (로봇 Edge Gateway → Cloud)

```python
import paho.mqtt.client as mqtt
import json
import time
import ssl
import threading

BROKER_HOST = 'iot.hospital.com'
BROKER_PORT = 8883
ROBOT_ID    = 'R001'

class RobotMQTTClient:
    def __init__(self):
        self.client = mqtt.Client(
            client_id=f'robot_{ROBOT_ID}',
            protocol=mqtt.MQTTv5
        )
        self._setup_tls()
        self._setup_will()
        self.client.on_connect    = self._on_connect
        self.client.on_disconnect = self._on_disconnect
        self.client.on_publish    = self._on_publish

    def _setup_tls(self):
        self.client.tls_set(
            ca_certs    = '/certs/ca.crt',
            certfile    = f'/certs/robot_{ROBOT_ID}.crt',
            keyfile     = f'/certs/robot_{ROBOT_ID}.key',
            tls_version = ssl.PROTOCOL_TLS_CLIENT,
            cert_reqs   = ssl.CERT_REQUIRED
        )

    def _setup_will(self):
        """비정상 종료 시 발행될 LWT"""
        self.client.will_set(
            topic   = f'robot/{ROBOT_ID}/status',
            payload = json.dumps({'online': False, 'reason': 'unexpected_disconnect'}),
            qos     = 1,
            retain  = True
        )

    def _on_connect(self, client, userdata, flags, rc, properties=None):
        if rc == 0:
            # 정상 연결 시 온라인 상태 발행 (Retain)
            client.publish(
                f'robot/{ROBOT_ID}/status',
                json.dumps({'online': True, 'timestamp': time.time()}),
                qos=1, retain=True
            )
        else:
            print(f"Connection failed: {rc}")

    def _on_disconnect(self, client, userdata, rc, properties=None):
        if rc != 0:
            print(f"Unexpected disconnect: {rc}, reconnecting...")
            time.sleep(5)
            client.reconnect()

    def _on_publish(self, client, userdata, mid):
        pass  # 발행 완료 확인 (디버깅용)

    def publish_status(self, joint_positions, temperature, errors):
        payload = {
            'timestamp'       : time.time(),
            'robot_id'        : ROBOT_ID,
            'joint_positions' : joint_positions,
            'temperature_C'   : temperature,
            'error_codes'     : errors,
            'uptime_s'        : int(time.monotonic())
        }
        self.client.publish(
            topic   = f'robot/{ROBOT_ID}/telemetry',
            payload = json.dumps(payload),
            qos     = 0,    # 텔레메트리는 QoS 0 (빈번, 손실 허용)
            retain  = False
        )

    def connect_and_run(self):
        self.client.connect(BROKER_HOST, BROKER_PORT, keepalive=60)
        self.client.loop_start()  # 별도 스레드에서 네트워크 루프

# 사용 예시
mqtt_client = RobotMQTTClient()
mqtt_client.connect_and_run()

while True:
    mqtt_client.publish_status(
        joint_positions=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6],
        temperature=42.3,
        errors=[]
    )
    time.sleep(1)  # 1Hz (클라우드 보고용, 제어는 DDS 1kHz)
```

### Subscriber (클라우드 대시보드)

```python
import paho.mqtt.client as mqtt
import json

def on_connect(client, userdata, flags, rc, properties=None):
    print(f"Connected to broker: rc={rc}")
    # 와일드카드 구독
    client.subscribe('robot/+/status',    qos=1)  # 온라인 상태
    client.subscribe('robot/+/telemetry', qos=0)  # 텔레메트리
    client.subscribe('robot/+/alert',     qos=2)  # 긴급 알림

def on_message(client, userdata, msg):
    parts     = msg.topic.split('/')
    robot_id  = parts[1]
    data_type = parts[2]
    payload   = json.loads(msg.payload.decode())

    if data_type == 'alert':
        send_notification(robot_id, payload)
    elif data_type == 'status' and not payload.get('online'):
        trigger_recovery_procedure(robot_id)

client = mqtt.Client(protocol=mqtt.MQTTv5)
client.on_connect = on_connect
client.on_message = on_message
client.tls_set(ca_certs='/certs/ca.crt')
client.connect('iot.hospital.com', 8883)
client.loop_forever()
```

---

## MQTT v5.0 주요 신기능

| 기능 | 설명 | 의료 로봇 활용 |
|------|------|-------------|
| Message Expiry Interval | 메시지 유효 시간 (초) | 오래된 알림 자동 폐기 |
| User Properties | 커스텀 헤더 (Key-Value 쌍) | 로봇 ID, 버전 태깅 |
| Topic Alias | 긴 토픽 이름을 숫자로 대체 | 대역폭 절약 |
| Shared Subscriptions | $share/group/topic | 다중 서버 로드 밸런싱 |
| Request-Response Pattern | 응답 토픽 지정 | 원격 명령 응답 |
| Flow Control | Receive Maximum 설정 | 브로커 과부하 방지 |
| Reason Codes | 상세 오류 코드 | 장애 원인 파악 |
| Session Expiry Interval | 세션 보존 기간 설정 | 오프라인 메시지 관리 |

---

## MQTT 브로커 선택 가이드

| 브로커 | 최대 연결 | 라이선스 | 클러스터 | 추천 용도 |
|--------|---------|---------|---------|---------|
| Mosquitto 2.x | ~수천 | EPL 2.0 | 미흡 | 소규모, 개발/테스트 |
| EMQX 5.x | 1억+ | Apache 2.0 / 상용 | 우수 | 대규모 IIoT |
| HiveMQ 4.x | 수백만 | 상용 | 우수 | 엔터프라이즈 |
| VerneMQ | 수십만 | Apache 2.0 | 우수 | 오픈소스 클러스터 |
| AWS IoT Core | 무제한 (관리형) | 종량제 | 자동 | AWS 기반 인프라 |
| Azure IoT Hub | 수백만 (관리형) | 종량제 | 자동 | Azure 기반 |

```bash
# Mosquitto 설치 및 TLS 설정 (Ubuntu)
apt-get install mosquitto mosquitto-clients

# TLS 설정 파일 (/etc/mosquitto/conf.d/tls.conf)
cat > /etc/mosquitto/conf.d/tls.conf << 'EOF'
listener 8883
cafile   /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile  /etc/mosquitto/certs/server.key
require_certificate true
allow_anonymous false
password_file /etc/mosquitto/passwd
EOF

# 클라이언트 발행 테스트 (TLS)
mosquitto_pub -h localhost -p 8883 \
  --cafile /etc/mosquitto/certs/ca.crt \
  --cert /certs/robot_R001.crt \
  --key  /certs/robot_R001.key \
  -t 'robot/R001/status' \
  -m '{"online":true}' -r  # -r: retain

# 구독 테스트
mosquitto_sub -h localhost -p 8883 \
  --cafile /etc/mosquitto/certs/ca.crt \
  -t 'robot/+/status' -v
```

---

## 의료 로봇 MQTT 통합 아키텍처

```
[로봇 내부 - 실시간 제어 계층]
  DDS (ROS 2, 1kHz)
  SOME/IP (vsomeip, 100Hz~1kHz)
       │
       │ 실시간 Ethernet (TSN)
       ▼
[Edge Gateway (로봇 메인 컨트롤러)]
  ├── DDS → JSON 변환 (bridge_node)
  ├── 데이터 집계 (1kHz → 1Hz 다운샘플링)
  └── MQTT 발행 (1Hz)
       │
       │ MQTT over TLS 8883
       │ (병원 내부망 or 인터넷)
       ▼
[Cloud MQTT Broker (EMQX / AWS IoT Core)]
  ├── 실시간 대시보드  (WebSocket + MQTT.js)
  ├── 시계열 DB 저장  (InfluxDB / TimescaleDB)
  ├── AI 이상 감지    (Azure ML / SageMaker)
  ├── 알림 시스템     (PagerDuty, SMS, 이메일)
  └── OTA 업데이트   (DoIP over TLS → 로봇)

MQTT 토픽 체계:
  robot/{id}/status          → 온라인/오프라인 (Retain, QoS 1)
  robot/{id}/telemetry       → 1Hz 상태 데이터 (QoS 0)
  robot/{id}/alert           → 긴급 알림 (QoS 2)
  robot/{id}/ota/command     → OTA 명령 수신 (QoS 2)
  robot/{id}/ota/progress    → OTA 진행률 (QoS 1)
  hospital/{hid}/robot/+/status → 병원 전체 로봇 현황
```

---

## Reference
- [MQTT v5.0 Specification (OASIS)](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- [MQTT v3.1.1 Specification (ISO/IEC 20922)](https://www.iso.org/standard/69466.html)
- [Eclipse Paho MQTT Python Client](https://github.com/eclipse/paho.mqtt.python)
- [Eclipse Mosquitto Broker](https://mosquitto.org/documentation/)
- [EMQX Documentation](https://www.emqx.io/docs/en/v5.0/)
- [AWS IoT Core MQTT Developer Guide](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
