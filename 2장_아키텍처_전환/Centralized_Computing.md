# 중앙집중형 컴퓨팅 (HPC / HPVC)

## 개요
차량 및 의료 로봇 시스템이 복잡해지면서 수십 개의 소형 ECU들이 분산 처리하던 구조에서, 고성능 컴퓨터(HPC, High Performance Computer) 한두 대가 모든 연산을 통합 처리하는 방식으로 진화하고 있습니다. 이를 통해 AI 기능 통합, OTA 업데이트, 소프트웨어 중심 설계가 가능해집니다.

---

## 아키텍처 발전 단계

```
1세대: 분산 ECU (2000년대)
  ECU #1 ─CAN─ ECU #2 ─CAN─ ECU #3 ─CAN─ ... ECU #100
  각 ECU: 단일 기능, 4~32MHz MCU

2세대: Domain Controller (2010년대)
  파워트레인 DC ─Ethernet─ 섀시 DC ─Ethernet─ 인포테인먼트 DC
  각 DC: 다기능, 수백MHz MCU

3세대: 중앙집중형 HPC (2020년대~)
  Zone ECU ─Ethernet─ HPC ─Ethernet─ Zone ECU
               │
              Cloud
  HPC: AI/GPU 수십 TOPS, 멀티코어 CPU
```

---

## HPC 내부 구성 요소

```
┌──────────────────────────────────────────────────────────────┐
│                    HPC (High Performance Computer)            │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────────┐ │
│  │  AP 클러스터 │  │  AI 가속기 │  │  Safety MCU             │ │
│  │  (CPU)     │  │  (GPU/NPU) │  │  (ASIL-D 인증)          │ │
│  │  멀티코어   │  │  ~100 TOPS │  │  시스템 감시             │ │
│  │  Linux/RTOS│  │            │  │  비상 제어               │ │
│  └────────────┘  └────────────┘  └─────────────────────────┘ │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────────┐ │
│  │  Hypervisor │  │  Ethernet  │  │  HSM                    │ │
│  │  (가상화)  │  │  Switch    │  │  (Hardware Security)    │ │
│  │  RTOS 격리 │  │  TSN 지원  │  │  SecureBoot, 암호화     │ │
│  └────────────┘  └────────────┘  └─────────────────────────┘ │
│                                                              │
│  인터페이스: PCIe Gen4, 10GbE, USB3, CAN FD                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 하이퍼바이저(Hypervisor)와 가상화

하나의 HPC에서 안전 제어(RTOS)와 사용자 서비스(Linux)를 동시에 격리 실행합니다.

```
┌─────────────────────────────────────────────────┐
│  HPC 하드웨어 (멀티코어 ARM/x86)                 │
├─────────────────────────────────────────────────┤
│  Type-1 Hypervisor (예: Xen, QNX Hypervisor)   │
├────────────────────┬────────────────────────────┤
│  Guest OS #1       │  Guest OS #2               │
│  RTOS (QNX/FreeRTOS│  Linux (Ubuntu/Yocto)      │
│  안전 제어          │  인포테인먼트, AI, OTA      │
│  실시간 응답 < 1ms  │  일반 애플리케이션          │
│  ASIL-B/D 인증     │  비안전 기능               │
└────────────────────┴────────────────────────────┘

핵심: 두 Guest OS는 메모리/인터럽트 완전 격리
→ Linux의 버그/크래시가 RTOS 제어에 영향 없음
```

### 가상화 대안: 컨테이너
```
Linux 기반 HPC (안전 기능 불필요한 부분):
  ┌─────────────────────────────────────┐
  │  Linux Kernel                       │
  ├────────────────────────────────────-┤
  │ Container Runtime (Docker/Podman)  │
  ├─────────────┬──────────────────────┤
  │ Container A │ Container B          │
  │ (AI 추론)   │ (OTA 관리)           │
  │ Python/     │ Node.js              │
  │ TensorFlow  │                      │
  └─────────────┴──────────────────────┘

컨테이너 장점: 빠른 배포, 격리, CI/CD 통합
단점: 하이퍼바이저 대비 격리 수준 낮음 (실시간성 미보장)
```

---

## AI/머신러닝 통합

HPC에 탑재된 NPU/GPU로 엣지 AI 추론을 실행합니다.

```
센서 데이터
(카메라, 라이다, 레이더)
      │
      ▼
┌─────────────────────────────────────────────┐
│  HPC NPU/GPU                                │
│                                             │
│  ① 객체 인식 (YOLOv8 등)        ~20ms 추론  │
│  ② 공간 인식 (Depth Estimation)             │
│  ③ 이상 감지 (Anomaly Detection)            │
│  ④ 동작 계획 (Motion Planning)             │
│                                             │
│  성능 요구: > 30 TOPS (자율주행 기준)         │
└─────────────────────────────────────────────┘
      │
      ▼
Zone ECU → 모터/액추에이터 제어
```

**의료 로봇 AI 활용 사례:**
- 수술 중 실시간 조직 인식 (종양 vs 정상 조직 경계)
- 수술 도구 위치 추적 및 안전 경계 모니터링
- 출혈 감지 및 즉각 알림
- 로봇 암 충돌 예측 및 회피

---

## 중앙집중형 HPC의 도전 과제

### 기능 안전 (Functional Safety)
```
분산 ECU 시대:
  ECU 하나 고장 → 해당 기능만 영향
  ASIL-D 달성: ECU 내부 이중화

HPC 시대:
  HPC 고장 → 모든 기능 영향 (Single Point of Failure)

해결 방법:
  ① HPC 이중화 (Dual HPC 구성)
  ② Safety MCU 별도 탑재 (독립 Watchdog)
  ③ Fail-Safe 모드: Safety MCU가 최소 기능 유지

  HPC #1 (주) ──────────────────────► Zone ECU
                    │
  HPC #2 (예비) ───┘ (Failover 수백ms 내)
```

### 열 관리
```
소비 전력 비교:
  기존 ECU 100개 × 평균 2W = 200W
  HPC 1개: 50~200W (동일하거나 적은 전력)

  하지만 HPC는 한 곳에 집중 → 냉각 설계 중요

냉각 방법:
  - 히트싱크 + 팬 냉각 (공냉)
  - 액랭 (고성능 HPC)
  - 열전달 패드 + 차량 냉각 시스템 연계
```

---

## 의료 로봇 HPC 구성 예시

```
수술 로봇 HPC 구성

┌─────────────────────────────────────────────────────┐
│                    Main HPC                          │
│  ┌──────────────────────────────────────────────┐   │
│  │ NVIDIA Orin / Qualcomm SA8295               │   │
│  │ CPU: 12코어 ARM A78AE                        │   │
│  │ GPU/DLA: 275 TOPS                            │   │
│  │ Memory: 64GB LPDDR5                          │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │ Safety MCU (TI TDA4VM / Renesas R-Car S4)   │   │
│  │ ASIL-D 인증, 독립 전원, BIST 자가진단        │   │
│  └──────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────┐   │
│  │ Ethernet Switch (TSN)                        │   │
│  │ 포트: 8× 1GbE Zone, 2× 10GbE Cloud/외부     │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

소프트웨어:
  - Hypervisor: QNX Hypervisor 또는 ACRN
  - 안전 OS: QNX Neutrino RTOS
  - 비안전 OS: Ubuntu 22.04 LTS / Yocto
  - 미들웨어: ROS 2 + AUTOSAR Adaptive
```

---

## Reference
- [NVIDIA DRIVE Orin - Autonomous Vehicle SoC](https://www.nvidia.com/en-us/self-driving-cars/drive-platform/)
- [Qualcomm Snapdragon Digital Chassis](https://www.qualcomm.com/products/automotive/snapdragon-digital-chassis)
- [ACRN Hypervisor - Open Source for Automotive](https://projectacrn.org/)
- [NXP S32G - Vehicle Network Processor](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/s32g-vehicle-network-processors:S32G)
