# DoIP (Diagnostics over IP)

## 개요
전통적인 차량 진단은 OBD-II 포트를 통해 CAN 통신을 사용하여 수행되었습니다. 하지만 전자 제어 장치(ECU)의 수가 늘어나고 소프트웨어의 복잡도가 증가하면서, 기존 방식으로는 진단 속도와 데이터 전송량의 한계에 부딪혔습니다. 이에 따라, Ethernet의 고속 대역폭을 활용하여 진단을 수행하는 DoIP(Diagnostics over IP) 표준(ISO 13400)이 제정되었습니다. 의료 로봇 분야에서도 복잡한 시스템의 유지보수와 펌웨어 업데이트를 위해 DoIP 도입이 활발히 이루어지고 있습니다.

### DoIP의 주요 기능
1.  **고속 데이터 전송**: 기존 CAN 기반 진단(UDS on CAN)보다 수백 배 빠른 속도(100Mbps ~ 1Gbps)로 대용량 데이터를 전송할 수 있습니다. 이는 펌웨어 업데이트(FOTA) 시간을 획기적으로 단축시킵니다.
2.  **원격 진단**: IP 기반 통신이므로, 인터넷이나 병원 내부망을 통해 원격지에서도 로봇의 상태를 진단하고 문제를 해결할 수 있습니다.
3.  **다중 접속**: 여러 진단 기기(Tester)가 동시에 로봇에 접속하여, 각기 다른 ECU를 진단하거나 모니터링할 수 있습니다.

### 동작 원리
DoIP는 TCP/UDP 포트 13400을 사용하며, 클라이언트(Tester)와 서버(Gateway/ECU) 간의 연결 설정, 인증, 데이터 전송 과정을 거칩니다.
-   **Vehicle Discovery**: UDP 브로드캐스트를 통해 네트워크에 연결된 차량(로봇)을 탐색합니다.
-   **Routing Activation**: TCP 연결 후, 인증 과정을 거쳐 진단 세션을 활성화합니다.
-   **Diagnostic Message**: UDS(Unified Diagnostic Services, ISO 14229) 메시지를 IP 패킷에 실어 전송합니다.

### 의료 로봇에서의 활용
수술 로봇이나 AI 진단 장비는 수시로 소프트웨어 업데이트가 필요하며, 고장이 발생했을 때 신속한 원인 파악이 중요합니다. DoIP를 적용하면 다음과 같은 이점을 얻을 수 있습니다.
-   **빠른 펌웨어 업데이트**: 기가비트 이더넷을 통해 수 기가바이트(GB)에 달하는 OS 이미지나 AI 모델을 몇 분 안에 업데이트할 수 있습니다.
-   **효율적인 유지보수**: 엔지니어가 현장에 방문하지 않고도 원격으로 로그 파일을 분석하거나, 실시간 센서 데이터를 모니터링하여 고장을 진단할 수 있습니다.
-   **표준화**: ISO 표준을 따르므로, 상용 진단 툴이나 스캐너를 활용하여 개발 및 검증 효율성을 높일 수 있습니다.

## Reference
- [ISO 13400-2:2019 - Road vehicles — Diagnostic communication over Internet Protocol (DoIP)](https://www.iso.org/standard/69527.html)
- [Vector - Diagnostics over IP](https://www.vector.com/kr/ko/know-how/protocols/doip/)
