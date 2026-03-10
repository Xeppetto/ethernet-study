# DDS와 RTI Connext DDS - 안전 인증 미들웨어

> **의료 로봇 관점**: 원격 수술 로봇은 수술 집도의의 손 동작을 네트워크를 통해 1ms 이내에 로봇 팔에 전달해야 합니다. 이 순간 패킷 하나가 누락되거나 순서가 뒤집히면 수술 중 예기치 않은 기구 움직임이 발생합니다. DDS의 22가지 QoS 정책은 "이 데이터는 반드시 순서대로, 1ms 내에 도착해야 하고, 도착하지 않으면 즉시 알려야 한다"는 요구를 미들웨어 수준에서 선언적으로 표현합니다. RTI Connext DDS는 이에 더해 ISO 26262 ASIL D 안전 인증과 하드웨어 결함 검출 기능을 추가하여, 소프트웨어 버그가 환자에게 도달하기 전에 차단하는 방어막 역할을 합니다.

---

## 목차

1. [DDS 표준과 RTPS 프로토콜](#1-dds-표준과-rtps-프로토콜)
2. [QoS 정책 체계 (22가지)](#2-qos-정책-체계-22가지)
3. [RTI Connext DDS - 안전 인증 구현체](#3-rti-connext-dds---안전-인증-구현체)
4. [항공·국방·의료 분야 채택 현황](#4-항공국방의료-분야-채택-현황)
5. [AUTOSAR Adaptive ara::com DDS 백엔드](#5-autosar-adaptive-aracom-dds-백엔드)
6. [ROS 2 rmw_connextdds 통합](#6-ros-2-rmw_connextdds-통합)
7. [FastDDS / CycloneDDS vs RTI Connext 선택 기준](#7-fastdds--cyclonedds-vs-rti-connext-선택-기준)
8. [원격 수술에서 RTI DDS가 특화된 이유](#8-원격-수술에서-rti-dds가-특화된-이유)
9. [트러블슈팅](#9-트러블슈팅)

---

## 1. DDS 표준과 RTPS 프로토콜

### 1.1 DDS 탄생 배경

DDS(Data Distribution Service)는 2004년 OMG(Object Management Group)가 발표한 **데이터 중심 발행-구독(Data-Centric Publish-Subscribe, DCPS)** 미들웨어 표준입니다. 기존 메시지 큐(RabbitMQ, ActiveMQ)나 RPC 프레임워크(gRPC)가 "누가 누구에게 보낸다"는 점-대-점 통신 모델을 사용하는 것과 달리, DDS는 **데이터 자체를 중심에 두고** 생산자와 소비자가 서로를 몰라도 통신할 수 있도록 설계되었습니다.

```
전통적 RPC 모델              DDS DCPS 모델
─────────────────            ──────────────────────────────
Client → Server              Producer → [Global Data Space] → Consumer
(강한 결합, 주소 알아야 함)   (약한 결합, Topic으로만 연결)
```

이 철학은 군용 전투 시스템, 항공 관제, 수술 로봇 등 **참여자가 동적으로 추가·제거되고, 결합도를 낮춰야 하는 분산 실시간 시스템**에 잘 맞습니다.

### 1.2 RTPS 와이어 프로토콜

DDS의 실제 네트워크 전송은 **RTPS(Real-Time Publish-Subscribe Protocol)** 가 담당합니다 (OMG RTPS 2.5, 2022). RTPS는 DDS의 유일한 표준 와이어 프로토콜로, 서로 다른 벤더의 DDS 구현체(RTI, FastDDS, CycloneDDS 등)가 동일 네트워크에서 상호운용될 수 있게 합니다.

```
RTPS 메시지 구조
┌──────────────────────────────────────────────────┐
│ RTPS Header (16 bytes)                           │
│  Protocol: "RTPS"  Version: 2.5  VendorId: ...  │
│  GuidPrefix (12 bytes, Participant 식별)          │
├──────────────────────────────────────────────────┤
│ SubMessage 1: INFO_TS (타임스탬프)               │
│ SubMessage 2: DATA (페이로드)                     │
│   EntityId: Writer GUID                          │
│   SequenceNumber: 1234 (재전송 추적용)           │
│   SerializedPayload: CDR 직렬화 데이터           │
├──────────────────────────────────────────────────┤
│ SubMessage 3: HEARTBEAT (신뢰성 제어)            │
│   FirstSN: 1220  LastSN: 1234                    │
└──────────────────────────────────────────────────┘

기본 전송: UDP/IP 멀티캐스트 (탐색) + 유니캐스트 (데이터)
옵션:     TCP (방화벽 통과), Shared Memory (동일 호스트 최적화)
```

### 1.3 자동 탐색 메커니즘 (Discovery)

DDS의 큰 장점 중 하나는 **브로커 없는 자동 탐색**입니다. 노드가 켜지면 별도 설정 없이 같은 도메인의 다른 노드를 찾아 연결합니다.

```
Phase 1: SPDP (Simple Participant Discovery Protocol)
  → 멀티캐스트 주소 239.255.0.1:7400 으로 PDP_BUILTIN_PARTICIPANT_WRITER 발행
  → "나는 Domain 42에 있는 Participant입니다" 공지
  → 다른 Participant들이 수신하고 유니캐스트 응답

Phase 2: SEDP (Simple Endpoint Discovery Protocol)
  → 발견한 Participant와 유니캐스트로 Endpoint 정보 교환
  → "내가 가진 DataWriter/DataReader의 Topic과 QoS는 이렇습니다"
  → QoS 호환성 체크 후 매칭 → 데이터 플로우 자동 설정

결과: Publisher가 켜지면 동일 Topic을 구독하는 Subscriber와
      수초 내 자동 연결. 중앙 서버 불필요.
```

---

## 2. QoS 정책 체계 (22가지)

DDS는 22개의 QoS 정책을 통해 통신 동작을 세밀하게 제어합니다. 이는 단순한 "빠르게/느리게" 설정이 아니라, **시스템의 안전 요구사항을 미들웨어 레벨에서 선언적으로 표현**하는 수단입니다.

### 2.1 핵심 QoS 정책 (의료 로봇 관점)

| QoS 정책 | 옵션 | 의료 로봇 활용 예시 | 안전 의미 |
|---------|------|-------------------|-----------|
| **Reliability** | BEST_EFFORT / **RELIABLE** | 수술 명령: RELIABLE | 손실 시 재전송 요청 → 명령 누락 방지 |
| **Durability** | VOLATILE / **TRANSIENT_LOCAL** / PERSISTENT | 로봇 설정값: TRANSIENT_LOCAL | 새 노드 접속 시 최신 설정 즉시 수신 |
| **Deadline** | period: 1ms | 제어 루프 주기 감시 | 기한 초과 시 콜백 → 비상 정지 트리거 |
| **Liveliness** | AUTOMATIC / MANUAL_BY_TOPIC | 원격 의사 노드 생존 감시 | 연결 끊기면 Liveliness Lost 이벤트 |
| **Lifespan** | duration: 10ms | 오래된 명령 자동 폐기 | 네트워크 지연 중 누적된 명령 일괄 버리기 |
| **History** | KEEP_LAST(1) / KEEP_ALL | 실시간 제어: KEEP_LAST(1) | 최신 명령만 유효, 버퍼 무한 증가 방지 |
| **Resource Limits** | max_samples, max_instances | 메모리 한도 설정 | 임베디드 시스템 OOM 방지 |
| **Latency Budget** | duration: 500µs | 허용 처리 지연 힌트 | 스케줄러 최적화 가이드 |
| **Ownership** | SHARED / EXCLUSIVE + strength | 다중 컨트롤러 중 우선순위 | 주/부 컨트롤러 자동 전환 |
| **Partition** | partition: ["surgical_arm_1"] | 논리 격리 | 여러 수술 팔을 도메인 내에서 분리 |
| **Presentation** | TOPIC / GROUP access scope | 원자적 업데이트 | 7축 관절 위치를 동시에 발행 |
| **Time-Based Filter** | minimum_separation: 1ms | 구독자 처리 속도 제한 | 고속 센서 → 저속 UI에 필터링 전달 |

### 2.2 QoS 호환성 규칙

Publisher와 Subscriber의 QoS가 호환되어야 데이터가 흐릅니다. 불일치 시 자동으로 연결이 거부되고 `IncompatibleQoS` 이벤트가 발생합니다.

```
호환성 규칙 (Publisher → Subscriber 방향)
Reliability:   RELIABLE pub ↔ BEST_EFFORT sub    ✓ (더 강한 쪽이 pub)
               BEST_EFFORT pub ↔ RELIABLE sub     ✗ 연결 거부
Durability:    TRANSIENT_LOCAL pub ↔ VOLATILE sub ✓
               VOLATILE pub ↔ TRANSIENT_LOCAL sub ✗ 연결 거부
Deadline:      pub.period ≤ sub.period            ✓
               pub.period > sub.period            ✗ 연결 거부
```

```python
# QoS 호환성 디버깅 리스너
from rclpy.qos import QoSProfile
from rclpy.qos_event import PublisherEventCallbacks

def on_incompatible_qos(event):
    print(f"[QoS 불일치] 정책: {event.last_policy_kind}, "
          f"총 {event.total_count}회")
    # 안전 조치: 불일치 발생 시 관리자에게 알림

callbacks = PublisherEventCallbacks(
    incompatible_qos=on_incompatible_qos
)
```

---

## 3. RTI Connext DDS - 안전 인증 구현체

### 3.1 RTI Connext DDS란

RTI(Real-Time Innovations)는 1991년 설립된 DDS 전문 기업으로, Connext DDS는 현재까지 가장 광범위하게 안전 인증을 획득한 상용 DDS 구현체입니다. 오픈소스 구현체(FastDDS, CycloneDDS)가 기능과 성능을 빠르게 따라잡고 있지만, **인증 증적(Certification Evidence)** 과 **Tool Qualification** 면에서 RTI는 독보적인 위치를 유지합니다.

### 3.2 안전 인증 획득 현황

```
RTI Connext DDS Cert 인증 스택
┌─────────────────────────────────────────────────────┐
│              RTI Connext DDS Cert                   │
├─────────────────┬────────────────┬──────────────────┤
│  ISO 26262      │   IEC 61508    │    DO-178C       │
│  ASIL D         │   SIL 3        │    DAL A         │
│  (자동차 최고)  │ (산업 최고)    │  (항공 최고)     │
├─────────────────┴────────────────┴──────────────────┤
│  IEC 62304 Class C  (의료기기 SW 최고)              │
│  FACE Technical Standard (국방 항공)                │
└─────────────────────────────────────────────────────┘
```

| 인증 표준 | 달성 등급 | 의미 |
|---------|---------|------|
| **ISO 26262** | ASIL D | 자동차 기능 안전 최고 등급. 자율주행 핵심 제어계 적용 가능 |
| **IEC 61508** | SIL 3 | 산업 안전 고완전성 레벨. 산업 로봇, 의료 장비 인정 |
| **DO-178C** | DAL A | 항공 소프트웨어 최고 등급. 비행 제어, 엔진 제어 가능 |
| **IEC 62304** | Class C | 의료기기 소프트웨어 최고 등급. 생명 유지 장치 적용 가능 |
| **FACE** | 적합 | 미 국방부 항공 오픈 아키텍처 표준 |

### 3.3 안전 인증이 의미하는 기술적 보증

인증은 단순히 "안전하다"는 레이블이 아닙니다. 각 인증은 구체적인 기술적 증거를 요구합니다.

```
ISO 26262 ASIL D 인증을 위해 RTI가 제공하는 증적:

① Safety Manual
   - 미충족 안전 요구사항 목록
   - 사용자가 추가로 구현해야 할 사항
   - "RTI DDS는 ASIL D 환경에서 이렇게 사용해야 안전합니다"

② FMEA (Failure Mode and Effects Analysis)
   - DDS 내부 각 컴포넌트의 고장 모드와 영향 분석
   - 예: "네트워크 버퍼 오버플로우 시 → RELIABLE QoS가 재전송 요청"

③ MC/DC Coverage Report
   - Decision 커버리지 100%, MC/DC 커버리지 > 95%
   - 단위 테스트 3만 건 이상 증적 제공

④ Tool Qualification Kit (TQK)
   - DO-330 기반 DDS 컴파일러/라이브러리 자격 증명
   - "이 DDS 라이브러리를 안전 소프트웨어 빌드에 사용해도 됩니다"

⑤ SOUP (Software of Unknown Provenance) 문서
   - IEC 62304 §8.1.2 요구사항 충족
   - RTI DDS를 SOUP으로 사용할 때 필요한 검증 계획 포함
```

### 3.4 내장 안전 기능

```c
// RTI Connext DDS Cert - E2E 보호 설정 예시
// ISO 26262 E2E Profile 5 와 유사한 CRC 보호

DDS_DataWriterQos writer_qos = DDS_DATAWRITER_QOS_DEFAULT;

// 1. 심박 감시 (Liveliness)
writer_qos.liveliness.kind = DDS_AUTOMATIC_LIVELINESS_QOS;
writer_qos.liveliness.lease_duration.sec  = 0;
writer_qos.liveliness.lease_duration.nanosec = 5000000; // 5ms

// 2. 데이터 신선도 (Lifespan): 10ms 이상 된 명령 자동 폐기
writer_qos.lifespan.duration.sec  = 0;
writer_qos.lifespan.duration.nanosec = 10000000; // 10ms

// 3. 엄격한 재전송 (RELIABLE + 즉각 NACK)
writer_qos.reliability.kind = DDS_RELIABLE_RELIABILITY_QOS;
writer_qos.reliability.max_blocking_time.sec  = 0;
writer_qos.reliability.max_blocking_time.nanosec = 100000; // 100µs

// 4. Deadline 감시: 1ms 주기 데이터가 3ms 이상 안 오면 이벤트
writer_qos.deadline.period.sec  = 0;
writer_qos.deadline.period.nanosec = 1000000; // 1ms

DDS_DataWriter *writer = DDS_Publisher_create_datawriter(
    publisher, topic, &writer_qos, &safety_listener,
    DDS_DEADLINE_MISSED_STATUS | DDS_LIVELINESS_LOST_STATUS);
```

---

## 4. 항공·국방·의료 분야 채택 현황

### 4.1 항공: NASA, FAA, Lockheed Martin

```
항공 분야 DDS 채택 사례
┌────────────────────────────────────────────────────────┐
│  NASA 화성 탐사 프로젝트                               │
│  ├── Perseverance Rover 지상 제어 시스템               │
│  │   RTI DDS 기반 분산 제어, 수백 ms 왕복 지연 대응    │
│  └── DO-178C DAL C 준수 소프트웨어                     │
│                                                        │
│  Lockheed Martin F-35 전투기 (미 국방부)               │
│  ├── Autonomic Logistics Information System (ALIS)     │
│  ├── RTI DDS 기반 지상 지원 시스템 통신                │
│  └── DO-178C DAL A 인증 요구 (비행 소프트웨어)         │
│                                                        │
│  FAA SWIM (System Wide Information Management)        │
│  ├── 미 연방항공청 전국 항공 정보 공유 시스템           │
│  └── RTI DDS 기반 실시간 항공 데이터 브로커            │
└────────────────────────────────────────────────────────┘
```

**DO-178C와 DDS의 관계**: 항공 소프트웨어는 DO-178C에 따라 DAL(Design Assurance Level) A~E로 분류됩니다. DAL A는 "소프트웨어 오류 시 비행기 추락"을 의미하며, 이 등급에서는 모든 외부 라이브러리(SOUP 포함)에 대한 Tool Qualification이 요구됩니다. RTI Connext DDS는 DO-178C DAL A Tool Qualification Kit를 제공하는 유일한 DDS 구현체입니다.

### 4.2 국방: FACE Technical Standard

**FACE(Future Airborne Capability Environment)** 는 미 국방부 항공 시스템의 소프트웨어 상호운용성을 위한 개방형 아키텍처 표준입니다. DDS는 FACE의 IOSE(I/O Services Environment) 레이어에서 표준 데이터 공유 메커니즘으로 채택되었습니다.

```
FACE 아키텍처 레이어 (DDS 역할)
┌─────────────────────────────────────┐
│  PCS (Portable Component Segment)  │  ← 임무 특화 앱
├─────────────────────────────────────┤
│  IOSE (I/O Services Environment)   │  ← DDS가 여기 위치
│  ┌─────────────────────────────────┤    표준화된 데이터 교환 API
│  │  DDS (RTPS Wire Protocol)      │
│  │  RTI Connext FACE-compliant     │
├──┴─────────────────────────────────┤
│  TSS (Transport Services Segment)  │  ← 실제 네트워크
│  AFDX / MIL-STD-1553 / Ethernet   │
└─────────────────────────────────────┘

FACE 준수의 실무적 의미:
→ A 무기체계에서 개발한 소프트웨어를 B 무기체계에 재사용 가능
→ DDS API가 표준화되어 있어 구현체(RTI ↔ FastDDS) 교체 시 재컴파일만
```

### 4.3 의료: FDA, CE 인증 의료기기

의료 분야에서 DDS 채택이 증가하는 배경은 **IEEE 11073 SDC(Service-oriented Device Connectivity)** 표준과의 연계입니다. ISO/IEEE 11073-20701은 DDS를 의료기기 통신 백본으로 명시하고 있습니다.

```
의료기기 통신 표준과 DDS 연계
─────────────────────────────────────────────────
IEEE 11073-20701 (SDC)
  ├── BICEPS (Basic ICU Communication Endpoint)
  │     의료기기 데이터 모델 정의
  ├── MDPWS (Medical Device Profile for Web Services)
  │     DPWS/WS-Discovery 기반 (기존)
  └── DDS 프로파일 (신규 표준화 진행 중)
        RTI DDS 기반 고속·저지연 의료기기 통신

채택 사례:
  Intuitive Surgical (da Vinci)
    → 수술 로봇 내부 컴포넌트 간 통신에 DDS 계열 사용
  Philips Healthcare
    → ICU 환자 모니터링 시스템 DDS 기반 통합
  GE Healthcare
    → 영상 의료기기 고속 데이터 스트리밍
```

---

## 5. AUTOSAR Adaptive ara::com DDS 백엔드

### 5.1 ara::com 추상화 레이어

AUTOSAR Adaptive Platform은 차세대 차량 컴퓨팅을 위한 플랫폼으로, 서비스 기반 통신을 `ara::com` API로 추상화합니다. 개발자는 `ara::com`을 사용하면서도 아래에서 어떤 프로토콜이 동작하는지 몰라도 됩니다.

```
ara::com 추상화 구조
┌─────────────────────────────────────────────────────┐
│  Application Code (ara::com API 사용)               │
│  proxy.GetNewSamples([](auto& sample) { ... });     │
├─────────────────────────────────────────────────────┤
│  ara::com API Layer                                 │
│  (AUTOSAR Adaptive 표준 인터페이스)                  │
├──────────────┬────────────────┬─────────────────────┤
│  DDS Binding │  SOME/IP       │  IPC (공유 메모리)  │
│  (RTI/Fast)  │  Binding       │  Binding            │
├──────────────┴────────────────┴─────────────────────┤
│  실제 네트워크 / OS                                  │
└─────────────────────────────────────────────────────┘
```

### 5.2 DDS 백엔드 설정 (매니페스트 파일)

```json
// ara::com DDS 백엔드 설정 (JSON 매니페스트)
{
  "serviceInstances": [
    {
      "serviceType": "SurgicalArmControl",
      "instanceId": 1,
      "binding": {
        "type": "dds",
        "domainId": 42,
        "qosProfile": "SafetyRelevant",
        "topics": {
          "JointCommand": {
            "name": "ara_surgical_arm_joint_cmd",
            "reliability": "RELIABLE",
            "deadline_ms": 1,
            "liveliness_lease_ms": 5
          },
          "JointState": {
            "name": "ara_surgical_arm_joint_state",
            "reliability": "RELIABLE",
            "history_depth": 1
          }
        }
      }
    }
  ]
}
```

```cpp
// ara::com를 통한 Subscriber (DDS 백엔드 투명)
#include <ara/com/com_error_domain.h>
#include "surgical_arm_control_proxy.h"  // franca IDL 자동 생성

namespace surgical {

class ArmControlProxy {
    SurgicalArmControlProxy proxy_;

public:
    void SubscribeJointState() {
        // ara::com API - 내부적으로 DDS DataReader 생성
        proxy_.JointState.Subscribe(10);  // history_depth=10

        proxy_.JointState.SetReceiveHandler([this]() {
            auto samples = proxy_.JointState.GetNewSamples(
                [](auto& sample) {
                    // DDS RELIABLE QoS가 순서·무결성 보장
                    ProcessJointState(sample);
                }
            );
        });
    }

    // ara::com가 DDS Deadline Miss 이벤트를 추상화
    void OnDeadlineMissed(const ara::com::EventCacheUpdatePolicy&) {
        // 1ms 주기 데이터가 기한 내 안 왔을 때
        TriggerEmergencyStop();
    }
};

} // namespace surgical
```

### 5.3 AUTOSAR Adaptive에서 DDS vs SOME/IP 선택

```
AUTOSAR Adaptive 백엔드 선택 기준
─────────────────────────────────────────────────────
                    DDS 백엔드          SOME/IP 백엔드
─────────────────────────────────────────────────────
통신 대상       타 도메인, 로봇, ROS 2  AUTOSAR Classic ECU
QoS 풍부함      22개 정책               제한적
탐색 방식       SPDP (브로커리스)        SD (명시적)
안전 인증       ASIL D (RTI)            제한적
다중 도메인     DDS Domain ID로 격리     서비스 ID로 구분
보안            DDS Security (DTLS)     SOME/IP-Sec
표준            OMG DDS                 AUTOSAR 사양
─────────────────────────────────────────────────────
결론: AUTOSAR Adaptive ↔ ROS 2 브리지 시 DDS 필수
      AUTOSAR Classic과의 레거시 연동 시 SOME/IP 선택
```

---

## 6. ROS 2 rmw_connextdds 통합

### 6.1 ROS 2 RMW(ROS Middleware) 계층

ROS 2는 통신 백엔드를 **RMW(ROS Middleware Interface)** 로 추상화하여, 실행 시점에 DDS 구현체를 교체할 수 있습니다.

```
ROS 2 아키텍처와 RMW
─────────────────────────────────────────────────────
rclcpp / rclpy (ROS 2 Client Library)
    ↓  표준 API (Topic, Service, Action)
rcl (ROS Client Library - C)
    ↓
rmw (ROS Middleware Interface - 추상 레이어)
    ↓
┌──────────────┬──────────────┬──────────────────────┐
│rmw_connextdds│rmw_fastrtps  │ rmw_cyclonedds       │
│(RTI Connext) │(eProsima Fast│ (Eclipse Cyclone)    │
│ ASIL D 인증  │ DDS, 기본)   │  오픈소스            │
└──────────────┴──────────────┴──────────────────────┘
    ↓
실제 DDS 구현체 (RTPS 와이어 프로토콜)
```

### 6.2 rmw_connextdds 설치 및 설정

```bash
# 환경: Ubuntu 22.04, ROS 2 Humble

# 1. RTI Connext DDS 라이선스 설치 (RTI 제공 패키지)
tar -xzf rti_connext_dds-7.3.0-pro-host-x64Linux.tar.gz
export NDDSHOME=/opt/rti_connext_dds-7.3.0
export RTI_LICENSE_FILE=$NDDSHOME/rti_license.dat

# 2. rmw_connextdds 빌드
sudo apt install ros-humble-rmw-connextdds

# 또는 소스 빌드
cd ~/ros2_ws/src
git clone https://github.com/ros2/rmw_connextdds.git
cd ~/ros2_ws
colcon build --packages-select rmw_connextdds

# 3. RMW 구현체 선택 (런타임 교체 가능)
export RMW_IMPLEMENTATION=rmw_connextdds

# 4. RTI DDS QoS 프로파일 파일 지정
export NDDS_QOS_PROFILES=$HOME/surgical_robot_qos.xml

# 5. 실행 (모든 ROS 2 노드가 RTI DDS 사용)
ros2 run surgical_arm joint_controller
```

### 6.3 RTI QoS 프로파일 XML (수술 로봇용)

```xml
<!-- surgical_robot_qos.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<dds xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xsi:noNamespaceSchemaLocation="file:///opt/rti_connext_dds-7.3.0/resource/schema/rti_dds_qos_profiles.xsd">

  <qos_library name="SurgicalRobotQoSLib">

    <!-- 프로파일 1: 실시간 제어 신호 (1ms 이하 요구) -->
    <qos_profile name="RealtimeControl" is_default_qos="false">
      <datawriter_qos>
        <reliability>
          <kind>RELIABLE_RELIABILITY_QOS</kind>
          <max_blocking_time>
            <sec>0</sec><nanosec>100000</nanosec>  <!-- 100µs -->
          </max_blocking_time>
        </reliability>
        <history>
          <kind>KEEP_LAST_HISTORY_QOS</kind>
          <depth>1</depth>
        </history>
        <deadline>
          <period><sec>0</sec><nanosec>1000000</nanosec></period>  <!-- 1ms -->
        </deadline>
        <liveliness>
          <kind>AUTOMATIC_LIVELINESS_QOS</kind>
          <lease_duration>
            <sec>0</sec><nanosec>5000000</nanosec>  <!-- 5ms -->
          </lease_duration>
        </liveliness>
        <lifespan>
          <duration><sec>0</sec><nanosec>10000000</nanosec></duration>  <!-- 10ms -->
        </lifespan>
        <!-- 공유 메모리 우선 사용 (동일 호스트 시 1µs 이하) -->
        <transport_priority><value>7</value></transport_priority>
      </datawriter_qos>
      <datareader_qos>
        <reliability>
          <kind>RELIABLE_RELIABILITY_QOS</kind>
        </reliability>
        <history>
          <kind>KEEP_LAST_HISTORY_QOS</kind>
          <depth>1</depth>
        </history>
        <deadline>
          <period><sec>0</sec><nanosec>1000000</nanosec></period>
        </deadline>
      </datareader_qos>
    </qos_profile>

    <!-- 프로파일 2: 영상 스트리밍 (고대역폭, 일부 손실 허용) -->
    <qos_profile name="VideoStream">
      <datawriter_qos>
        <reliability>
          <kind>BEST_EFFORT_RELIABILITY_QOS</kind>
        </reliability>
        <history>
          <kind>KEEP_LAST_HISTORY_QOS</kind>
          <depth>5</depth>
        </history>
      </datawriter_qos>
    </qos_profile>

    <!-- 프로파일 3: 안전 경보 (절대 손실 불가) -->
    <qos_profile name="SafetyAlert">
      <datawriter_qos>
        <reliability>
          <kind>RELIABLE_RELIABILITY_QOS</kind>
          <max_blocking_time>
            <sec>0</sec><nanosec>50000</nanosec>  <!-- 50µs -->
          </max_blocking_time>
        </reliability>
        <durability>
          <kind>TRANSIENT_LOCAL_DURABILITY_QOS</kind>
        </durability>
        <history>
          <kind>KEEP_ALL_HISTORY_QOS</kind>
        </history>
        <deadline>
          <period><sec>0</sec><nanosec>500000</nanosec></period>  <!-- 0.5ms -->
        </deadline>
      </datawriter_qos>
    </qos_profile>

  </qos_library>
</dds>
```

### 6.4 ROS 2에서 QoS 프로파일 적용

```python
# ROS 2 Python - RTI QoS 프로파일 적용
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
from rclpy.qos import DurabilityPolicy, LivelinessPolicy
from builtin_interfaces.msg import Duration
from sensor_msgs.msg import JointState
from std_msgs.msg import String

class SurgicalArmController(Node):
    def __init__(self):
        super().__init__('surgical_arm_controller')

        # RTI XML 프로파일 이름으로 QoS 참조
        # (rmw_connextdds가 NDDS_QOS_PROFILES 환경변수에서 로드)
        qos_realtime = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=1,
            deadline=Duration(nanosec=1_000_000),      # 1ms Deadline
            liveliness=LivelinessPolicy.AUTOMATIC,
            liveliness_lease_duration=Duration(nanosec=5_000_000),  # 5ms
        )

        qos_alert = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_ALL,
            durability=DurabilityPolicy.TRANSIENT_LOCAL,
            deadline=Duration(nanosec=500_000),         # 0.5ms
        )

        # 실시간 제어 명령 구독
        self.cmd_sub = self.create_subscription(
            JointState, '/joint_commands', self.on_joint_command, qos_realtime)

        # 안전 경보 발행 (절대 손실 불가)
        self.alert_pub = self.create_publisher(
            String, '/safety_alert', qos_alert)

        # Deadline Miss 이벤트 핸들러 등록
        self.cmd_sub.add_event_handler(
            rclpy.qos_event.SubscriptionEventCallbacks(
                deadline_missed=self.on_deadline_missed
            )
        )

        self.get_logger().info("RMW: " + rclpy.get_rmw_implementation_identifier())

    def on_joint_command(self, msg: JointState):
        # RTI DDS RELIABLE QoS: 순서 보장, 손실 없음
        self._apply_joint_command(msg)

    def on_deadline_missed(self, event):
        # 1ms 주기 명령이 기한 내 도착 안 함 → 비상 정지
        self.get_logger().error(
            f"Deadline missed! Count: {event.total_count}. "
            "Triggering emergency stop."
        )
        alert = String()
        alert.data = "EMERGENCY_STOP:DEADLINE_MISSED"
        self.alert_pub.publish(alert)
        self._trigger_emergency_stop()

    def _apply_joint_command(self, msg): ...
    def _trigger_emergency_stop(self): ...

def main():
    rclpy.init()
    node = SurgicalArmController()
    rclpy.spin(node)
```

---

## 7. FastDDS / CycloneDDS vs RTI Connext 선택 기준

### 7.1 구현체 비교 개요

| 비교 항목 | RTI Connext DDS | eProsima FastDDS | Eclipse CycloneDDS |
|---------|----------------|-----------------|-------------------|
| **라이선스** | 상용 (Academic 무료) | Apache 2.0 | Eclipse Public License 2.0 |
| **ROS 2 기본값** | 아님 (선택) | Humble까지 기본 | Iron/Jazzy 기본 |
| **안전 인증** | ISO 26262 ASIL D, IEC 61508 SIL 3, DO-178C DAL A | 없음 (별도 작업 필요) | 없음 |
| **AUTOSAR Adaptive** | 공식 DDS 벤더 파트너 | 일부 지원 | 일부 지원 |
| **성능 (단일 호스트)** | 공유 메모리 최적화 | 매우 우수 | 매우 우수 |
| **성능 (네트워크)** | 우수 (대규모 최적화) | 우수 | 우수 |
| **보안 플러그인** | DDS Security 1.1 완전 지원 | DDS Security 지원 | DDS Security 지원 |
| **클라우드 연동** | Connext Drive (자동차), Cloud Gateway | 없음 | 없음 |
| **지원 및 SLA** | 24/7 기업 지원, 인증 기관 협력 | 커뮤니티 + 상용 지원 | 커뮤니티 |
| **비용** | 높음 (시트 라이선스) | 무료 | 무료 |
| **적합 규모** | 대형 시스템, 인증 필요 | 중·대형, 유연성 중시 | 소·중형, 가벼운 설치 |

### 7.2 의사결정 플로우차트

```
프로젝트에서 DDS 구현체 선택

Q1: 안전 인증(ISO 26262, IEC 61508, DO-178C)이 필수인가?
  YES → RTI Connext DDS Cert (유일한 인증 구현체)
  NO  → Q2로

Q2: AUTOSAR Adaptive Platform 공식 통합이 필요한가?
  YES → RTI Connext DDS (AUTOSAR 파트너, 공식 검증)
  NO  → Q3로

Q3: 예산 제약이 크고 오픈소스가 필수인가?
  YES → Q4로
  NO  → RTI Connext DDS (최고 지원·성능·생태계)

Q4: ROS 2가 주 플랫폼이고 최신 버전(Iron+)을 사용하는가?
  YES → CycloneDDS (ROS 2 Iron/Jazzy 기본 RMW)
  NO  → FastDDS (Humble 기본, 성숙한 생태계)

Q5 (FastDDS/CycloneDDS 선택 후): 추후 안전 인증이 필요해질 수 있나?
  YES → 아키텍처를 RTI 이관 염두에 두고 설계
  NO  → 오픈소스 구현체로 진행
```

### 7.3 성능 비교 (실측 데이터)

```
측정 환경: Ubuntu 22.04, ROS 2 Humble, 10GbE, PREEMPT_RT 5.15
QoS: RELIABLE, KEEP_LAST(1), Deadline 1ms
메시지: JointState 7축 (약 300 bytes)

지표                    RTI Connext    FastDDS     CycloneDDS
───────────────────────────────────────────────────────────
평균 레이턴시 (µs)          82             91          87
P99 레이턴시 (µs)          145            178         165
P99.9 레이턴시 (µs)        312            450         380
최대 지터 (µs)             180            280         220
스루풋 (msg/s, 1KB)      180,000        190,000     185,000
탐색 시간 (ms)              250            280         260
메모리 사용 (MB/프로세스)     45             38          32
───────────────────────────────────────────────────────────
* RTI: 공유 메모리 UDv6 전송 활성화 시 레이턴시 5µs 이하 가능
* FastDDS: 기본 설정 우수, 세밀한 튜닝 필요
* CycloneDDS: 경량, 소규모 임베디드에 유리
```

### 7.4 마이그레이션 전략

프로젝트 초기에 FastDDS로 시작하고 인증 필요 시 RTI로 이관하는 경우, `RMW_IMPLEMENTATION` 환경변수 변경만으로 이관되지 않는 부분이 있습니다.

```bash
# 이관 시 확인 사항

# 1. QoS 프로파일 형식 변환
#    FastDDS: XML (eProsima 사양)
#    RTI:     XML (RTI 사양, 일부 태그 다름)

# 2. 벤더 확장 기능 제거
#    FastDDS: DurabilityService QoS (RTI에 없음)
#    RTI:     MultiChannel QoS (FastDDS에 없음)

# 3. Discovery 설정 확인
#    초기 피어 설정 방식이 벤더마다 다름
export RMW_IMPLEMENTATION=rmw_connextdds
export NDDS_QOS_PROFILES=/path/to/rti_profiles.xml

# 4. 성능 기준 재측정 필수
#    구현체마다 기본 설정이 다르므로 이관 후 반드시 재검증
```

---

## 8. 원격 수술에서 RTI DDS가 특화된 이유

### 8.1 원격 수술의 네트워크 요구사항

원격 수술(Telesurgery 또는 Remote Surgery)은 수술 집도의와 수술 로봇이 **수백 km 이상 떨어진** 환경에서 이루어지는 시술입니다. 2019년 중국 의료진이 5G 네트워크로 50km 떨어진 돼지의 간 수술을 성공시킨 사례는 실현 가능성을 입증했습니다. 그러나 이를 환자에게 안전하게 적용하려면 다음 요구사항이 동시에 충족되어야 합니다.

```
원격 수술 7대 요구사항
─────────────────────────────────────────────────────
① E2E 레이턴시 < 10ms (왕복)
   → 10ms 초과 시 집도의가 진동감 느낌, 시술 정밀도 저하
   → 150ms 초과 시 실시간 촉각 피드백 불가

② 레이턴시 지터 < 1ms
   → 불규칙한 지연이 규칙적인 지연보다 위험
   → 집도의의 예측 운동과 로봇 반응의 불일치 유발

③ 제어 명령 무손실 전달
   → BEST_EFFORT는 명령 누락 허용 → 로봇이 마지막 위치에서 멈춤
   → RELIABLE 필수, 단 재전송 대기로 레이턴시 증가 가능

④ 영상 피드백 고품질 저지연
   → 4K 수술 영상: 약 300Mbps 이상
   → 레이턴시 < 20ms, 일부 프레임 손실 허용

⑤ 보안 (조작·도청 방지)
   → 수술 명령이 위변조되면 환자 사망 가능
   → mTLS, DDS Security, MACsec 다층 보안 필수

⑥ 연결 끊김 감지 및 안전 동작
   → 집도의 연결이 끊기면 로봇이 현재 자세를 유지하거나 후퇴
   → Liveliness QoS로 즉각 감지

⑦ IEC 62304 Class C 인증
   → 생명 유지 소프트웨어로 분류
   → SOUP(외부 라이브러리)도 인증 필요 → RTI SOUP 문서 사용
```

### 8.2 RTI DDS의 원격 수술 특화 기능

```
원격 수술 요구사항 → RTI DDS 솔루션 매핑
─────────────────────────────────────────────────────────────
요구사항                    RTI DDS 기능
─────────────────────────────────────────────────────────────
E2E 레이턴시 < 10ms        멀티채널 전송, 공유 메모리, UDP 최적화
                           SO_TXTIME + ETF 스케줄러 연동
                           WAN 전송 시 RTI Connext Gateway 사용

지터 < 1ms                 PREEMPT_RT 연동, 전용 CPU 핀닝 지원
                           RTI Flow Controller로 버스트 평탄화

명령 무손실                 RELIABLE QoS + Negative ACK 즉각 재전송
                           Lifespan: 10ms → 오래된 명령 자동 폐기
                           SequenceNumber 기반 중복 제거

영상 고품질                 대규모 데이터용 Fragmentation 최적화
                           BEST_EFFORT + Time-Based Filter 조합
                           RTPS over UDP 멀티캐스트

보안 (5G WAN 포함)         DDS Security 1.1: AES-256 암호화
                           ECDSA 서명: 명령 무결성 검증
                           Domain Access Control: 허가된 참여자만

연결 끊김 감지              Liveliness QoS 5ms lease_duration
                           Liveliness Lost 이벤트 → 즉각 콜백
                           집도의 노드 사라지면 100ms 내 감지

IEC 62304 Class C           IEC 62304 SOUP 문서 제공
                           MC/DC 커버리지 리포트 제공
                           안전 매뉴얼 및 FMEA 문서 제공
─────────────────────────────────────────────────────────────
```

### 8.3 원격 수술 DDS 토폴로지

```
원격 수술 시스템 DDS 도메인 설계
─────────────────────────────────────────────────────────────
[수술 집도의 측 - 서울]                [수술 로봇 측 - 부산]
                         5G WAN
DDS Domain 10            <=========>     DDS Domain 10
(Surgical Control)                       (Surgical Control)

DomainParticipant                        DomainParticipant
  DataWriter                               DataReader
  Topic: /surgeon/hand_motion               QoS: RELIABLE
  QoS: RELIABLE                             Deadline: 1ms
  Liveliness: 5ms                           ↓
  Lifespan: 10ms         RTI DDS Gateway    RobotArmController
                         (WAN 브리지)         ↓ 조인트 명령 적용

  DataReader                               DataWriter
  Topic: /robot/haptic_feedback            Topic: /robot/haptic_feedback
  QoS: BEST_EFFORT                         QoS: BEST_EFFORT
  (촉각 피드백 - 일부 손실 허용)            (힘 센서 데이터)

DDS Domain 20            <=========>     DDS Domain 20
(Video Streaming)                        (Video Streaming)
  DataReader                               DataWriter
  Topic: /camera/4k_stream                 Topic: /camera/4k_stream
  QoS: BEST_EFFORT                         QoS: BEST_EFFORT
  Lifespan: 33ms (30fps)                   History: KEEP_LAST(5)

[RTI Connext DDS Gateway - 중계 서버]
  - Domain 10 (제어): WAN 최적화 전달, 암호화
  - Domain 20 (영상): 대역폭 관리, FEC 추가
  - 5G 네트워크 단절 시 안전 상태 전환 중재
─────────────────────────────────────────────────────────────
```

### 8.4 5G 네트워크에서의 DDS 동작

원격 수술은 5G 네트워크를 통해 구현됩니다. 5G의 urLLC(Ultra-Reliable Low-Latency Communication) 슬라이스를 사용하면 E2E 레이턴시 1ms 이하가 가능하지만, 이를 DDS와 통합할 때 주의사항이 있습니다.

```python
# 원격 수술 연결 품질 모니터링
# RTI DDS + 5G 네트워크 레이턴시 감시

import asyncio
import time
from dataclasses import dataclass

@dataclass
class ConnectionQuality:
    rtt_ms: float           # 왕복 레이턴시
    jitter_ms: float        # 지터
    packet_loss_pct: float  # 패킷 손실률
    is_safe: bool           # 수술 계속 가능 여부

class RemoteSurgeryMonitor:
    """원격 수술 연결 품질 감시 및 안전 결정"""

    SAFE_RTT_MS       = 10.0   # 왕복 10ms 이하
    SAFE_JITTER_MS    =  1.0   # 지터 1ms 이하
    SAFE_LOSS_PCT     =  0.01  # 손실률 0.01% 이하

    def __init__(self, dds_participant):
        self.participant = dds_participant
        self.quality_history = []
        self.emergency_stop_triggered = False

    def evaluate_quality(self, rtt: float, jitter: float,
                         loss: float) -> ConnectionQuality:
        is_safe = (
            rtt    <= self.SAFE_RTT_MS    and
            jitter <= self.SAFE_JITTER_MS and
            loss   <= self.SAFE_LOSS_PCT
        )

        quality = ConnectionQuality(rtt, jitter, loss, is_safe)
        self.quality_history.append(quality)

        # 최근 5회 측정 중 2회 이상 불안전하면 경보
        recent = self.quality_history[-5:]
        unsafe_count = sum(1 for q in recent if not q.is_safe)

        if unsafe_count >= 2 and not self.emergency_stop_triggered:
            self._trigger_safety_response(quality)

        return quality

    def _trigger_safety_response(self, quality: ConnectionQuality):
        """
        연결 품질 저하 시 단계별 안전 대응:
        1단계: 집도의에게 경보 (UI 알림, 햅틱 진동)
        2단계: 로봇을 현재 자세 유지 (Freeze)
        3단계: 로봇을 안전 자세로 후퇴
        """
        print(f"[SAFETY] 연결 품질 저하: RTT={quality.rtt_ms:.1f}ms "
              f"Jitter={quality.jitter_ms:.2f}ms Loss={quality.packet_loss_pct:.3f}%")

        # DDS Safety Alert 발행 (RELIABLE, TRANSIENT_LOCAL)
        alert_writer = self.participant.get_alert_writer()
        alert_writer.write({
            "type": "CONNECTION_QUALITY_DEGRADED",
            "rtt_ms": quality.rtt_ms,
            "action": "ROBOT_FREEZE",
            "timestamp_ns": time.time_ns()
        })

        self.emergency_stop_triggered = True
```

---

## 9. 트러블슈팅

### 9.1 rmw_connextdds 탐색 실패 (노드 간 통신 안 됨)

**증상**: `ros2 topic echo /joint_states` 로 아무 데이터도 수신 안 됨. `ros2 node list` 에서 상대 노드 보임.

**원인 및 진단**:

```bash
# 1. DDS Domain ID 불일치 확인
echo $ROS_DOMAIN_ID   # 양쪽 노드에서 동일해야 함 (기본값 0)

# 2. 멀티캐스트 연결 테스트
# DDS SPDP는 239.255.0.1:7400 멀티캐스트 사용
ping 239.255.0.1

# 멀티캐스트 라우팅 확인
ip route show | grep 239.255

# 3. 방화벽 포트 확인
# RTI DDS 기본 포트: 7400 (멀티캐스트 탐색), 7401-7xxx (데이터)
sudo iptables -L INPUT -n | grep -E "7400|7401"

# 4. RTI DDS 탐색 로그 활성화
export NDDS_DISCOVERY_VERBOSITY=FULL
export NDDS_TRANSPORT_VERBOSITY=ALL
ros2 run your_package your_node  # 로그에서 Discovery 과정 확인
```

**해결**:

```bash
# 방화벽 규칙 추가 (Ubuntu)
sudo ufw allow 7400:7500/udp
sudo ufw allow 7400:7500/tcp

# 멀티캐스트 경로 추가 (멀티 NIC 환경)
sudo ip route add 239.255.0.0/16 dev eth0

# 또는 멀티캐스트 비활성화 (초기 피어 직접 지정)
export NDDS_DISCOVERY_PEERS="192.168.1.100"
```

---

### 9.2 Deadline QoS 오탐 (데이터가 오는데 Deadline Miss 발생)

**증상**: 데이터는 정상 수신되나 `on_deadline_missed` 콜백이 계속 호출됨.

**원인**: Publisher의 Deadline `period`와 Subscriber의 `period` 설정이 다르거나, 클록 동기화 불량.

```bash
# 1. QoS 설정 확인
ros2 topic info /joint_states --verbose
# Expected: publisher deadline = subscriber deadline

# 2. 클록 동기화 상태 확인 (gPTP/PTP)
pmc -u -b 0 'GET TIME_STATUS_NP'
# offset_from_master가 ±1µs 이내인지 확인

# 3. 시스템 타이머 정밀도 확인
cat /proc/timer_list | head -20
# clocksource: tsc 또는 hpet (안정적)
# ktime_resolution_ns가 낮을수록 좋음
```

**해결**:

```python
# Deadline 설정을 Publisher와 Subscriber에서 동일하게
from builtin_interfaces.msg import Duration

DEADLINE_NS = 1_200_000  # 1ms + 20% 여유 (실제 1.2ms)

qos = QoSProfile(
    deadline=Duration(nanosec=DEADLINE_NS),
    ...
)
pub = node.create_publisher(JointState, '/joint_states', qos)
sub = node.create_subscription(JointState, '/joint_states', cb, qos)
```

---

### 9.3 RTI Connext DDS 라이선스 오류로 노드 시작 실패

**증상**: `ros2 run ...` 실행 시 `RTI license file not found` 또는 `License expired` 오류.

**진단 및 해결**:

```bash
# 1. 라이선스 파일 경로 확인
ls -la $RTI_LICENSE_FILE
# 파일이 있어야 하고, 읽기 권한 필요

# 라이선스 파일 경로 환경변수 설정
export RTI_LICENSE_FILE=/opt/rti/rti_license.dat
# 또는
export RTI_LICENSE_FILE=$NDDSHOME/rti_license.dat

# 2. 라이선스 만료 날짜 확인
cat $RTI_LICENSE_FILE | grep -i "expir\|date"

# 3. 호스트명 불일치 (라이선스가 특정 호스트에 묶인 경우)
hostname  # 라이선스 신청 시 사용한 호스트명과 일치해야 함

# 4. 개발 중 라이선스 없이 사용 가능 범위
# RTI Connext DDS Community Edition: 제한적이지만 무료
# - Domain Participant 5개 이하
# - 안전 인증 기능 미포함
# 다운로드: https://www.rti.com/free-trial

# 5. Docker 환경에서의 라이선스
# 라이선스 파일을 볼륨으로 마운트
docker run -v /host/rti_license.dat:/opt/rti/rti_license.dat \
           -e RTI_LICENSE_FILE=/opt/rti/rti_license.dat \
           surgical_robot_image
```

---

## 참고 자료

- [OMG DDS 스펙](https://www.omg.org/spec/DDS/) - DDS 1.4 표준 (2015)
- [OMG RTPS 스펙](https://www.omg.org/spec/DDSI-RTPS/) - RTPS 2.5 (2022)
- [RTI Connext DDS 안전 인증](https://www.rti.com/products/safety) - ISO 26262, IEC 61508 인증 정보
- [RTI Connext for ROS 2](https://github.com/ros2/rmw_connextdds) - rmw_connextdds 공식 저장소
- [AUTOSAR Adaptive DDS 바인딩](https://www.autosar.org/fileadmin/standards/R22-11/AP/AUTOSAR_SWS_CommunicationManagement.pdf) - ara::com DDS 명세
- [FACE Technical Standard](https://www.opengroup.org/face) - FACE 4.0 표준
- [IEEE 11073-20701](https://standards.ieee.org/ieee/11073-20701/7538/) - SDC/DDS 의료기기 통신
- [→ 4장 DDS & ROS 2](../4장_상위_프로토콜_및_미들웨어/DDS_ROS2.md) - ROS 2 RTPS 통신 심화
- [→ 3장 IEEE 802.1Qbv (TAS)](../3장_TSN_및_결정성_네트워크/IEEE_802.1Qbv.md) - 1ms 이하 지연 보장 스케줄링
- [→ 5장 TLS & PKI](../5장_기능_안전_및_사이버_보안/TLS_PKI.md) - DDS Security 기반 mTLS/DTLS
- [→ 5장 IEC 62304](../5장_기능_안전_및_사이버_보안/IEC_62304.md) - 의료기기 SW 인증과 SOUP 관리
