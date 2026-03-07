# Zonal Architecture (영역 기반 아키텍처)

## 개요
전통적인 Domain Architecture에서 물리 위치 기반의 Zonal Architecture로 전환하는 것은 현대 E/E(Electrical/Electronic) 아키텍처 혁신의 핵심입니다. 배선 최적화, 확장성 향상, 소프트웨어 중심 제어로의 전환을 가능하게 합니다.

---

## Domain Architecture vs Zonal Architecture 비교

| 특성 | Domain Architecture | Zonal Architecture |
|------|--------------------|--------------------|
| 구분 원칙 | 기능별 (파워트레인, 샤시, 인포테인먼트) | 물리적 위치별 (전방, 중앙, 후방) |
| 배선 구조 | 각 ECU → 중앙 도메인 DCU (장거리 배선) | 센서 → 인접 Zone ECU (짧은 배선) |
| ECU 수 | 70~150개 개별 ECU | Zone ECU 4~6개 + HPC 1~2개 |
| 배선 무게 | 40~70kg (고급차 기준) | 20~35kg (40~50% 감소) |
| 통신 백본 | CAN/LIN/FlexRay + 일부 Ethernet | TSN Ethernet 전면 (1Gbps+) |
| 확장성 | 낮음 (도메인별 별도 설계) | 높음 (Zone ECU 추가로 확장) |
| OTA 지원 | 제한적 (개별 ECU 업데이트) | 전체 소프트웨어 OTA (HPC 통합) |
| 도입 시기 | 2000년대 ~ 현재 | 2022년~ (신규 플랫폼) |

---

## 기존 Domain Architecture (기능 중심)
```
                   CAN/LIN/FlexRay 버스
                          │
      ┌───────────────────┼───────────────────┐
      │                   │                   │
┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
│ 파워트레인  │       │  샤시     │       │ 인포테인먼트│
│  Domain   │       │  Domain   │       │  Domain   │
│           │       │           │       │           │
│ 엔진 ECU  │       │ ABS ECU   │       │ 오디오 ECU │
│ 변속기 ECU │       │ ESP ECU   │       │ 내비 ECU   │
│ BMS ECU   │       │ 조향 ECU  │       │ 통신 ECU   │
└───────────┘       └───────────┘       └───────────┘

문제: 고급차 ECU 150개+, 배선 하네스 70kg, 센서와 ECU 간 최대 10m 배선
```

## 신규 Zonal Architecture (위치 중심)
```
                    TSN Ethernet 백본 (1Gbps+)
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌───▼─────────┐
    │  Zone A   │   │  Zone B   │   │  Zone C     │
    │  (전방좌) │   │  (중앙)   │   │  (후방우)   │
    │           │   │           │   │             │
    │ Zone ECU  │   │ HPC/      │   │ Zone ECU    │
    │ (로컬 I/O)│   │ Central   │   │ (로컬 I/O)  │
    │  전방 카메라│  │ Computer  │   │  후방 센서들 │
    │  범퍼 센서 │   │  (AI/OTA) │   │  후방 조명  │
    └───────────┘   └───────────┘   └─────────────┘

이점: 각 Zone은 물리적으로 가까운 장치만 연결
      Zone 내부: 0.3~0.5m 배선
      Zone ↔ HPC: 1GbE Ethernet 단일 링크
```

---

## Zone Controller 역할과 기능

```
Zone Controller 내부 구조:
┌──────────────────────────────────────────────────────┐
│                     Zone Controller                    │
│                                                        │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────┐  │
│  │  Ethernet    │  │  Safety MCU   │  │  I/O 관리  │  │
│  │  Switch      │  │  (ASIL-B/D)   │  │ (로컬처리) │  │
│  │  TSN 지원    │  │  CAN FD 컨트롤│  │ GPIO/ADC   │  │
│  └──────┬───────┘  └──────────────┘  └────────────┘  │
│         │                                              │
│  UpLink:  1GbE TSN (HPC 방향, IEEE 802.1Qbv)          │
│  DownLink: CAN FD 4Mbps (레거시 ECU/센서 지원)         │
│            10BASE-T1S (IP 기반 다중 센서)              │
│            100BASE-T1 (카메라 / 라이다)                │
│            LIN 20kbps (저속 액추에이터)                │
│            PoDL: IEEE 802.3bu (전원 + 데이터 통합)    │
└──────────────────────────────────────────────────────┘
```

### 주요 기능

| 기능 | 설명 | 기술 |
|------|------|------|
| 프로토콜 변환 | CAN/LIN 신호 → SOME/IP Event 변환 | ComM, Signal Gateway |
| 로컬 처리 | 센서 데이터 필터링, 집계, 간단한 PID | FreeRTOS / AUTOSAR Classic |
| 전원 관리 | PoDL, Partial Networking, 슬립/웨이크 | ISO 11898-6 |
| 보안 경계 | 트래픽 필터링, 방화벽 역할 | VLAN, ACL |
| 진단 지원 | DoIP 프록시, DTC 관리 | ISO 13400 |

---

## 배선 최적화 효과

```
기존 Domain 아키텍처 (수술 로봇 예시):
  관절 인코더 #1 (팔 끝) ──── 5.0m 케이블 ──► 중앙 Motion DC
  관절 인코더 #2 (팔 끝) ──── 4.8m 케이블 ──► 중앙 Motion DC
  힘 센서 #1    (팔 끝) ──── 5.2m 케이블 ──► 중앙 Motion DC
  힘 센서 #2    (팔 중간)──── 2.5m 케이블 ──► 중앙 Motion DC
  총 배선 길이: 17.5m, 무게: 약 2.1kg (0.12kg/m × 4선)

Zonal 아키텍처:
  관절 인코더 #1 ──── 0.3m ──►
  관절 인코더 #2 ──── 0.3m ──► Zone ECU (팔 중간) ──── 5.0m Ethernet ──► HPC
  힘 센서 #1    ──── 0.3m ──►
  힘 센서 #2    ──── 0.5m ──►
  총 배선 길이: 6.4m, 무게: 약 0.7kg

이점:
  배선 길이:  17.5m → 6.4m (63% 감소)
  배선 무게:  2.1kg → 0.7kg (66% 감소) → 가반하중 증가
  커넥터 수:  8개 → 5개 (신뢰성 향상)
  제조 공정:  개별 배선 4개 → Zone ECU 단일 커넥터
```

---

## 의료 로봇 Zonal Architecture 설계 예시

```
수술 로봇 Zonal 설계 (7자유도 로봇 암)

                      ┌──────────────────────────┐
                      │  Patient-Side HPC         │
                      │  NVIDIA Orin + Safety MCU │
                      │  QNX Hypervisor + Ubuntu  │
                      │  ROS 2 + AUTOSAR Adaptive │
                      └────────────┬─────────────┘
             TSN Ethernet 백본 (1Gbps, IEEE 802.1Qbv)
       ┌──────────────┬────────────┴──────────────┐
       │              │                           │
  ┌────▼────┐    ┌─────▼─────┐           ┌────────▼───────┐
  │  Zone 1  │   │  Zone 2    │           │    Zone 3       │
  │ (상완부) │   │(하완/손목) │           │  (내시경/비전)  │
  │          │   │            │           │                 │
  │ 조인트1  │   │ 조인트4,5  │           │ 4K 카메라 ×2   │
  │ 조인트2  │   │ 조인트6,7  │           │ 조명 제어       │
  │ 조인트3  │   │ 수술도구   │           │ 초음파 센서     │
  │ 힘센서×2 │   │ 그리퍼     │           │ AI 전처리       │
  │ 온도센서 │   │ 힘센서×3   │           │ 4K 영상 압축    │
  └──────────┘   └────────────┘           └─────────────────┘
     100BASE-T1       100BASE-T1               1000BASE-T1
     PoDL 24V         PoDL 24V                 (고화질 영상)
     15m 케이블       10m 케이블               5m 케이블

Zone 1,2: 100BASE-T1 (Single Pair, PoDL 전원 공급 포함)
Zone 3:   1000BASE-T1 (4K 30fps × 2 = 약 400Mbps 필요)
```

---

## TSN과의 통합

Zonal Architecture는 Ethernet 백본 위에 TSN을 적용하여 실시간성을 보장합니다.

### TSN 트래픽 분류 (VLAN + QoS)
```
HPC ←──────────── TSN Ethernet 백본 (1Gbps) ──────────────► Zone ECU

VLAN 10 (안전 제어):  Priority 7, TAS Gate 제어
  Safety MCU ↔ Zone ECU E-Stop 신호
  Cycle Time: 1ms, 지연 보장: < 100µs
  대역폭 예약: 10Mbps

VLAN 20 (실시간 제어): Priority 6, CBS(Credit-Based Shaping)
  관절 제어 명령 / 엔코더 피드백
  Cycle Time: 1ms, 지연: < 500µs
  대역폭: 50Mbps

VLAN 30 (영상): Priority 4, Best Effort + 대역폭 제한
  4K 카메라 스트림 (H.265 압축)
  대역폭: 400~800Mbps

VLAN 40 (진단): Priority 1, Best Effort
  DoIP, OTA 업데이트, 로그 전송
  대역폭: 나머지 대역폭
```

### TSN Gate Control List (GCL) 예시
```python
# IEEE 802.1Qbv TAS (Time-Aware Shaper) GCL 설정
# Cycle Time: 1ms (1,000,000 ns)
gcl_config = {
    "cycle_time_ns": 1_000_000,
    "gate_control_list": [
        # [시작 오프셋(ns), 열린 큐 비트마스크(Q7..Q0)]
        {"time_ns":       0, "gate_mask": 0b10000000},  # Q7: Safety 100µs
        {"time_ns":  100_000, "gate_mask": 0b01000000},  # Q6: Control 200µs
        {"time_ns":  300_000, "gate_mask": 0b00001111},  # Q3~Q0: Video/Diag 700µs
    ]
}

# Zone ECU에 ethtool 명령으로 적용
import subprocess
subprocess.run([
    "ethtool", "--set-taprio-qopt", "eth0",
    "num_tc", "8",
    "map", "0", "1", "2", "3", "4", "5", "6", "7",
    "queues", "1@0", "1@1", "1@2", "1@3", "1@4", "1@5", "1@6", "1@7",
    "base-time", "0",
    "sched-entry", "S", "80", "100000",   # Q7 100µs
    "sched-entry", "S", "40", "200000",   # Q6 200µs
    "sched-entry", "S", "0f", "700000",   # Q3~Q0 700µs
    "clockid", "CLOCK_TAI"
])
```

---

## Zone Controller 제품 선택 기준

| 항목 | 요구사항 | 추천 제품 |
|------|---------|---------|
| 프로세서 | ASIL-B 이상 Safety MCU | NXP S32K3, Infineon TC3xx |
| Ethernet | TSN 지원 PHY | NXP TJA1120, Marvell 88Q2112 |
| 레거시 버스 | CAN FD × 4이상, LIN × 2 | SoC 내장 또는 외장 |
| 전원 공급 | PoDL (IEEE 802.3bu) 지원 | 별도 PoDL IC |
| 동작 온도 | -40°C ~ +85°C (차량) / +5°C ~ +40°C (의료) | 등급별 선택 |
| 안전 인증 | ISO 26262 ASIL-B, IEC 62304 Class B/C | TÜV 인증 필요 |
| 방수/방진 | IP54 이상 (의료 세척 환경) | 의료 로봇 특화 |

### Zone Controller 소프트웨어 스택
```
┌────────────────────────────────────────────┐
│     Application                            │
│  로컬 PID 제어, 게이트웨이 변환 로직        │
├────────────────────────────────────────────┤
│     Middleware                             │
│  AUTOSAR Classic (BSW + RTE)               │
│  또는 Micro-ROS (ROS 2 경량 클라이언트)    │
├────────────────────────────────────────────┤
│     Security                               │
│  SecOC (CMAC-AES-128), TLS 1.3 (업링크)   │
├────────────────────────────────────────────┤
│     BSP / Drivers                          │
│  CAN FD, LIN, 100BASE-T1, GPIO, ADC        │
├────────────────────────────────────────────┤
│     RTOS                                   │
│  FreeRTOS / Zephyr / AUTOSAR OS (OSEK)     │
└────────────────────────────────────────────┘
```

---

## CAN → SOME/IP 게이트웨이 변환 예시

```c
/* Zone ECU: CAN 수신 → SOME/IP Event 변환 */
#include "CanIf.h"
#include "SomeIpGw.h"

/* CAN ID 0x100: joint_angle (uint16, 10ms 주기) */
void CanIf_RxIndication(uint16_t canId, const uint8_t* data, uint8_t len) {
    if (canId == 0x100 && len >= 2) {
        uint16_t raw = (data[0] << 8) | data[1];
        float angle_deg = raw * 0.01f;  /* 0.01°/LSB */
        float angle_rad = angle_deg * M_PI / 180.0f;

        /* SOME/IP Event 0x8001 (Service 0x0101) 발행 */
        SomeIpGw_SendEvent(
            0x0101,  /* Service ID */
            0x8001,  /* Event ID */
            &angle_rad, sizeof(float));
    }
}
```

---

## Reference
- [Aptiv Smart Vehicle Architecture](https://www.aptiv.com/en/insights/article/smart-vehicle-architecture)
- [NXP Zonal Architecture Solutions](https://www.nxp.com/applications/automotive/zonal-architecture:ZONAL-ARCHITECTURE)
- [IEEE 802.3cg - 10BASE-T1S](https://standards.ieee.org/ieee/802.3cg/7438/)
- [IEEE 802.3bu - PoDL (Power over Data Line)](https://standards.ieee.org/ieee/802.3bu/5545/)
- [IEEE 802.1Qbv - TAS Gate Control](https://standards.ieee.org/ieee/802.1Qbv/6068/)
- [NXP S32K3 - Zone ECU MCU](https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/s32k-automotive-mcus/s32k3-microcontrollers-for-automotive-general-purpose:S32K3)
