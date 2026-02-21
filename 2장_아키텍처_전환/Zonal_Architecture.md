# Zonal Architecture (영역 기반 아키텍처)

## 개요
전통적인 Domain Architecture에서 물리 위치 기반의 Zonal Architecture로 전환하는 것은 현대 E/E(Electrical/Electronic) 아키텍처 혁신의 핵심입니다. 배선 최적화, 확장성 향상, 소프트웨어 중심 제어로의 전환을 가능하게 합니다.

---

## Domain Architecture vs Zonal Architecture

### 기존 Domain Architecture (기능 중심)
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

문제점: ECU 수 증가 → 배선 복잡도 폭발적 증가
        예: 고급 차량 100개 이상의 ECU, 케이블 하네스 50kg 이상
```

### 신규 Zonal Architecture (위치 중심)
```
                    Ethernet 백본 (1Gbps+)
                          │
      ┌───────────────────┼───────────────────┐
      │                   │                   │
┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
│  Zone A   │       │  Zone B   │       │  Zone C   │
│  (전방)   │       │  (중앙)   │       │  (후방)   │
│           │       │           │       │           │
│ Zone ECU  │       │ HPC/      │       │ Zone ECU  │
│ (로컬 I/O)│       │ Central   │       │ (로컬 I/O)│
│  센서들    │       │ Computer  │       │  센서들    │
│  액추에이터│       │           │       │  액추에이터│
└───────────┘       └───────────┘       └───────────┘

이점: 각 Zone은 물리적으로 가까운 장치들만 연결
      Zone 내부: 짧은 배선
      Zone ↔ HPC: 고속 Ethernet 단일 링크
```

---

## Zone Controller 역할과 기능

```
Zone Controller 내부 구조:
┌──────────────────────────────────────────────────────┐
│                     Zone Controller                    │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  Ethernet    │  │  MCU (Safety) │  │  I/O 관리   │ │
│  │  Switch      │  │  ASIL-B/D    │  │  (로컬 처리) │ │
│  │  (TSN 지원)  │  │              │  │             │ │
│  └──────┬───────┘  └──────────────┘  └─────────────┘ │
│         │                                             │
│  UpLink │  (HPC로 1Gbps+ Ethernet)                   │
│                                                        │
│  DownLink: CAN/LIN/FlexRay (레거시 센서 지원)          │
│            10BASE-T1S (IP 기반 센서)                   │
│            100BASE-T1 (고속 카메라/라이다)              │
└──────────────────────────────────────────────────────┘
```

### 주요 기능
1. **프로토콜 변환 (Gateway)**: CAN/LIN 레거시 버스 데이터를 Ethernet 패킷으로 변환
2. **로컬 처리 (Edge Computing)**: 간단한 제어 로직, 데이터 필터링, 집계를 로컬에서 처리하여 HPC 부하 감소
3. **전원 관리**: 구역 내 장치들의 전원 공급 및 슬립/웨이크업 관리
4. **보안 경계**: 구역 내부 트래픽을 HPC로 전달하기 전 필터링 역할

---

## 배선 최적화 효과

```
기존 Domain 아키텍처:
  센서 A (로봇 팔 끝) ──── 5m 케이블 ──► 중앙 ECU
  센서 B (로봇 팔 끝) ──── 5m 케이블 ──► 중앙 ECU
  센서 C (로봇 팔 끝) ──── 5m 케이블 ──► 중앙 ECU
  총 배선: 15m

Zonal 아키텍처:
  센서 A ─ 0.3m ─►
  센서 B ─ 0.3m ─► Zone ECU ─── 5m Ethernet ──► 중앙 HPC
  센서 C ─ 0.3m ─►
  총 배선: 5.9m (61% 감소)

이점:
  - 케이블 무게 감소 (로봇 팔 가반하중 증가)
  - 커넥터 수 감소 (신뢰성 향상)
  - 제조 공정 단순화 (조립 비용 감소)
```

---

## 의료 로봇 Zonal Architecture 설계 예시

```
수술 로봇 Zonal 설계

                         ┌─────────────────┐
                         │   Patient-Side  │
                         │   HPC (주 컴퓨터)│
                         │   CPU/GPU/Safety│
                         └────────┬────────┘
                    TSN 백본 (1Gbps) │
         ┌──────────────┬───────────┴──────────────┐
         │              │                          │
    ┌────▼────┐    ┌─────▼─────┐           ┌───────▼───────┐
    │  Zone 1  │   │  Zone 2    │           │   Zone 3       │
    │ (암 #1)  │   │ (암 #2,#3) │           │ (내시경/비전)  │
    │          │   │            │           │                │
    │ 조인트   │   │ 조인트     │           │ 4K 카메라      │
    │ 제어 #1  │   │ 제어 #2,3  │           │ 조명 제어      │
    │ 힘 센서  │   │ 수술도구   │           │ 초음파 센서    │
    │ 인코더   │   │ 그리퍼     │           │ AI 처리        │
    └──────────┘   └────────────┘           └────────────────┘

    Zone 1,2: 100BASE-T1 (15m, PoDL 전원 공급 가능)
    Zone 3: 1000BASE-T1 (고화질 영상)
```

---

## TSN과의 통합

Zonal Architecture는 Ethernet 백본 위에 TSN을 적용하여 실시간성을 보장합니다.

```
HPC ←────────── TSN Ethernet 백본 ────────────► Zone ECU

트래픽 분류 (VLAN + TSN):
  VLAN 10 (안전 제어): 802.1Qbv TAS 적용, 지연 < 100µs
  VLAN 20 (센서):     CBS(802.1Qav), 대역폭 보장
  VLAN 30 (진단):     Best Effort

Zone ECU 인터페이스:
  업링크: TSN 지원 Ethernet → HPC
  다운링크 (레거시): CAN FD (최대 8Mbps)
  다운링크 (IP 기반): 10BASE-T1S (센서 버스)
  다운링크 (고속): 100BASE-T1 (카메라)
```

---

## Zone Controller 선택 기준

| 항목 | 요구사항 |
|------|---------|
| 프로세서 | 실시간 MCU (ASIL-B 이상) + 통신 SoC |
| Ethernet | TSN 지원 PHY (100BASE-T1 또는 1000BASE-T1) |
| 레거시 버스 | CAN FD, LIN, 필요시 FlexRay |
| 전원 공급 | 12V/24V 입력, PoDL 출력 지원 |
| 환경 조건 | 동작 온도 -40°C ~ +85°C (차량), 멸균 호환 (의료) |
| 안전 인증 | ISO 26262 ASIL-B/D, IEC 62304 (의료) |

### Zone Controller 소프트웨어 스택
```
┌───────────────────────────────────┐
│     Application Layer             │
│  (로컬 제어 로직, 게이트웨이 로직) │
├───────────────────────────────────┤
│     Middleware                    │
│  AUTOSAR Classic (Zone ECU 권장)  │
│  또는 Micro-ROS (ROS 2 기반)       │
├───────────────────────────────────┤
│     BSP / Drivers                 │
│  CAN, LIN, Ethernet (TSN), GPIO   │
├───────────────────────────────────┤
│     RTOS                          │
│  FreeRTOS / Zephyr / AUTOSAR OS   │
└───────────────────────────────────┘
```

---

## Reference
- [Aptiv - Smart Vehicle Architecture](https://www.aptiv.com/en/insights/article/smart-vehicle-architecture)
- [NXP - Zonal Architecture](https://www.nxp.com/applications/automotive/zonal-architecture:ZONAL-ARCHITECTURE)
- [IEEE 802.3cg - 10BASE-T1S for Zonal Networks](https://standards.ieee.org/ieee/802.3cg/7438/)
- [AUTOSAR - Adaptive Platform for Zonal ECU](https://www.autosar.org/standards/adaptive-platform/)
