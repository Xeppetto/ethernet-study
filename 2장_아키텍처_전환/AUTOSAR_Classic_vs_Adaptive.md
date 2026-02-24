# AUTOSAR Classic vs Adaptive Platform

## AUTOSAR가 존재하는 이유

AUTOSAR가 없던 시절, 자동차 부품 공급업체마다 독자적인 소프트웨어 구조를 사용했다. Bosch의 ABS ECU 소프트웨어를 Siemens의 엔진 ECU와 연결하려면, 두 회사 엔지니어가 수개월에 걸쳐 통신 프로토콜을 맞춰야 했다. 완성차 업체(OEM)는 수십 개 공급업체의 ECU를 통합하면서 같은 문제를 반복했다.

AUTOSAR(AUTomotive Open System ARchitecture)는 2003년 BMW, Bosch, Continental, DaimlerChrysler, Siemens VDO, Volkswagen이 모여 만든 업계 표준이다. "표준화된 인터페이스 위에서, 혁신적인 기능으로 경쟁하자"가 모토였다. 그 결과로 탄생한 Classic Platform은 이후 20년간 수백 만 개의 ECU에 탑재됐다.

그런데 Ethernet, AI, OTA 업데이트, 고성능 AP(Application Processor)가 등장하면서 Classic Platform의 설계 전제가 흔들렸다. 정적 구성, 빌드 타임 확정, 수MB RAM 환경을 위해 설계된 Classic은 수GB RAM, 동적 서비스, 런타임 업데이트를 요구하는 새 세계에서 뿌리를 내리기 어려웠다. 2017년 AUTOSAR는 이 간극을 메우기 위해 Adaptive Platform을 발표했다.

---

## 두 플랫폼의 핵심 차이

```
Classic Platform (2003~):              Adaptive Platform (2017~):
─────────────────────────────────────  ─────────────────────────────────────
대상 하드웨어: MCU (Cortex-M, TriCore) 대상 하드웨어: AP/MPU (Cortex-A, x86)
메모리: 수KB ~ 수MB                    메모리: 수백MB ~ 수GB
OS: AUTOSAR OS (경량 RTOS)             OS: POSIX (Linux, QNX)
구성: 정적 (컴파일 타임 확정)           구성: 동적 (런타임 Service Discovery)
통신: CAN, LIN, FlexRay (PDU 기반)     통신: SOME/IP, DDS (Ethernet)
소프트웨어 배포: 정적 바이너리          소프트웨어 배포: 동적 (OTA, 컨테이너)
기능 안전: ASIL-D 달성 가능            기능 안전: ASIL-B (ASIL-D 개발 중)
언어: C (MISRA-C:2012)                 언어: C++14/17 (ara:: 네임스페이스)
핵심 장점: 결정론적, 안전 인증 성숙    핵심 장점: 유연성, AI/OTA, Ethernet
```

두 플랫폼은 경쟁 관계가 아니다. 서로 다른 문제를 해결하고, 실제 시스템에서는 공존한다.

---

## AUTOSAR Classic Platform 상세

### 계층 구조

```
AUTOSAR Classic 소프트웨어 계층:

┌──────────────────────────────────────────────────────────────────┐
│  Application Layer                                               │
│  SWC A ←──────── RTE ──────────► SWC B                          │
│  (Joint Control)  (인터페이스 추상화) (Safety Monitor)           │
├──────────────────────────────────────────────────────────────────┤
│  Runtime Environment (RTE)                                       │
│  SWC 간 통신, SWC↔BSW 통신 중개                                   │
│  ARXML 설정에서 자동 생성됨 (Vector DaVinci, EB tresos 사용)     │
├──────────────────────────────────────────────────────────────────┤
│  Basic Software (BSW)                                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Services: Com, PduR, CanIf, NvM, Dem, Dcm, WdgM, Os        │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ ECU Abstraction: IoHwAb (I/O 추상화), Crc, E2E             │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │ MCAL: CAN, SPI, ADC, PWM, GPT 드라이버 (칩 종속)           │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│  Hardware: MCU (Infineon TriCore, NXP S32, Renesas RH850)        │
└──────────────────────────────────────────────────────────────────┘
```

**E2E(End-to-End) 보호:** Classic Platform의 Com 모듈은 E2E 보호 프로파일을 지원한다. 각 PDU(Protocol Data Unit)에 CRC, 카운터, 데이터 ID를 추가해 데이터 손상, 손실, 순서 오류를 감지한다. IEC 62304나 ISO 26262 ASIL-D를 달성하기 위한 필수 메커니즘이다.

```c
// AUTOSAR E2E Profile 2 사용 예시 (개념 코드)
// Com 모듈이 자동으로 추가하는 헤더 (ARXML 설정)
// Length: 1 byte (Counter), 2 bytes (CRC-16)
// → PDU 크기가 3 byte 늘어남, 하지만 안전 보호 확보

// SWC에서는 정상 데이터만 읽음 (E2E 검사는 BSW가 자동 수행)
Std_ReturnType status;
JointAngle angle;
status = Rte_Read_JointControlPort_angle(&angle);
if (status == RTE_E_OK) {
    // 정상 데이터
} else if (status == RTE_E_COM_STOPPED) {
    // 통신 오류 → 안전 조치
}
```

### AUTOSAR OS의 실시간 보장

AUTOSAR OS는 OSEK/VDX 표준 기반의 경량 RTOS다. 태스크(Task)가 컴파일 타임에 정의되고, 우선순위가 고정되며, 최악 실행 시간(WCET)이 정적 분석으로 계산될 수 있다.

```
AUTOSAR OS 태스크 설계 예시 (관절 제어 ECU):

Task Name        Priority  Period   WCET   Stack
─────────────────────────────────────────────────────────
SafetyMonitor    255       1ms      50µs   512B
JointControl     200       1ms      200µs  1KB
EncoderRead      100       1ms      30µs   256B
CommunicationTx  50        1ms      20µs   256B
DiagnosticTask   10        100ms    5ms    2KB
─────────────────────────────────────────────────────────
CPU 이용률 = Σ(WCET/Period) = (50+200+30+20)/1000 + 5/100
           = 30% (1ms 주기) + 5% (100ms 주기) = 35%
→ 여유 65%, 안전 요구사항 충족
```

---

## AUTOSAR Adaptive Platform 상세

### ARA 기반 아키텍처

```
AUTOSAR Adaptive 소프트웨어 계층:

┌──────────────────────────────────────────────────────────────────┐
│  Adaptive Application (C++ 서비스들)                              │
│  Service A ◄──── ara::com ────► Service B                        │
│  (관절 제어)      (미들웨어 API)  (AI 추론)                       │
├──────────────────────────────────────────────────────────────────┤
│  AUTOSAR Runtime for Adaptive (ARA)                               │
│                                                                  │
│  ara::com    통신 (SOME/IP, DDS 추상화)                          │
│  ara::exec   프로세스 생명주기 (시작/종료/재시작)                 │
│  ara::diag   진단 (UDS over DoIP)                                │
│  ara::crypto 암호화, 키 관리, TLS                                │
│  ara::iam    신원 및 접근 제어                                    │
│  ara::phm    플랫폼 헬스 관리 (Watchdog 통합)                    │
│  ara::tsync  시간 동기화 (PTP/gPTP 연동)                         │
│  ara::log    구조화 로깅 (DLT 프로토콜)                           │
│  ara::per    영구 데이터 저장 (Key-Value Store)                   │
│  ara::nm     네트워크 관리 (슬립/웨이크업)                        │
├──────────────────────────────────────────────────────────────────┤
│  Foundation (POSIX OS + 미들웨어)                                 │
│  Linux (Yocto) 또는 QNX Neutrino + SOME/IP 스택 (vsomeip)       │
├──────────────────────────────────────────────────────────────────┤
│  Hardware: AP/MPU (ARM Cortex-A, x86, RISC-V)                    │
└──────────────────────────────────────────────────────────────────┘
```

**ara::phm (Platform Health Management):** Classic의 WdgM(Watchdog Manager)에 대응하는 Adaptive 모듈이다. 각 Adaptive Application이 주기적으로 Checkpoint를 보고하고, ara::phm이 타임아웃을 감지하면 해당 프로세스를 재시작하거나 Fail-Safe를 실행한다. 의료 로봇에서 관절 제어 서비스가 응답하지 않으면 ara::phm이 이를 감지한다.

```cpp
// ara::com을 이용한 SOME/IP 서비스 구현
#include "ara/com/sample/joint_control_skeleton.h"
#include "ara/phm/supervised_entity.h"

class JointControlService : public ara::com::sample::JointControlSkeleton {
    ara::phm::SupervisedEntity supervised_;  // Health Management 등록

public:
    JointControlService()
        : supervised_("JointControl", std::chrono::milliseconds(10)) {}

    // Method: SetPosition
    ara::core::Future<SetPositionOutput>
    SetPosition(uint8_t joint_id, float angle_rad) override {
        supervised_.ReportCheckpoint(CHECKPOINT_SET_POSITION);  // PHM 보고

        bool ok = hardware_driver_.MoveJoint(joint_id, angle_rad);

        SetPositionOutput out;
        out.result = ok ? 0x00 : 0x01;
        ara::core::Promise<SetPositionOutput> p;
        p.set_value(out);
        return p.get_future();
    }
};

// ara::exec: 프로세스 실행 환경 진입점
int main() {
    ara::core::Initialize();
    auto service = std::make_shared<JointControlService>();
    service->OfferService();  // SD에 서비스 광고

    ara::core::RunApplicationLoop();  // 이벤트 루프
    return 0;
}
```

### ara::tsync: 시간 동기화와 TSN 연동

Adaptive Platform의 ara::tsync API는 하드웨어 PTP 클록에 접근해 나노초 정밀도의 타임스탬프를 제공한다. TSN TAS 스케줄링의 기반이 된다.

```cpp
#include "ara/tsync/time_sync.h"

// 현재 동기화된 네트워크 시각 읽기
ara::tsync::SynchronizedTimeBase ts_base;
auto current_time = ts_base.GetCurrentTime();

// 타임스탬프가 포함된 관절 상태 발행
JointStateEvent event;
event.timestamp = current_time;  // PTP 동기화된 시각
event.position = encoder_.Read();
PositionChanged.Send(event);
// → 수신 측이 동일 시간축에서 지연을 정확히 계산 가능
```

---

## 공존 아키텍처: 실제 의료 로봇의 선택

현실적인 의료 로봇은 두 플랫폼을 계층적으로 배치한다.

```
의료 로봇 E/E 아키텍처 (공존 설계):

┌─────────────────────────────────────────────────────────────────┐
│  HPC (Adaptive Platform)                                        │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ AI 추론 서비스, OTA 클라이언트, 원격 진단, ROS 2 플래닝    │ │
│  │ ara::com (SOME/IP + DDS), ara::exec, ara::phm              │ │
│  │ Ubuntu 22.04 + QNX Hypervisor                              │ │
│  └───────────────────────────────┬──────────────────────────── ┘ │
│                                  │ TSN Ethernet 1GbE (VLAN 분리) │
├──────────────────────────────────┼──────────────────────────────┤
│  Zone ECU × 3 (Classic Platform) │                              │
│  ┌────────────────────────────────▼───────────────────────────┐ │
│  │ Zone ECU A (로봇 팔 #1)                                     │ │
│  │ AUTOSAR Classic (R21-11)                                   │ │
│  │ SWC: JointControl, SafetyMonitor, E2E보호                  │ │
│  │ ASIL-D, QNX Neutrino RTOS 격리                             │ │
│  │ CAN FD DownLink (관절 모터 드라이버)                        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  COM→SOME/IP Gateway (Classic↔Adaptive 브리지)                 │
│  CAN FD 신호 → SOME/IP Event 자동 변환 (ARXML 기반)             │
└─────────────────────────────────────────────────────────────────┘
```

**COM-to-SOME/IP 변환 세부 설계:**

```
Classic ECU (CAN ID 0x200)          Gateway                Adaptive HPC
  joint_angle_1: uint16, 0.01°/LSB ─────────────────────► Service 0x0101
  joint_velocity_1: int16, 0.001rpm/LSB                    Event 0x8001
  joint_torque_1: int16, 0.01 Nm/LSB                       Payload: float32×3

변환 규칙 (ARXML SomeIpXf 모듈):
  [CAN 신호 → PDU → PDUR → SomeIpXf → SOME/IP]
  scaling: angle = raw_value × 0.01 × π/180  (degree → radian)
  endian: CAN (big-endian) → SOME/IP (big-endian, 동일)
  타이밍: CAN 10ms 주기 → SOME/IP Event on-change + 주기 최대 100ms
```

---

## IEC 62304 관점에서의 플랫폼 선택

의료기기 소프트웨어로서 두 플랫폼은 다른 인증 성숙도를 가진다.

**Classic Platform:** 2003년부터 20년간 자동차 안전 인증(ISO 26262 ASIL-D)을 받아왔고, 그 경험이 의료 분야에 전용 가능하다. IEC 62304 Class C 인증을 받은 Classic Platform BSW 제품(예: Vector MICROSAR, EB tresos)이 존재한다. Classic BSW를 SOUP(Software of Unknown Provenance)로 사용할 때도 SOUP 목록화 및 검증 절차가 AUTOSAR 생태계에서 잘 정의되어 있다.

**Adaptive Platform:** IEC 62304 Class C 경험은 Classic보다 적다. 그러나 ara::phm의 Watchdog, ara::crypto의 HSM 통합, ara::tsync의 시간 추적 같은 기능들이 의료 안전 요구사항을 염두에 두고 설계되었다. 2024년 기준으로 Adaptive Platform 기반 의료기기 인증 사례가 점증하고 있다.

**실용적 결론:** 새로 설계하는 수술 로봇에서 안전 임계(ASIL-D/IEC 62304 Class C) 관절 제어는 Classic Platform이나 검증된 RTOS로 구현하고, AI·OTA·원격 진단은 Adaptive Platform으로 구현하는 혼합 접근이 현재 최선이다.

---

## 개발 도구 생태계

| 도구 | 제조사 | 플랫폼 | 주요 용도 |
|---|---|---|---|
| Vector DaVinci Developer | Vector | Classic | SWC 개발, ARXML, RTE 코드 생성 |
| EB tresos Studio | Elektrobit | Classic | BSW 모듈 설정, MCAL |
| Vector Adaptive DaVinci | Vector | Adaptive | ara::com 코드 생성, 서비스 설계 |
| ETAS ISOLAR | ETAS | Classic/Adaptive | 통합 개발 환경 |
| AUTOSAR Builder | Arccore | Classic | 오픈소스 기반 Classic 도구 |
| Eclipse Leda | Eclipse SDV | Adaptive | 오픈소스 SDV 스택 |

**주의:** AUTOSAR 도구 라이선스 비용이 상당하다. 중소규모 의료기기 업체에서는 Eclipse 기반 오픈소스 도구(Eclipse ARXML, ara SDK)와 상용 도구를 선택적으로 조합하는 것이 현실적이다.

---

## 마이그레이션: Classic에서 Adaptive로

기존 Classic 기반 의료 로봇을 Adaptive로 완전 전환하는 것은 급진적이고 위험하다. 점진적 마이그레이션이 현실적이다.

**Step 1:** Adaptive HPC를 추가해 Classic 시스템 옆에 배치. COM→SOME/IP 게이트웨이를 통해 Classic 데이터를 HPC에서 읽기만 한다. Adaptive 서비스(AI 추론, OTA 관리)만 HPC에서 실행.

**Step 2:** 비안전 기능(HMI, 로깅, 원격 진단)을 Adaptive로 이전. Classic ECU는 안전 제어만 담당.

**Step 3:** (수년 후) Adaptive Platform의 안전 인증이 성숙하면, ASIL-B 기능 일부를 Adaptive로 이전. ASIL-D 기능은 Classic 또는 독립 Safety MCU로 유지.

---

## 참고 문헌

- AUTOSAR Classic Platform R21-11 Specification Suite: https://www.autosar.org
- AUTOSAR Adaptive Platform R22-11 Specification Suite: https://www.autosar.org
- Vector MICROSAR: https://www.vector.com/int/en/products/products-a-z/software/microsar/
- IEC 62304:2006/AMD1:2015 §8: Software Configuration Management
- Zeller, M. et al. "Safety in AUTOSAR Adaptive Platform" IEEE ETFA (2021)
- Eclipse Leda: https://eclipse-leda.github.io/leda/

---

*관련: [2장 SOA](./SOA.md) | [2장 SDV](./SDV.md)*
