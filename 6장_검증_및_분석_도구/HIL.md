# HIL (Hardware in the Loop)

## 개요

HIL(Hardware in the Loop)은 실제 제어기 하드웨어를 실시간 시뮬레이터와 연결하여 테스트하는 검증 방법론입니다. 자동차, 항공우주, 의료 기기 분야에서 표준화된 기술로, 복잡한 수술 로봇 시스템에서 실제 환자에게 위험을 가하지 않고 극한 시나리오를 안전하게 검증할 수 있습니다.

규제 관점에서 HIL은 ISO 26262, IEC 62304, DO-178C 모두 유효한 검증 방법으로 명시적으로 인정합니다. 특히 IEC 62304 Class C(생명 유지 관련) 소프트웨어는 시스템 레벨 테스트를 실제 또는 시뮬레이션 환경에서 수행해야 하며, HIL은 두 요건을 모두 만족하는 최선의 방법입니다.

> **의료 로봇 관점**: 수술 로봇의 안전 기능(E-Stop, 충돌 감지, 토크 제한)은 실제 환자 환경에서 고의적으로 오류를 주입하여 테스트할 수 없습니다. HIL은 가상 환자/수술 환경 모델을 실시간 시뮬레이터에서 구동하면서, 실제 Safety ECU가 올바르게 반응하는지 검증하는 유일한 현실적 방법입니다. IEC 62304의 소프트웨어 통합 테스트(8.4절) 요구를 HIL로 충족할 수 있으며, 규제 제출용 테스트 증거도 자동으로 생성됩니다.

---

## 검증 레벨: MIL → SIL → PIL → HIL

소프트웨어 개발 초기부터 양산까지, 검증 레벨은 단계적으로 현실에 가까워집니다. 각 단계는 독립적으로 가치가 있으며, HIL은 최종 통합 검증 단계입니다.

```
검증 레벨 비교:

MIL (Model in the Loop)  ─── 가장 이른 단계, 가장 빠름
  ┌──────────────┐          ┌──────────────────┐
  │ 제어 알고리즘  │ ────────► │  플랜트 모델      │
  │ (Simulink 등)│          │  (가상 로봇)      │
  └──────────────┘          └──────────────────┘
  · 순수 소프트웨어 시뮬레이션
  · 알고리즘 설계 검증 (수백 시나리오 수분 내 완료)
  · 실제 코드/하드웨어와 차이 존재
  · 실시간성 보장 없음

SIL (Software in the Loop)
  ┌──────────────────┐         ┌──────────────────┐
  │ 생성된 C/C++ 코드 │ ────────► │  플랜트 모델      │
  │ (MBD 자동생성)   │          │  (소프트웨어)     │
  └──────────────────┘          └──────────────────┘
  · 실제 생성 코드로 MIL과 동등성 검증 (Back-to-Back)
  · MISRA C 준수 코드 생성 검증
  · 실시간 타이밍 제약 없음
  · IEC 62304 소프트웨어 단위 테스트 충족

PIL (Processor in the Loop)
  ┌────────────────────┐  JTAG/UART  ┌──────────────────┐
  │  실제 MCU/ECU      │◄───────────►│  PC 플랜트 모델   │
  │  (타겟 하드웨어)    │             │  (소프트웨어)     │
  └────────────────────┘             └──────────────────┘
  · 실제 프로세서에서 실행 → 타이밍/메모리 영향 검증
  · 인터럽트 레이턴시, 실행 시간 측정
  · 플랜트 모델은 PC에서 실행 (하드웨어 I/O 없음)

HIL (Hardware in the Loop)  ─── 가장 현실적, 가장 느림
  ┌────────────────────┐  I/O (Eth/CAN/Analog)  ┌─────────────────────┐
  │  실제 ECU/제어기    │◄──────────────────────►│  실시간 시뮬레이터   │
  │  (타겟 HW + SW)    │                         │  (dSPACE/Speedgoat) │
  └────────────────────┘                         └─────────────────────┘
  · 실제 하드웨어 + 실제 버스(Ethernet/CAN) + 실시간 플랜트 모델
  · 실제 I/O 신호 (아날로그/디지털/Ethernet)
  · IEC 62304 시스템 통합 테스트, ISO 26262 HARA 검증 충족

검증 레벨별 규제 매핑:
┌──────┬───────────────────────┬────────────────────────┐
│ 레벨 │ IEC 62304 적용        │ ISO 26262 적용         │
├──────┼───────────────────────┼────────────────────────┤
│ MIL  │ 소프트웨어 설계 검토   │ SW 설계 검증           │
│ SIL  │ SW 단위/통합 테스트   │ SW 통합 테스트         │
│ PIL  │ SW-HW 통합 검증       │ HW-SW 통합 테스트      │
│ HIL  │ 시스템 레벨 테스트     │ 차량 레벨 검증(V&V)   │
└──────┴───────────────────────┴────────────────────────┘
```

---

## HIL 시스템 구성

### 수술 로봇 HIL 구성 예시

```
수술 로봇 HIL 전체 구성도:

┌─────────────────────────────────────────────────────────────────┐
│                      HIL 테스트 환경                              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │          실시간 시뮬레이터 (dSPACE SCALEXIO)              │    │
│  │                                                         │    │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │    │
│  │  │   로봇 다이나믹스 모델 │  │  환경/인터랙션 모델      │  │    │
│  │  │  · 역기구학 (IK/FK)  │  │  · 조직 저항 모델        │  │    │
│  │  │  · 관절 동역학       │  │  · 충돌/접촉 모델        │  │    │
│  │  │  · 마찰 모델         │  │  · 의사 입력 시뮬레이션  │  │    │
│  │  └─────────────────────┘  └──────────────────────────┘  │    │
│  │  ┌─────────────────────┐  ┌──────────────────────────┐  │    │
│  │  │  센서/액추에이터 모델  │  │  고장 주입 모듈          │  │    │
│  │  │  · 토크 센서 (가상)  │  │  · 지연/손실/변조        │  │    │
│  │  │  · 인코더 (가상)     │  │  · 전원 이상             │  │    │
│  │  │  · F/T 센서 (가상)   │  │  · 센서 오프셋           │  │    │
│  │  └─────────────────────┘  └──────────────────────────┘  │    │
│  │                                                         │    │
│  │  I/O 인터페이스:                                        │    │
│  │    · Ethernet (1GbE TSN, IEEE 802.1Qbv)                │    │
│  │    · CAN FD (8Mbps)                                    │    │
│  │    · Analog I/O (±10V, 1MS/s)                         │    │
│  │    · Digital I/O (3.3/5/24V 선택)                     │    │
│  └────────────┬───────────────────────┬────────────────────┘    │
│               │ TSN Ethernet          │ CAN FD / Analog I/O     │
│               ▼                       ▼                          │
│  ┌────────────────────────────────────────────────────────┐     │
│  │           실제 Robot Safety ECU                         │     │
│  │           (검증 대상: 생산 동일 HW + SW)                 │     │
│  │           · 관절 제어 알고리즘                           │     │
│  │           · E-Stop/안전 모니터링                        │     │
│  │           · E2E 보호 (AUTOSAR E2E Profile)             │     │
│  │           · TSN 수신 (Qbv Gate 제어)                   │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  테스트 관리 PC                                          │     │
│  │  · MATLAB Simulink + Stateflow (시나리오 편집)           │     │
│  │  · Vector CANoe (네트워크 시뮬레이션/캡처)               │     │
│  │  · Python (테스트 자동화 스크립트)                       │     │
│  │  · 커버리지 측정 도구 (Cantata, VectorCAST)             │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Network HIL (Ethernet/TSN 검증)

TSN 기반 의료 로봇에서는 네트워크 계층의 결함 주입이 특히 중요합니다. Safety Critical 메시지(관절 제어, E-Stop)가 TSN 우선순위 큐를 통해 보장된 지연 내에 전달되는지 검증합니다.

```
Ethernet Network HIL 구성:

              TSN 스위치 (의료 로봇 실제 스위치)
              · VLAN 10: Safety Network (E-Stop, 제어)
              · VLAN 20: Diagnostics/Telemetry
              · IEEE 802.1Qbv GCL 활성화
                │                    │
     Physical   │                    │  Physical Ethernet
     Ethernet   │                    │
                ▼                    ▼
        ┌──────────────┐    ┌────────────────────────────────┐
        │ 실제 Safety  │    │  HIL 시뮬레이터 Ethernet 포트  │
        │   ECU        │    │                               │
        │  · E-Stop    │    │  가상 노드들:                  │
        │    처리       │    │  · Joint Sensor Node (SOME/IP)│
        │  · E2E 보호  │    │  · Actuator Node (DDS/ROS2)   │
        │  · TSN 수신  │    │  · Fault Injector Node        │
        └──────────────┘    └────────────────────────────────┘

네트워크 결함 주입 (Linux tc netem 기반):
  # HIL PC에서 네트워크 계층 결함 주입
  # 지연 주입: 100ms 지연 + 10ms 지터
  tc qdisc add dev eth0 root netem delay 100ms 10ms

  # 패킷 손실: 5% 랜덤 드롭
  tc qdisc change dev eth0 root netem loss 5%

  # CRC 변조 (Bit Error Rate 시뮬레이션)
  tc qdisc change dev eth0 root netem corrupt 1%

  # 패킷 재정렬 (Reorder)
  tc qdisc change dev eth0 root netem delay 10ms reorder 25% 50%

  # 정리
  tc qdisc del dev eth0 root
```

---

## HIL 테스트 시나리오 (의료 로봇)

### 핵심 테스트 시나리오 목록

IEC 62304와 ISO 26262의 요구사항에서 도출된 테스트 시나리오입니다. 각 테스트 케이스는 특정 SRS(Software Requirements Specification) 항목을 추적합니다.

```
테스트 시나리오 매핑 테이블:

┌──────────────┬──────────────────────────────┬──────────────┬──────────┐
│ 테스트 ID    │ 시나리오                      │ SRS 항목     │ ASIL/CAL │
├──────────────┼──────────────────────────────┼──────────────┼──────────┤
│ TC-ES-001    │ E-Stop 응답 시간 (< 10ms)     │ SRS-03       │ ASIL D   │
│ TC-ES-002    │ 전원 차단 시 E-Stop 상태 유지 │ SRS-03.1     │ ASIL D   │
│ TC-CM-001    │ 통신 손실 > 100ms → 정지      │ SRS-04       │ ASIL C   │
│ TC-CM-002    │ E2E CRC 오류 → Safe State    │ SRS-05       │ ASIL D   │
│ TC-CM-003    │ TSN 지연 과도 → Watchdog     │ SRS-06       │ ASIL C   │
│ TC-TQ-001    │ 토크 한계 초과 → 정지         │ SRS-07       │ ASIL D   │
│ TC-TQ-002    │ 센서 고장 시 안전 동작        │ SRS-08       │ ASIL D   │
│ TC-OTA-001   │ OTA 중 E-Stop 인터록 동작     │ SRS-12       │ CAL 4    │
│ TC-PTP-001   │ PTP 동기화 손실 → 오류 처리  │ SRS-15       │ ASIL B   │
│ TC-BI-001    │ Babbling Idiot 방어           │ SRS-16       │ ASIL C   │
└──────────────┴──────────────────────────────┴──────────────┴──────────┘
```

### 자동화 테스트 프레임워크

```python
#!/usr/bin/env python3
"""
HIL 테스트 시나리오 자동화
IEC 62304 §8.4 시스템 통합 테스트 + ISO 26262 검증 증거 생성
"""
import time
import json
import subprocess
from datetime import datetime
from typing import NamedTuple
from dataclasses import dataclass, field


@dataclass
class TestResult:
    test_id: str
    name: str
    passed: bool
    measured_value: float
    threshold: float
    unit: str
    detail: str
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    srs_ref: str = ""


class HILTestFramework:
    """
    HIL 테스트 자동화 프레임워크
    실제 환경에서는 dSPACE/Speedgoat Python API 또는
    MATLAB Engine for Python으로 시뮬레이터 제어
    """

    def __init__(self, hil_ip: str = "192.168.100.10", hil_port: int = 5000):
        self.hil_ip = hil_ip
        self.hil_port = hil_port
        self.results: list[TestResult] = []
        # 실제 구현: import ds1401 또는 import speedgoat

    def set_param(self, name: str, value: float):
        """시뮬레이터 파라미터 설정"""
        # 실제: ds1401.set_var(name, value)
        print(f"  [SIM] {name} = {value}")

    def get_signal(self, name: str) -> float:
        """ECU 출력 신호 읽기"""
        # 실제: return ds1401.get_var(name)
        return 0.0

    def inject_network_fault(self, fault: str, params: dict):
        """Linux tc netem으로 네트워크 결함 주입"""
        if fault == "delay":
            cmd = f"tc qdisc change dev eth1 root netem delay {params['ms']}ms"
        elif fault == "loss":
            cmd = f"tc qdisc change dev eth1 root netem loss {params['pct']}%"
        elif fault == "corrupt":
            cmd = f"tc qdisc change dev eth1 root netem corrupt {params['pct']}%"
        elif fault == "clear":
            cmd = "tc qdisc del dev eth1 root"
        else:
            return
        subprocess.run(cmd.split(), check=False)

    # ─── 테스트 케이스 ────────────────────────────────────────────────

    def tc_estop_response(self) -> TestResult:
        """
        TC-ES-001: E-Stop 응답 시간 검증
        SRS-03: E-Stop 신호 발생 후 10ms 이내 모든 관절 정지
        IEC 62304 Class C / ISO 26262 ASIL D
        """
        print("[TC-ES-001] E-Stop 응답 시간 테스트")

        # 정상 동작 상태 설정
        self.set_param("motion_enable", 1.0)
        self.set_param("joint_speed_setpoint", 0.5)  # 50% 속도
        time.sleep(0.1)  # 안정화

        # E-Stop 발생 → 타이밍 측정
        t_start = time.perf_counter()
        self.set_param("estop_signal", 1.0)

        # 10ms + 여유 10ms = 20ms 타임아웃
        latency_ms = None
        for _ in range(200):  # 0.1ms 간격으로 폴링
            state = self.get_signal("motion_state")
            if state == 0.0:  # 정지 확인
                latency_ms = (time.perf_counter() - t_start) * 1000
                break
            time.sleep(0.0001)

        if latency_ms is None:
            latency_ms = 20.0  # 타임아웃

        # 복구
        self.set_param("estop_signal", 0.0)
        self.set_param("motion_enable", 0.0)
        time.sleep(0.1)

        passed = latency_ms < 10.0
        return TestResult(
            test_id="TC-ES-001", name="E-Stop Response Time",
            passed=passed, measured_value=latency_ms,
            threshold=10.0, unit="ms",
            detail=f"응답 시간 {latency_ms:.2f}ms (요구: < 10ms)",
            srs_ref="SRS-03"
        )

    def tc_comm_loss_stop(self) -> TestResult:
        """
        TC-CM-001: Ethernet 통신 손실 → 안전 정지 검증
        SRS-04: 100ms 이상 통신 손실 시 안전 상태(Safe State) 전환
        """
        print("[TC-CM-001] 통신 손실 안전 정지 테스트")
        self.set_param("motion_enable", 1.0)
        time.sleep(0.05)

        # Ethernet 링크 차단 (tc netem 또는 시뮬레이터)
        self.inject_network_fault("loss", {"pct": 100})
        t_start = time.perf_counter()

        # 110ms 대기 후 상태 확인
        time.sleep(0.110)
        state = self.get_signal("safety_state")
        elapsed_ms = (time.perf_counter() - t_start) * 1000

        # 복구
        self.inject_network_fault("clear", {})
        self.set_param("motion_enable", 0.0)

        passed = state == 1.0  # 1 = Safe State
        return TestResult(
            test_id="TC-CM-001", name="Communication Loss Safe Stop",
            passed=passed, measured_value=elapsed_ms,
            threshold=100.0, unit="ms",
            detail=f"{elapsed_ms:.0f}ms 후 상태: {'SAFE' if passed else '비정상'}",
            srs_ref="SRS-04"
        )

    def tc_e2e_crc_error(self) -> TestResult:
        """
        TC-CM-002: E2E CRC 오류 주입 → Safe State 전환 검증
        SRS-05: E2E Profile 4 오류 감지 후 50ms 이내 Safe State
        """
        print("[TC-CM-002] E2E CRC 오류 탐지 테스트")
        self.set_param("motion_enable", 1.0)
        time.sleep(0.05)

        t_start = time.perf_counter()
        self.inject_network_fault("corrupt", {"pct": 100})  # 100% CRC 오류

        # E2E 오류 감지 및 반응 대기
        latency_ms = None
        for _ in range(500):
            state = self.get_signal("safety_state")
            e2e_flag = self.get_signal("e2e_error_flag")
            if state == 1.0 and e2e_flag == 1.0:
                latency_ms = (time.perf_counter() - t_start) * 1000
                break
            time.sleep(0.0001)

        latency_ms = latency_ms or 50.0

        # 복구
        self.inject_network_fault("clear", {})
        self.set_param("motion_enable", 0.0)

        passed = latency_ms < 50.0
        return TestResult(
            test_id="TC-CM-002", name="E2E CRC Error Detection",
            passed=passed, measured_value=latency_ms,
            threshold=50.0, unit="ms",
            detail=f"E2E 오류 탐지 + Safe State: {latency_ms:.1f}ms (요구: < 50ms)",
            srs_ref="SRS-05"
        )

    def tc_tsn_delay_watchdog(self) -> TestResult:
        """
        TC-CM-003: TSN 지연 과도 → Watchdog 반응 검증
        SRS-06: 관절 제어 패킷 지연 > 2ms 시 Watchdog 카운터 증가
                5회 연속 지연 시 Safe State
        """
        print("[TC-CM-003] TSN 지연 Watchdog 테스트")
        self.set_param("motion_enable", 1.0)
        time.sleep(0.05)

        # 5ms 지연 주입 (주기 1ms의 5배)
        self.inject_network_fault("delay", {"ms": 5})
        time.sleep(0.010)  # 10ms = 10 사이클

        watchdog_cnt = self.get_signal("watchdog_count")
        safety_state = self.get_signal("safety_state")

        self.inject_network_fault("clear", {})
        self.set_param("motion_enable", 0.0)

        passed = watchdog_cnt >= 5 and safety_state == 1.0
        return TestResult(
            test_id="TC-CM-003", name="TSN Delay Watchdog",
            passed=passed, measured_value=watchdog_cnt,
            threshold=5.0, unit="count",
            detail=f"Watchdog 카운트: {watchdog_cnt}, Safe State: {safety_state}",
            srs_ref="SRS-06"
        )

    def tc_torque_limit(self) -> TestResult:
        """
        TC-TQ-001: 토크 한계 초과 → 안전 정지 검증
        SRS-07: 관절 토크 > 30Nm 시 즉각 정지 (< 5ms)
        """
        print("[TC-TQ-001] 토크 한계 초과 테스트")
        self.set_param("motion_enable", 1.0)
        time.sleep(0.05)

        t_start = time.perf_counter()
        # 과토크 시뮬레이션 (환경 모델에서 큰 저항 주입)
        self.set_param("env_torque_disturbance", 35.0)  # 35Nm > 30Nm 한계

        latency_ms = None
        for _ in range(100):
            state = self.get_signal("torque_fault_state")
            if state == 1.0:
                latency_ms = (time.perf_counter() - t_start) * 1000
                break
            time.sleep(0.0001)

        latency_ms = latency_ms or 10.0
        self.set_param("env_torque_disturbance", 0.0)
        self.set_param("motion_enable", 0.0)

        passed = latency_ms < 5.0
        return TestResult(
            test_id="TC-TQ-001", name="Torque Limit Safe Stop",
            passed=passed, measured_value=latency_ms,
            threshold=5.0, unit="ms",
            detail=f"과토크 감지 후 정지: {latency_ms:.2f}ms (요구: < 5ms)",
            srs_ref="SRS-07"
        )

    def tc_babbling_idiot(self) -> TestResult:
        """
        TC-BI-001: Babbling Idiot 방어 (PSFP/IDS 차단)
        SRS-16: 과도 트래픽(기준 10배) 감지 후 차단, 정상 동작 유지
        """
        print("[TC-BI-001] Babbling Idiot 방어 테스트")
        self.set_param("motion_enable", 1.0)
        time.sleep(0.05)

        # 과도 트래픽 주입 (시뮬레이터에서 flood 패킷 생성)
        self.set_param("traffic_flood_enable", 1.0)
        time.sleep(0.1)

        # Safety 메시지가 여전히 수신되는지 확인
        safety_msg_rx = self.get_signal("safety_msg_rx_count")
        flood_blocked = self.get_signal("psfp_block_count")
        motion_normal = self.get_signal("motion_state")

        self.set_param("traffic_flood_enable", 0.0)
        self.set_param("motion_enable", 0.0)

        passed = flood_blocked > 0 and motion_normal == 1.0
        return TestResult(
            test_id="TC-BI-001", name="Babbling Idiot Defense",
            passed=passed, measured_value=float(flood_blocked),
            threshold=1.0, unit="packets",
            detail=f"차단 패킷: {flood_blocked}, 정상 동작: {motion_normal}",
            srs_ref="SRS-16"
        )

    def run_all(self) -> list[TestResult]:
        """전체 테스트 실행 및 결과 저장"""
        tests = [
            self.tc_estop_response,
            self.tc_comm_loss_stop,
            self.tc_e2e_crc_error,
            self.tc_tsn_delay_watchdog,
            self.tc_torque_limit,
            self.tc_babbling_idiot,
        ]

        print(f"\n{'='*60}")
        print(f"HIL 테스트 시작: {datetime.now().isoformat()}")
        print(f"{'='*60}\n")

        for test_fn in tests:
            result = test_fn()
            self.results.append(result)
            status = "PASS ✓" if result.passed else "FAIL ✗"
            print(f"  [{status}] {result.test_id} {result.name}")
            print(f"         측정값: {result.measured_value:.2f}{result.unit} "
                  f"(기준: {result.threshold}{result.unit})")
            print(f"         {result.detail}\n")

        # 결과 요약
        total = len(self.results)
        passed = sum(1 for r in self.results if r.passed)
        print(f"{'='*60}")
        print(f"결과: {passed}/{total} 통과 "
              f"({'OK' if passed == total else 'FAIL'})")
        print(f"{'='*60}\n")

        # IEC 62304 증거 생성
        self.export_report()
        return self.results

    def export_report(self, path: str = "/tmp/hil_report.json"):
        """IEC 62304 / ISO 26262 규제 제출용 리포트 생성"""
        report = {
            "tool": "HIL Test Framework v1.0",
            "timestamp": datetime.now().isoformat(),
            "total": len(self.results),
            "passed": sum(1 for r in self.results if r.passed),
            "results": [
                {
                    "id": r.test_id, "name": r.name,
                    "passed": r.passed,
                    "measured": r.measured_value,
                    "threshold": r.threshold,
                    "unit": r.unit,
                    "detail": r.detail,
                    "srs_ref": r.srs_ref,
                    "timestamp": r.timestamp
                }
                for r in self.results
            ]
        }
        with open(path, "w") as f:
            json.dump(report, f, indent=2, ensure_ascii=False)
        print(f"[리포트] {path} 저장 완료")
```

---

## Fault Injection 기법

결함 주입(Fault Injection)은 HIL의 핵심 가치입니다. 현실에서 재현하기 어렵거나 위험한 시나리오를 제어된 환경에서 반복 재현합니다.

```
HIL Fault Injection 종류 및 적용:

┌─────────────────────┬──────────────────────┬───────────────────────┐
│ Fault 유형          │ 주입 방법            │ 검증 목표             │
├─────────────────────┼──────────────────────┼───────────────────────┤
│ 통신 지연           │ tc netem delay       │ Watchdog, 타임아웃     │
│ 패킷 손실           │ tc netem loss        │ E2E 오류 처리          │
│ CRC 변조            │ tc netem corrupt     │ 안전 상태 전환         │
│ 링크 다운           │ ip link set down     │ 통신 손실 대응         │
│ Babbling Idiot      │ 과부하 트래픽 생성   │ PSFP/IDS 차단          │
│ Replay Attack       │ 패킷 재전송          │ E2E Sequence 검증      │
│ PTP 동기화 손실     │ ptp4l 중단           │ 타임아웃 처리          │
│ 전압 저하           │ 전원 조작 H/W        │ 저전압 감지             │
│ 과토크              │ 환경 모델 파라미터   │ 토크 한계 알고리즘     │
│ 센서 오프셋         │ 가상 센서 바이어스   │ 센서 진단 알고리즘     │
│ 클록 드리프트       │ PTP offset 인위 증가  │ 시간 동기화 오류 처리  │
│ 메모리 부족         │ 메모리 점유 프로세스 │ OOM 대응, Safe State   │
└─────────────────────┴──────────────────────┴───────────────────────┘

ISO 26262 FMEA 연계:
  각 Fault는 FMEA의 Failure Mode와 1:1 매핑
  예: "관절 제어 Ethernet 링크 다운"
      → FMEA ID: FM-ETH-003
      → 위험: 관절 제어 손실 → 환자 충돌 가능
      → 대응: 통신 손실 감지 → 100ms 내 안전 정지
      → HIL TC: TC-CM-001로 검증
```

---

## 코드 커버리지 측정

IEC 62304 Class C와 ISO 26262 ASIL C/D는 MC/DC(Modified Condition/Decision Coverage) 100% 달성을 요구합니다.

```
커버리지 요구 수준:
┌───────────┬──────────────┬────────────┬──────────────┐
│ 표준      │ 등급         │ 최소 커버리지│ HIL 기여     │
├───────────┼──────────────┼────────────┼──────────────┤
│ IEC 62304 │ Class A      │ Statement  │ 시스템 테스트 │
│ IEC 62304 │ Class B      │ Branch     │ 시스템 테스트 │
│ IEC 62304 │ Class C      │ MC/DC      │ 필수          │
│ ISO 26262 │ ASIL A/B     │ Statement  │ 보완          │
│ ISO 26262 │ ASIL C       │ Branch     │ 시스템 레벨   │
│ ISO 26262 │ ASIL D       │ MC/DC      │ 필수          │
└───────────┴──────────────┴────────────┴──────────────┘

MC/DC 예시:
  // 안전 정지 조건: E-Stop OR (토크 초과 AND 센서 유효)
  bool should_stop = estop || (torque_exceed && sensor_valid);

  MC/DC 요구 테스트 케이스:
  #  estop  torque_exceed  sensor_valid  결과  테스트 목적
  1    F         F              F          F   기본 동작
  2    T         F              F          T   estop 단독 영향 확인
  3    F         T              F          F   sensor_valid 영향 확인
  4    F         T              T          T   토크+센서 조합 확인
  5    F         F              T          F   sensor_valid 단독 영향 없음

커버리지 측정 도구 (임베디드):
  VectorCAST:  DO-178C, IEC 62304, MISRA 통합 지원
  Cantata++:   C/C++ 단위/통합 테스트 + MC/DC
  LDRA:        코드 분석 + 커버리지 + RTM

커버리지 수집 명령 예시 (gcov 기반):
  # 빌드 시 계측 코드 삽입
  gcc -fprofile-arcs -ftest-coverage -o safety_ecU safety_ecu.c

  # HIL 테스트 실행 후 커버리지 수집
  gcov safety_ecu.c
  gcovr --html-details coverage_report.html
  # → 미커버 줄(branch) 표시 → 추가 테스트 케이스 도출
```

---

## HIL + CI/CD 통합

HIL 테스트를 CI/CD 파이프라인에 통합하면, SW 변경이 발생할 때마다 자동으로 안전 검증을 수행하고 규제 제출용 증거를 생성할 수 있습니다.

```
자동화 CI/CD 파이프라인:

  ┌─────────────────────────────────────────────────────────────┐
  │  개발자 Git Push (feature branch)                            │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Stage 1: Static Analysis (< 5분)                            │
  │  · MISRA C 2012 검사 (Cppcheck, PC-lint)                     │
  │  · CERT C 보안 규칙 검사 (Clang-Tidy)                        │
  │  · 코드 복잡도 측정 (McCabe ≤ 10)                            │
  │  · SIL 단위 테스트 (Google Test)                             │
  └──────────────────────────┬──────────────────────────────────┘
                             │ 통과
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Stage 2: SIL / PIL 테스트 (< 30분)                          │
  │  · Software Integration Test (SIL)                          │
  │  · 실제 MCU에서 코드 실행 검증 (PIL, JTAG)                   │
  │  · 타이밍 프로파일링                                          │
  └──────────────────────────┬──────────────────────────────────┘
                             │ 통과
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Stage 3: HIL 테스트 (< 2시간)                               │
  │  · 전체 시나리오 자동 실행                                    │
  │  · Fault Injection 테스트                                    │
  │  · MC/DC 커버리지 측정                                       │
  │  · IEC 62304 §8.4 증거 리포트 생성                           │
  └──────────────────────────┬──────────────────────────────────┘
                             │ 모두 통과
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Stage 4: 아티팩트 보관                                       │
  │  · 빌드 아티팩트 (ECU 펌웨어 이미지)                          │
  │  · 테스트 리포트 (JSON + PDF)                                │
  │  · 커버리지 리포트 (HTML)                                    │
  │  · 규제 제출용 추적성 매트릭스 (RTM) 자동 갱신               │
  └─────────────────────────────────────────────────────────────┘

GitHub Actions 예시:
  # .github/workflows/hil_test.yml
  hil_test:
    runs-on: self-hosted  # HIL 장비 연결된 자체 호스팅 Runner
    needs: [static_analysis, sil_test]
    steps:
      - name: Run HIL Tests
        run: |
          python3 hil_test_runner.py \
            --hil-ip $HIL_IP \
            --report-dir artifacts/hil

      - name: Check Pass Rate
        run: |
          PASS_RATE=$(jq '.passed / .total' artifacts/hil/report.json)
          if [ $(echo "$PASS_RATE < 1.0" | bc) -eq 1 ]; then
            echo "HIL 테스트 실패"; exit 1
          fi

      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          name: hil-report
          path: artifacts/hil/

HIL 도구별 특징:
┌─────────────────┬───────────────────────────────────────────┐
│ 도구            │ 특징                                       │
├─────────────────┼───────────────────────────────────────────┤
│ dSPACE SCALEXIO │ 자동차/의료 표준, AUTOSAR 완전 지원        │
│ Speedgoat       │ MATLAB/Simulink 네이티브 통합              │
│ NI VeriStand    │ 유연한 플러그인 구조, 다양한 I/O           │
│ OPAL-RT         │ 전력/전기 특화, 최소 25µs 스텝             │
│ Vector CANoe    │ 차량 네트워크(CAN/Ethernet/LIN) 특화       │
│ Renode          │ 오픈소스 임베디드 시뮬레이터 (저비용)       │
└─────────────────┴───────────────────────────────────────────┘
```

---

## 트러블슈팅

### 문제 1: HIL 시뮬레이터와 ECU 동기화 오류

```
증상: 시뮬레이터 타임스텝과 ECU 제어 주기가 맞지 않아
      결과가 불안정하거나 재현 불가능

원인:
  1. 시뮬레이터 스텝 크기 vs ECU 제어 주기 불일치
  2. PTP 동기화 없이 독립 클록 운용
  3. 네트워크 지연으로 시뮬레이터 입력 지연

해결:
  # 시뮬레이터와 ECU 모두 gPTP로 동기화
  # 시뮬레이터: ptp4l 마스터로 설정
  ptp4l -i eth0 -f /etc/ptp4l_grandmaster.conf &
  phc2sys -s eth0 -c CLOCK_REALTIME -w &

  # ECU: ptp4l 슬레이브로 설정
  # → 동기화 후 시뮬레이터 타임스텝 = ECU 제어 주기 정렬

  # 확인: 시뮬레이터-ECU 오프셋 모니터링
  pmc -u -b 0 'GET TIME_STATUS_NP' | grep offsetFromMaster
  # 목표: ±1µs 이내
```

### 문제 2: Fault Injection 후 ECU 복구 실패

```
증상: tc netem으로 패킷 손실 주입 후 "clear" 해도
      ECU가 Safe State에서 복구되지 않음

원인:
  1. ECU 안전 상태 복구 조건 미충족 (설계 의도)
  2. 복구 인터록: 수동 리셋 요구 (IEC 62304 요구일 수 있음)
  3. tc 설정이 완전히 해제되지 않음

확인:
  # tc 규칙 완전 삭제 확인
  tc qdisc show dev eth1
  # → "qdisc noqueue" 또는 기본 규칙만 표시되어야 함

  # 아직 규칙 남아있다면:
  tc qdisc del dev eth1 root 2>/dev/null || true
  tc qdisc del dev eth1 ingress 2>/dev/null || true

  # ECU 수동 리셋 신호
  hil.set_param("manual_reset", 1.0)
  time.sleep(0.1)
  hil.set_param("manual_reset", 0.0)
```

### 문제 3: 코드 커버리지가 목표에 미달

```
증상: HIL 테스트 후 MC/DC 커버리지 85% → 목표 100% 미달

분석:
  # 미커버 브랜치 확인
  gcovr --html-details coverage.html
  # coverage.html에서 빨간색(미커버) 라인 확인

  # 미커버 케이스 유형 예시:
  # 1. 오류 처리 코드 (else 분기) - 정상 시나리오에서 미실행
  # 2. 예외 상황 (overflow, NaN 체크)
  # 3. 컴파일 시 최적화로 제거된 데드 코드

해결:
  # 1. Fault Injection으로 오류 처리 코드 커버
  tc_estop_response() → else 분기 커버
  tc_torque_limit()   → 예외 처리 커버

  # 2. 데드 코드 제거 (미도달 분기 삭제 또는 defensive 코드 검토)
  # IEC 62304: 도달 불가 코드는 제거 또는 정당화 문서 필요

  # 3. 추가 테스트 케이스 도출
  # 미커버 라인 분석 → SRS 추가 항목 검토 → 테스트 케이스 추가
```

---

## Reference
- [dSPACE - HIL Simulation](https://www.dspace.com/en/inc/home/products/systems/hil.cfm)
- [MathWorks - Hardware-in-the-Loop Simulation](https://www.mathworks.com/solutions/verification-validation/hil-simulation.html)
- [NI - HIL Testing Basics](https://www.ni.com/en-us/innovations/white-papers/11/hardware-in-the-loop--hil--simulation-basics.html)
- [Vector - CANoe Network Simulation](https://www.vector.com/int/en/products/products-a-z/software/canoe/)
- [IEC 62304:2006+AMD1:2015 - Software lifecycle for medical devices](https://www.iec.ch/standards/iec-62304/)
- [ISO 26262-6:2018 - Product development at the software level](https://www.iso.org/standard/68385.html)
- [OPAL-RT - Real-Time Simulation](https://www.opal-rt.com/)
- [Speedgoat - Real-Time Target Machines](https://www.speedgoat.com/)
