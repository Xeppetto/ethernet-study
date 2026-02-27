# IEEE 802.1DG (Automotive Profile for TSN)

## 개요
TSN(Time Sensitive Networking)은 다양한 표준들의 집합체입니다. IEEE 802.1DG는 차량용 Ethernet에서 **반드시 지원해야 하는 TSN 기능을 정의하는 Automotive Profile**입니다. 상호 호환성(Interoperability)을 보장하고 과도한 옵션으로 인한 혼란을 방지합니다.

의료 로봇을 포함한 산업용 시스템에서도 IEEE 802.1DG를 기반으로 프로파일을 설계합니다.

---

## TSN 표준 전체 지도

```
TSN 표준 생태계:

시간 동기화:
  IEEE 802.1AS   ← gPTP (필수)

트래픽 스케줄링:
  IEEE 802.1Qbv  ← TAS (Time Aware Shaper, 필수)
  IEEE 802.1Qav  ← CBS (Credit-Based Shaper, 선택)
  IEEE 802.1Qbu  ← Frame Preemption (선택)
  IEEE 802.3br   ← Interspersing Express Traffic (선택)

신뢰성:
  IEEE 802.1CB   ← FRER 이중화 (선택)
  IEEE 802.1Qci  ← PSFP 폴리싱 (권장)

설정/관리:
  IEEE 802.1Qcc  ← 중앙집중형 네트워크 설정 (CNC/CUC)
  IEEE 802.1Qcx  ← YANG 데이터 모델

보안:
  IEEE 802.1AE   ← MACsec 암호화 (권장)

프로파일:
  IEEE 802.1DG   ← Automotive Profile (이 문서)
  IEC 60802      ← Industrial (공장) Profile
  ARINC 664 P7   ← Avionics (항공) Profile
```

---

## IEEE 802.1DG 필수/선택 요구사항

### 종단 장치(End Station) 요구사항

| TSN 기능 | 요구 수준 | 설명 |
|---------|---------|------|
| 802.1AS gPTP | **필수** | 시간 동기화 |
| 802.1Qbv TAS | 선택 | 스케줄된 트래픽 전송 |
| 802.1Qbu Preemption | 선택 | 프레임 선점 (eMAC/pMAC) |
| 802.1CB FRER | 선택 | 이중화 (ASIL-D 요구 시) |
| 802.1Qci PSFP | 선택 | 수신 폴리싱 |
| 802.1p Priority | **필수** | 8단계 우선순위 (PCP 0~7) |
| VLAN (802.1Q) | **필수** | 트래픽 격리 |
| NETCONF/YANG | 권장 | 원격 설정 관리 |

### 브리지(Bridge/Switch) 요구사항

| TSN 기능 | 요구 수준 | 설명 |
|---------|---------|------|
| 802.1AS Transparent/Boundary Clock | **필수** | 시간 동기화 전파 |
| 802.1Qbv TAS | **필수** | GCL 스케줄링 |
| 802.1Qci PSFP | **필수** | 트래픽 폴리싱 |
| 802.1CB FRER | 선택 | 이중화 |
| 802.1Qbu Preemption | 선택 | 프레임 선점 |
| 802.1AE MACsec | 선택 | 링크 암호화 |
| NETCONF/YANG | **필수** | 원격 설정 관리 |

---

## 성능 요구사항 (802.1DG 기준)

```
지연(Latency):
  Scheduled Traffic (TC7): < 100µs (단일 스위치 홉 기준)
  Best Effort Traffic:      제한 없음

지터(Jitter):
  Scheduled Traffic: < 1µs (TAS + 802.1AS 조합)

패킷 손실률(Packet Loss Rate):
  Scheduled Traffic:  0% (FRER 적용 시 링크 장애에도)
  Best Effort:        1% 이하 권장

시간 동기화 정밀도:
  동일 네트워크 내: < 1µs
  최대 7홉:         < 1µs (Transparent Clock 사용 시)
  802.1DG 목표:     < 900ns (end-to-end, 7홉)

링크 속도:
  최소: 100BASE-T1 (100Mbps)
  권장: 1000BASE-T1 (1Gbps)
  고속: 2.5GBASE-T1 / 5GBASE-T1 (차세대)
  MultiGig: 10BASE-T1S (멀티드롭, 저속 센서용)
```

---

## 차량 Ethernet 토폴로지

```
차량 내 TSN 네트워크 구성:

         ┌─────────────────────────────────────────┐
         │  Central Computing Unit (CCU/HPC)        │
         │  - 802.1AS GM (Primary)                  │
         │  - 802.1Qbv TAS 종단                     │
         └──────────────┬──────────────────────────┘
                        │ 1000BASE-T1 (1Gbps)
           ┌────────────┼────────────┐
           │            │            │
           ▼            ▼            ▼
     Zone Switch A  Zone Switch B  Zone Switch C
     (TSN Bridge)   (TSN Bridge)   (TSN Bridge)
           │            │            │
      ┌────┤        ┌───┤        ┌───┤
      │    │        │   │        │   │
  카메라  ADAS   LIDAR 레이더   모터 브레이크
  100BASE-T1    1000BASE-T1   1000BASE-T1

각 Zone Switch:
  802.1AS Transparent Clock
  802.1Qbv TAS (GCL: 안전>제어>센서>BE)
  802.1Qci PSFP (Babbling Idiot 방지)
```

---

## 네트워크 설정 관리 (CNC/CUC 아키텍처)

802.1DG는 NETCONF 프로토콜과 YANG 데이터 모델로 중앙에서 TSN 설정을 관리하도록 요구합니다.

### CUC/CNC 계층 구조

```
┌──────────────────────────────────────────────────────────────┐
│  CUC (Centralized User Configuration)                        │
│  - 애플리케이션 요구사항 정의 (지연, 대역폭, 신뢰성)          │
│  - "제어 트래픽 100µs 지연 보장" 요청                         │
│  - SOME/IP 서비스 디스커버리 통합                             │
└──────────────────────┬───────────────────────────────────────┘
                       │ YANG 모델 (NETCONF/RESTCONF)
┌──────────────────────▼───────────────────────────────────────┐
│  CNC (Centralized Network Configuration)                     │
│  - GCL 최적화 계산                                            │
│  - 최적 경로 결정 (토폴로지 기반)                             │
│  - PSFP 정책 계산                                            │
│  - FRER 경로 설정                                            │
└──────┬──────────────┬──────────────┬──────────────────────────┘
       │              │              │ NETCONF 설정 배포
  Switch A       Switch B       End Station
  (GCL 적용)    (GCL 적용)     (TX 설정)
```

### NETCONF/YANG으로 GCL 설정

```python
# NETCONF/YANG으로 GCL 설정 예시 (ncclient 사용)
from ncclient import manager

with manager.connect(
    host='192.168.1.10',  # TSN 스위치 IP
    port=830,
    username='admin',
    password='password',
    hostkey_verify=False
) as m:

    # 802.1Qbv GCL 설정 YANG XML
    config = """
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>eth0</name>
          <scheduled-traffic
              xmlns="urn:ieee:std:802.1Q:yang:ieee802-dot1q-sched">
            <gate-parameters>
              <cycle-time>
                <numerator>1000000</numerator>
                <denominator>1000000000</denominator>
              </cycle-time>
              <admin-control-list>
                <gate-control-entry>
                  <index>0</index>
                  <operation-name>set-gate-states</operation-name>
                  <time-interval-value>100000</time-interval-value>
                  <gate-states-value>128</gate-states-value>
                </gate-control-entry>
                <gate-control-entry>
                  <index>1</index>
                  <operation-name>set-gate-states</operation-name>
                  <time-interval-value>900000</time-interval-value>
                  <gate-states-value>127</gate-states-value>
                </gate-control-entry>
              </admin-control-list>
            </gate-parameters>
          </scheduled-traffic>
        </interface>
      </interfaces>
    </config>
    """
    m.edit_config(target='running', config=config)
    print("GCL 설정 완료")
```

---

## AUTOSAR와의 통합 (ara::tsync)

AUTOSAR Adaptive Platform은 TSN 시간 동기화를 **ara::tsync** 서비스로 추상화합니다.

```
AUTOSAR Adaptive 스택:

Application Layer
  └── ara::tsync API
        ├── GetCurrentTime()       ← 현재 시각 조회 (TAI 기반)
        ├── GetSyncStatus()        ← 동기화 상태 확인
        └── RegisterCallback()     ← 시간 점프 이벤트 콜백

ara::tsync Service
  └── IEEE 802.1AS (gPTP)          ← 하드웨어 시간 동기화
        └── linuxptp / phc2sys     ← Linux 구현

사용 예시 (C++):
  auto& tsync = ara::tsync::SynchronizedTimeBaseConsumer::Create("ClockDomain0");
  ara::core::Result<ara::tsync::BaseTime> time = tsync.GetCurrentTime();
  if (time.HasValue()) {
      uint64_t tai_ns = time.Value().ToNanoseconds();
  }

ara::tsync 도메인:
  ClockDomain0 = 차량 기준 시계 (802.1AS 도메인 0)
  ClockDomain1 = 인포테인먼트 (802.1AS 도메인 1)
```

---

## SOME/IP와 TSN 연동

```
SOME/IP (Scalable service-Oriented MiddlewarE over IP):
  차량 SOA(Service-Oriented Architecture) 미들웨어

TSN 연동:
  SOME/IP 서비스 발견 → CUC에 스트림 요구사항 전달
  CUC → CNC → GCL/CBS/FRER 자동 설정

흐름:
  앱 A: "제어 서비스 500Hz, 10Mbps, 2ms 지연" 등록
       │
       ▼
  SOME/IP SD (Service Discovery)
       │
       ▼
  CUC: 요구사항 수집 → CNC 전달
       │
       ▼
  CNC: 최적 GCL 계산 → 스위치에 NETCONF 배포
       │
       ▼
  TSN 스위치: GCL 적용 → 2ms 지연 보장
```

---

## 802.1DG와 산업 프로파일 비교

| 특성 | 802.1DG (Automotive) | IEC 60802 (Industrial) |
|------|---------------------|----------------------|
| 목표 도메인 | 차량 내 네트워크 | 공장 자동화, 프로세스 산업 |
| 최대 홉 수 | 7 | 제한 없음 |
| 최소 사이클 타임 | 250µs | 31.25µs |
| 신뢰성 | FRER 선택 | FRER 필수 (안전 클래스) |
| 보안 | 802.1AE MACsec 권장 | 선택 |
| 시간 동기화 | 802.1AS (< 900ns) | 802.1AS (< 1µs) |
| 스케줄링 | TAS | TAS + CBS |
| 설정 관리 | NETCONF/YANG | NETCONF/YANG |

---

## 의료 로봇 커스텀 TSN 프로파일

의료 로봇 업계에 특화된 TSN 프로파일은 표준화 진행 중입니다. 현재는 802.1DG를 기반으로 의료 안전 요구사항을 추가합니다.

```
의료 로봇 TSN 프로파일 (비공식):

필수 요구사항:
  ✓ 802.1AS: < 500ns 동기화 정밀도 (7홉)
  ✓ 802.1Qbv TAS: 사이클 타임 ≤ 2ms (500Hz 제어)
  ✓ 802.1Qci PSFP: Babbling Idiot 방어
  ✓ 802.1CB FRER: 안전 제어 스트림 이중화
  ✓ VLAN 격리: 안전(VLAN 10)/제어(VLAN 20)/진단(VLAN 40)
  ✓ 802.1AE MACsec: 수술 데이터 암호화

안전 규제:
  ✓ IEC 62304: 의료 소프트웨어 수명주기
  ✓ IEC 62443: 산업 사이버보안
  ✓ IEC 60601-1: 의료 기기 전기 안전
  ✓ IEC 61508 / ISO 26262: ASIL-B 이상 (관절 제어)

성능 목표:
  제어 루프 지연: < 1ms end-to-end
  안전 신호 지연: < 2ms
  시간 동기화:    < 1µs
  네트워크 가용성: 99.9999% (FRER)
  패킷 손실률:    0% (안전 스트림)
```

---

## Reference
- [IEEE P802.1DG - TSN Profile for Automotive](https://1.ieee802.org/tsn/802-1dg/)
- [IEC 60802 - TSN Profile for Industrial Automation](https://webstore.iec.ch/)
- [Avnu Alliance - Automotive TSN](https://avnu.org/automotive/)
- [RFC 6241 - NETCONF Protocol](https://datatracker.ietf.org/doc/html/rfc6241)
- [IEEE 802.1Qcc - Stream Reservation Protocol (SRP) Extensions](https://standards.ieee.org/ieee/802.1Qcc/6135/)
- [AUTOSAR - ara::tsync Specification](https://www.autosar.org/standards/adaptive-platform/)
- [OPEN Alliance - TC10 Automotive Ethernet](https://opensig.org/automotive-ethernet/)
