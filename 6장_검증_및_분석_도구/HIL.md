# HIL (Hardware in the Loop)

## 개요
HIL(Hardware in the Loop)은 실제 제어기 하드웨어를 실시간 시뮬레이터와 연결하여 테스트하는 검증 방법론입니다. 자동차, 항공우주 분야에서 표준이 된 기술로, 복잡한 수술 로봇 시스템에서 실제 환자에게 위험을 가하지 않고 극한 시나리오를 검증할 수 있습니다. ISO 26262, IEC 62304 모두 HIL을 유효한 검증 방법으로 인정합니다.

---

## HIL vs SIL vs PIL 비교

```
검증 레벨 비교:

MIL (Model in the Loop):
  ┌──────────────┐     ┌──────────────────┐
  │ 제어 알고리즘  │────►│  플랜트 모델      │
  │ (Simulink 등)│     │  (가상 로봇)      │
  └──────────────┘     └──────────────────┘
  → 순수 시뮬레이션, 빠르지만 실제와 차이

SIL (Software in the Loop):
  ┌──────────────────┐     ┌──────────────────┐
  │ 생성된 코드       │────►│  플랜트 모델      │
  │ (C 코드 자동생성) │     │  (소프트웨어)     │
  └──────────────────┘     └──────────────────┘
  → 코드 수준 검증, 실시간 보장 없음

PIL (Processor in the Loop):
  ┌────────────────────┐  JTAG/UART  ┌──────────────────┐
  │  실제 MCU/ECU      │◄───────────►│  PC 플랜트 모델   │
  │  (타겟 하드웨어)    │             │                  │
  └────────────────────┘             └──────────────────┘
  → 실제 프로세서 검증, 타이밍 포함

HIL (Hardware in the Loop):
  ┌────────────────────┐  I/O (Eth/CAN/Analog)  ┌─────────────────────┐
  │  실제 ECU/제어기    │◄──────────────────────►│  실시간 시뮬레이터   │
  │  (타겟 HW + SW)    │                         │  (dSPACE/Speedgoat) │
  └────────────────────┘                         └─────────────────────┘
  → 가장 현실적, IEC 62304/ISO 26262 검증 인정
```

---

## HIL 시스템 구성

```
수술 로봇 HIL 구성:

┌────────────────────────────────────────────────────────────────┐
│                    HIL 테스트 환경                              │
│                                                                │
│  ┌───────────────────────────────────────────────────────┐    │
│  │  실시간 시뮬레이터 (dSPACE DS1007 또는 Speedgoat)      │    │
│  │                                                       │    │
│  │  ┌──────────────────┐    ┌───────────────────────┐   │    │
│  │  │  로봇 다이나믹스  │    │  센서/액추에이터 모델  │   │    │
│  │  │  - 역기구학       │    │  - 토크 센서 (가상)   │   │    │
│  │  │  - 관절 동역학    │    │  - 인코더 (가상)      │   │    │
│  │  │  - 충돌 감지      │    │  - F/T 센서 (가상)    │   │    │
│  │  └──────────────────┘    └───────────────────────┘   │    │
│  │                   │                │                  │    │
│  │              Ethernet (TSN)      Analog/Digital I/O  │    │
│  └───────────────────┼────────────────┼──────────────────┘    │
│                      │                │                        │
│  ┌───────────────────▼────────────────▼──────────────────┐    │
│  │           실제 Robot Safety ECU                        │    │
│  │  (검증 대상: 안전 모니터링 SW + 관절 제어 SW)           │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  테스트 관리 PC (Vector CANoe / MATLAB Simulink)      │     │
│  │  - 시나리오 실행 자동화                               │     │
│  │  - 결과 기록 및 분석                                  │     │
│  │  - 커버리지 측정                                      │     │
│  └──────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

---

## Network HIL (Ethernet/TSN 검증)

```
Ethernet HIL 구성:

┌──────────────────────────────────────────────────────────────┐
│  TSN 스위치 (의료 로봇 실제 스위치)                           │
│  - VLAN 10: Safety Network                                  │
│  - VLAN 20: Control Network                                 │
└──────┬──────────────────────────────┬────────────────────────┘
       │ (Physical Ethernet)          │
       ▼                              ▼
┌─────────────────┐          ┌─────────────────────────────────┐
│ 실제 Safety ECU  │          │  HIL 시뮬레이터 Ethernet 포트   │
│  - E-Stop 처리  │          │  - 가상 센서 노드 (SOME/IP)      │
│  - E2E 보호     │          │  - 가상 액추에이터 (DDS)         │
│  - TSN 수신     │          │  - Fault Injection 기능          │
└─────────────────┘          └─────────────────────────────────┘

Fault Injection 종류:
  ① 패킷 지연 주입 (Latency Injection)
     예: SOME/IP 패킷 100ms 지연 → Watchdog 반응 확인
  ② 패킷 손실 (Packet Loss)
     예: 5% 랜덤 드롭 → E2E 시퀀스 오류 처리 확인
  ③ 잘못된 데이터 (Data Corruption)
     예: E2E CRC 변조 → 안전 상태 전환 확인
  ④ 재전송 공격 (Replay)
     예: 동일 패킷 반복 → Sequence Number 탐지 확인
  ⑤ Babbling Idiot
     예: 과도한 패킷 전송 → IDS/PSFP 차단 확인
```

---

## HIL 테스트 시나리오 (의료 로봇)

```python
#!/usr/bin/env python3
"""
HIL 테스트 시나리오 자동화 (Python)
실시간 시뮬레이터 API를 통해 시나리오 실행 및 결과 검증
"""
import time
import socket
import struct
from typing import NamedTuple

class TestResult(NamedTuple):
    name: str
    passed: bool
    latency_ms: float
    detail: str

class HILTestFramework:
    """HIL 테스트 자동화 클래스"""

    def __init__(self, hil_ip: str, hil_port: int = 5000):
        self.hil_ip = hil_ip
        self.hil_port = hil_port
        self.results = []

    def set_simulator_param(self, param_name: str, value: float):
        """시뮬레이터 파라미터 설정 (예: 지연 주입)"""
        # 실제 구현에서는 dSPACE/Speedgoat API 사용
        # ds1401.set_var(param_name, value)
        print(f"[HIL] Set {param_name} = {value}")

    def get_ecu_signal(self, signal_name: str) -> float:
        """ECU 출력 신호 읽기"""
        # 실제 구현에서는 CAN/Ethernet 또는 JTAG API
        # return ds1401.get_var(signal_name)
        return 0.0

    def inject_fault(self, fault_type: str, params: dict):
        """Fault Injection"""
        if fault_type == "latency":
            self.set_simulator_param("eth_delay_ms", params['delay_ms'])
        elif fault_type == "packet_loss":
            self.set_simulator_param("eth_loss_rate", params['loss_rate'])
        elif fault_type == "data_corruption":
            self.set_simulator_param("eth_corrupt_enable", 1.0)

    # ─── 테스트 케이스 ───────────────────────────────────────────

    def tc_estop_response(self) -> TestResult:
        """
        TC-ES-001: E-Stop 응답 시간 검증
        요구사항: SRS-03 (E-Stop 응답 < 10ms)
        방법: 가상 E-Stop 신호 발생 → ECU 정지 반응 시간 측정
        """
        # E-Stop 신호 발생
        t_start = time.perf_counter()
        self.set_simulator_param("estop_signal", 1.0)

        # ECU가 Safe State로 전환되는 시간 측정
        timeout = 0.1  # 100ms 타임아웃
        while time.perf_counter() - t_start < timeout:
            state = self.get_ecu_signal("safety_state")
            if state == 1.0:  # Safe State
                break
            time.sleep(0.0001)

        latency_ms = (time.perf_counter() - t_start) * 1000
        self.set_simulator_param("estop_signal", 0.0)  # 복구

        passed = latency_ms < 10.0
        return TestResult(
            name="TC-ES-001 E-Stop Response",
            passed=passed,
            latency_ms=latency_ms,
            detail=f"응답 시간: {latency_ms:.2f}ms (목표: < 10ms)"
        )

    def tc_comm_timeout(self) -> TestResult:
        """
        TC-CM-001: 통신 손실 시 안전 정지 검증
        요구사항: SRS-04 (통신 손실 > 100ms → 정지)
        방법: Ethernet 링크 차단 → 100ms 후 정지 확인
        """
        # 통신 차단
        self.set_simulator_param("eth_link_down", 1.0)
        t_start = time.perf_counter()

        # 100ms 이내 안전 정지 확인 (여유 10ms)
        time.sleep(0.110)

        state = self.get_ecu_signal("motion_state")
        latency_ms = (time.perf_counter() - t_start) * 1000

        self.set_simulator_param("eth_link_down", 0.0)  # 복구

        passed = state == 0.0  # 0 = Stopped
        return TestResult(
            name="TC-CM-001 Comm Timeout Stop",
            passed=passed,
            latency_ms=latency_ms,
            detail=f"통신 차단 후 {latency_ms:.0f}ms: 상태={'정지' if passed else '비정상'}"
        )

    def tc_e2e_crc_error(self) -> TestResult:
        """
        E2E CRC 오류 주입 → Safe State 전환 확인
        """
        self.inject_fault("data_corruption", {})
        time.sleep(0.05)

        state = self.get_ecu_signal("safety_state")
        e2e_error = self.get_ecu_signal("e2e_error_flag")

        self.set_simulator_param("eth_corrupt_enable", 0.0)

        passed = state == 1.0 and e2e_error == 1.0
        return TestResult(
            name="TC-E2E-001 CRC Error Detection",
            passed=passed,
            latency_ms=50.0,
            detail=f"CRC 오류 감지: {'OK' if e2e_error else 'FAIL'}, "
                   f"Safe State: {'OK' if state else 'FAIL'}"
        )

    def run_all(self) -> list:
        """전체 테스트 실행"""
        tests = [
            self.tc_estop_response,
            self.tc_comm_timeout,
            self.tc_e2e_crc_error,
        ]
        for test in tests:
            result = test()
            self.results.append(result)
            status = "PASS" if result.passed else "FAIL"
            print(f"[{status}] {result.name}: {result.detail}")

        passed = sum(1 for r in self.results if r.passed)
        print(f"\n결과: {passed}/{len(self.results)} 통과")
        return self.results


# 실행
hil = HILTestFramework(hil_ip="192.168.100.10")
hil.run_all()
```

---

## Fault Injection 기법

```
HIL Fault Injection 종류 및 적용:

┌──────────────────────┬────────────────────┬───────────────────┐
│ Fault 유형           │ 주입 방법          │ 검증 목표         │
├──────────────────────┼────────────────────┼───────────────────┤
│ 통신 지연            │ 소프트웨어 딜레이   │ Watchdog 반응     │
│ 패킷 손실            │ NIC 드롭 규칙       │ E2E 오류 처리     │
│ CRC 오류             │ 데이터 변조         │ 안전 상태 전환    │
│ 전압 저하            │ 전원 조작 H/W      │ 저전압 감지        │
│ 모터 고장            │ 토크 제한 시뮬     │ 모터 오류 처리    │
│ 센서 이상            │ 가상 값 주입        │ 센서 진단 알고리즘│
│ Babbling Idiot       │ 과부하 트래픽       │ PSFP/IDS 차단    │
│ Replay Attack        │ 패킷 재전송         │ Sequence 검증     │
│ 클록 드리프트        │ PTP offset 조작    │ 타임아웃 처리     │
└──────────────────────┴────────────────────┴───────────────────┘
```

---

## HIL + CI/CD 통합

```
자동화 파이프라인:

  Git Push
      │
      ▼
  CI 서버 (Jenkins / GitHub Actions)
      │
      ├── SIL 테스트 (빠름, < 5분)
      │     - 단위 테스트
      │     - MISRA 정적 분석
      │     - 코드 커버리지
      │
      ├── PIL 테스트 (중간, < 30분)
      │     - 실제 MCU에서 코드 실행
      │     - 타이밍 검증
      │
      └── HIL 테스트 (느림, < 2시간)
            - 전체 시나리오 검증
            - Fault Injection
            - 커버리지 리포트
            - IEC 62304 증거 생성
                  │
                  ▼
            테스트 리포트 + 커버리지 리포트
            → 규제 기관 제출용 문서 자동 생성

HIL 도구별 특징:
  dSPACE SCALEXIO:  자동차/의료 표준, AUTOSAR 지원
  Speedgoat:        MATLAB/Simulink 통합 우수
  NI VeriStand:     유연한 플러그인 구조
  OPAL-RT:          전력 계통 특화, 고속 (25µs)
  Vector CANoe:     네트워크 시뮬레이션 특화
```

---

## Reference
- [dSPACE - HIL Simulation](https://www.dspace.com/en/inc/home/products/systems/hil.cfm)
- [MathWorks - Hardware-in-the-Loop Simulation](https://www.mathworks.com/solutions/verification-validation/hil-simulation.html)
- [NI - HIL Testing](https://www.ni.com/en-us/innovations/white-papers/11/hardware-in-the-loop--hil--simulation-basics.html)
- [Vector - CANoe Network Simulation](https://www.vector.com/int/en/products/products-a-z/software/canoe/)
- [IEC 62304 - Software lifecycle processes for medical device software](https://www.iec.ch/standards/iec-62304/)
