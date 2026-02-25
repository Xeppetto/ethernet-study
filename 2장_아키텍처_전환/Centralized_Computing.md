# 중앙집중형 컴퓨팅: HPC 아키텍처

## 왜 중앙집중인가

분산 ECU 시대의 문제는 단순히 "ECU가 너무 많다"가 아니었다. 진짜 문제는 ECU들 사이에 지능이 없다는 것이었다. 각 ECU는 자신의 임무만 수행하고, 시스템 전체의 맥락을 볼 수 없었다. 수술 로봇에서 로봇 팔이 갑자기 빨리 움직일 때, 관절 ECU는 그것이 의사의 의도적인 동작인지, 소프트웨어 버그인지, 외부 충격인지 구분할 수 없었다. 그 판단은 모든 ECU의 상태를 동시에 볼 수 있는 상위 시스템에서만 가능하다.

고성능 컴퓨터(HPC) 하나 혹은 두 대가 모든 연산을 통합하면, 시스템 전체의 상태를 하나의 일관된 모델로 유지할 수 있다. AI가 내시경 영상을 분석하면서 동시에 관절 힘 데이터와 연결지어 "이 방향으로 힘이 증가하는 것은 의심 조직 경계에 근접했기 때문"이라고 판단하는 것—이것은 중앙집중 없이 불가능하다.

---

## 아키텍처 진화: 세 단계

```
1세대: 분산 ECU (2000년대)
───────────────────────────────────────────────────
관절 #1 ECU ─CAN─ 관절 #2 ECU ─CAN─ 안전 ECU ─CAN─ HMI ECU
  ↑4MHz MCU    ↑4MHz MCU     ↑16MHz MCU    ↑32MHz MCU

각 ECU는 단일 기능, 수KB RAM, 실시간 보장 쉬움
단점: 시스템 전체 인지 불가, AI 추론 불가, 소프트 업데이트 어려움

2세대: Domain Controller (2010년대)
───────────────────────────────────────────────────
Motion DC ─100BASE-T1─ Vision DC ─100BASE-T1─ Safety DC
↑800MHz AP           ↑1GHz AP              ↑200MHz MCU
각 DC: 수십 MB RAM, 영역 내 AI 부분 가능
단점: 여전히 도메인 간 데이터 공유 복잡, 수백 ECU 통합 미완

3세대: 중앙집중형 HPC (2020년대~)
───────────────────────────────────────────────────
HPC ─TSN Ethernet─ Zone ECU ─ 센서/액추에이터
↑수백 TOPS AI, ↑수십코어 CPU, 수십GB RAM
Zone ECU: 경량 MCU (프로토콜 변환, 로컬 안전 감시)
이점: 전체 상태 통합, AI 추론, OTA, 소프트웨어 정의
```

---

## HPC 내부 구조: 계층별 역할

HPC는 단일 칩이 아니라 여러 이종(heterogeneous) 처리 유닛의 집합이다.

```
┌─────────────────────────────────────────────────────────────────┐
│                      HPC (Patient-Side Computer)                 │
│                                                                 │
│  ┌───────────────────────┐   ┌───────────────────────────────┐  │
│  │   Application SoC     │   │   Safety MCU                  │  │
│  │  (NVIDIA Orin / NXP)  │   │   (TI TDA4VM / Renesas R-Car) │  │
│  │                       │   │                               │  │
│  │  CPU: 12× Cortex-A78  │   │  CPU: Cortex-R52 Lockstep    │  │
│  │  GPU: Ampere 2048 CUDA│   │  ASIL-D 인증                  │  │
│  │  DLA: 2× 설계         │   │  독립 전원 도메인             │  │
│  │  Memory: 32GB LPDDR5  │   │  내장 WDT, BIST 자가진단      │  │
│  │  Storage: NVMe 512GB  │   │  Ethernet PHY (모니터링)      │  │
│  └──────────┬────────────┘   └──────────────┬────────────────┘  │
│             │ PCIe Gen4 ×8                   │ SPI/UART/Ethernet│
│  ┌──────────▼────────────────────────────────▼────────────────┐  │
│  │          내부 TSN 이더넷 스위치 (8× 1GbE + 2× 10GbE)       │  │
│  │          → Zone ECU 연결, 외부 콘솔 연결                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────┐  ┌──────────────────────────────────┐    │
│  │  HSM               │  │  전원 관리 IC (PMIC)             │    │
│  │  (Hardware Security│  │  입력: 24V DC                    │    │
│  │   Module)          │  │  출력: SoC 0.8V/1.8V, MCU 3.3V  │    │
│  │  Secure Boot 키    │  │  Safety MCU: 별도 레귤레이터    │    │
│  │  TLS 인증서 저장   │  └──────────────────────────────────┘    │
│  └───────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

**SoC와 Safety MCU의 분리가 중요한 이유:** SoC(Application Processor)는 Linux/RTOS를 돌리고 AI 추론을 수행하는 고성능 칩이지만, 소프트웨어 버그, 메모리 오류, 열 스로틀링 등으로 인해 일시적으로 응답이 없어질 수 있다. Safety MCU는 이것을 감시하다가 HPC에서 50밀리초 동안 heartbeat가 오지 않으면 Fail-Safe를 실행한다. 이 두 칩은 독립적인 전원 레일과 독립적인 리셋 회로를 가져야 한다—하나가 죽어도 다른 하나는 살아있어야 한다.

---

## 하이퍼바이저: 하나의 HPC에서 실시간과 비실시간을 격리

의료 로봇 HPC에서 가장 어려운 과제 중 하나는 실시간 제어 코드와 비실시간 서비스(AI, OTA, 로깅)를 같은 하드웨어에서 안전하게 분리하는 것이다. 하이퍼바이저(Hypervisor)가 이 문제를 해결한다.

```
HPC 하드웨어: ARM Cortex-A78AE × 12코어
─────────────────────────────────────────────────────────────────
Type-1 Hypervisor (QNX Hypervisor 2.2 / ACRN / Xen)
─────────────────────────────────────────────────────────────────
  파티션 A: Safety Partition        파티션 B: General Partition
  ─────────────────────────         ─────────────────────────────
  QNX Neutrino RTOS 7.1             Ubuntu 22.04 LTS / Yocto
  코어: CPU 0, 1, 2 (격리됨)        코어: CPU 3~11
  메모리: 2GB (전용, 고정)           메모리: 30GB (가변)
  인터럽트: 직접 핸들링              인터럽트: 가상화됨
  ─────────────────────────         ─────────────────────────────
  • 관절 제어 루프 (1kHz)           • ROS 2 플래닝 노드
  • 힘/토크 처리                    • AI 추론 (PyTorch)
  • Safety watchdog                 • OTA 클라이언트
  • FRER/PTP 스택                   • 원격 진단 서버
  • 비상 정지 로직                   • 로그 수집 및 전송
  ─────────────────────────         ─────────────────────────────
  WCL: < 50µs (실측)                지터: 수ms 허용 가능
  ASIL-B 소프트웨어                  비안전(ASIL-QM)
─────────────────────────────────────────────────────────────────
  공유 메모리 채널 (IVSHMEM, 1µs 미만 IPC)
  → 파티션 A가 제어 명령 publish, 파티션 B가 읽음
```

**하이퍼바이저 타이밍 오버헤드:** Type-1 하이퍼바이저도 실시간 파티션에 지연을 추가한다. QNX Hypervisor의 경우 파티션 스케줄링 오버헤드가 약 5~15마이크로초다. 이것이 1ms 제어 루프에서 무시할 수 없다면, CPU 코어를 파티션에 전용 배정(pinning)하고 하이퍼바이저 스케줄러가 개입하지 않게 구성할 수 있다.

---

## CPU 코어 격리: 실시간 태스크에 코어를 독점시키기

하이퍼바이저 없이 Linux 단일 커널로 구성할 때, 실시간 태스크가 OS의 일반 스케줄링 압력을 받지 않도록 CPU 코어를 격리해야 한다.

```bash
# 1. 부트 파라미터로 코어 격리 (grub 설정)
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=0,1,2 nohz_full=0,1,2 rcu_nocbs=0,1,2"
# isolcpus: 일반 스케줄러에서 제외
# nohz_full: 해당 코어의 타이머 인터럽트 최소화
# rcu_nocbs: RCU 콜백을 다른 코어로 오프로드

# 2. 격리된 코어에 실시간 프로세스 배정
taskset -c 0 chrt --fifo 99 ./joint_control_loop

# 3. IRQ 어피니티 조정 (실시간 코어에서 인터럽트 제거)
for irq in $(ls /proc/irq); do
    [ -d /proc/irq/$irq ] && echo "c" > /proc/irq/$irq/smp_affinity  # 코어 2,3에만
done

# 4. 실시간 태스크 메모리 락 (페이지 폴트 방지)
# C 코드에서:
# mlockall(MCL_CURRENT | MCL_FUTURE);
# → 전체 프로세스 메모리를 물리 메모리에 고정

# 5. 결과 확인: cyclictest로 지터 측정
cyclictest --mlockall -p99 -t1 -a0 -n -i1000 -l100000 -h200
# -a0: CPU 코어 0에서 실행
# -i1000: 1000µs (1ms) 간격
# -l100000: 100,000회 측정
```

**실측 비교 (1GHz ARM Cortex-A72, 1ms 주기):**

| 구성 | 평균 지터 | 최악 지터 |
|---|---|---|
| 표준 Linux (4.19) | 50~200µs | 5~10ms |
| PREEMPT_RT 패치 적용 | 10~50µs | 100~300µs |
| isolcpus + PREEMPT_RT | 5~15µs | 30~80µs |
| RTOS (QNX) 단독 | 1~5µs | 10~20µs |
| QNX Hypervisor (전용 코어) | 2~8µs | 15~30µs |

> 의료 로봇 1kHz 제어 루프의 목표 지터는 ±50µs 이하이므로, 표준 Linux 단독으로는 요구사항을 만족하지 못한다. PREEMPT_RT + isolcpus 조합이 최소 요구사항이며, 더 엄격한 경우 RTOS 파티션이 필요하다.

---

## IOMMU/SMMU: DMA 격리로 메모리 안전 보장

HPC에서 여러 게스트 OS가 동시에 동작하면, 하나의 게스트가 DMA(Direct Memory Access)를 통해 다른 게스트의 메모리에 접근할 위험이 있다. 버그가 있는 네트워크 드라이버가 잘못된 DMA 주소를 쓰면 안전 파티션의 메모리를 덮어쓸 수 있다.

IOMMU(Input/Output Memory Management Unit, x86) 또는 SMMU(System MMU, ARM)가 이를 방지한다. 각 장치의 DMA 접근을 I/O 페이지 테이블로 제어해, 허가된 메모리 영역에만 접근할 수 있게 한다.

```bash
# Intel IOMMU 활성화 (GRUB)
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"

# ARM SMMU 상태 확인
dmesg | grep -i smmu
# [    0.456789] arm-smmu-v3 arm-smmu-v3.0: probed (6 master devices)

# IOMMU 그룹 확인 (어떤 장치가 함께 격리되는지)
ls /sys/kernel/iommu_groups/
for d in /sys/kernel/iommu_groups/*/devices/*; do
    echo "Group: $(basename $(dirname $(dirname $d))): $(basename $d)"
done
```

**의료 로봇에서의 실제 적용:** TSN NIC를 안전 파티션에만 바인딩하고 SMMU로 해당 NIC의 DMA를 안전 파티션 메모리 영역으로 제한한다. 이렇게 하면 일반 파티션의 네트워크 드라이버가 설령 오작동해도 안전 파티션의 제어 패킷 처리에 영향을 줄 수 없다.

---

## AI 추론과 실시간 제어의 공존

HPC에서 AI 추론과 1kHz 실시간 제어가 동시에 돌 때 가장 큰 위험은 AI 추론의 가변 실행 시간이 실시간 태스크의 CPU 자원을 빼앗는 것이다. 이를 막는 방법:

**방법 1: AI를 별도 코어/GPU에 격리.** AI 추론은 GPU나 NPU(Neural Processing Unit)에서 수행하고, CPU 코어는 제어 루프에 전용 배정한다. NVIDIA Orin의 DLA(Deep Learning Accelerator)는 CPU와 독립적으로 동작해 이 목적에 적합하다.

**방법 2: AI 추론에 CPU 버짓 제한.** Linux `cgroup`으로 AI 프로세스의 CPU 사용률 상한을 설정한다.

```bash
# AI 추론 프로세스에 CPU 30% 상한 (cgroup v2)
mkdir /sys/fs/cgroup/ai_inference
echo "300000 1000000" > /sys/fs/cgroup/ai_inference/cpu.max
# 1,000,000µs 주기에 300,000µs (30%) 허용

# AI 프로세스를 해당 cgroup에 추가
echo $AI_PID > /sys/fs/cgroup/ai_inference/cgroup.procs
```

**방법 3: 결과 유효 기간 설계.** AI 추론 결과(예: 조직 경계 위치)는 1~5ms 단위로 갱신되어도 충분하다. 추론 결과를 공유 메모리에 넣어두고 제어 루프가 항상 최신 결과를 읽는 방식으로, 추론 완료를 기다리지 않아도 된다.

---

## Safety MCU Watchdog 통합

HPC의 Application SoC가 응답을 멈췄을 때 Safety MCU가 감지하고 Fail-Safe로 전환하는 메커니즘이다.

```
Watchdog 동작 흐름:

Application SoC                   Safety MCU
─────────────────                 ─────────────────────────────
제어 루프 실행 (1ms)  ─heartbeat─►  타이머 리셋 (50ms 윈도우)
                                  타이머 카운트 중...
                      ─heartbeat─►  타이머 리셋
                      ─heartbeat─►  타이머 리셋
(SoC 응답 없음)
                                  타이머 만료 (50ms 경과)
                                  → Fail-Safe 실행:
                                     1. 모든 관절에 브레이크 명령
                                     2. HPC 전원 사이클 (재시작 시도)
                                     3. 비상 정지 인터락 활성화
                                     4. 오류 상태 LED 및 알람
```

```c
// Safety MCU 코드 (의사 코드)
#define WATCHDOG_TIMEOUT_MS  50
#define HEARTBEAT_PERIOD_MS  10

volatile uint32_t last_heartbeat_time = 0;

void heartbeat_receive_isr(void) {
    last_heartbeat_time = get_system_time_ms();
}

void safety_monitor_task(void) {
    while (1) {
        uint32_t now = get_system_time_ms();
        if ((now - last_heartbeat_time) > WATCHDOG_TIMEOUT_MS) {
            trigger_fail_safe();  // 브레이크, 전원차단, 알람
        }
        task_delay_ms(5);  // 5ms 주기 감시
    }
}
```

IEC 62304 관점에서 이 Watchdog 로직은 Class C 소프트웨어로 분류된다. Watchdog 타임아웃 값(50ms)의 도출 근거와 Fail-Safe 동작의 검증 결과가 위험 관리 문서에 포함되어야 한다.

---

## HPC 이중화: Single Point of Failure 제거

환자에게 위험이 생길 수 있는 고가용성 구간에서는 HPC를 이중화한다.

```
HPC #1 (주)                     HPC #2 (예비)
───────────                     ────────────────
정상 동작                        대기(Standby) 상태
Zone ECU 제어                   HPC #1 상태 수신만 (passive)
Safety MCU 감시                 HPC #1 heartbeat 모니터링

HPC #1 장애 감지 (50ms 내):
  → HPC #2가 Active로 전환
  → Zone ECU 제어 인계
  → 전환 시간 목표: < 100ms

전환 중 갭 처리:
  Zone ECU에서 Hold-Last-Command (최대 100ms)
  Zone Safety MCU: 100ms 이상 갭 → 로봇 정지

주의: HPC 절체 100ms 동안 로봇 팔은 Hold 상태
      수술 중 절체는 의사가 인지해야 함 (HMI 알림)
```

HPC 이중화의 비용: 하드웨어 두 배, 통신 동기화 오버헤드, SW 복잡도 증가. 따라서 HPC 이중화가 필요한지는 수술 단계별 위험 분석에 근거해야 한다. 모든 수술 단계에 Fail-Operational이 필요한 것은 아니다.

---

## 열 관리: 고성능 컴퓨팅의 현실적 제약

NVIDIA Orin NX(10W~25W), Orin AGX(60W~60W 이상)처럼 HPC용 SoC는 상당한 열을 발생시킨다. 수술실 환경에서의 열 관리는 일반 서버실과 다른 제약이 있다.

- **팬 소음**: 수술실에서 팬 소음은 의료진의 집중을 방해할 수 있다. 팬리스 혹은 저속 팬 설계가 선호된다.
- **열 경계**: 수술 로봇 하우징이 환자와 가까이 있으면 표면 온도 제한(IEC 60601-1: 43°C 이하 손이 닿는 부위)을 지켜야 한다.
- **성능 스로틀링 방지**: 열이 임계값에 가까워지면 SoC가 동작 주파수를 낮춘다. 이것이 제어 루프 실행 시간에 영향을 주지 않도록 열 설계 여유가 필요하다.

실용적 접근: HPC의 TDP 소비 전력의 150%를 방열 용량으로 설계하고, 지속적인 AI 추론 부하 상태에서 2시간 이상 동작해도 스로틀링이 발생하지 않는지 검증한다.

---

## 참고 문헌

- NVIDIA Jetson Orin AGX Technical Reference Manual
- QNX Hypervisor 2.2 User's Guide — BlackBerry QNX
- Intel VT-d: Virtualization Technology for Directed I/O, Intel Manual
- ARM SMMU Architecture Specification v3.3
- IEC 62304:2006/AMD1:2015 §5.3: Software Architectural Design
- ACRN Project: https://projectacrn.org
- Lelli, J. et al. "Deadline Scheduling in the Linux Kernel" OSPERT (2012)

---

*관련: [2장 Zonal Architecture](./Zonal_Architecture.md) | [2장 RTOS 및 실시간 OS](./RTOS_and_Realtime_OS.md)*
