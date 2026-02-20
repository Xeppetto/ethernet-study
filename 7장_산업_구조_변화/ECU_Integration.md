# ECU 통합 감소 (ECU Integration and Reduction)

## 개요
과거 자동차와 로봇 시스템은 새로운 기능이 추가될 때마다 해당 기능을 담당하는 ECU(Electronic Control Unit)를 하나씩 추가하는 방식(Add-on)으로 발전해 왔습니다. 이로 인해 고급 차량의 경우 100개가 넘는 ECU가 탑재되기도 했습니다. 하지만 이러한 구조는 배선의 복잡성, 중량 증가, 비용 상승, 소프트웨어 관리의 어려움이라는 한계에 부딪혔습니다. 이에 따라, 고성능 ECU 하나가 여러 개의 기능을 통합하여 수행하는 'ECU 통합(ECU Consolidation)' 트렌드가 가속화되고 있습니다.

### 통합의 배경
- **복잡성 증가**: 수많은 ECU 간의 통신(CAN, LIN 등)이 복잡하게 얽혀 있어, 문제 발생 시 원인을 찾기 어렵고 유지보수가 힘듭니다.
- **비용 효율성**: 개별 ECU마다 하우징, 전원부, 커넥터, 통신 칩이 필요하므로, 이를 하나로 합치면 부품 원가(BOM Cost)를 절감할 수 있습니다.
- **공간 제약**: 의료 로봇과 같이 협소한 공간에 많은 기능을 넣어야 하는 시스템에서는 물리적인 공간 확보가 중요합니다.

### 통합 방식
1.  **Domain Controller**: 기능별(도메인)로 ECU를 묶어 하나의 제어기로 통합합니다. (예: 바디 제어 모듈(BCM) + 게이트웨이 -> 바디 도메인 컨트롤러)
2.  **Zone Controller**: 물리적 위치(Zone)를 기준으로 센서와 액추에이터를 연결하는 입출력(I/O) 모듈로 단순화하고, 제어 로직은 상위 중앙 컴퓨터로 이동시킵니다.
3.  **Virtualization (Hypervisor)**: 하나의 강력한 프로세서 위에서 여러 개의 OS를 가상화하여 실행함으로써, 기존에 물리적으로 분리되어 있던 ECU 기능들을 소프트웨어적으로 통합합니다. (예: 인포테인먼트(Android) + 계기판(QNX) -> 콕핏 도메인 컨트롤러)

### 의료 로봇 분야의 변화
의료 로봇 시스템에서도 모터 제어기, 센서 인터페이스 보드, 전원 관리 보드 등이 개별적으로 존재하던 구조에서, 고성능 SoC(System on Chip)를 탑재한 메인 보드 하나로 통합되는 추세입니다.
- **경량화**: 로봇 팔이나 이동형 플랫폼의 무게를 줄여 에너지 효율과 기동성을 높일 수 있습니다.
- **신뢰성 향상**: 커넥터와 케이블 수가 줄어들면 접촉 불량이나 단선 고장의 가능성이 낮아집니다.
- **개발 용이성**: 모든 데이터가 중앙 프로세서 메모리 상에서 공유되므로, 통신 지연 없이 데이터를 융합하고 복잡한 제어 알고리즘을 쉽게 구현할 수 있습니다.

## Reference
- [McKinsey - Automotive software and electronics 2030](https://www.mckinsey.com/industries/automotive-and-assembly/our-insights/automotive-software-and-electronics-2030)
- [NXP - ECU Consolidation](https://www.nxp.com/applications/automotive/ecu-consolidation:ECU-CONSOLIDATION)
