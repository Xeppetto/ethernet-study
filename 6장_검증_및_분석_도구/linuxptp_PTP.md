# linuxptp PTP 설정

## 개요
Linux 환경에서 TSN의 핵심 기술인 시간 동기화(IEEE 802.1AS, IEEE 1588)를 구현하기 위해 가장 널리 사용되는 오픈 소스 프로젝트가 바로 'linuxptp'입니다. 이 도구는 `ptp4l`과 `phc2sys`라는 두 가지 주요 프로그램으로 구성되어 있으며, 이를 적절히 설정하고 운영하는 방법을 익히는 것은 정밀한 로봇 제어 시스템을 구축하는 데 필수적입니다.

### ptp4l (PTP for Linux)
`ptp4l`은 네트워크 인터페이스(NIC)의 하드웨어 타임스탬핑(Hardware Timestamping) 기능을 사용하여, 외부 Grandmaster Clock(GM)과 시스템의 PTP 하드웨어 시계(PHC, PTP Hardware Clock)를 동기화하는 데몬입니다.

#### 주요 설정 옵션 (/etc/linuxptp/ptp4l.conf)
-   `interface`: PTP를 사용할 네트워크 인터페이스 이름 (예: eth0)
-   `delay_mechanism`: 지연 측정 방식 (E2E: End-to-End, P2P: Peer-to-Peer). 802.1AS는 P2P 방식을 사용합니다.
-   `transportSpecific`: 전송 계층 종류 (1: IEEE 802.1AS, 0: IEEE 1588)
-   `priority1`, `priority2`: BMCA(Best Master Clock Algorithm)에서 GM 선출 우선순위를 결정합니다. 값이 낮을수록 우선순위가 높습니다.

### phc2sys (PHC to System Clock Synchronization)
`phc2sys`는 `ptp4l`에 의해 동기화된 PHC(NIC 시계)를 리눅스 시스템 커널 시계(System Clock, CLOCK_REALTIME)에 동기화시키는 프로그램입니다. 애플리케이션(Application)은 주로 시스템 시계를 참조하므로, 이 과정이 반드시 필요합니다.

#### 실행 예시
```bash
# eth0 인터페이스의 PHC를 시스템 시계(CLOCK_REALTIME)에 동기화 (-s: source, -c: destination)
sudo phc2sys -s eth0 -c CLOCK_REALTIME -O 0 -m
```
옵션 설명:
-   `-s`: 소스 시계 (Master)
-   `-c`: 타겟 시계 (Slave)
-   `-O`: 오프셋(Offset) 보정 값 (초 단위)
-   `-m`: 로그 메시지 출력

### 의료 로봇 분야 활용
로봇의 제어 소프트웨어(ROS 2 등)는 센서 데이터의 타임스탬프를 기준으로 동작합니다. `linuxptp`를 사용하여 모든 제어기와 센서의 시간을 마이크로초(µs) 단위로 동기화하면, 다음과 같은 효과를 얻을 수 있습니다.
-   **정밀한 궤적 제어**: 여러 관절이 동시에 움직여야 하는 경우, 각 관절의 구동 시점을 정확히 일치시킬 수 있습니다.
-   **데이터 융합**: 서로 다른 센서(카메라, LIDAR, IMU)에서 들어온 데이터의 시점을 정확히 맞춰(Align), 왜곡 없는 3D 환경 맵을 생성할 수 있습니다.
-   **이벤트 분석**: 시스템 오류 발생 시, 로그 파일의 타임스탬프를 비교하여 정확한 인과 관계를 분석할 수 있습니다.

## Reference
- [LinuxPTP Project](http://linuxptp.sourceforge.net/)
- [Red Hat Enterprise Linux - Configuring PTP Using ptp4l](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-configuring_ptp_using_ptp4l)
