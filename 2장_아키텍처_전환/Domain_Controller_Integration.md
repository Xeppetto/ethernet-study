# Domain Controller 통합

## 완전 중앙집중이 당장 불가능한 이유

HPC 단일 시스템이 이상적이지만, 현실에서 모든 시스템이 즉시 HPC로 전환하지 못하는 이유가 있다. 안전 인증이 그 하나다. Zone ECU가 처리하던 ASIL-D 안전 기능을 HPC의 소프트웨어로 옮기면, 그 소프트웨어도 ASIL-D 인증을 받아야 한다. ASIL-D 인증에는 수년의 개발 기간과 독립 검증 비용이 든다. 때문에 현실적인 의료 로봇 아키텍처는 완전 중앙집중이 아니라 **도메인별 통합**을 거쳐 진화한다.

Domain Controller 통합은 비슷한 기능의 ECU들을 하나의 Domain Controller(DC)로 묶는 과도기적 단계다. 이것이 "과도기"라는 표현은 틀렸다—많은 의료 로봇 시스템에서 도메인 기반 아키텍처는 장기간 유지될 현실적인 선택이다.

---

## 의료 로봇 도메인 분류와 통신 구조

수술 로봇의 기능을 안전 특성과 실시간성 요구에 따라 분류하면 네 개의 도메인으로 구분된다.

```
┌─────────────────────────────────────────────────────────────────┐
│                     의료 로봇 시스템 전체                         │
│                                                                 │
│  ┌──────────────────┐   TSN 1GbE   ┌───────────────────────┐   │
│  │  Motion DC        │◄────────────►│  Vision DC             │   │
│  │                  │              │                       │   │
│  │  관절 제어 (12축) │              │  내시경 4K 스트리밍    │   │
│  │  역기구학 연산    │              │  AI 조직 인식         │   │
│  │  힘/임피던스 제어 │              │  수술 도구 추적       │   │
│  │  ASIL-B          │              │  ASIL-QM (비안전)     │   │
│  │  주기: 1kHz      │              │  주기: 30-60fps       │   │
│  └──────┬───────────┘              └───────────┬───────────┘   │
│         │ TSN 1GbE                             │ TSN 1GbE      │
│  ┌──────▼───────────┐              ┌───────────▼───────────┐   │
│  │  HMI DC           │◄────────────►│  Safety DC             │   │
│  │                  │  TSN 1GbE    │                       │   │
│  │  외과의 콘솔      │              │  E-Stop 인터락        │   │
│  │  햅틱 피드백      │              │  시스템 상태 감시     │   │
│  │  수술 영상 표시   │              │  안전 구역 위반 감지  │   │
│  │  ASIL-QM         │              │  ASIL-D              │   │
│  │  주기: 30fps     │              │  주기: 1ms           │   │
│  └──────────────────┘              └───────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**도메인별 안전 등급이 다른 이유:** Safety DC는 비상 정지 판단을 내리고 실행한다. 이 소프트웨어가 오작동하면 로봇이 잘못된 상황에서 계속 움직이거나, 반대로 정상 수술 중에 갑자기 정지할 수 있다—둘 다 환자에게 위험하다. ASIL-D가 필요하다. Vision DC의 AI 추론이 틀려도 의사가 최종 판단을 내리므로 ASIL-QM(인증 불필요)으로 충분하다.

---

## 도메인 간 통신: 프로토콜 선택의 근거

같은 Ethernet 백본을 공유하더라도 도메인마다 통신 패턴이 다르기 때문에 프로토콜을 다르게 선택한다.

```
도메인 간 주요 통신 경로:

Motion DC → HMI DC
  프로토콜: DDS (ROS 2)
  토픽: /joint_states (위치/속도/토크, 1kHz)
  QoS: BEST_EFFORT (손실 1~2개 허용), KEEP_LAST(1)
  VLAN 20, PCP=5, CBS 보장

HMI DC → Motion DC
  프로토콜: DDS (ROS 2)
  토픽: /joint_commands (목표 위치, 1kHz)
  QoS: RELIABLE, DEADLINE(1ms)
  VLAN 20, PCP=6, TAS 스케줄링

Safety DC ↔ 모든 DC
  프로토콜: SOME/IP over UDP (경량)
  서비스: SafetyStatusService (상태 보고)
  서비스: EmergencyStopService (E-Stop 명령)
  주기: 1ms heartbeat + 이벤트 트리거
  VLAN 10, PCP=7, TAS 최우선

Vision DC → HMI DC
  프로토콜: RTP/UDP (영상 스트리밍)
  포맷: H.265 또는 JPEG2000
  대역폭: 50~200Mbps (4K@30fps 압축 후)
  VLAN 30, PCP=3, CBS 대역폭 예약
```

**DDS DEADLINE QoS의 의미:** `/joint_commands` 토픽에 DEADLINE(1ms)을 설정하면, DDS 미들웨어가 1ms 이내에 데이터가 도착하지 않으면 `on_requested_deadline_missed` 콜백을 호출한다. Motion DC는 이 콜백에서 Hold-Last-Command 또는 안전 정지를 실행한다. 네트워크 지연이 DEADLINE을 초과하는 순간을 소프트웨어가 즉시 감지할 수 있다.

---

## Domain Controller 내부: 소프트웨어 스택 상세

```
Motion Domain Controller 소프트웨어 아키텍처:

┌─────────────────────────────────────────────────────────────────┐
│  Application Layer                                              │
│                                                                 │
│  ┌───────────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │ JointControlNode  │  │ ImpedanceCtrl│  │ CollisionDetect  │  │
│  │ (ROS 2 Lifecycle) │  │ (Force Ctrl) │  │ (Safety Monitor) │  │
│  └───────────────────┘  └──────────────┘  └─────────────────┘   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Middleware Layer                                               │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  ROS 2 (Humble) + CycloneDDS                              │  │
│  │  Publisher/Subscriber, Service/Client, Action Server       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  OS / Runtime                                                   │
│                                                                 │
│  QNX Neutrino 7.1 (Safety Partition) + Linux RT (일반 파티션)   │
│  CPU 0,1: 제어 루프 (isolcpus), CPU 2~7: ROS 2 노드           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  BSP / Drivers                                                  │
│                                                                 │
│  TSN Ethernet Driver (taprio, PTP hwtimestamp)                  │
│  CAN FD Driver (SocketCAN, 4채널)                               │
│  SPI/I2C (인코더, ADC)                                          │
│  GPIO (Emergency Stop 인터락)                                   │
└─────────────────────────────────────────────────────────────────┘
```

**ROS 2 Lifecycle Node의 의미:** Motion DC가 시작될 때 단순히 프로세스를 실행하는 것이 아니라, Lifecycle 상태 머신을 거쳐 단계적으로 초기화된다. `Unconfigured → Inactive → Active` 전환 시 하드웨어 상태를 확인하고, 이상이 있으면 `Unconfigured`로 돌아간다. 수술 시작 전 모든 DC가 `Active` 상태임을 Safety DC가 확인하는 체계가 가능해진다.

---

## 도메인 간 지연 예산

Motion DC에서 출발한 관절 상태 데이터가 외과의 콘솔에 표시되기까지 경로를 분석한다.

```
관절 상태 → HMI 표시까지 지연 분석 (목표: < 5ms 단방향)

┌─────────────────────────────────────────────────────────────────┐
│ Motion DC 내부 처리                                              │
│  인코더 읽기 (SPI): 100µs                                        │
│  제어 루프 연산: 200µs                                           │
│  ROS 2 publish: 50µs                                            │
│  DDS 직렬화: 30µs                                               │
│  NIC 큐잉 + TAS 스케줄링: 0~100µs                               │
│  소계: ~480µs                                                   │
├─────────────────────────────────────────────────────────────────┤
│ 네트워크 전송 (TSN 스위치 1홉)                                   │
│  직렬화 지연 (64B@1GbE): 0.672µs                                │
│  스위치 큐잉 (TAS 보장): 14~18µs                                │
│  소계: ~20µs                                                    │
├─────────────────────────────────────────────────────────────────┤
│ HMI DC 수신 처리                                                 │
│  NIC → 소켓 버퍼: 20µs                                          │
│  DDS 역직렬화: 30µs                                             │
│  ROS 2 callback: 50µs                                           │
│  렌더링 파이프라인: 2~4ms                                        │
│  소계: ~4.1ms                                                   │
├─────────────────────────────────────────────────────────────────┤
│ 총 지연: ~4.6ms (목표 5ms 이내 달성)                             │
└─────────────────────────────────────────────────────────────────┘

병목: 렌더링 파이프라인 (2~4ms)
→ HMI DC에 GPU가 있어야 실시간 렌더링 가능
→ 영상 표시와 상태 표시를 별도 파이프라인으로 처리
```

---

## 도메인 고장 시나리오와 FMEA

각 도메인의 고장이 전체 시스템에 미치는 영향과 완화 방법:

| 고장 도메인 | 즉각 영향 | 허용 시간 | 완화 조치 |
|---|---|---|---|
| Motion DC 완전 고장 | 관절 제어 불가 | 0ms | Hold-Last + 50ms 내 안전 정지 |
| Motion DC 소프트웨어 행 | 명령 중단 | 1ms (DEADLINE) | Watchdog → 재시작 또는 정지 |
| Vision DC 고장 | 내시경 영상 없음 | - | 수동 내시경으로 전환 (의사 판단) |
| HMI DC 고장 | 콘솔 응답 없음 | 100ms | Safety DC가 자동 정지 트리거 |
| Safety DC 고장 | E-Stop 기능 없음 | 즉각 | 하드웨어 인터락으로 즉시 전원 차단 |
| 네트워크 링크 단선 | 도메인 간 통신 끊김 | 50ms | FRER 이중화 경로로 전환 |

**Safety DC의 특별 취급:** Safety DC의 고장은 시스템 레벨에서 즉각적으로 처리되어야 한다. 이를 위해 Safety DC의 출력은 소프트웨어 신호만이 아니라 하드와이어드 릴레이를 통해 직접 전원 차단 회로에 연결된다. Safety DC 소프트웨어가 죽어도 하드웨어 watchdog이 릴레이를 열어 모터 전원을 차단한다.

---

## 인터도메인 IPC: 공유 메모리와 IVSHMEM

같은 HPC 내에서 여러 도메인(파티션)이 동작할 때, Ethernet 소켓을 통한 통신보다 공유 메모리가 훨씬 낮은 지연을 제공한다.

```
IVSHMEM (Inter-VM Shared Memory) 방식:

┌─────────────────┐     공유 물리 메모리      ┌─────────────────┐
│  파티션 A (QNX)  │ ◄──────────────────────► │  파티션 B (Linux)│
│  제어 루프       │      1µs 미만 IPC         │  ROS 2 노드      │
│  관절 상태 쓰기  │                          │  관절 상태 읽기  │
└─────────────────┘                          └─────────────────┘

구현: POSIX shm_open + mmap (Linux), shm_create (QNX)
동기화: 락프리 링버퍼 (멀티쓰기 없는 경우) 또는 원자적 교환
보호: IOMMU로 각 파티션이 자신의 공유 메모리 영역만 접근 가능

지연 비교:
  IVSHMEM 공유 메모리: < 1µs
  Unix Domain Socket: 5~15µs
  loopback TCP: 30~80µs
  물리 Ethernet 루프: 100~300µs
```

도메인 간 데이터 흐름 설계 시, 실시간 고빈도 데이터(1kHz 관절 상태)는 공유 메모리로, 비빈도 이벤트(상태 변화, 오류 보고)는 네트워크로 분리하면 최적이다.

---

## 마이그레이션 경로: ECU에서 Domain Controller로

기존 분산 ECU 시스템을 Domain Controller로 통합하는 것은 하룻밤에 이루어지지 않는다. 실용적인 마이그레이션 경로:

**Phase 1 (1~2년):** 기존 ECU 유지, 관찰만 추가. 새 Ethernet 백본을 병렬로 깔고, 모든 CAN 메시지를 게이트웨이를 통해 Ethernet으로 미러링. 트래픽 패턴과 타이밍을 2~3개월 분석.

**Phase 2 (2~3년):** 비안전 기능부터 DC로 이전. HMI, 로깅, 진단을 새 DC로 옮기고 기존 ECU는 Redundancy로 유지하며 결과 검증.

**Phase 3 (3~5년):** 안전 기능을 DC로 통합. IEC 62304 검증·인증을 거쳐 기존 ECU를 DC의 SW로 교체. 이 단계가 가장 오래 걸리고 가장 많은 비용이 드는 단계다.

---

## 참고 문헌

- AUTOSAR Adaptive Platform Foundation: Communication Management (ara::com)
- ROS 2 Lifecycle: https://design.ros2.org/articles/node_lifecycle.html
- CycloneDDS QoS Reference: Eclipse Cyclone DDS Documentation
- IEC 62304:2006/AMD1:2015: §5.3 Software Architectural Design
- ISO 14971:2019: §4.4 Risk Analysis
- VDA: AUTOSAR Adaptive Platform — Architecture Overview (2022)

---

*관련: [2장 중앙집중형 컴퓨팅](./Centralized_Computing.md) | [2장 SOA](./SOA.md)*
