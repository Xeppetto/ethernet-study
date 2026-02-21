# MQTT (Message Queuing Telemetry Transport)

## 개요
MQTT는 IoT 환경을 위한 경량 발행/구독(Pub/Sub) 메시징 프로토콜입니다(OASIS 표준). 최소 2 Byte 헤더, TCP 기반, 중앙 브로커 아키텍처가 특징입니다. 의료 로봇의 **클라우드 연동, 원격 모니터링, 알림 시스템**에 적합하며, 실시간 로컬 제어에는 SOME/IP나 DDS를 사용하고 MQTT는 클라우드 레이어에 활용합니다.

---

## MQTT 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│  MQTT Broker (예: Mosquitto, EMQX, HiveMQ, AWS IoT Core)   │
│  - 메시지 라우팅 (토픽 기반)                                  │
│  - QoS 보장                                                  │
│  - 세션 관리 / 클린 세션                                      │
│  - 인증/인가 (TLS + Username/Password)                       │
└─────┬────────────────────────────┬──────────────────────────┘
      │                            │
Publisher                      Subscriber
(Robot ECU)                    (클라우드, 모니터링 PC)
  Topic: robot/joint/status        Topic: robot/joint/status
  QoS: 1                           QoS: 1
```

---

## MQTT 패킷 구조

```
MQTT Fixed Header (최소 2 Byte):
┌────────────────────────────────┬────────────────────────────┐
│  Packet Type (4b) + Flags (4b) │  Remaining Length (1~4B)   │
└────────────────────────────────┴────────────────────────────┘

주요 패킷 타입:
  0x10: CONNECT
  0x20: CONNACK (연결 응답)
  0x30: PUBLISH (데이터 발행)
  0x40: PUBACK (QoS 1 ACK)
  0x80: SUBSCRIBE (구독 요청)
  0x90: SUBACK (구독 응답)
  0xC0: PINGREQ (연결 유지)
  0xE0: DISCONNECT
```

---

## QoS (Quality of Service) 레벨

```
QoS 0: At Most Once (최대 한 번)
  Publisher → Broker → Subscriber
  재전송 없음, 손실 가능
  최저 오버헤드, 실시간 센서 스트리밍 적합

QoS 1: At Least Once (최소 한 번)
  Publisher → Broker: PUBLISH
  Broker → Publisher: PUBACK
  Subscriber → Broker: PUBACK
  중복 수신 가능 (수신 측에서 중복 처리 필요)
  대부분의 알림/상태 전송에 적합

QoS 2: Exactly Once (정확히 한 번)
  4-way Handshake (PUBLISH → PUBREC → PUBREL → PUBCOMP)
  중복 없음, 가장 높은 오버헤드
  중요한 명령/설정 변경에 적합
```

---

## 토픽 설계

```
MQTT 토픽 계층 구조 (의료 로봇):

robot/{robot_id}/joint/{joint_id}/position   → 조인트 위치
robot/{robot_id}/joint/{joint_id}/velocity   → 조인트 속도
robot/{robot_id}/status                      → 전체 상태
robot/{robot_id}/error                       → 오류 알림
robot/{robot_id}/diagnostics/dtc             → DTC 목록
robot/{robot_id}/ota/progress                → OTA 진행률
hospital/{hospital_id}/robot/+/status        → 병원 내 모든 로봇 상태

와일드카드:
  +: 단일 레벨 (예: robot/+/status = 모든 로봇의 상태)
  #: 다중 레벨 (예: robot/R001/# = R001 로봇의 모든 데이터)
```

---

## Python MQTT 예시 (paho-mqtt)

### Publisher (로봇 → 클라우드)
```python
import paho.mqtt.client as mqtt
import json
import time
import ssl

BROKER_HOST = 'iot.hospital.com'
BROKER_PORT = 8883  # TLS 포트
ROBOT_ID = 'R001'
CA_CERT = '/certs/ca.crt'
CLIENT_CERT = '/certs/robot.crt'
CLIENT_KEY = '/certs/robot.key'

def create_client():
    client = mqtt.Client(client_id=f'robot_{ROBOT_ID}',
                         protocol=mqtt.MQTTv5)

    # TLS 설정 (상호 인증)
    client.tls_set(
        ca_certs=CA_CERT,
        certfile=CLIENT_CERT,
        keyfile=CLIENT_KEY,
        tls_version=ssl.PROTOCOL_TLS_CLIENT
    )

    # Last Will Testament (로봇 비정상 종료 시 발행)
    client.will_set(
        topic=f'robot/{ROBOT_ID}/status',
        payload=json.dumps({'online': False, 'reason': 'unexpected_disconnect'}),
        qos=1, retain=True
    )

    client.connect(BROKER_HOST, BROKER_PORT, keepalive=60)
    return client

client = create_client()
client.loop_start()

# 로봇 상태 주기적 발행
while True:
    robot_status = {
        'timestamp': time.time(),
        'joint_positions': [0.1, 0.2, 0.3, 0.4, 0.5, 0.6],
        'temperature': 42.3,
        'battery_voltage': 24.1,
        'error_code': 0
    }

    client.publish(
        topic=f'robot/{ROBOT_ID}/status',
        payload=json.dumps(robot_status),
        qos=1,
        retain=False
    )
    time.sleep(1)  # 1Hz (클라우드용, 제어는 DDS로)
```

### Subscriber (클라우드 대시보드)
```python
import paho.mqtt.client as mqtt
import json

def on_connect(client, userdata, flags, rc, properties=None):
    print(f"Connected: {rc}")
    # 모든 로봇 상태 구독
    client.subscribe('robot/+/status', qos=1)
    client.subscribe('robot/+/error', qos=2)

def on_message(client, userdata, msg):
    topic_parts = msg.topic.split('/')
    robot_id = topic_parts[1]
    data_type = topic_parts[2]

    payload = json.loads(msg.payload.decode())
    print(f"Robot {robot_id} {data_type}: {payload}")

    # 오류 시 알림 전송 로직
    if data_type == 'error' and payload.get('error_code', 0) != 0:
        send_alert(robot_id, payload)

client = mqtt.Client(protocol=mqtt.MQTTv5)
client.on_connect = on_connect
client.on_message = on_message
client.connect('iot.hospital.com', 8883)
client.loop_forever()
```

---

## MQTT v5.0 주요 신기능

| 기능 | 설명 |
|------|------|
| Message Expiry Interval | 메시지 유효 시간 설정 |
| User Properties | 커스텀 헤더 추가 (Key-Value) |
| Topic Alias | 긴 토픽 이름을 숫자로 대체 (대역폭 절약) |
| Shared Subscriptions | 로드 밸런싱 구독 (여러 구독자 중 한 명에게) |
| Request-Response | 응답 토픽 지정 가능 |
| Flow Control | 최대 수신 메시지 수 제어 |

---

## MQTT 브로커 선택 가이드

| 브로커 | 특징 | 추천 용도 |
|--------|------|---------|
| Mosquitto | 경량, 오픈소스 | 소규모, 개발용 |
| EMQX | 고성능, 클러스터 | 대규모 배포 |
| HiveMQ | 엔터프라이즈, 플러그인 | 기업 환경 |
| AWS IoT Core | 클라우드 관리형 | AWS 기반 IoT |
| Azure IoT Hub | 클라우드 관리형 | Azure 기반 IoT |

```bash
# Mosquitto 설치 및 실행 (개발용)
apt-get install mosquitto mosquitto-clients

# TLS 없이 기본 실행
mosquitto -p 1883

# TLS 적용 설정 (/etc/mosquitto/conf.d/tls.conf)
# listener 8883
# cafile /etc/mosquitto/ca.crt
# certfile /etc/mosquitto/server.crt
# keyfile /etc/mosquitto/server.key
# require_certificate true

# 메시지 발행 테스트
mosquitto_pub -h localhost -p 1883 -t 'robot/R001/status' -m '{"online":true}'

# 메시지 구독 테스트
mosquitto_sub -h localhost -p 1883 -t 'robot/+/status'
```

---

## 의료 로봇 MQTT 통합 아키텍처

```
로봇 내부 (실시간):
  DDS/SOME/IP ─► 조인트 제어 (1kHz, < 1ms)
  TSN Ethernet

Edge Gateway (변환):
  DDS 메시지 → JSON 변환 → MQTT 발행

클라우드 (모니터링):
  MQTT Broker (TLS 암호화)
  └─► 실시간 대시보드 (WebSocket)
  └─► 데이터 저장 (InfluxDB, TimescaleDB)
  └─► AI 분석 (이상 감지, 예측 유지보수)
  └─► 알림 (이메일, SMS, 앱 푸시)
```

---

## Reference
- [MQTT v5.0 Specification (OASIS)](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- [paho-mqtt Python Client](https://github.com/eclipse/paho.mqtt.python)
- [Eclipse Mosquitto Broker](https://mosquitto.org/)
- [EMQX Enterprise](https://www.emqx.com/en)
- [AWS IoT Core MQTT](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
