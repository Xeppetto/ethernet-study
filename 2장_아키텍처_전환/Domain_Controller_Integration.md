# Domain Controller 통합

## 개요
기존의 수십~수백 개 개별 ECU 분산 아키텍처에서, 유사한 기능을 묶어 고성능 Domain Controller로 통합하는 추세입니다. 이는 완전한 중앙집중형 HPC로 가는 과도기적 단계이자, 현실적이고 실용적인 최적화 방안입니다.

---

## 아키텍처 진화 단계

```
1단계: 분산 ECU (2000년대)
  엔진 ECU ─CAN─ 변속기 ECU ─CAN─ ABS ECU ─CAN─ ... ECU #100
  통신: CAN 2Mbps (포화 위험), 각 ECU: 단일 기능

2단계: Domain Controller 통합 (2015년~현재)
  파워트레인 DC ─Ethernet 100BASE-T1─ 샤시 DC ─Ethernet─ ADAS DC
  통신: Domain 내부 CAN FD, Domain 간 Ethernet
  DC당 기능 통합: 5~20개 ECU → 1개 DC

3단계: 중앙집중형 HPC + Zone ECU (2025년~)
  Zone ECU ─TSN Ethernet 1Gbps─ HPC
  HPC: AI/GPU 수십~수백 TOPS, OTA, 클라우드 연동
```

---

## 주요 Domain Controller 제품 비교

| 제품 | 제조사 | CPU | AI 성능 | 안전 인증 | 대상 도메인 |
|------|--------|-----|---------|----------|------------|
| S32G2/G3 | NXP | 4× A53 + 3× M7 | - | ASIL-D | Zone/Gateway |
| TDA4VM (Jacinto 7) | Texas Instruments | 2× A72 + 6× R5F | 8 TOPS | ASIL-D | ADAS |
| R-Car S4 | Renesas | 8× A55 + CR52 | 12 TOPS | ASIL-B | Gateway/HPC |
| Drive Orin | NVIDIA | 12× A78AE | 254 TOPS | ASIL-B | ADAS/HPC |
| SA8295P | Qualcomm | 8× Kryo Gold | 30 TOPS | - | Cockpit |
| TC4xx (AURIX) | Infineon | 4× TriCore | - | ASIL-D | Safety DC |

---

## Domain Controller 구성 예시

### 의료 로봇 Domain Controller 구조
```
┌─────────────────────────────────────────────────────────────┐
│                     의료 로봇 시스템                          │
│                                                             │
│  ┌──────────────────┐  Ethernet  ┌──────────────────────┐   │
│  │ Motion Control DC │◄──────────►│  Vision Domain DC    │   │
│  │                  │  1GbE TSN  │                      │   │
│  │ • 7축 관절 제어   │            │ • 4K 내시경 처리     │   │
│  │ • 역기구학 연산   │            │ • 3D 재구성 (SfM)    │   │
│  │ • 임피던스 제어   │            │ • AI 병변 인식       │   │
│  │ • 충돌 회피       │            │ • 조명 제어          │   │
│  │  NXP TDA4VM      │            │  NVIDIA Orin NX      │   │
│  │  ASIL-D          │            │  (AI 특화)           │   │
│  └──────────────────┘            └──────────────────────┘   │
│           │                              │                   │
│           └─────── TSN Ethernet ─────────┘                   │
│                         │                                    │
│  ┌──────────────────┐  Ethernet  ┌──────────────────────┐   │
│  │   HMI Domain DC   │◄──────────►│   Safety Domain DC   │   │
│  │                  │  1GbE      │                      │   │
│  │ • 의사용 콘솔     │            │ • 비상 정지 (E-Stop)  │   │
│  │ • 햅틱 피드백 처리│            │ • 시스템 상태 감시    │   │
│  │ • 수술 영상 표시  │            │ • 이중화 제어         │   │
│  │ • 수술 기록       │            │ • ASIL-D 인증        │   │
│  │  Qualcomm SA8295 │            │  Infineon TC4xx      │   │
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
│  │ ARM A72/A78│  │ ASIL-B/D   │  │ LPDDR4/5          │   │
│  │ 800MHz~4GHz│  │ Watchdog   │  │ 4~16 GB           │   │
│  │ 멀티코어   │  │ BIST       │  │ ECC 지원           │   │
│  └────────────┘  └────────────┘  └──────────────────┘   │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │ CAN FD     │  │ Ethernet   │  │ HSM               │   │
│  │ 인터페이스  │  │ TSN PHY    │  │ (보안 모듈)       │   │
│  │ 4~8채널    │  │ 100BASE-T1 │  │ SecureBoot        │   │
│  │ 최대 8Mbps │  │ 1000BASE-T1│  │ AES/ECC 엔진      │   │
│  └────────────┘  └────────────┘  └──────────────────┘   │
│                                                          │
│  스토리지: eMMC 32~128GB (OS + 모델 + 로그)              │
└──────────────────────────────────────────────────────────┘

소프트웨어 스택:
┌──────────────────────────────────────────────────────────┐
│ Application: 도메인별 제어 알고리즘, AI 모델, 서비스       │
├──────────────────────────────────────────────────────────┤
│ Middleware: AUTOSAR Adaptive (ara::com) / ROS 2          │
├──────────────────────────────────────────────────────────┤
│ OS: QNX Neutrino / Linux (Yocto) / AUTOSAR OS            │
├──────────────────────────────────────────────────────────┤
│ BSP: CAN FD, 100BASE-T1 TSN, GPIO, I2C, SPI 드라이버    │
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
```

```bash
# CPU Affinity 및 실시간 우선순위 설정
# 실시간 제어 프로세스를 Core 2,3 고정
taskset -c 2,3 ./realtime_control &
CTRL_PID=$(pgrep realtime_control)

# SCHED_FIFO (실시간 스케줄링) 우선순위 90 설정
chrt -f 90 -p $CTRL_PID

# 메모리 페이지 폴트 방지
cat > /tmp/lock_memory.c << 'EOF'
#include <sys/mman.h>
int main() {
    mlockall(MCL_CURRENT | MCL_FUTURE);
    /* 실시간 제어 루프 진입 */
    return 0;
}
EOF

# IRQ Affinity: Ethernet 인터럽트를 비실시간 코어로
ETH_IRQ=$(grep eth0 /proc/interrupts | cut -d: -f1 | tr -d ' ')
echo "fc" > /proc/irq/${ETH_IRQ}/smp_affinity  # Core 2~7만 허용
```

### 메모리 격리 (SMMU/IOMMU)
```
ARM SMMU (System Memory Management Unit) 활용:
  ┌───────────────────────────────────────────┐
  │  SMMU 격리 구성                            │
  │                                           │
  │  Safety 태스크 → Region A (Read/Write)    │
  │  AI 태스크    → Region B (Read/Write)     │
  │  I/O DMA     → Region C (DMA 전용)       │
  │  Region A ↔ B: 접근 차단 (SMMU가 거부)  │
  └───────────────────────────────────────────┘

IOMMU 설정 (Linux):
  dma-ranges = <0x00 0x80000000 0x00 0x80000000 0x00 0x40000000>
  iommu-map = <0x0 &smmu 0x0 0x1>
```

---

## 도메인 간 통신 프로토콜

```
Motion Control DC ◄──── 제어 명령 (SOME/IP UDP) ────────► HMI DC
                  ◄──── 관절 상태 (DDS RTPS) ────────────►
                  ◄──── 안전 신호 (SOME/IP TCP) ───────────► Safety DC

Ethernet 백본 트래픽 분류 (VLAN + IEEE 802.1p):
  VLAN 10 (안전):  Safety DC ↔ 모든 DC
    PCP=7, TAS 스케줄링, 지연 < 1ms, 대역폭 10Mbps
  VLAN 20 (제어):  Motion DC ↔ HMI DC
    PCP=5, CBS 대역폭 보장, 100Mbps 예약
  VLAN 30 (영상):  Vision DC ↔ HMI DC
    PCP=3, 대역폭 300Mbps 이상 확보
  VLAN 40 (진단):  DoIP 클라이언트 ↔ 모든 DC
    PCP=0, Best Effort
```

### 도메인 간 통신 지연 측정
```python
import socket, time, struct

def measure_latency(server_ip: str, port: int, count: int = 1000) -> dict:
    """SOME/IP 응답 지연 측정 (µs)"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(0.1)

    latencies = []
    SOMEIP_HEADER = struct.pack(
        ">HHLHBBBB",
        0x0101, 0x0002,  # Service ID, Method ID (GetPosition)
        8,               # Length (헤더 제외 없음)
        0x0001, 0x0001,  # Client ID, Session ID
        0x01, 0x01,      # Proto Version, Interface Version
        0x00, 0x00       # Msg Type (Request), Return Code
    )

    for i in range(count):
        t0 = time.perf_counter_ns()
        sock.sendto(SOMEIP_HEADER, (server_ip, port))
        try:
            sock.recv(256)
            t1 = time.perf_counter_ns()
            latencies.append((t1 - t0) / 1000)  # µs 변환
        except socket.timeout:
            pass

    return {
        "min_us": min(latencies),
        "avg_us": sum(latencies) / len(latencies),
        "max_us": max(latencies),
        "p99_us": sorted(latencies)[int(len(latencies) * 0.99)]
    }
```

---

## 기술적 과제와 해결 방안

### 과제 1: 단일 실패 지점 (Single Point of Failure)
```
문제: Domain Controller 하나 고장 시 도메인 전체 영향

해결:
  ① 이중화 (Hot Standby): 주 DC + 예비 DC, 자동 절체 (< 100ms)
     Heartbeat 모니터링: 10ms 간격, 3회 연속 실패 시 절체
  ② Safety MCU 독립 운용: DC 고장 감지 후 안전 상태로 전환
     (E-Stop 활성화, 브레이크 고정, 안전 위치로 이동)
  ③ 계층적 Watchdog:
     SW WDG → OS WDG → External HW WDG → 전원 리셋
```

### 과제 2: 소프트웨어 복잡도
```
문제: 여러 기능 통합 → 코드 복잡도 증가, 기능 간 간섭

해결:
  AUTOSAR Adaptive: ara::exec로 프로세스 격리
  Process Isolation: 각 서비스는 독립 프로세스 (메모리 분리)
  계약 기반 인터페이스: Skeleton/Proxy로 명확한 API 경계
  IEC 62304 요구: 모듈별 독립 검증 가능 설계
```

### 과제 3: 열 관리
```
문제: 고성능 AP의 발열 → 신뢰성 저하, 성능 스로틀링

해결:
  방열판 + 서멀 패드 + 능동 팬 (소음 < 45dB 의료 환경)
  DVFS: 부하에 따른 동적 클럭/전압 조절
  Thermal Throttling 정책:
    85°C: AI 추론 주파수 50% 감소
    95°C: 비안전 서비스 일시 중단
    100°C+: 비상 정지 활성화
```

---

## 의료 로봇 Domain Controller 진단 인터페이스

```
DoIP (ISO 13400) 기반 원격 진단:

진단 PC / OTA 서버
       │ TCP 13400
       ▼
  Motion Control DC (DoIP Entity)
  ├── UDS 0x22 ReadDataByIdentifier
  │   ├── DID 0xF186: ActiveDiagnosticSession
  │   ├── DID 0xF18C: ECU Serial Number
  │   ├── DID 0xF189: SW Version Number
  │   └── DID 0xD001: Joint Angle [7× float32]
  ├── UDS 0x14 ClearDiagnosticInformation
  ├── UDS 0x19 ReadDTCInformation
  │   └── DTC 0x110011: JointMotor_1_OverCurrent
  └── UDS 0x31 RoutineControl
      └── Routine 0x0201: Joint Calibration
```

---

## Reference
- [AUTOSAR Adaptive Platform](https://www.autosar.org/standards/adaptive-platform/)
- [NXP S32G Vehicle Network Processors](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/s32g-vehicle-network-processors:S32G)
- [TI TDA4VM - Jacinto 7 ADAS SoC](https://www.ti.com/product/TDA4VM)
- [Renesas R-Car S4 - Automotive SoC](https://www.renesas.com/en/products/automotive-products/automotive-system-chips-socs/r-car-s4)
- [NVIDIA DRIVE Orin](https://www.nvidia.com/en-us/self-driving-cars/drive-platform/)
- [Infineon AURIX TC4xx - Safety MCU](https://www.infineon.com/cms/en/product/microcontroller/32-bit-tricore-microcontroller/32-bit-tricore-aurix-tc4x/)
