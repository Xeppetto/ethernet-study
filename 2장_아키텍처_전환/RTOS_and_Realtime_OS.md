# RTOS와 실시간 운영체제: 의료 로봇의 선택

## 실시간이란 무엇인가

"실시간(real-time)"이라는 단어는 자주 오해된다. 빠르다는 뜻이 아니다. 예측 가능하다는 뜻이다.

100밀리초 안에 응답하는 시스템도 실시간이 될 수 있고, 1밀리초 안에 응답하지만 가끔 10밀리초가 걸리는 시스템은 실시간이 아니다. 실시간의 정의는 **최악의 경우 응답 시간(WCRT: Worst-Case Response Time)이 데드라인을 항상 만족하는 것**이다. "거의 항상"은 실시간이 아니다.

의료 로봇의 1kHz 제어 루프에서 데드라인은 1밀리초다. 0.999밀리초에 응답하는 날이 999번이고 1.001밀리초에 응답하는 날이 1번이어도, 그 1번이 수술 중 환자 위험으로 이어질 수 있다. IEC 62304 Class C 요구사항은 이 WCRT를 측정하고 문서화할 것을 요구한다.

---

## 실시간 운영체제 분류

```
실시간 OS 분류:

Hard Real-Time:
  WCRT 위반 = 시스템 실패 (허용 불가)
  예: 관절 제어 루프, 안전 인터락, E-Stop
  요구 WCRT: 수µs ~ 수ms
  OS: QNX, VxWorks, INTEGRITY, FreeRTOS, Zephyr

Soft Real-Time:
  WCRT 위반 = 성능 저하 (허용되는 경우 존재)
  예: 내시경 영상 스트리밍, AI 추론, 햅틱 피드백
  요구 WCRT: 수ms ~ 수십ms
  OS: Linux PREEMPT_RT, macOS, Windows

Firm Real-Time:
  WCRT 위반 = 해당 결과 무효화 (폐기하고 계속)
  예: 오디오/비디오 버퍼, 데이터 수집
  요구 WCRT: 가변적 (버퍼 크기에 따라)
```

의료 로봇에서 관절 제어는 Hard Real-Time, 영상 표시는 Soft Real-Time, 진단 로그는 요구사항 없음. 이 세 가지를 하나의 OS에서 처리하는 것은 어렵고 위험하다—Hard RT와 Soft RT가 CPU를 공유하면 Hard RT가 침범받을 수 있다.

---

## 주요 RTOS 비교

### QNX Neutrino RTOS

BlackBerry QNX의 마이크로커널 기반 RTOS다. "마이크로커널"이 의미하는 것은 OS 서비스(파일 시스템, 네트워크 스택, 드라이버)가 커널 밖의 별도 프로세스로 동작한다는 것이다. 드라이버가 크래시해도 커널과 다른 프로세스는 살아남는다.

```
QNX 마이크로커널 구조:

┌────────────────────────────────────────────────────────────────┐
│  Application Space (Ring 3)                                    │
│  Joint Control App   Safety Monitor   AI Service              │
│       │                   │               │                   │
│  io-net (네트워크)   proc (프로세스)   pps (이벤트)           │
│  devb-... (디스크)  io-usb (USB)      ...                     │
├────────────────────────────────────────────────────────────────┤
│  QNX Microkernel (Ring 0)                                      │
│  스케줄링, IPC, 메모리 관리, 인터럽트 핸들링만 담당             │
│  크기: ~100KB                                                  │
└────────────────────────────────────────────────────────────────┘
```

**의료 로봇에서의 QNX 장점:**
- 마이크로커널 격리: 드라이버 버그가 관절 제어에 영향 없음
- 기능 안전 인증: ISO 26262 ASIL-D, IEC 61508 SIL 3 인증 받은 제품
- 결정론적 IPC: 메시지 패싱의 WCRT 측정 가능
- QNX Hypervisor와 네이티브 통합 (같은 제조사)

**단점:**
- 상용 라이선스 비용 (대형 프로젝트에서 수억 원)
- 리눅스 생태계 소프트웨어 재사용 어려움
- ROS 2 공식 지원 없음 (비공식 포팅 존재)

**cyclictest 실측 결과 (Cortex-A72, 1ms 주기):**

| 구성 | 평균 지연 | 최악 지연 |
|---|---|---|
| QNX Neutrino 7.1 단독 | 3µs | 12µs |
| QNX + 네트워크 부하 | 5µs | 25µs |

### Linux PREEMPT_RT

리눅스 커널에 PREEMPT_RT 패치를 적용하면 커널 내부 대부분의 코드를 선점(preempt) 가능하게 만든다. 표준 Linux에서 스핀락과 인터럽트 핸들러는 선점이 불가능해 실시간 태스크를 수십 밀리초 블로킹할 수 있는데, PREEMPT_RT는 이것들을 뮤텍스와 스레드로 변환한다.

```bash
# PREEMPT_RT 적용 확인
uname -v | grep PREEMPT_RT
# 또는
cat /proc/version | grep PREEMPT

# 실시간 프로세스로 실행 (FIFO 스케줄러, 최고 우선순위)
chrt --fifo 99 ./joint_control_loop

# 결과 측정
cyclictest -m -p99 -i1000 -l1000000 -h200 --histfile=latency.txt
# -m: mlockall (페이지 폴트 방지)
# -p99: FIFO 우선순위 99
# -i1000: 1000µs 간격
# -l1000000: 100만 회 측정
# -h200: 히스토그램 최대 200µs
```

**실측 비교 (x86-64, Intel i7-1165G7, 1ms 주기):**

| 구성 | 평균 지연 | 최악 지연 |
|---|---|---|
| 표준 Linux 5.15 | 15µs | 8,000µs (8ms!) |
| PREEMPT_RT 5.15-rt | 8µs | 85µs |
| PREEMPT_RT + isolcpus | 5µs | 35µs |

> **최악 지연 8ms** (표준 Linux)는 1kHz 제어 루프에서 치명적이다. PREEMPT_RT 적용 후 85µs는 여전히 높지만, isolcpus와 IRQ 어피니티 조정을 더하면 35µs까지 낮출 수 있다. 의료 로봇의 허용 지터 기준(±50µs)에 근접한다.

**의료 로봇에서의 PREEMPT_RT 장점:**
- ROS 2, DDS, Python 등 리눅스 생태계 완전 활용
- 무료 (오픈소스)
- 방대한 커뮤니티와 문서
- Yocto로 의료 로봇 전용 이미지 구성 가능

**단점:**
- QNX 대비 최악 지연이 높음
- 안전 인증(SIL/ASIL) 공식 지원 없음 (Linux Foundation ELISA 프로젝트 진행 중)
- isolcpus 설정 등 튜닝 노하우 필요

### FreeRTOS / Zephyr: MCU 레벨 RTOS

Zone ECU MCU(Cortex-M, RISC-V)에는 FreeRTOS 또는 Zephyr가 적합하다. 수KB의 RAM에서 동작하는 초경량 RTOS다.

```c
// FreeRTOS 관절 제어 태스크 예시
void JointControlTask(void *pvParameters) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(1);  // 1ms 주기

    for(;;) {
        // 관절 인코더 읽기
        float angle = ReadEncoder(JOINT_1);

        // PID 제어 계산
        float torque = PID_Compute(&pid1, target_angle, angle);

        // 모터 드라이버 출력
        SetMotorTorque(JOINT_1, torque);

        // 정확히 1ms 간격으로 실행 (drift 보정 포함)
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}

// 태스크 생성 (우선순위 10, 스택 512 byte)
xTaskCreate(JointControlTask, "JointCtrl", 512, NULL, 10, NULL);
```

**Zephyr의 장점:** Zephyr는 FreeRTOS보다 풍부한 기능(Bluetooth, LoRa, USB, CAN, 10BASE-T1S 드라이버 포함)과 기능 안전 인증(IEC 61508 SIL 3) 작업이 진행 중이다. NXP, STMicroelectronics, Nordic Semiconductor 등 주요 MCU 제조사가 공식 지원한다.

### VxWorks: 엔터프라이즈 RTOS

Wind River의 VxWorks는 항공우주, 방산, 의료 분야에서 오래된 실적을 가진 상용 RTOS다. DO-178C(항공 소프트웨어), IEC 62304(의료기기), IEC 61508(기능 안전) 인증을 받은 제품이 있다.

QNX와 유사한 포지셔닝이지만, VxWorks는 전통적으로 항공 분야에 강점이 있고 의료 쪽에서도 점유율을 확보하고 있다. 로봇공학에서는 QNX보다 덜 사용된다.

---

## 의료 로봇 OS 선택 가이드

```
선택 매트릭스:

역할                    권장 OS              이유
────────────────────────────────────────────────────────────────────
Zone ECU (관절 제어)   FreeRTOS / Zephyr    MCU, 수KB RAM, ASIL 경로
Zone ECU (안전 감시)   AUTOSAR OS / RTOS    ASIL-D 인증 필요
HPC 안전 파티션        QNX / VxWorks        ASIL-B, 하이퍼바이저 격리
HPC 일반 파티션        Linux PREEMPT_RT     ROS 2, AI, 리눅스 생태계
HPC AI/OTA 서비스      표준 Linux (Ubuntu)  GPU 드라이버, Docker 지원
```

```
실용적 의료 로봇 HPC 구성 예시:

Hardware: NXP S32G2 + NVIDIA Orin NX
  └─ S32G2: AUTOSAR Classic 실행, CAN FD 게이트웨이, ASIL-D 안전
  └─ Orin NX: QNX Hypervisor
      ├─ 파티션 A: QNX Neutrino (관절 제어, FRER 스택)
      └─ 파티션 B: Linux RT-Preempt (ROS 2, AI, OTA)

네트워크:
  └─ S32G2 TSN 포트: 안전 VLAN 10, 제어 VLAN 20
  └─ Orin NX I225 NIC: TAS 오프로드 지원
```

---

## 성능 측정과 IEC 62304 검증

RTOS를 선택한 후 성능이 요구사항을 만족하는지 측정하고, 그 결과를 IEC 62304 소프트웨어 검증 문서에 포함해야 한다.

### cyclictest: 지터 측정의 표준 도구

```bash
# 실시간 지터 측정 (1시간 테스트)
sudo cyclictest \
    --mlockall \
    --priority=99 \
    --interval=1000 \     # 1ms 주기
    --distance=0 \
    --nsecs \
    --loops=3600000 \     # 1시간 × 1kHz
    --affinity=0 \        # CPU 0에서 실행 (isolcpus=0 설정 후)
    --histfile=jitter_1hr.txt \
    --histogram=1000      # 히스토그램 최대 1000µs

# 결과 분석
# Min: 최소 지연 (이상적: < 5µs)
# Avg: 평균 지연 (이상적: < 10µs)
# Max: 최악 지연 (의료 로봇 목표: < 50µs)
```

### 부하 하에서의 최악 지연 측정

RTOS 지터는 시스템이 유휴 상태일 때는 매우 낮지만, 실제 운영 부하(AI 추론, 네트워크 트래픽, I/O)가 걸릴 때 올라간다. 반드시 최악 조건에서 측정해야 한다.

```bash
# 부하 생성 스크립트 (테스트 중 동시 실행)
stress-ng --cpu 8 --io 4 --vm 2 --vm-bytes 1G &  # CPU/IO 부하
iperf3 -c 192.168.10.1 -t 3600 -b 900M &          # 네트워크 부하 900Mbps
docker run -d ai-inference:latest &                # AI 추론 부하

# 이 상태에서 cyclictest 측정
# → 부하 없음 대비 최악 지연이 얼마나 증가하는지 확인
```

IEC 62304 검증 보고서에는 이 두 조건(유휴, 최대 부하)에서의 측정 결과가 모두 포함되어야 하고, "최대 부하에서도 최악 지연이 [X µs] 이내"를 증명해야 한다.

---

## 듀얼 커널 접근: Xenomai / RTAI

Xenomai와 RTAI는 Linux 커널과 실시간 커널을 병렬로 실행하는 이중 커널(dual-kernel) 방식이다. 실시간 태스크는 실시간 커널에서 돌고, Linux는 가장 낮은 우선순위로 동작한다.

```
이중 커널 구조 (Xenomai Cobalt):

하드웨어 인터럽트
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Xenomai Cobalt (실시간 커널, 최고 우선순위)                      │
│  관절 제어 태스크, POSIX 실시간 API                              │
│  인터럽트 직접 처리, 지터 < 10µs                                 │
├──────────────────────────────────────────────────────────────────┤
│  Linux 커널 (I-pipe / Dovetail 패치 적용)                        │
│  ROS 2, AI, 일반 애플리케이션                                    │
│  Linux가 Xenomai에 선점당함 (가장 낮은 우선순위)                  │
└──────────────────────────────────────────────────────────────────┘
```

**장점:** QNX에 근접한 실시간 성능(최악 지연 10~20µs)을 리눅스 생태계와 함께 제공.
**단점:** 두 커널 유지보수 부담, 최신 Linux 커널 따라가기 어려움, 안전 인증 없음.

현재 추세는 PREEMPT_RT + isolcpus 조합이 Xenomai를 대체하는 방향이다. PREEMPT_RT가 Linux 메인라인에 점차 통합되면서 별도 패치 유지 부담이 줄고 있기 때문이다.

---

## 참고 문헌

- Buttazzo, G. "Hard Real-Time Computing Systems" Springer (3rd ed., 2011)
- Linux PREEMPT_RT patch: https://wiki.linuxfoundation.org/realtime/start
- cyclictest: https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest
- QNX Neutrino RTOS Product Page: https://blackberry.qnx.com/
- FreeRTOS Documentation: https://www.freertos.org/
- Zephyr Project: https://www.zephyrproject.org/
- Linux Foundation ELISA (Safety Linux): https://elisa.tech/
- Reghenzani, F. et al. "The Real-Time Linux Kernel: A Survey" JRTIP (2019)

---

*관련: [2장 중앙집중형 컴퓨팅](./Centralized_Computing.md) | [1장 결정성 네트워크와 의료 안전 기초](../1장_Ethernet_기초/Determinism_and_Safety_Basics.md)*
