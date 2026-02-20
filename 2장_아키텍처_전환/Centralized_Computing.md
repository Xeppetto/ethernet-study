# 중앙집중형 컴퓨팅 (HPC / HPVC)

## 개요
차량 및 의료 로봇 시스템이 복잡해지면서, 기존의 수십 개의 소형 ECU(Electronic Control Unit)들이 각자 맡은 기능을 수행하던 분산 구조에서, 고성능 컴퓨터 한두 대가 모든 연산을 통합하여 처리하는 '중앙집중형 아키텍처(Centralized Architecture)'로 변화하고 있습니다. 이를 가능하게 하는 핵심 기술이 바로 HPC(High Performance Computing) 또는 HPVC(High Performance Vehicle Computer)입니다.

### 중앙집중형 아키텍처의 필요성
과거에는 각 기능(엔진 제어, 브레이크 제어, 인포테인먼트 등)마다 별도의 ECU가 존재했습니다. 하지만 자율주행, AI, V2X(Vehicle to Everything) 통신 등 고도화된 기능이 요구되면서, 개별 ECU의 성능 한계와 복잡한 통신 구조가 문제가 되었습니다. 이에 따라, 고성능 프로세서(AP, GPU, NPU 등)를 탑재한 중앙 컴퓨터가 전체 시스템을 통합 제어하는 방식이 도입되었습니다.

#### 주요 특징
- **고성능 연산**: 수십 TOPS(Tera Operations Per Second) 이상의 연산 능력을 가진 프로세서를 사용하여, 복잡한 알고리즘과 AI 모델을 실시간으로 처리합니다.
- **소프트웨어 중심**: 하드웨어에 종속되지 않고 소프트웨어 업데이트(OTA)를 통해 기능을 개선하거나 추가할 수 있습니다. (SDV, Software Defined Vehicle)
- **가상화 기술**: 하나의 강력한 하드웨어 위에서 여러 운영체제(OS)나 애플리케이션을 동시에 실행하기 위해 Hypervisor 기술이 필수적으로 사용됩니다. (예: 실시간 제어용 RTOS와 사용자 인터페이스용 Linux/Android 동시 구동)
- **초고속 통신**: 대용량 데이터를 처리하기 위해 PCIe, 10Gbps Ethernet 등 고속 인터페이스를 지원합니다.

### HPC / HPVC의 구성 요소
- **AP (Application Processor)**: 고성능 CPU 코어를 내장하여 OS 및 애플리케이션 실행을 담당합니다.
- **GPU / NPU**: 딥러닝 추론, 영상 처리 등 병렬 연산 가속을 담당합니다.
- **Safety Controller**: 기능 안전(ISO 26262)을 만족시키기 위해 별도의 MCU(Micro Controller Unit)가 탑재되어 시스템 상태를 감시하고 비상 상황 시 안전하게 제어권을 가져옵니다.
- **Ethernet Switch**: 외부 센서 및 Zone Controller들과의 통신을 위한 고속 스위치가 내장되기도 합니다.

### 의료 로봇 분야 적용
의료 로봇, 특히 수술 로봇이나 AI 진단 로봇에서는 실시간 영상 분석과 정밀 제어가 동시에 이루어져야 합니다. HPC 기반의 중앙집중형 아키텍처를 도입하면 다음과 같은 이점이 있습니다.

- **AI 기반 기능 구현**: 수술 중 실시간 장기 인식, 출혈 감지, 경로 안내 등 고성능 AI 기능을 로봇 자체(Edge)에서 처리할 수 있습니다.
- **데이터 융합 (Sensor Fusion)**: 여러 센서(카메라, 레이더, 힘 센서 등) 데이터를 한곳에서 모아 처리하므로 더 정확하고 빠른 판단이 가능합니다.
- **원격 제어 및 클라우드 연동**: 5G/6G 통신 모듈과 결합하여 원격 수술이나 클라우드 기반 데이터 분석 서비스를 제공할 수 있습니다.

## Reference
- [NVIDIA DRIVE - Autonomous Vehicle Development Platform](https://www.nvidia.com/en-us/self-driving-cars/drive-platform/)
- [Qualcomm Snapdragon Digital Chassis](https://www.qualcomm.com/products/automotive/snapdragon-digital-chassis)
