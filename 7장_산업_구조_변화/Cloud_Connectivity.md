# 클라우드 연동 (Cloud Connectivity)

## 개요
Ethernet, 5G, TSN 기술의 발전으로 의료 로봇이 병원 내부망을 넘어 클라우드와 실시간으로 연결되는 **Cloud Connected Robot** 시대가 열리고 있습니다. 단순한 원격 제어(Teleoperation)를 넘어, AI 연산 자원을 클라우드에서 빌려 쓰는 **Cloud-Native Robotics** 패러다임으로 진화하고 있습니다.

---

## 클라우드 연결 아키텍처

```
의료 로봇 클라우드 아키텍처:

┌──────────────────────────────────────────────────────────────┐
│                     Cloud Platform                           │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │
│  │ AI/ML      │  │ Digital    │  │ Fleet Management       │ │
│  │ Training   │  │ Twin       │  │ - OTA 업데이트         │ │
│  │ - 모델 학습 │  │ - 가상 로봇 │  │ - 원격 모니터링        │ │
│  │ - 데이터   │  │ - 시뮬레이션│  │ - 알람 관리            │ │
│  │   라벨링   │  │ - 예측 유지 │  │ - 성능 대시보드        │ │
│  └────────────┘  └────────────┘  └────────────────────────┘ │
│          │               │                    │              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │           Data Lake / Analytics Platform               │  │
│  │  - 수술 로그, 센서 데이터, 영상 (FHIR 표준)            │  │
│  │  - Spark/Flink 스트림 처리                             │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────┬──────────────────────────────────┘
                            │ TLS 1.3 + mTLS
                            │ 5G / Wi-Fi 6 / Fiber
┌───────────────────────────▼──────────────────────────────────┐
│                   Edge Gateway (병원 내)                       │
│  - 데이터 전처리 + 필터링                                      │
│  - 로컬 AI 추론 (지연 민감 작업)                               │
│  - HIPAA/GDPR 익명화 처리                                      │
│  - QoS 정책 (의료 데이터 우선)                                 │
└───────────────────────────┬──────────────────────────────────┘
                            │ Ethernet TSN (내부망)
                    ┌───────▼────────┐
                    │  수술 로봇      │
                    │  (Robot ECU)   │
                    └────────────────┘
```

---

## 데이터 연동 계층 (Edge-Fog-Cloud)

```
3계층 데이터 처리 모델:

Robot (Edge, 1ms 이내):
  - 실시간 제어 (관절 서보, E-Stop)
  - 로컬 센서 처리 (1kHz)
  - Safety Critical (TSN, ASIL D)
  - 클라우드 연결 불필요

Hospital Network (Fog, 10ms~1s):
  - 의료 영상 처리 (DICOM)
  - 환자 데이터 연동 (EHR)
  - 로컬 AI 추론 (Edge GPU)
  - PACS 서버 연동

Cloud (Seconds~Minutes):
  - AI 모델 학습 (GPU 클러스터)
  - 빅데이터 분석
  - OTA 업데이트 배포
  - 원격 전문의 텔레콘설팅

데이터 흐름:
  Robot → Fog → Cloud
          ↑       │
         AI 모델 배포 ← 학습 완료
```

---

## MQTT 기반 클라우드 연결 구현

```python
#!/usr/bin/env python3
"""
의료 로봇 → AWS IoT Core / Azure IoT Hub 연동
MQTT v5.0 + TLS 1.3 + mTLS
"""
import json
import ssl
import time
import threading
from datetime import datetime, timezone
import paho.mqtt.client as mqtt

# 클라우드 MQTT 엔드포인트 설정
CLOUD_HOST = "your-iot-core.iot.ap-northeast-2.amazonaws.com"
CLOUD_PORT = 8883
ROBOT_ID = "surgical_robot_R001"
CERT_DIR = "/etc/robot/certs"

class RobotCloudConnector:
    """
    로봇 - 클라우드 양방향 통신 클래스
    - 실시간 센서 데이터 업로드
    - AI 모델 업데이트 수신
    - 원격 진단 명령 처리
    """

    def __init__(self):
        self.client = mqtt.Client(
            client_id=ROBOT_ID,
            protocol=mqtt.MQTTv5
        )
        self._setup_tls()
        self._setup_callbacks()

        # 토픽 정의
        self.TOPICS = {
            'telemetry': f"robot/{ROBOT_ID}/telemetry",
            'diagnostics': f"robot/{ROBOT_ID}/diagnostics",
            'ai_model': f"robot/{ROBOT_ID}/model/update",
            'commands': f"robot/{ROBOT_ID}/commands",
            'alerts': f"robot/{ROBOT_ID}/alerts",
        }

    def _setup_tls(self):
        """mTLS 설정 (AWS IoT Core 요구)"""
        self.client.tls_set(
            ca_certs=f"{CERT_DIR}/AmazonRootCA1.pem",
            certfile=f"{CERT_DIR}/robot-certificate.pem.crt",
            keyfile=f"{CERT_DIR}/robot-private.pem.key",
            tls_version=ssl.PROTOCOL_TLS_CLIENT
        )

    def _setup_callbacks(self):
        self.client.on_connect = self._on_connect
        self.client.on_message = self._on_message
        self.client.on_disconnect = self._on_disconnect

    def _on_connect(self, client, userdata, flags, rc, properties=None):
        if rc == 0:
            print(f"[클라우드] 연결 성공")
            # 명령 토픽 구독
            client.subscribe(self.TOPICS['commands'], qos=1)
            client.subscribe(self.TOPICS['ai_model'], qos=1)
        else:
            print(f"[클라우드] 연결 실패: {rc}")

    def _on_message(self, client, userdata, msg):
        """클라우드 명령 수신 처리"""
        topic = msg.topic
        try:
            payload = json.loads(msg.payload.decode())
        except Exception:
            return

        if topic == self.TOPICS['commands']:
            self._handle_remote_command(payload)
        elif topic == self.TOPICS['ai_model']:
            self._handle_model_update(payload)

    def _on_disconnect(self, client, userdata, rc, properties=None):
        print(f"[클라우드] 연결 해제 (rc={rc}), 재연결 시도...")
        # 지수 백오프 재연결
        for delay in [2, 4, 8, 16, 32]:
            try:
                time.sleep(delay)
                self.client.reconnect()
                return
            except Exception:
                continue

    def _handle_remote_command(self, cmd: dict):
        """원격 진단/모니터링 명령 처리"""
        action = cmd.get('action', '')
        if action == 'get_diagnostics':
            # DTC 목록 읽기 → 클라우드에 보고
            dtcs = self._read_dtc()
            self.publish_alert({'type': 'diagnostics_response', 'dtcs': dtcs})
        elif action == 'set_log_level':
            level = cmd.get('level', 'INFO')
            print(f"[명령] 로그 레벨 변경: {level}")
        # OTA 명령은 UPTANE 프레임워크가 별도 처리

    def _handle_model_update(self, update: dict):
        """AI 모델 업데이트 처리"""
        model_url = update.get('model_url')
        model_hash = update.get('sha256')
        print(f"[AI 모델] 업데이트 수신: {model_url}")
        # 실제 구현: 다운로드 → 해시 검증 → 모델 로드

    def _read_dtc(self) -> list:
        """UDS DTC 읽기 (가상)"""
        return []  # 실제: DoIP/UDS 0x19 호출

    def publish_telemetry(self, data: dict):
        """
        센서 데이터 주기적 업로드 (1Hz)
        데이터 필드: 관절 위치/속도, 온도, 부하
        """
        payload = {
            'robot_id': ROBOT_ID,
            'timestamp': datetime.now(timezone.utc).isoformat(),
            'joint_positions': data.get('joints', [0]*6),
            'joint_temperatures': data.get('temps', [36.5]*6),
            'battery_voltage': data.get('battery', 24.0),
            'emergency_stop': data.get('estop', False),
        }
        # QoS 1: 전달 보장 (클라우드 데이터 손실 방지)
        self.client.publish(
            self.TOPICS['telemetry'],
            json.dumps(payload),
            qos=1
        )

    def publish_alert(self, alert: dict):
        """심각 이벤트 즉시 알림 (QoS 2)"""
        payload = {
            'robot_id': ROBOT_ID,
            'timestamp': datetime.now(timezone.utc).isoformat(),
            **alert
        }
        self.client.publish(
            self.TOPICS['alerts'],
            json.dumps(payload),
            qos=2  # 정확히 1번 전달 보장
        )

    def start(self):
        """연결 시작"""
        self.client.connect(CLOUD_HOST, CLOUD_PORT, keepalive=60)
        self.client.loop_start()
        print(f"[클라우드] 연결 중... {CLOUD_HOST}")

    def stop(self):
        """연결 종료"""
        self.client.loop_stop()
        self.client.disconnect()
```

---

## 클라우드별 IoT 서비스 비교

```
주요 클라우드 IoT 플랫폼:

┌────────────────┬──────────────────┬──────────────────┬──────────────┐
│                │ AWS IoT Core     │ Azure IoT Hub    │ GCP IoT Core │
├────────────────┼──────────────────┼──────────────────┼──────────────┤
│ 프로토콜       │ MQTT, HTTP, WS   │ MQTT, AMQP, HTTP │ MQTT, HTTP   │
│ 보안           │ X.509 mTLS       │ SAS Token + X.509│ JWT + mTLS   │
│ OTA            │ FreeRTOS OTA     │ Azure DPS        │ Cloud IoT OTA│
│ AI/ML          │ SageMaker        │ Azure ML         │ Vertex AI    │
│ Digital Twin   │ AWS TwinMaker    │ Azure DT         │ Cloud IoT    │
│ 의료 규정      │ HIPAA Eligible   │ HIPAA, HITRUST   │ HIPAA BAA    │
│ 지역 데이터    │ 서울 리전         │ Korea Central    │ 아시아 리전  │
└────────────────┴──────────────────┴──────────────────┴──────────────┘

의료 기기 클라우드 선택 기준:
  - HIPAA/GDPR/개인정보보호법 준수
  - 국내 데이터 보관 (서울 리전)
  - HL7 FHIR 지원 (의료 데이터 표준)
  - ISO 27001 인증
  - 지연 시간 (한국: AWS 서울 < 20ms)
```

---

## HIPAA/GDPR 준수 (의료 데이터 보안)

```python
# 개인정보 익명화 처리 (클라우드 전송 전)
import hashlib
import json
from typing import Any

def anonymize_medical_data(patient_data: dict) -> dict:
    """
    HIPAA De-identification (Safe Harbor 방법)
    18개 식별자 제거/해시 처리
    """
    PHI_FIELDS = [
        'patient_name', 'date_of_birth', 'address',
        'phone', 'email', 'ssn', 'mrn',
        'hospital_account', 'ip_address'
    ]

    anonymized = patient_data.copy()

    for field in PHI_FIELDS:
        if field in anonymized:
            # 직접 식별자 → 단방향 해시 (추적 방지)
            original = str(anonymized[field])
            anonymized[field] = hashlib.sha256(
                original.encode()
            ).hexdigest()[:8]  # 8자리로 단축

    # 날짜 → 연도만 유지 (나이 90세 이상은 '90+')
    if 'date_of_birth' in patient_data:
        import datetime
        dob = datetime.datetime.fromisoformat(str(patient_data['date_of_birth']))
        age = (datetime.datetime.now() - dob).days // 365
        anonymized['age_group'] = '90+' if age >= 90 else f"{(age//10)*10}s"
        del anonymized['date_of_birth']

    return anonymized

# 전송 전 암호화
def encrypt_and_send(data: dict, cloud_connector) -> bool:
    """
    클라우드 전송 전 처리 파이프라인:
    원본 데이터 → 익명화 → 압축 → 전송 (TLS 암호화)
    """
    import gzip

    # 1. 익명화
    anon_data = anonymize_medical_data(data)

    # 2. 직렬화 + 압축
    json_bytes = json.dumps(anon_data).encode()
    compressed = gzip.compress(json_bytes)

    # 3. 메타데이터 추가
    envelope = {
        'version': '1.0',
        'compressed': True,
        'size_original': len(json_bytes),
        'size_compressed': len(compressed),
        'data': compressed.hex()  # Base64가 더 효율적이지만 예시용
    }

    # 4. 업로드 (TLS는 paho-mqtt가 처리)
    cloud_connector.publish_telemetry(envelope)
    return True
```

---

## 디지털 트윈 (Digital Twin)

```
수술 로봇 디지털 트윈 구현:

실제 로봇 (Physical Twin)
  │ 실시간 데이터 스트림 (1~10Hz)
  │ 위치, 속도, 힘, 온도
  ▼
Digital Twin Platform (Cloud)
  ┌────────────────────────────────────────────┐
  │  가상 로봇 모델                             │
  │  - 동역학 시뮬레이션 (실제와 동기화)         │
  │  - 가상 센서 (예측값 생성)                  │
  │  - 물리 기반 고장 모델                      │
  │                                            │
  │  활용:                                     │
  │  ① 예측 유지보수: 마모 시뮬레이션            │
  │  ② 수술 계획: 로봇 동작 사전 시뮬레이션      │
  │  ③ 트레이닝: 외과의 훈련용 가상 환경          │
  │  ④ 원격 진단: 현장 없이 문제 분석            │
  └────────────────────────────────────────────┘

도구:
  AWS IoT TwinMaker: 3D 시각화 + 데이터 연동
  Azure Digital Twins: 그래프 기반 모델
  Siemens MindSphere: 제조 특화
  ROS 2 + Gazebo: 오픈소스, 시뮬레이션
  NVIDIA Isaac Sim: GPU 가속 물리 시뮬레이션
```

---

## Reference
- [AWS IoT Core](https://aws.amazon.com/iot-core/)
- [Azure IoT Hub](https://azure.microsoft.com/en-us/products/iot-hub/)
- [AWS Robotics](https://aws.amazon.com/robotics/)
- [HIPAA on AWS](https://aws.amazon.com/compliance/hipaa-compliance/)
- [HL7 FHIR Standard](https://hl7.org/fhir/)
- [NIST SP 800-66 - HIPAA Security Rule Guidance](https://csrc.nist.gov/publications/detail/sp/800-66/rev-2/final)
