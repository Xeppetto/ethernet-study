# MQTT (Message Queuing Telemetry Transport)

## 개요
MQTT는 IoT(Internet of Things) 및 M2M(Machine-to-Machine) 통신을 위해 설계된 경량(Lightweight) 메시징 프로토콜입니다. 발행/구독(Publish/Subscribe) 패턴을 따르며, TCP/IP 위에서 동작합니다. 네트워크 대역폭이 제한되거나 불안정한 환경(예: 3G/4G, 저속 위성 통신)에서도 안정적으로 데이터를 주고받을 수 있도록 최적화되어 있습니다.

### MQTT의 핵심 특징
1.  **경량 헤더 (Lightweight Header)**: 최소 2바이트의 헤더를 사용하여 오버헤드가 매우 적습니다.
2.  **Topic 기반 라우팅**: 클라이언트(Publisher)는 특정 주제(Topic)에 메시지를 발행하고, 다른 클라이언트(Subscriber)는 관심 있는 주제를 구독합니다. 브로커(Broker)가 이들 사이의 메시지 전달을 중개합니다.
3.  **QoS (Quality of Service) 레벨**: 메시지 전송 품질을 세 단계로 조절할 수 있습니다.
    -   Level 0 (At most once): "보내고 잊기(Fire and Forget)", 손실 가능성 있음.
    -   Level 1 (At least once): 최소 한 번은 전달됨을 보장, 중복 수신 가능성 있음.
    -   Level 2 (Exactly once): 정확히 한 번만 전달됨을 보장, 오버헤드 큼.
4.  **Last Will and Testament (LWT)**: 클라이언트가 비정상적으로 연결이 끊어졌을 때, 브로커가 미리 지정된 유언(Will) 메시지를 구독자들에게 대신 전송해 줍니다. (예: "Device Offline")

### 의료 로봇 분야 활용
의료 로봇은 수술 중에는 고속 이더넷이나 TSN과 같은 실시간 통신이 필요하지만, 로봇의 상태 모니터링이나 로그 데이터 전송과 같은 비실시간 작업에는 MQTT가 효과적일 수 있습니다.

-   **원격 모니터링**: 수술 로봇이 전 세계 병원에 설치되어 있을 때, 각 로봇의 가동 현황, 오류 로그, 소모품 잔량 등을 중앙 관제 센터로 전송하는 데 적합합니다.
-   **클라우드 연동**: AWS IoT Core나 Azure IoT Hub와 같은 클라우드 서비스와 쉽게 연동되어 빅데이터 분석 및 AI 모델 학습에 필요한 데이터를 수집할 수 있습니다.
-   **알림 시스템**: 로봇에 이상 징후가 감지되었을 때, 담당 엔지니어의 스마트폰 앱으로 즉시 푸시 알림을 보내는 기능을 구현하기 쉽습니다.

### 고려사항
MQTT는 중앙 브로커(Broker)를 거쳐 통신하므로, 실시간성이 중요한 제어 명령(Control Loop)에는 적합하지 않습니다. 따라서 제어 루프는 실시간 Ethernet/TSN을 사용하고, 모니터링 및 클라우드 통신은 MQTT를 사용하는 이원화된 아키텍처가 권장됩니다.

## Reference
- [MQTT - The Standard for IoT Messaging](https://mqtt.org/)
- [HiveMQ - MQTT Essentials](https://www.hivemq.com/mqtt-essentials/)
