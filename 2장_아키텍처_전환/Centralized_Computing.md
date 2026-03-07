# 중앙집중형 컴퓨팅 (HPC / HPVC)

## 개요
차량 및 의료 로봇 시스템이 복잡해지면서 수십 개의 소형 ECU들이 분산 처리하던 구조에서, 고성능 컴퓨터(HPC, High Performance Computer) 한두 대가 모든 연산을 통합 처리하는 방식으로 진화하고 있습니다. 이를 통해 AI 기능 통합, OTA 업데이트, 소프트웨어 중심 설계가 가능해집니다.

---

## 아키텍처 발전 단계

```
1세대: 분산 ECU (2000년대)
  ECU #1 ─CAN─ ECU #2 ─CAN─ ECU #3 ─CAN─ ... ECU #100
  각 ECU: 단일 기능, 4~32MHz MCU, 16~512KB Flash

2세대: Domain Controller (2010년대)
  파워트레인 DC ─Ethernet─ 섀시 DC ─Ethernet─ 인포테인먼트 DC
  각 DC: 다기능 통합, 수백MHz AP

3세대: 중앙집중형 HPC (2020년대~)
  Zone ECU ─TSN Ethernet─ HPC ─10GbE─ Cloud
                │
           AI/GPU/NPU (100+ TOPS)
  HPC: 멀티코어 CPU + GPU/NPU 통합 SoC
```

---

## 주요 HPC SoC 비교

| SoC | 제조사 | CPU | AI 성능 | 메모리 | 주요 용도 |
|-----|--------|-----|---------|--------|----------|
| Drive Orin X | NVIDIA | 12× Cortex-A78AE | 254 TOPS | 64GB LPDDR5 | 자율주행 L4/L5 |
| Drive Thor | NVIDIA | 다중 클러스터 | 2000 TOPS | - | 차세대 SDV |
| Snapdragon SA8295P | Qualcomm | 8× Kryo Gold | 30 TOPS | 16GB LPDDR5 | Cockpit/IVI |
| TDA4VM (Jacinto 7) | Texas Instruments | 2× Cortex-A72 + MCU | 8 TOPS | 4GB LPDDR4 | ADAS ASIL-D |
| R-Car S4 | Renesas | 8× Cortex-A55 | 12 TOPS | 16GB LPDDR5 | 게이트웨이/HPC |
| S32G3 | NXP | 4× Cortex-A53 + 3× M7 | - | 4GB LPDDR4 | 네트워크 HPC |
| i.MX 95 | NXP | 6× Cortex-A55 | 4 TOPS (NPU) | 8GB LPDDR5 | 의료/로봇 |

---

## HPC 내부 구성 요소

```
┌──────────────────────────────────────────────────────────────┐
│                    HPC (High Performance Computer)            │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  AP 클러스터  │  │  AI 가속기   │  │  Safety MCU        │  │
│  │  (멀티코어)  │  │  (GPU/NPU)   │  │  (ASIL-D 인증)     │  │
│  │  4~12 Cores  │  │  8~254 TOPS  │  │  독립 전원         │  │
│  │  Linux/QNX   │  │  TensorRT    │  │  Watchdog/BIST     │  │
│  └──────────────┘  └──────────────┘  └────────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  Hypervisor   │  │  TSN Switch  │  │  HSM               │  │
│  │  (가상화)    │  │  (Ethernet)  │  │  (보안 모듈)       │  │
│  │  RTOS 격리   │  │  8× 1GbE     │  │  SecureBoot        │  │
│  │  메모리 분리  │  │  2× 10GbE   │  │  AES/ECC 엔진      │  │
│  └──────────────┘  └──────────────┘  └────────────────────┘  │
│                                                              │
│  인터페이스: PCIe Gen4×8, 10GbE (Cloud), CAN FD×4, USB 3.1   │
│  스토리지: NVMe SSD 256GB (OS + 모델), eMMC 32GB (백업)       │
└──────────────────────────────────────────────────────────────┘
```

---

## 하이퍼바이저(Hypervisor)와 가상화

하나의 HPC에서 안전 제어(RTOS)와 사용자 서비스(Linux)를 동시에 격리 실행합니다.

### Hypervisor 유형 비교

| 구분 | Type-1 (Bare-metal) | Type-2 (Hosted) | 컨테이너 |
|------|---------------------|-----------------|---------|
| 실행 위치 | 하드웨어 직접 | Host OS 위 | OS 커널 공유 |
| 격리 수준 | 완전 (하드웨어 MMU) | 부분 | 프로세스 격리 |
| 실시간성 | 우수 (RTOS 직접 실행) | 제한적 | OS 스케줄러 의존 |
| 안전 인증 | ASIL-B/D 가능 | 불가 | 불가 |
| 예시 | QNX Hypervisor, Xen, ACRN | KVM, VirtualBox | Docker, Podman |
| 부팅 속도 | 느림 | 느림 | 빠름 (수백ms) |
| 오버헤드 | 2~5% | 10~20% | 1~2% |

### Type-1 Hypervisor 파티셔닝 구조
```
┌─────────────────────────────────────────────────┐
│  HPC 하드웨어 (ARM Cortex-A78AE 12코어, ARM64)   │
├─────────────────────────────────────────────────┤
│  Type-1 Hypervisor (QNX Hypervisor / ACRN)      │
│  ARM Stage-2 MMU 기반 메모리 격리               │
├──────────────────────┬──────────────────────────┤
│  Guest OS A          │  Guest OS B              │
│  QNX Neutrino RTOS   │  Ubuntu 22.04 LTS        │
│  Core 0,1,2 전용     │  Core 3,4,5,6,7 할당     │
│  ─────────────────── │  ─────────────────────── │
│  안전 제어 서비스     │  AI 추론 서비스           │
│  관절 PID 제어        │  TensorRT 모델           │
│  힘/토크 제한         │  OTA 관리 (ara::ucm)     │
│  Emergency Stop       │  Cloud 연동 (MQTT TLS)   │
│  응답: < 1ms          │  ROS 2 노드              │
│  ASIL-B 격리          │  일반 서비스             │
├──────────────────────┴──────────────────────────┤
│  공유 메모리 통신 (격리된 IPC 채널, 512MB 영역)  │
└─────────────────────────────────────────────────┘

메모리 격리 보장:
  Guest A: 4GB 전용 (QNX는 16GB 이상 필요 없음)
  Guest B: 16GB 전용 (AI 모델 로딩용)
  Shared: 512MB (제어 명령/상태 교환용)
```

### 컨테이너 기반 서비스 배포
```bash
# 의료 로봇 AI 서비스 컨테이너화 예시
docker run -d \
  --name surgical-ai \
  --runtime=nvidia \                    # NVIDIA GPU/DLA 접근
  --cpuset-cpus="3,4,5" \              # CPU 코어 고정
  --memory="8g" \                       # 메모리 제한
  --device /dev/ttyUSB0 \              # 수술 도구 인터페이스
  --network=host \                      # Ethernet DDS 직접 접근
  -v /opt/models:/opt/models:ro \      # AI 모델 마운트
  registry.hospital.com/surgical-ai:v2.1.0

# 롤링 업데이트 (OTA 완료 후)
docker pull registry.hospital.com/surgical-ai:v2.2.0
docker stop surgical-ai
docker run -d ... surgical-ai:v2.2.0
```

---

## AI/머신러닝 통합

HPC에 탑재된 NPU/GPU로 엣지 AI 추론을 실행합니다.

### AI 추론 파이프라인
```
센서 데이터 입력
(4K 카메라 × 3, 초음파, 힘 센서)
        │
        ▼
┌─────────────────────────────────────────────┐
│  전처리 (CUDA/OpenCV, GPU)                  │
│  리사이즈, 정규화, 배치 구성                 │
│  처리 시간: 2~5ms                           │
└──────────────────┬──────────────────────────┘
                   │
        ▼
┌─────────────────────────────────────────────┐
│  HPC NPU/GPU 추론                           │
│  ① YOLOv8: 수술 도구/조직 객체 인식  15ms  │
│  ② DepthNet: 내시경 3D 깊이 추정     20ms  │
│  ③ AnomalyNet: 출혈/조직 이상 감지   10ms  │
│  ④ PathPlanner: 로봇 팔 동작 계획    5ms   │
│  총 추론 시간: < 50ms (20 FPS 목표)         │
│  요구 성능: > 50 TOPS                       │
└──────────────────┬──────────────────────────┘
                   │
        ▼
Zone ECU → 모터/액추에이터 제어 (1ms 주기)
```

### AI 프레임워크 비교

| 프레임워크 | 목표 플랫폼 | 최적화 방식 | 의료 로봇 활용 |
|----------|-----------|-----------|----------------|
| TensorRT | NVIDIA GPU/DLA | INT8/FP16 양자화 | 최고 추론 속도 |
| OpenVINO | Intel VPU/iGPU | IR 모델 최적화 | x86 기반 HPC |
| ONNX Runtime | 범용 | 다중 백엔드 | 이식성 필요 시 |
| TFLite | ARM NPU/MCU | 8-bit 양자화 | Zone ECU 엣지 AI |
| PyTorch Mobile | ARM CPU | 동적 그래프 | 프로토타이핑 |

---

## CPU 코어 분리 및 실시간성 보장

```bash
# Linux PREEMPT_RT 패치 + CPU isolation
# /etc/default/grub 커널 파라미터
GRUB_CMDLINE_LINUX="isolcpus=0,1 nohz_full=0,1 rcu_nocbs=0,1"

# 실시간 프로세스 Core 0,1 고정 + FIFO 스케줄링
sudo taskset -c 0,1 ./realtime_joint_control &
sudo chrt -f 90 -p $(pgrep realtime_joint_control)

# 메모리 페이지 락 (페이지 폴트 방지)
mlockall(MCL_CURRENT | MCL_FUTURE);

# IRQ Affinity: 네트워크 인터럽트를 비실시간 코어로 분리
echo 0xF0 > /proc/irq/$(cat /proc/interrupts | grep eth0 | awk '{print $1}')/smp_affinity
```

```
코어 배분 전략 (12코어 예시):
  Core 0, 1   : Safety MCU 통신 전용 (IRQ 전담, FIFO priority 90)
  Core 2, 3   : 관절 PID 제어 (1ms 주기, FIFO priority 80)
  Core 4, 5   : ROS 2 실시간 노드 (5ms 주기, FIFO priority 60)
  Core 6, 7   : AI 추론 (TensorRT, 50ms 주기)
  Core 8, 9   : OTA / 클라우드 연동 (일반 CFS 스케줄링)
  Core 10, 11 : OS 커널 + 드라이버 + 시스템 서비스
```

---

## 중앙집중형 HPC의 도전 과제

### 기능 안전 (단일 실패 지점 해결)
```
분산 ECU 구조:
  ECU #1 고장 → 해당 기능만 중단 (다른 ECU 정상)

HPC 집중 구조 위험:
  HPC 고장 → 전체 기능 마비 (Single Point of Failure)

해결 방법:

방법 1: 이중화 HPC (Lockstep 방식)
  HPC #1 (주) ── 결과 비교 ── HPC #2 (감시)
    둘 다 동일 입력 처리 → 출력 불일치 시 안전 상태 전환

방법 2: Safety MCU 독립 운용
  HPC ── Ethernet/CAN ── Safety MCU (ASIL-D)
  HPC 비정상 감지 시 Safety MCU가 최소 기능 유지
  (비상 정지, 브레이크 고정, 안전 위치 복귀)

방법 3: Watchdog 계층화
  Software WDG → OS WDG → External HW WDG (독립 IC)
  HPC 응답 없음 시 순차 리셋 → 최종 하드웨어 리셋
```

### 열 관리
```
소비 전력 비교:
  기존 ECU 100개 × 평균 2W = 200W (분산)
  HPC 1개: 50~200W (집중)

  동일 전력이지만 HPC는 한 점에 집중 → 열 밀도 높음

냉각 방법:
  1. 공냉: 히트싱크 + 팬 (< 50W HPC)
  2. 수냉: 냉각판 + 쿨런트 펌프 (50~150W HPC)
  3. 차량 냉각수 연동: 엔진 냉각 시스템 활용 (> 150W)
  4. 의료 로봇: 능동 팬 + 열전소자 (소음 최소화 필요)

DVFS (동적 전압/주파수 스케일링):
  부하 낮음: 1.2GHz @ 0.9V → 30W
  부하 높음: 2.4GHz @ 1.2V → 80W
  열 임계치 (95°C): 주파수 강제 감소 (Thermal Throttling)
```

---

## 의료 로봇 HPC 구성 예시

```
수술 로봇 Main HPC 구성

┌─────────────────────────────────────────────────────────┐
│                    Main HPC                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │ NVIDIA Orin NX 16GB (또는 Qualcomm SA8295P)     │   │
│  │ CPU: 8× Cortex-A78AE @ 2.2GHz                  │   │
│  │ GPU: 1024 CUDA cores, 32 DLA TOPS               │   │
│  │ Memory: 16GB LPDDR5 (CPU+GPU 공유)              │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Safety MCU (TI TDA4VM / Renesas R-Car S4)       │   │
│  │ ASIL-D 인증, 독립 전원 (배터리 백업)             │   │
│  │ BIST 자가진단, 주기적 Heartbeat → HPC 감시       │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │ TSN Ethernet Switch (NXP SJA1110 또는 Marvell)  │   │
│  │ 포트: 8× 1GbE (Zone ECU), 2× 2.5GbE (HPC ↔)   │   │
│  │ IEEE 802.1Qbv TAS, 802.1AS gPTP 지원           │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘

소프트웨어 스택:
  Layer 3: Surgical AI (TensorRT), ROS 2 (Humble), MQTT TLS
  Layer 2: AUTOSAR Adaptive (ara::com SOME/IP), ara::ucm OTA
  Layer 1: QNX Hypervisor + Guest: QNX RTOS + Ubuntu 22.04
  Layer 0: UEFI Secure Boot → U-Boot → Hypervisor
```

### IEC 62304 / ISO 26262 규제 매핑

| HPC 컴포넌트 | 안전 등급 | 적용 표준 | 인증 요구사항 |
|-------------|---------|----------|-------------|
| Safety MCU (E-Stop) | ASIL-D / Class C | ISO 26262 + IEC 62304 | 독립 감사, MC/DC 커버리지 |
| 관절 제어 소프트웨어 | ASIL-C / Class C | ISO 26262 + IEC 62304 | 형식 검증, E2E Protection |
| AI 추론 엔진 | Class B | IEC 62304 | 알고리즘 검증, 성능 시험 |
| OTA 관리 | Class B | IEC 62304 + R156 | 롤백 기능, 서명 검증 |
| 클라우드 연동 | Class A | IEC 62304 | 기본 기능 시험 |

---

## Reference
- [NVIDIA DRIVE Orin Platform](https://www.nvidia.com/en-us/self-driving-cars/drive-platform/)
- [Qualcomm Snapdragon Digital Chassis](https://www.qualcomm.com/products/automotive/snapdragon-digital-chassis)
- [ACRN Hypervisor for Automotive](https://projectacrn.org/)
- [QNX Hypervisor for Safety](https://blackberry.qnx.com/en/products/hypervisor)
- [NXP S32G Vehicle Network Processor](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/s32g-vehicle-network-processors:S32G)
- [Linux PREEMPT_RT Patch](https://wiki.linuxfoundation.org/realtime/start)
- [ELISA Project - Safety Linux](https://elisa.tech/)
