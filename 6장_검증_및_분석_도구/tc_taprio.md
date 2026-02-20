# tc taprio

## 개요
Linux Traffic Control(TC)의 `taprio` qdisc(Queueing Discipline)는 소프트웨어적으로 IEEE 802.1Qbv(Time Aware Shaper, TAS)를 구현하는 기능입니다. TSN 하드웨어 스위치 없이도 일반적인 리눅스 시스템에서 TAS의 동작 원리를 이해하고 실험해 볼 수 있는 강력한 도구이며, 실제 하드웨어 오프로드(Offload)를 지원하는 NIC를 사용하면 정밀한 스케줄링을 구현할 수 있습니다.

### tc taprio 설정 방법
`taprio`를 사용하기 위해서는 `tc` 명령어와 루트 권한이 필요합니다. 기본적으로 네트워크 인터페이스(NIC)에 qdisc를 추가하고, 스케줄(Schedule)을 정의합니다.

#### 기본 명령어 구조
```bash
tc qdisc add dev eth0 parent root handle 100 taprio \
    num_tc 3 \
    map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 \
    queues 1@0 1@1 2@2 \
    base-time 1554432000000000000 \
    sched-entry S 01 300000 \
    sched-entry S 02 300000 \
    sched-entry S 04 400000 \
    flags 0x2 \
    clockid CLOCK_TAI
```

#### 주요 파라미터 설명
-   `num_tc`: 트래픽 클래스(Traffic Class)의 개수 (예: 3개 - TC0, TC1, TC2)
-   `map`: 우선순위(Priority 0~15)를 트래픽 클래스(TC)에 매핑 (예: Priority 3 -> TC0)
-   `queues`: 각 트래픽 클래스가 사용할 큐(Queue)의 개수와 범위 지정 (예: 1@0 -> 1개 큐 사용, 0번 큐부터 시작)
-   `base-time`: 스케줄이 시작되는 기준 시각 (나노초 단위, PTP 시간 등)
-   `sched-entry`: 스케줄 엔트리 (Gate Operation)
    -   `S`: Set Gate Operation (게이트 상태 설정)
    -   `<mask>`: 16진수 비트마스크로 어떤 게이트(TC)를 열지(Open) 결정 (예: 01 -> TC0 Open, 02 -> TC1 Open)
    -   `<interval>`: 해당 상태 유지 시간 (나노초 단위)
-   `flags`: 동작 모드 (예: 0x2 -> TXTIME_OFFLOAD)
-   `clockid`: 스케줄링에 사용할 시계 종류 (CLOCK_TAI 등)

### 의료 로봇 분야 활용
로봇 제어 시스템에서 `taprio`를 사용하면 다음과 같은 효과를 얻을 수 있습니다.
-   **실시간 트래픽 보장**: 제어 주기(Control Loop)에 맞춰 정확한 시간에 제어 명령(Control Command)을 전송하도록 스케줄링하여, 지터(Jitter)를 최소화할 수 있습니다.
-   **트래픽 격리**: 비디오 스트리밍이나 로그 전송과 같은 대용량 트래픽이 제어 트래픽에 영향을 주지 않도록, 시간적으로 분리(Time Slicing)할 수 있습니다.
-   **하드웨어 검증 전 시뮬레이션**: 고가의 TSN 스위치 장비 없이도 리눅스 PC 2대를 연결하여 TAS 기능을 테스트하고 애플리케이션의 동작을 검증할 수 있습니다.

## Reference
- [Linux Kernel Networking - Traffic Control (tc-taprio)](https://man7.org/linux/man-pages/man8/tc-taprio.8.html)
- [OpenAvnu - taprio example](https://github.com/Avnu/OpenAvnu/blob/master/daemons/mrpd/tests/simple/taprio_setup.sh)
