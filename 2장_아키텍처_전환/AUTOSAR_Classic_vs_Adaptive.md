# AUTOSAR Classic vs Adaptive Platform

## 개요
AUTOSAR(AUTomotive Open System ARchitecture)는 차량용 소프트웨어 표준 플랫폼입니다. 기존의 Classic Platform이 마이크로컨트롤러(MCU) 기반 실시간 제어에 최적화되어 있다면, Adaptive Platform은 고성능 프로세서(AP)에서 동적이고 유연한 서비스 지향 아키텍처를 지원합니다.

Ethernet으로의 전환과 함께 Adaptive Platform의 중요성이 크게 높아졌으며, 두 플랫폼이 공존하는 환경을 이해하는 것이 중요합니다.

---

## 플랫폼 비교

| 특성 | AUTOSAR Classic | AUTOSAR Adaptive |
|------|----------------|-----------------|
| 최초 출시 | 2003년 (R4.x 현재) | 2017년 (R17-03) |
| 대상 하드웨어 | MCU (수MHz ~ 수백MHz) | AP/MPU (ARM Cortex-A, x86) |
| 운영체제 | AUTOSAR OS (OSEK 기반 RTOS) | POSIX 기반 OS (Linux, QNX) |
| 통신 표준 | COM/PDU 라우터 (CAN, LIN, FlexRay) | SOME/IP, DDS (Ethernet) |
| 메모리 요구량 | 수KB ~ 수MB | 수백MB ~ 수GB |
| 런타임 설정 | 정적 (컴파일 타임 ARXML 확정) | 동적 (런타임 Service Discovery) |
| 소프트웨어 배포 | 정적 바이너리 (플래시) | 동적 (OTA, 컨테이너 기반) |
| 기능 안전 | ASIL-D (완전 지원) | ASIL-B (현재) → ASIL-D 발전 중 |
| 프로그래밍 언어 | MISRA C, C++ (제한적) | C++14/17, Rust (검토 중) |
| 빌드 방식 | ARXML 기반 코드 생성 (RTE 자동) | aracom 코드 생성 + CMake |
| 주요 용도 | 파워트레인, 안전, Zone ECU | ADAS, 인포테인먼트, HPC |

---

## AUTOSAR Classic Platform

### 계층 아키텍처
```
┌─────────────────────────────────────────────────────┐
│                  Application Layer                   │
│  SWC (Software Component) ↔ SWC ↔ SWC             │
│  Port-Based Interface: Sender/Receiver, Client/Server│
├─────────────────────────────────────────────────────┤
│              RTE (Runtime Environment)               │
│  (SWC 간, SWC-BSW 간 통신 추상화 레이어)             │
│  ARXML에서 자동 생성됨 (코드 생성 도구)              │
├─────────────────────────────────────────────────────┤
│           BSW (Basic Software)                       │
│  ┌───────────────────────────────────────────────┐   │
│  │ Services:  OS, WDG, NvM, Dcm, ComM, SecOC    │   │
│  ├───────────────────────────────────────────────┤   │
│  │ ECU Abstraction: IoHwAb, Crc, E2E            │   │
│  ├───────────────────────────────────────────────┤   │
│  │ MCAL: CAN, LIN, SPI, ADC, PWM, Eth 드라이버  │   │
│  └───────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────┤
│                    Hardware (MCU)                    │
│  Cortex-M7 (STM32), TriCore (Infineon), RH850        │
└─────────────────────────────────────────────────────┘
```

### BSW 주요 모듈 상세

| 모듈 그룹 | 모듈명 | 역할 |
|-----------|--------|------|
| OS Services | Os | OSEK 기반 Task/ISR 스케줄링 |
| Communication | Com, PduR, CanIf, CanSM | CAN 통신 스택 |
| Diagnostics | Dcm, Dem, Fim | UDS 진단, DTC 관리 |
| Memory | MemIf, Fee, Fls, NvM | 비휘발성 메모리 추상화 |
| System | WdgM, WdgIf | Watchdog 관리 |
| Security | SecOC, Csm, KeyM | 메시지 인증, 키 관리 |
| E/E Management | EcuM, ComM | ECU 전원, 통신 관리 |

### SWC 인터페이스 예시 (ARXML 개념)
```xml
<!-- AUTOSAR Classic SWC 포트 정의 -->
<SWC name="JointControl">
  <RequiredPorts>
    <DataReceiverPort name="AngleCommand">
      <DataElement type="float32" semantics="angle_rad"
                   init="0.0" invalidation="timeout_50ms"/>
    </DataReceiverPort>
    <ServerCallPoint name="SafetyCheck" timeout_ms="5"/>
  </RequiredPorts>
  <ProvidedPorts>
    <DataSenderPort name="CurrentAngle">
      <DataElement type="float32" init="0.0"/>
    </DataSenderPort>
    <ServicePort name="SetPosition"/>
  </ProvidedPorts>
  <InternalBehavior>
    <RunableEntity period="1ms" category="TIMING"/>
  </InternalBehavior>
</SWC>
```

### Classic RTE 호출 패턴
```c
/* AUTOSAR Classic - Rte_Call / Rte_Write 예시 (자동 생성된 RTE 사용) */
#include "Rte_JointControl.h"

FUNC(void, JOINTCONTROL_CODE) JointControl_MainFunction(void) {
    float32 angle_cmd = 0.0f;
    float32 current_angle = 0.0f;

    /* Sender/Receiver 포트로 데이터 수신 */
    Std_ReturnType ret = Rte_Read_AngleCommand_Data(&angle_cmd);

    if (ret == RTE_E_OK) {
        current_angle = HW_GetJointAngle();
        float32 error = angle_cmd - current_angle;

        /* PID 제어 */
        float32 torque = Pid_Calculate(&pid_ctx, error);
        HW_SetMotorTorque(torque);

        /* 결과 발행 */
        Rte_Write_CurrentAngle_Data(&current_angle);
    }

    /* Safety 서비스 호출 */
    Rte_Call_SafetyCheck_IsOperationAllowed(JOINT_MOTION_OP);
}
```

---

## AUTOSAR Adaptive Platform

### 계층 아키텍처
```
┌─────────────────────────────────────────────────────────────────┐
│                   Application Layer                              │
│  Adaptive Application (C++17 서비스 / ROS 2 노드)               │
│  Service A ◄──── ara::com (SOME/IP / DDS) ────► Service B       │
├─────────────────────────────────────────────────────────────────┤
│              ARA (AUTOSAR Runtime for Adaptive)                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ │
│  │ara::com │ │ara::exec│ │ara::diag│ │ara::crypto│ │ara::phm  │ │
│  │SOME/IP  │ │프로세스 │ │DoIP/UDS │ │암호화/키  │ │Watchdog  │ │
│  │DDS      │ │생명주기 │ │진단     │ │관리       │ │헬스모니터│ │
│  └─────────┘ └─────────┘ └─────────┘ └──────────┘ └──────────┘ │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐             │
│  │ara::iam │ │ara::log │ │ara::nm  │ │ara::tsync│             │
│  │접근제어 │ │구조화   │ │네트워크 │ │PTP 시간  │             │
│  │권한관리 │ │로깅     │ │관리     │ │동기화    │             │
│  └─────────┘ └─────────┘ └─────────┘ └──────────┘             │
├─────────────────────────────────────────────────────────────────┤
│              Foundation (OS / 미들웨어)                          │
│  POSIX OS (Linux Yocto / QNX Neutrino) + C++17 Runtime          │
├─────────────────────────────────────────────────────────────────┤
│              Hardware (AP/MPU)                                   │
│  ARM Cortex-A78AE, x86 (Intel Atom), RISC-V                     │
└─────────────────────────────────────────────────────────────────┘
```

### Adaptive Platform 주요 Functional Cluster

| Cluster | Namespace | 역할 | 비고 |
|---------|-----------|------|------|
| Communication Management | ara::com | SOME/IP, DDS, 로컬 IPC | vsomeip, CycloneDDS 백엔드 |
| Execution Management | ara::exec | 프로세스 생명주기, 의존성 | systemd 유사 |
| Diagnostics | ara::diag | UDS over DoIP, DTC 관리 | ISO 14229 기반 |
| Cryptography | ara::crypto | AES/RSA/ECC, 키 저장소 | HSM 연동 |
| Identity & Access Mgmt | ara::iam | 서비스 인가, RBAC | |
| Log and Trace | ara::log | 구조화 로깅, 원격 전송 | DLT 호환 |
| Platform Health Mgmt | ara::phm | Watchdog, 생존 확인 | |
| Network Management | ara::nm | 버스 슬립/웨이크업 | |
| Time Synchronization | ara::tsync | gPTP (IEEE 802.1AS) 연동 | |
| Persistent Storage | ara::per | Key-Value / 파일 스토어 | |
| Update & Config Mgmt | ara::ucm | OTA 업데이트 수신/적용 | UPTANE 연동 |

### C++ 서비스 Skeleton 구현 (ara::com)
```cpp
#include "ara/com/sample/joint_control_skeleton.h"
#include "ara/core/future.h"
#include <cmath>

using namespace ara::com::sample;

class JointControlService : public JointControlSkeleton {
public:
    JointControlService(ara::core::InstanceSpecifier spec)
        : JointControlSkeleton(spec) {}

    /* Method Handler: SetPosition (Client/Server 패턴) */
    ara::core::Future<SetPositionOutput>
    SetPosition(uint8_t joint_id, float angle_rad) override {
        ara::core::Promise<SetPositionOutput> promise;
        SetPositionOutput output;

        if (joint_id >= MAX_JOINTS || std::abs(angle_rad) > M_PI) {
            output.error_code = 0x01;  /* E_INVALID_PARAMETER */
        } else {
            bool ok = hardware_->MoveJoint(joint_id, angle_rad);
            output.error_code = ok ? 0x00 : 0x02;
        }

        promise.set_value(output);
        return promise.get_future();
    }

    /* Event 발행: PositionChanged (Publish/Subscribe 패턴) */
    void PublishPositionUpdate(uint8_t joint_id, float current_rad) {
        PositionChangedEvent event_data;
        event_data.joint_id = joint_id;
        event_data.angle_rad = current_rad;
        event_data.timestamp = ara::core::SteadyClock::now();
        PositionChanged.Send(event_data);
    }

    /* Field: 최대 속도 (Getter/Setter/Notifier) */
    ara::core::Future<GetMaxVelocityOutput>
    GetMaxVelocity() override {
        ara::core::Promise<GetMaxVelocityOutput> p;
        p.set_value({max_velocity_});
        return p.get_future();
    }

private:
    static constexpr uint8_t MAX_JOINTS = 7;
    float max_velocity_ = 1.5f;  /* m/s */
};
```

### Execution Manifest (프로세스 선언)
```json
{
  "executableRef": "JointControlExe",
  "startupConfigs": [
    {
      "startupOption": "automatic",
      "functionGroupStates": ["Running"],
      "startupArguments": ["--instance=0", "--config=/opt/robot/joint.json"]
    }
  ],
  "resourceGroups": [
    {
      "cpuAffinity": [2, 3],
      "schedulingPolicy": "SCHED_FIFO",
      "schedulingPriority": 80,
      "memoryLimit": "256MB"
    }
  ]
}
```

---

## Classic ↔ Adaptive 공존 아키텍처

현실적인 시스템에서는 두 플랫폼이 함께 동작합니다.

```
┌──────────────────────────────────────────────────────────────────┐
│  의료 로봇 E/E 아키텍처 (Hybrid 공존 구성)                        │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              HPC (Adaptive Platform, Linux + QNX)           │ │
│  │  AI 추론, OTA 관리, 클라우드 연동, DoIP 진단                 │ │
│  │  ara::com (SOME/IP SD / DDS), ara::ucm (OTA)                │ │
│  └───────────────────────┬─────────────────────────────────────┘ │
│                          │ TSN Ethernet 백본 (1Gbps)             │
│         ┌────────────────┼─────────────────┐                    │
│         │                │                 │                    │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐            │
│  │ Zone ECU A  │  │ Zone ECU B  │  │ Zone ECU C  │            │
│  │ (Classic)   │  │ (Classic)   │  │ (Classic)   │            │
│  │ 조인트 제어  │  │ 안전 감시   │  │ 전원 관리   │            │
│  │ ASIL-D      │  │ ASIL-D      │  │ ASIL-B      │            │
│  │ CAN FD      │  │ CAN FD      │  │ LIN         │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
│  Gateway: Classic COM 신호 → Adaptive SOME/IP Event 변환         │
└──────────────────────────────────────────────────────────────────┘
```

### COM-to-SOME/IP 게이트웨이 변환 규칙
```
Classic ECU (CAN PDU)            Gateway (자동 변환)      Adaptive HPC (SOME/IP)
  ID:0x100, joint_angle_1  ──────────────────────────► Service 0x0101
  Signal: uint16, 0.01°/LSB                              Event 0x8001
  주기: 10ms                                              payload: float32 (rad)
                                                          발행: 변경 시 (이벤트)

  ID:0x200, tool_force      ──────────────────────────► Service 0x0102
  Signal: uint16, 0.1N/LSB                               Event 0x8001
  주기: 5ms                                               payload: float32 (N)
```

---

## 개발 도구 생태계

| 도구 | 제조사 | 지원 Platform | 용도 |
|------|--------|--------------|------|
| DaVinci Developer | Vector | Classic | SWC 개발, RTE 생성 |
| DaVinci Configurator | Vector | Classic | BSW 모듈 설정 |
| EB tresos Studio | Elektrobit | Classic | BSW 전체 설정 |
| Adaptive DaVinci | Vector | Adaptive | ara::com 코드 생성 |
| EB corbos Studio | Elektrobit | Adaptive | 서비스 모델링 |
| CARIAD VW.OS | CARIAD (VW) | Adaptive | 차량 OS 배포판 |
| Eclipse Leda | Eclipse SDV | Adaptive | 오픈소스 SDV 스택 |
| Capicxx-core-tools | COVESA | Adaptive | Franca IDL → ara::com |

---

## 빌드 및 배포 파이프라인 비교

### Classic Platform 빌드 흐름
```
ARXML 모델 (SWC, BSW 설정)
      │
      ▼
RTE 코드 생성기 (DaVinci 등)
      │ (Rte_*.c, Rte_*.h 자동 생성)
      ▼
크로스 컴파일러 (GCC ARM-none-eabi / HighTec TriCore)
      │ MISRA-C 검사 포함
      ▼
링커 → 플래시 이미지 (.hex / .s19)
      │
      ▼
UDS/XCP 다운로드 → ECU 플래시 (CAN Bootloader)
```

### Adaptive Platform 빌드 흐름
```
Service Interface 정의 (.fidl / .fdepl)
      │
      ▼
capicxx / ara-gen 코드 생성 (Skeleton/Proxy C++)
      │
      ▼
CMake 빌드 (aarch64-linux-gnu cross-compile)
      │ clang-tidy + cppcheck + AddressSanitizer
      ▼
OCI 컨테이너 이미지 (.tar.gz) 또는 Debian 패키지
      │
      ▼
OTA 서버 → ara::ucm → Adaptive App 업데이트
```

---

## 플랫폼 선택 가이드

```
Classic Platform 선택 기준:
  ✓ 마이크로컨트롤러 기반 (128KB ~ 수MB RAM)
  ✓ ASIL-D 인증 필요 (최고 수준 기능 안전)
  ✓ 결정론적 실시간 제어 (< 1ms 주기)
  ✓ CAN FD / LIN / FlexRay 인터페이스
  ✓ 저전력 환경 (배터리, 슬립 모드 필요)
  ✓ 정적 구성 (런타임 변경 불필요)

Adaptive Platform 선택 기준:
  ✓ 고성능 AP/MPU (수GB RAM)
  ✓ Ethernet 기반 통신 (SOME/IP, DDS)
  ✓ AI/ML 기능 통합 필요 (NPU/GPU)
  ✓ OTA 무선 업데이트 필요
  ✓ 동적 서비스 구성 (Plug & Play)
  ✓ 클라우드 / 원격 연동 필요

공존 아키텍처 (현실적 배치):
  Zone ECU → Classic Platform (ASIL-D, 실시간 제어)
  HPC → Adaptive Platform (AI, OTA, 서비스)
  Gateway ECU → Classic + Adaptive 브리지 역할
```

---

## 의료 로봇 적용: AUTOSAR 플랫폼 분담

```
수술 로봇 AUTOSAR 분담 구조

Classic Platform (Zone ECU):
  ┌─────────────────────────────────────────────────────┐
  │  Joint Control SWC → ASIL-D → 1ms 주기              │
  │  Torque Limiter SWC → ASIL-D → 5ms 주기             │
  │  E-Stop Monitor SWC → ASIL-D → 1ms 주기             │
  │  SecOC: CMAC-AES-128 메시지 인증 (CAN FD)            │
  │  E2E Profile 2: 체크섬 + 카운터 보호                  │
  └─────────────────────────────────────────────────────┘

Adaptive Platform (HPC):
  ┌─────────────────────────────────────────────────────┐
  │  JointControlService (ara::com, SOME/IP)             │
  │  SurgicalAI Service (TensorRT 추론, 50ms 주기)       │
  │  OTA Manager (ara::ucm, UPTANE 검증)                 │
  │  Diagnostics (ara::diag, DoIP/UDS)                  │
  │  Cloud Telemetry (MQTT TLS 1.3)                      │
  └─────────────────────────────────────────────────────┘

규제 매핑:
  IEC 62304 Class C → Classic + Adaptive 전체 적용
  ISO 26262 ASIL-D  → Classic Platform (Zone ECU)
  ISO/SAE 21434     → 전체 시스템 (TARA 수행)
```

---

## Reference
- [AUTOSAR Classic Platform Specification](https://www.autosar.org/standards/classic-platform/)
- [AUTOSAR Adaptive Platform Specification](https://www.autosar.org/standards/adaptive-platform/)
- [AUTOSAR Adaptive R22-11 Release Notes](https://www.autosar.org/fileadmin/user_upload/standards/adaptive/22-11/AUTOSAR_PRS_AdaptivePlatformCommunicationProtocol.pdf)
- [ara::com API Specification SWS_CM](https://www.autosar.org/fileadmin/user_upload/standards/adaptive/17-03/AUTOSAR_SWS_CommunicationManagement.pdf)
- [Eclipse Leda - Open Source SDV](https://eclipse-leda.github.io/leda/)
- [capicxx-core-tools (COVESA)](https://github.com/COVESA/capicxx-core-tools)
