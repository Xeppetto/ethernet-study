# Domain Controller 통합

## 개요
기존의 수십~수백 개 개별 ECU 분산 아키텍처에서, 유사한 기능을 묶어 고성능 Domain Controller로 통합하는 추세입니다. 이는 완전한 중앙집중형 HPC로 가는 과도기적 단계이자, 현실적이고 실용적인 최적화 방안입니다.

---

## 아키텍처 진화 단계

```
1단계: 분산 ECU (2000년대)
  엔진 ECU, 변속기 ECU, ABS ECU, 에어백 ECU ... (각각 독립)
  통신: CAN 버스 (2Mbps)

2단계: Domain Controller 통합 (2015~현재)
  파워트레인 DC + 샤시 DC + ADAS DC + 인포테인먼트 DC
  통신: CAN (내부) + Ethernet 100BASE-T1 (도메인 간)

3단계: 완전 중앙집중형 HPC (2025년~)
  1~2개 HPC + Zone ECU
  통신: TSN Ethernet 백본 (1Gbps+)
```

---

## Domain Controller 구성 예시

### 의료 로봇 Domain Controller 구조
```
┌─────────────────────────────────────────────────────────────┐
│                     의료 로봇 시스템                          │
│                                                             │
│  ┌──────────────────┐  Ethernet  ┌──────────────────────┐   │
│  │ Motion Control DC │◄──────────►│  Vision Domain DC    │   │
│  │                  │  (1Gbps)   │                      │   │
│  │ • 관절 모터 제어  │            │ • 내시경 카메라 처리  │   │
│  │ • 역기구학 연산   │            │ • 3D 재구성           │   │
│  │ • 힘/토크 제어    │            │ • AI 병변 인식        │   │
│  │ • 충돌 감지       │            │ • 조명 제어           │   │
│  └──────────────────┘            └──────────────────────┘   │
│           │                              │                   │
│           └─────── Ethernet ─────────────┘                   │
│                         │                                    │
│  ┌──────────────────┐  Ethernet  ┌──────────────────────┐   │
│  │   HMI Domain DC   │◄──────────►│   Safety Domain DC   │   │
│  │                  │            │                      │   │
│  │ • 의사용 콘솔     │            │ • 비상 정지 (E-Stop)  │   │
│  │ • 햅틱 피드백     │            │ • 시스템 상태 감시    │   │
│  │ • 수술 영상 표시  │            │ • 이중화 제어         │   │
│  │ • UI 관리         │            │ • ASIL-D 인증        │   │
│  └──────────────────┘            └──────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Domain Controller 내부 구조

```
Domain Controller 하드웨어:
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │ AP (CPU)   │  │ Safety MCU │  │ Memory            │   │
│  │ 멀티코어   │  │ ASIL-B/D   │  │ LPDDR4/5          │   │
│  │800MHz~4GHz │  │ Watchdog   │  │ 4~16 GB           │   │
│  └────────────┘  └────────────┘  └──────────────────┘   │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │ CAN FD     │  │ Ethernet   │  │ HSM               │   │
│  │ 인터페이스  │  │ PHY        │  │ (보안 모듈)       │   │
│  │ 4~8채널    │  │ 100BASE-T1 │  │ SecureBoot        │   │
│  └────────────┘  └────────────┘  └──────────────────┘   │
│                                                          │
│  스토리지: eMMC 32~128GB (OS + 데이터)                   │
└──────────────────────────────────────────────────────────┘

소프트웨어 스택:
┌──────────────────────────────────────────────────────────┐
│ Application: 도메인별 제어 알고리즘, AI 모델              │
├──────────────────────────────────────────────────────────┤
│ Middleware: AUTOSAR Adaptive / ROS 2                      │
├──────────────────────────────────────────────────────────┤
│ OS: QNX / Linux (Yocto)                                  │
├──────────────────────────────────────────────────────────┤
│ BSP: 드라이버 (CAN, Ethernet TSN, GPIO, I2C, SPI)        │
└──────────────────────────────────────────────────────────┘
```

---

## 실시간성 보장 전략

Domain Controller는 여러 기능을 하나의 프로세서에서 처리하므로 실시간성 관리가 핵심입니다.

### CPU 코어 분리 (Core Isolation)
```
멀티코어 프로세서 (8코어 예시):
  Core 0, 1: Safety MCU 통신 전용 (고우선순위, 인터럽트 전용)
  Core 2, 3: 실시간 제어 태스크 (QNX RTOS, 1ms 주기)
  Core 4, 5: AI 추론 / 영상 처리 (Linux, 일반 스케줄링)
  Core 6, 7: OS 커널 + 드라이버 (Linux)

CPU Affinity 설정:
  taskset -c 2,3 ./realtime_control_task
  chrt -f 90 ./realtime_control_task  # FIFO 스케줄링, 우선순위 90
```

### 메모리 격리
```
IOMMU: DMA 접근 격리 → 버그가 있는 드라이버의 메모리 침범 방지
SMMU: ARM 기반 시스템에서의 동등한 기술

실시간 태스크용 메모리 락:
  mlockall(MCL_CURRENT | MCL_FUTURE);  // 페이지 폴트 방지
```

---

## 도메인 간 통신 프로토콜

```
Motion Control DC ◄──── 제어 명령 (SOME/IP, UDP) ──────► HMI DC
                   ◄──── 상태 데이터 (DDS) ─────────────►
                   ◄──── 안전 신호 (SOME/IP, TCP) ────────► Safety DC

Ethernet 백본 트래픽 분류:
  VLAN 10 (안전): Safety DC ↔ 모든 DC
    PCP=7, TAS 스케줄링, 지연 < 1ms
  VLAN 20 (제어): Motion DC ↔ HMI DC
    PCP=5, CBS 대역폭 보장
  VLAN 30 (영상): Vision DC ↔ HMI DC
    PCP=3, 대역폭 300Mbps 이상
  VLAN 40 (진단): 진단 PC ↔ 모든 DC
    PCP=0, Best Effort
```

---

## 기술적 과제와 해결 방안

### 과제 1: 단일 실패 지점 (Single Point of Failure)
```
문제: Domain Controller 하나 고장 시 도메인 전체 영향

해결:
  ① 이중화 (Redundancy): 주 DC + 예비 DC, 자동 절체
  ② Safety MCU: DC 고장 감지 후 안전 상태로 전환
  ③ Watchdog: DC 비정상 응답 시 시스템 재시작
```

### 과제 2: 소프트웨어 복잡도
```
문제: 여러 기능 통합 → 코드 복잡도 증가, 인터페이스 문제

해결:
  AUTOSAR Adaptive: 표준화된 서비스 인터페이스
  모듈화 설계: 각 기능을 독립 서비스로 구현
  형식 검증: 인터페이스 계약(Contract) 기반 개발
```

### 과제 3: 열 관리
```
문제: 고성능 AP의 발열 → 신뢰성 저하, 성능 스로틀링

해결:
  방열판 설계, 서멀 패드, 능동 냉각
  동적 전압/주파수 스케일링 (DVFS)
  열 이벤트 발생 시 성능 제한 정책 수립
```

---

## 주요 Domain Controller 제품 예시

| 제품 | 제조사 | 특징 | 대상 |
|------|--------|------|------|
| S32G | NXP | 차량 네트워크 특화, TSN, ASIL-D | Zone/Domain ECU |
| TDA4VM | Texas Instruments | Jacinto 7, ASIL-D, 8 TOPS | ADAS DC |
| R-Car S4 | Renesas | 12 TOPS, SoC 통합 | Gateway/DC |
| Drive Orin | NVIDIA | 254 TOPS, AI 특화 | HPC/ADAS |
| SA8295P | Qualcomm | 차량 인포테인먼트, AI | Cockpit DC |

---

## Reference
- [AUTOSAR Adaptive Platform](https://www.autosar.org/standards/adaptive-platform/)
- [NXP S32G Vehicle Network Processors](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/s32g-vehicle-network-processors:S32G)
- [TI TDA4VM - Jacinto 7](https://www.ti.com/product/TDA4VM)
- [Renesas R-Car Series](https://www.renesas.com/en/products/automotive-products/automotive-system-chips-socs/r-car)
