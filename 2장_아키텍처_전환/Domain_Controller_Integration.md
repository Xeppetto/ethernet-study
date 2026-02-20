# Domain Controller 통합

## 개요
기존 차량 및 로봇 시스템은 수십~수백 개의 개별 ECU(Electronic Control Unit)가 각자의 기능을 수행하는 분산 아키텍처였습니다. 하지만 기능이 고도화됨에 따라 배선 복잡도와 통신 부하가 증가하여, 유사한 기능을 묶어 하나의 고성능 제어기인 'Domain Controller'로 통합하는 추세입니다. 이는 중앙집중형 아키텍처(HPC)로 가는 과도기적인 단계이자, 현실적인 최적화 방안입니다.

### Domain Controller의 개념
Domain Controller는 특정 기능 도메인(Domain) 내의 여러 ECU를 통합 관리하는 제어기입니다. 예를 들어, 파워트레인 도메인, 인포테인먼트 도메인, 바디 도메인 등으로 나누어 각 도메인마다 하나의 강력한 MCU나 AP를 탑재한 컨트롤러를 배치합니다.

#### 주요 특징
- **기능 통합**: 여러 개의 작은 ECU 기능을 하나의 Domain Controller에서 소프트웨어적으로 구현합니다. (예: 엔진 제어, 변속기 제어, 배터리 관리 등을 파워트레인 도메인 컨트롤러 하나로 통합)
- **배선 단순화**: 각 ECU 간의 복잡한 배선을 줄이고, 도메인 간 통신은 고속 백본망(Ethernet)을 통해 이루어집니다.
- **성능 향상**: 고성능 프로세서를 사용하여 복잡한 연산을 빠르게 처리하고, 데이터 공유를 원활하게 합니다.
- **확장성**: 새로운 기능을 추가할 때 하드웨어를 변경하지 않고 소프트웨어 업데이트만으로 가능합니다.

### 의료 로봇 분야 적용 (Medical Domain Controller)
의료 로봇 시스템에서도 유사하게 기능별로 제어기를 통합할 수 있습니다.
1.  **Motion Control Domain**: 로봇 팔의 관절 모터 제어, 힘 센서 처리, 역기구학 연산 등을 담당합니다.
2.  **Vision Domain**: 내시경 카메라 영상 처리, 3D 재구성, AI 병변 인식 등을 담당합니다.
3.  **HMI (Human Machine Interface) Domain**: 의사용 콘솔, 터치스크린 UI, 햅틱 피드백 등을 담당합니다.
4.  **Safety Domain**: 비상 정지, 시스템 상태 감시, 이중화 제어 등 안전 관련 기능을 전담합니다.

### 기술적 과제
Domain Controller 통합을 위해서는 다음과 같은 기술적 과제를 해결해야 합니다.
- **실시간성 보장**: 여러 기능을 하나의 프로세서에서 동시에 수행하므로, 실시간 운영체제(RTOS)와 멀티코어 프로세싱 기술이 중요합니다.
- **열 관리**: 고성능 프로세서 사용으로 인한 발열 문제를 해결해야 합니다.
- **보안 강화**: 외부 네트워크와 연결되는 도메인 컨트롤러는 해킹 위험에 노출될 수 있으므로, 하드웨어 보안 모듈(HSM)이나 시큐어 부팅 기술을 적용해야 합니다.
- **표준화**: 서로 다른 제조사의 부품을 통합하기 위해 AUTOSAR나 ROS 2와 같은 표준 소프트웨어 플랫폼을 사용해야 합니다.

## Reference
- [AUTOSAR - Adaptive Platform](https://www.autosar.org/standards/adaptive-platform/)
- [NXP - Domain Controller](https://www.nxp.com/applications/automotive/domain-controller:DOMAIN-CONTROLLER)
