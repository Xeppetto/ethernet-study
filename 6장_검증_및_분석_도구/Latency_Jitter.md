# Latency와 Jitter 분석

## 개요
Ethernet 네트워크를 사용하는 의료 로봇 시스템에서, 데이터가 얼마나 빨리 도착하는지(Latency)와 얼마나 일정하게 도착하는지(Jitter)는 성능을 결정하는 가장 중요한 지표입니다. 이 두 가지 요소를 정확히 측정하고 분석하는 것은 시스템의 실시간성을 보장하기 위해 필수적입니다.

### Latency (지연 시간)
Latency는 데이터 패킷이 송신자(Sender)를 떠나 수신자(Receiver)에 도착하기까지 걸리는 시간입니다.

1.  **구성 요소**:
    -   **Processing Delay**: 패킷을 만들고(Encapsulation), 처리하는 시간 (CPU, OS)
    -   **Transmission Delay**: 패킷을 물리 계층(PHY)으로 보내는 시간 (패킷 크기 / 대역폭)
    -   **Propagation Delay**: 케이블을 통해 신호가 이동하는 시간 (빛의 속도 대비 약 2/3)
    -   **Queueing Delay**: 스위치나 라우터의 큐(Queue)에서 대기하는 시간 (네트워크 혼잡 시 증가)

2.  **측정 방법**:
    -   **Ping (ICMP)**: 가장 간단하지만, OS 스케줄링 등의 영향으로 정확도가 떨어질 수 있습니다.
    -   **Timestamping**: 송신 시점(t1)과 수신 시점(t2)에 정밀한 타임스탬프를 찍어 차이(t2 - t1)를 계산합니다. (PTP 동기화 필요)
    -   **Hardware Timestamping**: NIC 하드웨어에서 직접 타임스탬프를 찍으면 OS 지연을 배제할 수 있습니다.

### Jitter (지터)
Jitter는 Latency의 변동폭(Variation)을 의미합니다. 즉, 데이터가 도착하는 시간의 불규칙성을 나타냅니다.

1.  **원인**:
    -   **네트워크 혼잡 (Congestion)**: 다른 트래픽 때문에 스위치 큐에서 대기하는 시간이 매번 달라집니다.
    -   **OS 스케줄링**: 송수신 프로세스가 CPU를 할당받는 시점이 일정하지 않습니다. (Context Switching)

2.  **영향**:
    -   **제어 불안정**: 로봇 관절 제어 주기가 일정하지 않으면 진동(Oscillation)이나 오버슈트(Overshoot)가 발생할 수 있습니다.
    -   **데이터 손실**: 버퍼(Buffer)가 넘치거나(Overflow) 비어서(Underflow) 데이터가 유실될 수 있습니다.

### 의료 로봇 분야 활용
수술 로봇의 경우, 의사가 조작한 명령이 로봇 팔에 전달되는 시간이 100ms를 넘으면 지연을 느끼게 되고 정밀한 수술이 어려워집니다. 또한, 햅틱 피드백(Haptic Feedback)을 위해서는 1ms 이하의 매우 낮은 Jitter가 요구됩니다.
-   **QoS 설정**: 802.1Qbv(TAS)나 802.1Q(VLAN Priority)를 사용하여 중요 트래픽의 Latency와 Jitter를 최소화해야 합니다.
-   **성능 튜닝**: 리눅스 커널 튜닝(PREEMPT_RT), 네트워크 카드 설정(Interrupt Coalescing), 케이블 길이 최적화 등을 통해 지연 요소를 제거해야 합니다.

## Reference
- [Cisco - Understanding Jitter in Packet Voice Networks](https://www.cisco.com/c/en/us/support/docs/voice/voice-quality/18902-jitter-packet-voice.html)
- [RFC 3393 - IP Packet Delay Variation Metric](https://datatracker.ietf.org/doc/html/rfc3393)
