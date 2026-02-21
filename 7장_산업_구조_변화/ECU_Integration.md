# ECU 통합 감소 (ECU Consolidation)

## 개요
과거 자동차와 로봇 시스템은 기능이 추가될 때마다 ECU(Electronic Control Unit)를 추가하는 방식으로 발전해왔습니다. 고급 차량에는 150개 이상의 ECU가 탑재되기도 했으며, 이는 배선 복잡성, 중량 증가, 비용 상승, 소프트웨어 관리 어려움이라는 한계를 낳았습니다. 현재는 고성능 SoC 기반의 **Domain Controller** 또는 **Zonal Controller**로 통합하는 ECU Consolidation 트렌드가 자동차·의료 로봇 분야에서 가속화되고 있습니다.

---

## ECU 진화 단계

```
ECU 통합 4단계 진화:

1단계: 분산형 (2000년대)
  ┌──────────────────────────────────────────────────┐
  │ ECU1  ECU2  ECU3  ECU4 ... ECU100               │
  │  │     │     │     │                            │
  │  └─────┴──┬──┘     │    CAN Bus (Low-speed)      │
  │            │         │                            │
  │          GW ECU ────┘    LIN/CAN 혼합             │
  └──────────────────────────────────────────────────┘
  특징: 기능당 1 ECU, CAN/LIN, 수백 개 케이블

2단계: 도메인 컨트롤러 (2015~)
  ┌──────────────────────────────────────────────────┐
  │ Powertrain  ADAS  Body  Chassis  Cockpit         │
  │   Domain    DC    DC     DC       DC             │
  │     │        │     │     │        │              │
  │     └────────┴──┬──┴─────┘        │  Ethernet    │
  │                Central GW──────── ┘              │
  └──────────────────────────────────────────────────┘
  특징: 도메인별 통합 ECU, Ethernet 등장

3단계: 존 컨트롤러 (2020~)
  ┌──────────────────────────────────────────────────┐
  │           Central HPC (AI + Safety)              │
  │               │    │    │                        │
  │        ┌──────┘    │    └──────┐                 │
  │  Zone  │      Zone │     Zone  │  (앞/뒤/좌/우)   │
  │  Ctrl  │      Ctrl │     Ctrl  │   Ethernet TSN   │
  │   │    │       │   │      │    │                  │
  │ Sensors/Actuators (국소 I/O)                      │
  └──────────────────────────────────────────────────┘
  특징: 위치 기반 Zone 통합, TSN 필수

4단계: 중앙화 (2025~)
  ┌──────────────────────────────────────────────────┐
  │     Vehicle Central Computer (VCC)               │
  │     - AI Compute: 2000+ TOPS                     │
  │     - VM 기반 기능 분리                           │
  │     - FOTA 일원화                                 │
  │             │                                    │
  │     I/O 박스 (하드웨어 인터페이스만)                │
  └──────────────────────────────────────────────────┘
  특징: 서버급 SoC, 클라우드 아키텍처 유사
```

---

## ECU 통합 방식

### 1. Domain Controller 통합

```
수술 로봇 Domain Controller 예시:

AS-IS (기존):
  모터 제어기 × 6개 (관절별)
  센서 허브 × 2개
  안전 모니터 ECU × 1개
  Vision ECU × 1개
  → 총 10개 ECU, 35개 케이블

TO-BE (통합):
  Safety Domain Controller:
    - 관절 모터 제어 × 6 (SIL 구현)
    - 센서 융합 (HW 가속)
    - E-Stop 회로 (Hardware)
    - TSN Ethernet 내장
  Vision/AI Controller:
    - 카메라 영상 처리
    - AI 추론 (NPU)
    - 클라우드 연결
  → 총 2개 ECU, 12개 케이블
```

### 2. Hypervisor 기반 통합

```
Type-1 Hypervisor (Bare-metal):

┌────────────────────────────────────────────────────────┐
│               SoC (NXP S32G / TI TDA4VM)              │
│  ┌──────────────────────────────────────────────────┐  │
│  │              Hypervisor (Type-1)                  │  │
│  │    QNX Neutrino RTOS / Green Hills Integrity      │  │
│  ├──────────────────────┬──────────────────────────┤  │
│  │   Safety VM (ASIL D)  │  Non-Safety VM (QM)      │  │
│  │   - 관절 제어          │  - ROS 2 / AI            │  │
│  │   - E2E 보호           │  - 클라우드 통신          │  │
│  │   - 안전 모니터링      │  - 영상 처리             │  │
│  │   CPU 코어 0,1 (격리) │  CPU 코어 2,3            │  │
│  │   메모리 영역 A        │  메모리 영역 B            │  │
│  └──────────────────────┴──────────────────────────┘  │
│               Hardware Abstraction Layer               │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐  │
│  │ LPDDR5  │ │  eMMC   │ │  TSN NIC │ │  CAN FD  │  │
│  └─────────┘ └─────────┘ └──────────┘ └──────────┘  │
└────────────────────────────────────────────────────────┘

하이퍼바이저 제품:
  QNX Hypervisor: 자동차 ASIL D 인증
  Wind River VxWorks: 항공/의료 DO-178C
  Green Hills Integrity: 의료 기기 인증
  Xen: 오픈소스, 서버/임베디드
  KVM + PREEMPT_RT: 비용 절감용
```

### 3. IOMMU 기반 메모리 격리

```bash
# IOMMU 기반 DMA 격리 (Safety VM 보호)
# Linux에서 IOMMU 확인
dmesg | grep -i iommu
# [    0.425] DMAR: IOMMU enabled

# vfio를 통한 PCIe 장치 VM 격리 (NIC을 Safety VM에 직접 할당)
# 1. NIC 드라이버 분리
sudo modprobe vfio-pci
echo "0000:01:00.0" | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver/unbind

# 2. VFIO에 바인딩
echo "8086 1533" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
# → Safety VM이 이 NIC를 직접 소유 (다른 VM 접근 불가)
```

---

## 케이블 절감 효과 계산

```python
# ECU 통합에 의한 케이블/비용 절감 계산

class ECU_Consolidation_Calculator:

    def calculate_savings(self, before: dict, after: dict) -> dict:
        """
        ECU 통합 전후 비용/중량/신뢰성 비교
        """
        # 비용 계산
        before_cost = (before['ecu_count'] * before['ecu_avg_cost'] +
                       before['cable_meters'] * before['cable_cost_per_m'] +
                       before['connector_count'] * before['connector_cost'])

        after_cost = (after['ecu_count'] * after['ecu_avg_cost'] +
                      after['cable_meters'] * after['cable_cost_per_m'] +
                      after['connector_count'] * after['connector_cost'])

        # 중량
        before_weight = (before['ecu_count'] * before['ecu_avg_weight_g'] +
                         before['cable_meters'] * before['cable_weight_g_per_m'])
        after_weight = (after['ecu_count'] * after['ecu_avg_weight_g'] +
                        after['cable_meters'] * after['cable_weight_g_per_m'])

        # MTBF (평균 무고장 시간) - 커넥터 수 반비례
        # 커넥터당 고장률: 10^-6 /h
        before_mtbf = 1e6 / before['connector_count']
        after_mtbf = 1e6 / after['connector_count']

        return {
            'cost_saving_usd': before_cost - after_cost,
            'cost_saving_pct': (before_cost - after_cost) / before_cost * 100,
            'weight_saving_g': before_weight - after_weight,
            'weight_saving_pct': (before_weight - after_weight) / before_weight * 100,
            'mtbf_before_h': before_mtbf,
            'mtbf_after_h': after_mtbf,
            'mtbf_improvement_pct': (after_mtbf - before_mtbf) / before_mtbf * 100,
        }

# 수술 로봇 예시
calc = ECU_Consolidation_Calculator()
result = calc.calculate_savings(
    before={
        'ecu_count': 10, 'ecu_avg_cost': 200,
        'cable_meters': 35, 'cable_cost_per_m': 5,
        'connector_count': 80, 'connector_cost': 3,
        'ecu_avg_weight_g': 150, 'cable_weight_g_per_m': 50
    },
    after={
        'ecu_count': 2, 'ecu_avg_cost': 800,
        'cable_meters': 12, 'cable_cost_per_m': 8,  # Ethernet 케이블
        'connector_count': 25, 'connector_cost': 5,
        'ecu_avg_weight_g': 300, 'cable_weight_g_per_m': 30
    }
)
print(f"비용 절감: ${result['cost_saving_usd']:.0f} ({result['cost_saving_pct']:.1f}%)")
print(f"중량 절감: {result['weight_saving_g']:.0f}g ({result['weight_saving_pct']:.1f}%)")
print(f"MTBF 향상: {result['mtbf_improvement_pct']:.0f}%")
```

---

## 의료 로봇 ECU 통합 로드맵

```
수술 로봇 ECU 통합 로드맵:

2020 (현재):
  ┌──────────────────────────────────────────────┐
  │  관절 MCU×6 + 안전 ECU + 비전 ECU + GW ECU   │
  │  총 10 ECU, CAN/Ethernet 혼합, 35m 케이블    │
  └──────────────────────────────────────────────┘

2023 (1단계 통합):
  ┌──────────────────────────────────────────────┐
  │  Safety DC (ASIL D) + AI/Vision DC           │
  │  총 2 ECU, Ethernet TSN, 12m 케이블          │
  │  NXP S32G3 + TI TDA4VM                      │
  └──────────────────────────────────────────────┘

2026 (완전 통합):
  ┌──────────────────────────────────────────────┐
  │  Robot Central Computer (RCC)                │
  │  - ARM Cortex-A + M 혼합                     │
  │  - Safety VM (ASIL D) + AI VM                │
  │  - TSN + 클라우드 통합                        │
  │  - OTA 일원화                                 │
  └──────────────────────────────────────────────┘
```

---

## Reference
- [McKinsey - Automotive software and electronics 2030](https://www.mckinsey.com/industries/automotive-and-assembly/our-insights/automotive-software-and-electronics-2030)
- [NXP - S32G Vehicle Network Processor](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/s32g-vehicle-network-processors:S32G)
- [TI - Jacinto TDA4VM Automotive SoC](https://www.ti.com/product/TDA4VM)
- [QNX Hypervisor for Automotive](https://www.qnx.com/products/hypervisor/)
- [Renesas R-Car H3 Automotive SoC](https://www.renesas.com/products/microcontrollers-microprocessors/r-car)
