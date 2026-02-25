# Zonal Architecture: 위치 기반 아키텍처

## 왜 Zonal인가

기능 중심(Domain) 아키텍처의 근본적인 문제는 케이블 경로에 있다. 파워트레인 Domain Controller가 차량 앞쪽에 있고, 트렁크 근처의 모터가 신호를 보내려면 케이블이 차량 전체를 가로질러 달린다. 2010년대 중반 프리미엄 자동차 한 대의 와이어 하네스 무게는 60킬로그램에 달했다. 이것은 성능 문제가 아니라 물리적 설계의 병목이다.

수술 로봇에서 이 문제는 더 직접적이다. 각 관절마다 개별 제어 케이블이 중앙 컴퓨터까지 연결되면, 로봇 팔이 움직일 때 케이블 다발이 꺾이고 당겨지며 기계적 신뢰성을 떨어뜨린다. 케이블 커넥터 하나가 느슨해지면 장시간 수술 도중 접촉 불량이 발생할 수 있다.

Zonal Architecture는 이 문제를 위치(Zone) 기준의 통합으로 해결한다. 물리적으로 가까운 장치들을 하나의 Zone Controller에 연결하고, Zone Controller와 중앙 컴퓨터 사이는 단 하나의 고속 Ethernet 링크로 연결한다.

---

## Domain Architecture vs Zonal Architecture

```
Domain Architecture (기능 중심):
┌──────────────────────────────────────────────────────────────┐
│ Motion Domain Controller                                     │
│    ↑ 5m 케이블   ↑ 5m 케이블   ↑ 5m 케이블   ↑ 5m 케이블    │
│  관절 #1       관절 #2       관절 #3       관절 #4           │
│  (팔 끝)       (팔 중간)     (어깨)        (베이스)          │
│                                                              │
│  총 케이블: 4관절 × 5m = 20m (관절당 다중 신호선 포함 60m+) │
└──────────────────────────────────────────────────────────────┘

Zonal Architecture (위치 중심):
┌───────────────────────────────────┐
│ 중앙 HPC                          │
│       ↑ 단일 Ethernet 5m          │
│  ┌────────────┐                   │
│  │ Zone ECU   │ ← 로봇 팔 위치    │
│  │ 0.3m ↑↑↑↑ │                   │
│  │ 관#1 관#2  │                   │
│  │ 관#3 관#4  │                   │
│  └────────────┘                   │
│                                   │
│ 총 케이블: 4×0.3m + 5m = 6.2m    │
│           (Domain 대비 ~90% 감소) │
└───────────────────────────────────┘
```

단순히 배선 길이만의 문제가 아니다. 커넥터 수가 줄고, 배선 조립 시간이 줄고, 잠재적 고장 지점이 줄어든다. 수술 로봇에서 케이블 가닥 수가 감소하면 로봇 팔의 관절 자유도에도 영향을 준다—유연성이 높아진다.

---

## Zone Controller의 역할: 단순한 허브가 아니다

Zone Controller를 "케이블을 모아주는 허브"로 이해하면 그 가치를 절반도 이해하지 못한 것이다. Zone Controller는 세 가지 핵심 기능을 수행한다.

**첫째, 프로토콜 변환.** 존 내부에는 CAN FD, LIN, 10BASE-T1S 같은 다양한 프로토콜로 통신하는 센서와 액추에이터가 연결된다. Zone Controller가 이것들을 표준 Ethernet 패킷으로 변환해 HPC에 전달한다. HPC는 CAN 드라이버 없이 표준 IP 스택만으로 모든 센서 데이터를 받을 수 있다.

**둘째, 에지 처리.** 모든 데이터를 원시 형태로 HPC에 올리면 네트워크 대역폭과 HPC 부하가 커진다. Zone Controller에서 1차 필터링, 집계, 이상 감지를 수행하고 요약된 정보만 HPC로 올린다. 예를 들어 관절 인코더 데이터를 Zone Controller가 미분해 속도로 변환하고, 속도가 안전 범위를 벗어날 때만 HPC에 알림을 보낼 수 있다.

**셋째, 전원 도메인 관리.** PoDL(Power over Data Line) 또는 PoE를 통해 Zone Controller가 존 내 센서에 전원을 공급한다. 존 단위로 전원을 켜고 끄는 것도 Zone Controller가 담당한다. 이것은 수술 로봇에서 사용하지 않는 팔을 저전력 상태로 유지하는 데 활용된다.

```
Zone Controller 내부 구조:
┌───────────────────────────────────────────────────────────────┐
│                         Zone Controller                        │
│                                                               │
│   UpLink 인터페이스              DownLink 인터페이스            │
│  ┌─────────────────┐    ┌────────────────────────────────┐    │
│  │ TSN-capable PHY  │    │ CAN FD × 4ch (레거시 센서)    │    │
│  │ 100BASE-T1 or   │    │ LIN × 2ch (보조 장치)          │    │
│  │ 1000BASE-T1     │    │ 10BASE-T1S (멀티드롭 버스)     │    │
│  │ → HPC           │    │ 100BASE-T1 (고속 카메라)       │    │
│  └────────┬────────┘    └──────────────┬─────────────────┘    │
│           │                            │                       │
│  ┌────────▼────────────────────────────▼──────────────────┐   │
│  │                 내부 스위치 패브릭                        │   │
│  │         (CAN FD 게이트웨이, TSN 큐 관리 포함)            │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌──────────────────────┐   ┌──────────────────────────────┐  │
│  │  Safety MCU          │   │  PoDL / PoE 전원 관리        │  │
│  │  ASIL-B, Watchdog   │   │  12V/24V 입력 → 존 내 분배   │  │
│  │  독립 전원           │   │  과전류/단락 보호            │  │
│  └──────────────────────┘   └──────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 수술 로봇 Zonal 설계: 3존 아키텍처

의료 로봇의 물리 구조와 기능 안전 요구사항을 함께 고려하면, 다음과 같은 3존 설계가 합리적이다.

```
                    ┌─────────────────────────────────┐
                    │         Patient-Side HPC         │
                    │  CPU: 12-core ARM A78AE          │
                    │  GPU/DLA: 275 TOPS               │
                    │  Safety MCU: TI TDA4VM (ASIL-D)  │
                    │  TSN Switch: 8×1GbE + 2×10GbE   │
                    └──────────────┬──────────────────┘
                                   │ TSN 백본 (1GbE)
               ┌───────────────────┼────────────────────┐
               │                   │                    │
    ┌──────────▼──────┐   ┌────────▼────────┐   ┌──────▼────────────┐
    │   Zone A         │   │   Zone B        │   │   Zone C          │
    │  로봇 팔 #1       │   │  로봇 팔 #2,#3  │   │  내시경 / 비전    │
    │                  │   │                 │   │                   │
    │  링크: 100BASE-T1│   │링크: 100BASE-T1 │   │링크: 1000BASE-T1  │
    │  거리: 최대 15m  │   │거리: 최대 15m   │   │거리: 최대 15m     │
    │  전원: PoDL 48V  │   │전원: PoDL 48V   │   │전원: PoE+ 30W    │
    │                  │   │                 │   │                   │
    │  DownLink 장치:  │   │DownLink 장치:   │   │DownLink 장치:     │
    │  - 관절 × 4      │   │- 관절 × 8      │   │- 4K 카메라 × 2   │
    │    (인코더+모터)  │   │  (인코더+모터)  │   │  (1GbE × 2)      │
    │  - 힘/토크센서×1 │   │- 힘/토크센서×2  │   │- 초음파 센서      │
    │  - 그리퍼        │   │- 수술 도구 I/F  │   │- 조명 제어 (LIN) │
    └──────────────────┘   └─────────────────┘   └───────────────────┘

Safety Domain (별도 독립 네트워크):
  Safety MCU → 직접 하드와이어드 인터락 (E-Stop, 전원 차단)
  Heartbeat: VLAN 99, PCP=7, 1ms 주기
```

**존 설계에서 중요한 점:** Zone C(내시경/비전)는 1000BASE-T1 링크를 사용하는데, 이것은 4K@60fps 비압축 영상이 약 6Gbps를 필요로 하기 때문이 아니다. H.265로 압축된 스트림도 50~100Mbps에 달하고, 여기에 조명 제어, 초음파 센서, 기타 보조 데이터까지 더하면 100Mbps로는 버스트 트래픽을 감당하기 어렵다. 1GbE는 여유 대역폭을 주고, TSN QoS로 카메라 스트림에 별도 대역폭을 예약한다.

---

## TSN과 Zonal Architecture의 통합

Zonal Architecture의 Ethernet 백본에 TSN을 적용하면, 같은 케이블 위에서 안전 제어 트래픽과 영상 트래픽이 충돌 없이 공존할 수 있다. 핵심은 VLAN + PCP + TAS의 조합이다.

```
트래픽 클래스 설계:

VLAN 10  PCP=7  안전 인터락 / E-Stop 신호
VLAN 20  PCP=6  관절 제어 명령 (1kHz, 실시간)
VLAN 30  PCP=5  힘/토크 센서 피드백 (1kHz, 실시간)
VLAN 40  PCP=3  카메라 스트림 (CBS 대역폭 보장)
VLAN 50  PCP=1  진단 / 로그 (Best Effort)
VLAN 99  PCP=7  Safety Heartbeat (전용, 모니터링)
```

TAS(802.1Qbv) Gate Control List 예시 (1ms 사이클):

```
GCL Entry  Duration  Gate State (큐 7~0, 열림=1 닫힘=0)
─────────────────────────────────────────────────────────
  0        100µs     0b11000000  (큐7,6 열림: 안전+제어)
  1        100µs     0b00100000  (큐5 열림: 센서 피드백)
  2        700µs     0b00001000  (큐3 열림: 카메라 CBS)
  3        100µs     0b11000000  (큐7,6 열림: 2차 사이클)
─────────────────────────────────────────────────────────
사이클 타임: 1,000µs (1ms)
Guard Band: 각 Gate 전환 전 1518byte @ 1GbE = 12.144µs
```

```bash
# Linux tc taprio로 Zone Controller 업링크 TAS 설정
tc qdisc replace dev eth_uplink parent root handle 100 taprio \
    num_tc 8 \
    map 0 1 2 3 4 5 6 7 \
    queues 1@0 1@1 1@2 1@3 1@4 1@5 1@6 1@7 \
    base-time 1000000000 \
    sched-entry S 0xC0 100000 \
    sched-entry S 0x20 100000 \
    sched-entry S 0x08 700000 \
    sched-entry S 0xC0 100000 \
    flags 0x2  # TAPRIO_FLAGS_FULL_OFFLOAD (HW 오프로드 필수)
```

> **주의**: `flags 0x2`(하드웨어 오프로드)가 없으면 소프트웨어 스케줄링으로 폴백하며 지터가 수백 마이크로초로 커진다. TSN NIC 선택 시 taprio HW 오프로드 지원을 반드시 확인해야 한다.

---

## 장애 도메인 분석: Zone 격리가 주는 안전 이점

Zonal Architecture에서 Zone Controller가 고장나면 해당 존의 장치들이 영향을 받지만, 다른 존과 HPC는 정상 동작을 계속한다. 이것이 Domain Architecture 대비 결정적 차이다.

```
Zone A Zone Controller 고장 시 영향 범위:

Domain Architecture:
  Motion Domain Controller 고장 → 모든 관절 제어 불가
  → 즉각적인 전체 수술 중단 필요

Zonal Architecture:
  Zone A Controller 고장 → Zone A 로봇 팔 #1만 영향
  → Zone B, C 정상 → 제한적 수술 계속 가능(의사 판단)
  → 혹은 Zone A Safety MCU가 팔 #1만 안전 정지

장애 감지 메커니즘:
  1. HPC ↔ Zone Controller Heartbeat (1ms 주기, PCP=7)
  2. Zone Controller Watchdog: HPC 신호 없으면 50ms 후 Fail-Safe
  3. Safety MCU: Zone Controller AP 무응답 감지 → 존 전원 차단
```

FMEA 관점에서 보면, 각 Zone Controller의 고장이 시스템 전체가 아닌 해당 존에만 영향을 미치도록 설계됨으로써, 단일 실패 지점(SPOF)이 제거된다. 이것은 IEC 62304 Class C 위험 관리 문서에서 "완화 조치"로 명시할 수 있는 아키텍처 결정이다.

---

## Zone Controller 하드웨어 선택 기준

의료 로봇용 Zone Controller를 선택할 때 고려해야 할 항목들:

| 항목 | 요구사항 | 근거 |
|---|---|---|
| Safety MCU | ASIL-B 이상, 독립 전원 | 존 장치 안전 감시 |
| 업링크 PHY | TSN 지원 (IEEE 802.1Qbv 하드웨어) | 제어 지연 보장 |
| 다운링크 | CAN FD, LIN, 10BASE-T1S 복합 지원 | 레거시 + 신규 센서 혼재 |
| PoDL 출력 | 48V, 최대 50W 존 내 분배 | 별도 전원 케이블 제거 |
| 동작 온도 | 0°C ~ +70°C (의료 실내), 멸균 호환 재질 | 수술실 환경 |
| 인증 | IEC 60601-1 기본 안전, IEC 62304 대상 | 의료기기 규제 |
| 크기 | 로봇 팔 내부 장착 가능 (소형화) | 케이블 단축 극대화 |

대표적인 칩: **NXP S32G2**(차량·산업 Zone Gateway SoC, CAN FD 4ch, 100BASE-T1 TSN 내장), **Renesas R-Car E3**(소형 Zone, TSN 지원), **TI AM6442**(산업용 TSN SoC, 8ch CAN FD).

---

## IEC 62304와 Zonal Architecture

IEC 62304 Class C 소프트웨어가 Zone Controller에도 탑재된다면, 다음 요구사항이 적용된다:

Zone Controller의 게이트웨이 로직(CAN → SOME/IP 변환)은 Class B 또는 C로 분류된다. 변환 오류가 제어 명령을 잘못 전달하면 의료 사고로 이어질 수 있기 때문이다. 따라서:

- 게이트웨이 변환 로직의 유닛 테스트 필수
- CAN 메시지 ID와 SOME/IP 페이로드 간 1:1 매핑 명세 문서화
- 변환 중 데이터 손상을 감지하는 CRC/체크섬 검증
- 경계 조건(최대값, 최소값, 오버플로우) 테스트 케이스

Safety MCU의 Watchdog 타이머 로직은 Class C다. 이것이 오작동하면 Fail-Safe 동작이 트리거되지 않아 로봇이 위험 상태에서 계속 동작할 수 있다.

---

## 참고 문헌

- IEEE 802.3cg-2019: 10BASE-T1S Multidrop Ethernet Standard
- IEEE 802.3bw-2015: 100BASE-T1 Single Pair Ethernet
- NXP AN13267: S32G2 Zonal Gateway Reference Design
- IEEE 802.1Qbv-2015: Enhancements for Scheduled Traffic
- IEC 62304:2006/AMD1:2015: Medical Device Software Lifecycle
- Zeller, M. et al. "Zonal Architecture for Highly Automated Vehicles" IEEE VTC (2020)

---

*관련: [2장 중앙집중형 컴퓨팅](./Centralized_Computing.md) | [1장 흐름 제어와 버퍼](../1장_Ethernet_기초/Flow_Control_and_Buffer.md)*
