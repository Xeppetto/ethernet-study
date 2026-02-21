# AUTOSAR Classic vs Adaptive Platform

## 개요
AUTOSAR(AUTomotive Open System ARchitecture)는 차량용 소프트웨어 표준 플랫폼입니다. 기존의 Classic Platform이 마이크로컨트롤러(MCU) 기반 실시간 제어에 최적화되어 있다면, Adaptive Platform은 고성능 프로세서(AP)에서 동적이고 유연한 서비스 지향 아키텍처를 지원합니다.

Ethernet으로의 전환과 함께 Adaptive Platform의 중요성이 크게 높아졌으며, 두 플랫폼이 공존하는 환경을 이해하는 것이 중요합니다.

---

## 비교 요약

| 특성 | AUTOSAR Classic | AUTOSAR Adaptive |
|------|----------------|-----------------|
| 출시 | 2003년 | 2017년 (R17-03) |
| 대상 하드웨어 | MCU (수MHz ~ 수백MHz) | AP/MPU (ARM Cortex-A, x86) |
| 운영체제 | AUTOSAR OS (RTOS) | POSIX 기반 OS (Linux, QNX) |
| 통신 | COM, PDU (CAN, LIN, FlexRay) | SOME/IP, DDS (Ethernet) |
| 메모리 | 수KB ~ 수MB | 수백MB ~ 수GB |
| 런타임 설정 | 정적 (컴파일 타임) | 동적 (런타임 Service Discovery) |
| 소프트웨어 배포 | 정적 바이너리 | 동적 (OTA, 컨테이너) |
| 기능 안전 | ASIL-D 지원 | ASIL-B (현재) → ASIL-D 발전 중 |
| 주요 용도 | 파워트레인, 안전, Zone ECU | ADAS, 인포테인먼트, HPC |

---

## AUTOSAR Classic Platform

### 아키텍처
```
┌─────────────────────────────────────────────────────┐
│                  Application Layer                   │
│  SWC (Software Component) ↔ SWC ↔ SWC             │
├─────────────────────────────────────────────────────┤
│              RTE (Runtime Environment)               │
│  (SWC 간, SWC-BSW 간 통신 추상화)                   │
├─────────────────────────────────────────────────────┤
│           BSW (Basic Software)                       │
│  ┌──────────────────────────────────────────────┐   │
│  │  Services: OS, WDG, NvM, Dcm, Com           │   │
│  ├──────────────────────────────────────────────┤   │
│  │  ECU Abstraction: IoHwAb, Crc              │   │
│  ├──────────────────────────────────────────────┤   │
│  │  MCAL: CAN, LIN, SPI, ADC, PWM 드라이버    │   │
│  └──────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────┤
│                    Hardware                          │
│  MCU (Cortex-M, TriCore, RH850 등)                  │
└─────────────────────────────────────────────────────┘
```

### Classic Platform 특징
- **정적 구성**: ARXML 파일로 컴파일 타임에 모든 구성 확정
- **결정론적**: 실시간 태스크 스케줄링, 우선순위 기반 선점
- **ASIL-D 인증**: 최고 수준의 기능 안전 달성 가능
- **주요 통신**: COM 모듈 → PDUR → CanIf → CAN 드라이버

### SWC 인터페이스 예시 (ARXML 개념)
```xml
<!-- AUTOSAR Classic SWC 포트 정의 개념 -->
<SWC name="JointControl">
  <RequiredPorts>
    <DataReceiverPort name="angleCommand">
      <DataElement type="float32" semantics="angle_rad"/>
    </DataReceiverPort>
  </RequiredPorts>
  <ProvidedPorts>
    <DataSenderPort name="currentAngle">
      <DataElement type="float32" semantics="angle_rad"/>
    </DataSenderPort>
  </ProvidedPorts>
</SWC>
```

---

## AUTOSAR Adaptive Platform

### 아키텍처
```
┌───────────────────────────────────────────────────────────────┐
│                   Application Layer                            │
│  Adaptive Application (C++ 서비스)                            │
│  Service A ◄──── ara::com ────► Service B                    │
├───────────────────────────────────────────────────────────────┤
│              ARA (AUTOSAR Runtime for Adaptive)                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ara::com  │ │ara::exec │ │ara::diag │ │ara::crypto       │ │
│  │(SOME/IP) │ │(프로세스 │ │(진단)    │ │(보안)            │ │
│  │(DDS)     │ │ 관리)    │ │          │ │                  │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘ │
├───────────────────────────────────────────────────────────────┤
│              Foundation (OS / 미들웨어)                        │
│  POSIX OS (Linux/QNX) + C++14/17 Runtime                      │
├───────────────────────────────────────────────────────────────┤
│              Hardware                                          │
│  AP/MPU (ARM Cortex-A, x86, RISC-V)                          │
└───────────────────────────────────────────────────────────────┘
```

### Adaptive Platform 주요 Functional Cluster

| Cluster | 역할 |
|---------|------|
| ara::com | 통신 (SOME/IP, DDS, 로컬 IPC) |
| ara::exec | 프로세스 생명주기 관리 |
| ara::diag | UDS 진단 (DoIP 기반) |
| ara::crypto | 암호화/키 관리 |
| ara::iam | 신원 및 접근 관리 |
| ara::log | 구조화 로깅 |
| ara::phm | 플랫폼 헬스 관리 (Watchdog) |
| ara::nm | 네트워크 관리 |
| ara::tsync | 시간 동기화 (PTP 연동) |
| ara::per | 영구 데이터 저장 |

### C++ 서비스 예시 (ara::com)
```cpp
// Adaptive Application 서비스 구현 예시
#include "ara/com/sample/joint_control_skeleton.h"
using namespace ara::com::sample;

class JointControlService : public JointControlSkeleton {
public:
    // Method Handler: SetPosition
    ara::core::Future<SetPositionOutput>
    SetPosition(uint8_t joint_id, float angle) override {
        // 실제 조인트 제어 로직
        bool success = hardware_->MoveJoint(joint_id, angle);

        SetPositionOutput output;
        output.result = success ? 0x00 : 0x01;

        ara::core::Promise<SetPositionOutput> promise;
        promise.set_value(output);
        return promise.get_future();
    }

    // Event: PositionChanged 발행
    void PublishPosition(uint8_t joint_id, float current_angle) {
        PositionChanged.Send({joint_id, current_angle,
                             ara::core::SteadyClock::now()});
    }
};
```

---

## Classic ↔ Adaptive 공존 아키텍처

현실적인 시스템에서는 두 플랫폼이 함께 동작합니다.

```
┌──────────────────────────────────────────────────────────────────┐
│  의료 로봇 E/E 아키텍처                                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              HPC (Adaptive Platform)                        │ │
│  │  AI 추론, OTA 관리, 클라우드 연동, DoIP 진단                 │ │
│  │  SOME/IP SD, DDS, ara::com                                  │ │
│  └───────────────────────┬─────────────────────────────────────┘ │
│                          │ Ethernet (1Gbps, TSN)                │
│         ┌────────────────┼─────────────────┐                    │
│         │                │                 │                    │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐            │
│  │ Zone ECU A  │  │ Zone ECU B  │  │ Zone ECU C  │            │
│  │ (Classic)   │  │ (Classic)   │  │ (Classic)   │            │
│  │ 조인트 제어  │  │ 안전 감시   │  │ 전원 관리   │            │
│  │ ASIL-D      │  │ ASIL-D      │  │ ASIL-B      │            │
│  │ CAN FD      │  │ CAN FD      │  │ LIN         │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└──────────────────────────────────────────────────────────────────┘

통신 게이트웨이:
  Classic ↔ Adaptive: COM-to-SOME/IP Gateway
  CAN Signal → SOME/IP Event 변환
```

### COM-to-SOME/IP 게이트웨이 개념
```
Classic ECU (CAN 신호)         Gateway             Adaptive HPC (SOME/IP)
  joint_angle [CAN ID:0x100] ──────────────────► JointAngleEvent [SOME/IP]
  (10ms 주기, uint16, 0.01° 해상도)               (변경 시 발행, float32)

Gateway 변환 규칙 (ARXML 기반 자동 생성):
  CAN ID:    0x100
  Signal:    joint_angle_1, offset=0, length=16
  Scaling:   0.01°/LSB
  ↓ 변환
  SOME/IP:   Service 0x0101, Event 0x8001
  Payload:   float32, 라디안 단위
```

---

## 개발 도구 생태계

| 도구 | 제조사 | 지원 Platform | 용도 |
|------|--------|--------------|------|
| Vector DaVinci Developer | Vector | Classic | SWC 개발, RTE 생성 |
| EB tresos Studio | Elektrobit | Classic | BSW 설정 |
| Vector Adaptive DaVinci | Vector | Adaptive | ara::com 코드 생성 |
| CARIAD VW.OS | CARIAD (VW) | Adaptive | 차량 OS |
| Eclipse Leda | Eclipse | Adaptive | 오픈소스 SDV 스택 |

---

## 선택 가이드

```
Classic Platform 선택:
  ✓ 마이크로컨트롤러 기반 (128KB ~ 수MB RAM)
  ✓ ASIL-D 인증 필요
  ✓ 결정론적 실시간 제어 (< 1ms 주기)
  ✓ CAN/LIN/FlexRay 인터페이스
  ✓ 저전력 환경

Adaptive Platform 선택:
  ✓ 고성능 AP/MPU (수GB RAM)
  ✓ Ethernet 기반 통신 (SOME/IP, DDS)
  ✓ AI/ML 기능 통합 필요
  ✓ OTA 업데이트 필요
  ✓ 동적 서비스 구성 (Plug & Play)
  ✓ 클라우드 연동

공존 아키텍처 (현실):
  Zone ECU → Classic Platform (ASIL-D, 실시간 제어)
  HPC → Adaptive Platform (AI, OTA, 서비스)
  Gateway → Classic + Adaptive 브리지
```

---

## Reference
- [AUTOSAR Classic Platform Specification](https://www.autosar.org/standards/classic-platform/)
- [AUTOSAR Adaptive Platform Specification](https://www.autosar.org/standards/adaptive-platform/)
- [AUTOSAR Explained - YouTube](https://www.youtube.com/c/AutosarExplained)
- [ara::com API Specification](https://www.autosar.org/fileadmin/user_upload/standards/adaptive/17-03/AUTOSAR_SWS_CommunicationManagement.pdf)
