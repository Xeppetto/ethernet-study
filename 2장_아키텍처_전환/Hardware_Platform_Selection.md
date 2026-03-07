# 하드웨어 플랫폼 선택 가이드: 의료 로봇 HPC와 네트워크

## 선택이 아닌 설계

의료 로봇의 하드웨어 플랫폼을 선택하는 것은 단순히 사양 시트를 비교하는 작업이 아니다. 선택한 SoC가 WCRT를 보장할 수 있는지, NIC가 TSN TAS를 하드웨어로 오프로드할 수 있는지, 스위치가 802.1AS 하드웨어 타임스탬핑을 지원하는지—이 모든 것이 네트워크 지터에 직접 영향을 미치고, 결국 수술 정밀도에 영향을 미친다.

"나중에 소프트웨어로 해결하면 된다"는 말은 실시간 시스템에서 통하지 않는다. 하드웨어 TAS 오프로드 없이 소프트웨어 taprio로 지터를 ±10µs 이내로 만드는 것은 불가능하다. 하드웨어 타임스탬핑 없이 gPTP를 100ns 정밀도로 맞추는 것도 불가능하다. 플랫폼 선택이 성능의 상한을 결정한다.

---

## HPC SoC 선택

### NVIDIA Jetson Orin 시리즈

NVIDIA의 Jetson Orin은 자율주행과 의료 로봇용 고성능 AI 플랫폼이다. CPU와 GPU가 동일 패키지에 통합되어 있고, CUDA 생태계가 완전히 지원된다.

```
Jetson Orin 라인업 비교:

              Orin NX 8G   Orin NX 16G  Orin AGX 32G  Orin AGX 64G
CPU           6× A78AE     8× A78AE     12× A78AE     12× A78AE
GPU           1024 CUDA    1024 CUDA    2048 CUDA     2048 CUDA
DLA           1× 25 TOPS   2× 25 TOPS   2× 40 TOPS    2× 40 TOPS
메모리        8GB          16GB         32GB          64GB
TDP           10~25W       10~25W       15~40W        15~60W
TSN NIC       외부 필요    외부 필요    외부 필요     외부 필요
의료 로봇 적합성 AI 보조  AI + 제어    전체 HPC      최고사양 HPC
```

**Orin의 한계:** Jetson Orin은 내장 TSN NIC가 없다. 외부 Intel I225 또는 Marvell AQTION NIC를 PCIe로 연결해야 TSN을 사용할 수 있다. 또한 Orin의 온보드 GbE(1000BASE-T)는 하드웨어 PTP 타임스탬핑을 지원하지 않아 gPTP 정밀도가 제한된다.

**권장 사용 사례:** Vision DC(AI 영상 분석), AI 추론 전용 파티션.

### NXP S32G2: 차량·의료 게이트웨이 SoC

S32G2는 NXP가 차량 네트워크 게이트웨이를 위해 설계한 SoC다. TSN이 SoC 내부에 통합되어 있어 추가 NIC 없이 하드웨어 TSN이 가능하다.

```
NXP S32G274A 주요 사양:
  CPU: 4× Cortex-A53 (1GHz, 범용) + 4× Cortex-M7 (400MHz, 실시간)
  메모리: 최대 4GB LPDDR4
  네트워크: 3× GMAC (TSN 802.1Qbv/AS/Qav 하드웨어 내장)
  CAN FD: 4채널
  FlexRay: 2채널 (선택)
  PTP: 하드웨어 타임스탬프 내장, < 10ns 정밀도
  기능 안전: ISO 26262 ASIL-D 달성 경로
  전력: 8~15W (차량 조건)

의료 로봇 적합 역할:
  - Zone Controller (CAN FD + TSN 게이트웨이)
  - Safety Domain Controller (ASIL-D Cortex-M7)
  - Network Gateway (다수의 TSN 포트 활용)
```

**S32G2의 강점:** Cortex-A53과 Cortex-M7이 같은 칩에 있어 하이퍼바이저 없이 코어 수준에서 안전/비안전 분리가 된다. M7 코어가 ASIL-D 제어를 담당하고 A53 코어가 Linux/ROS 2를 돌린다.

### Qualcomm SA8295P: 차량 인포테인먼트/AI

SA8295P는 Qualcomm의 Snapdragon Ride 플랫폼 최고 사양으로, AI 추론과 멀티미디어에 특화되어 있다. 의료 로봇에서 HMI DC나 Vision DC 역할에 적합하다.

```
SA8295P 주요 사양:
  CPU: 8× Cortex-A78AE (최대 3.0GHz)
  GPU: Adreno 735 (높은 그래픽 성능)
  AI: Hexagon DSP + AI Engine (30+ TOPS)
  메모리: 32GB LPDDR5
  영상: 12개 카메라 동시 처리 ISP
  TSN: 외부 NIC 필요
  의료 로봇 역할: Vision DC, HMI DC
```

### Renesas R-Car V4H: 차량 ADAS

R-Car V4H는 Renesas의 차세대 ADAS SoC로, 안전 기능과 AI를 균형 있게 갖췄다.

```
R-Car V4H 주요 사양:
  CPU: 8× Cortex-A76 + 2× Cortex-R52 (안전)
  AI: 연산 효율 중심 (16 TOPS)
  네트워크: Ethernet AVB/TSN 내장
  안전: ISO 26262 ASIL-B (일부 IP)
  의료 로봇 역할: Motion DC, Safety Domain (ASIL-B 기능)
```

---

## TSN NIC 선택

Zone ECU나 HPC에서 외부 TSN NIC가 필요할 때 선택 기준:

### Intel I225-V / I226-V

현재 가장 많이 사용되는 PCIe TSN NIC다. 특히 x86 기반 HPC에서 표준에 가깝다.

```
Intel I225-V/I226-V 특징:
  속도: 2.5GbE (2.5GBASE-T)
  TSN 지원: 802.1Qbv (TAS), 802.1Qav (CBS), 802.1AS (PTP)
  PTP 정밀도: 하드웨어 타임스탬프 < 5ns
  TAS 오프로드: 최대 4 Gate Entry (Linux taprio HW 모드 지원)
  PCIe: Gen3 x1
  Linux 드라이버: igc (upstream 5.5+)
  가격: $10~15 (소매)

확인 사항:
  # TAS 하드웨어 오프로드 지원 확인
  ethtool -T eth0 | grep "hardware-transmit"
  # hardware-transmit 있어야 HW TAS 가능

  # taprio HW 모드 설정
  tc qdisc add dev eth0 parent root taprio \
      ... flags 0x2  # 0x2 = TAPRIO_FLAGS_FULL_OFFLOAD
```

### Intel I210 / I211

이전 세대지만 1GbE에서 검증된 TSN NIC다. 산업 자동화와 임베디드 시스템에서 폭넓게 사용된다.

```
Intel I210 특징:
  속도: 1GbE
  TSN 지원: 802.1Qav (CBS), 802.1AS (PTP) — TAS 하드웨어 미지원
  PTP 정밀도: 하드웨어 타임스탬프 < 10ns
  Linux 드라이버: igb
  주의: taprio는 소프트웨어 모드만 가능 (지터 높음)

  # I210에서 CBS (802.1Qav) 설정
  tc qdisc replace dev eth0 root handle 100 mqprio \
      num_tc 3 map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 \
      queues 1@0 1@1 2@2 hw 0

  tc qdisc add dev eth0 parent 100:1 cbs \
      idleslope 98688 sendslope -901312 hicredit 153 locredit -1389 \
      offload 1
```

### NXP TJA1103 / TJA1120: 차량용 100/1000BASE-T1

Zone ECU와 HPC 사이 링크가 100BASE-T1이나 1000BASE-T1(Single Pair Ethernet)인 경우, NXP TJA1103/1120이 대표적인 PHY 칩이다.

```
NXP TJA1103 (100BASE-T1):
  속도: 100Mbps, 단일 꼬임선
  케이블 거리: 최대 15m (비차폐), 40m (차폐)
  PoDL: 지원 (전원 + 데이터 동시)
  TSN: 802.1AS 하드웨어 타임스탬프 지원
  온도: -40°C ~ +125°C (차량 등급)
  인터페이스: RGMII / MII to SoC

NXP TJA1120 (1000BASE-T1):
  속도: 1Gbps, 단일 꼬임선
  TSN: 802.1AS, 802.1Qbv 지원
  의료 로봇 Zone 3 (내시경 카메라 링크)에 적합
```

### Marvell 88Q2221 / 88Q5050: 차량용 TSN PHY/스위치

```
Marvell 88Q2221 (PHY):
  100BASE-T1 / 1000BASE-T1 듀얼 모드
  TSN PHY: 802.1AS 하드웨어 지원
  EMI 저항: AEC-Q100 Grade 0 (가장 혹독한 조건)
  의료 적합: 의료 환경의 전기 수술 기구 EMI에 강건

Marvell 88Q5050 (TSN 스위치):
  5포트 기가비트 TSN 스위치
  802.1Qbv TAS, 802.1Qav CBS, 802.1AS 모두 하드웨어 지원
  PCIe 인터페이스 (HPC에 직접 연결 가능)
  낮은 레이턴시: < 1µs cut-through 모드
```

---

## TSN 스위치 선택

수술실 네트워크의 TSN 스위치는 Zone ECU와 HPC 사이, 그리고 여러 Domain Controller를 연결하는 핵심 인프라다.

### 필수 확인 항목

```
TSN 스위치 선택 체크리스트:

기능 지원:
  ☐ 802.1AS (gPTP) — 나노초 하드웨어 타임스탬프
  ☐ 802.1Qbv (TAS) — 하드웨어 GCL, 128 entry 이상
  ☐ 802.1Qav (CBS) — 포트당 CBS 큐
  ☐ 802.1Qbu (프레임 선점) — 권장
  ☐ 802.1CB (FRER) — 의료 로봇 이중화 필수
  ☐ 802.1Q VLAN — 다수 VLAN 지원 (최소 16개)
  ☐ 802.1X 포트 인증 — 보안 요구사항

성능:
  ☐ 포트당 큐 수: 8개 이상
  ☐ TAS GCL 엔트리: 256개 이상 (복잡한 스케줄에)
  ☐ 컷스루(Cut-Through) 지연: < 2µs
  ☐ PTP 타임스탬프 정밀도: < 10ns

관리:
  ☐ NETCONF/YANG 관리 (자동화 설정)
  ☐ SNMP/RMON 모니터링
  ☐ 이중화 전원 입력
  ☐ DIN 레일 또는 패널 마운트 옵션
```

### 주요 제품 비교

| 제품 | 제조사 | 포트 | 특징 | 적합 용도 |
|---|---|---|---|---|
| IE3400H | Cisco | 8× 1GbE + 2× 10GbE | NETCONF, 강력한 관리 | 병원 MES 네트워크 |
| PT-7528 | Moxa | 최대 28포트 | 산업용 강건성, DIN 레일 | 수술실 제어반 |
| FL SWITCH 2xxx | Phoenix Contact | 4~24포트 | IEC 61850, 의료 인증 | 의료기기 네트워크 |
| VSC9953 (칩) | Microchip | 4포트 TSN | 임베디드 Zone용 | Zone Controller 내장 |
| SJA1110 (칩) | NXP | 5~10포트 | 차량용, ASIL-B | Zone ECU 내장 스위치 |
| Marvell 88Q5050 (칩) | Marvell | 5포트 | PCIe 연결, HPC 내장 | HPC 내부 스위치 |

**의료 로봇 권장 구성:**
- 수술실 백본: Cisco IE3400H 또는 Moxa PT-7528 (관리 기능 중요)
- Zone Controller 내장: NXP SJA1110 또는 Microchip VSC9953 (소형화)
- HPC 내부 스위치: Marvell 88Q5050 (PCIe, 낮은 지연)

---

## 메모리 선택과 크기 산정

HPC 메모리 요구사항 계산:

```
메모리 용도별 산정 (의료 로봇 HPC):

AI 모델 (GPU 메모리):
  ResNet-50 (영상 분류): ~100MB
  YOLOv8-x (실시간 객체 인식): ~170MB
  Depth Estimation 모델: ~250MB
  합계: ~520MB GPU 메모리 (+ 실행 버퍼 × 2)
  필요: 최소 2GB GPU 메모리

ROS 2 노드 (CPU 메모리):
  각 노드: 50~200MB RSS
  10개 노드: ~1GB
  DDS 히스토리 버퍼: ~200MB
  합계: ~1.2GB

OS 및 런타임:
  Linux OS (Yocto): ~500MB
  QNX 파티션: ~128MB
  페이지 캐시 여유: ~2GB

안전 여유 (×1.5):
  (1.2GB + 0.5GB + 2GB) × 1.5 = ~5.5GB

권장: 8GB 이상 (Orin NX 8G: 최소, 16G: 권장)
```

**LPDDR5 vs LPDDR4X:** LPDDR5는 더 높은 대역폭(~68GB/s)과 낮은 전력 소비를 제공한다. AI 추론은 메모리 대역폭에 민감하므로, 동일한 TOPS라도 LPDDR5 시스템이 실효 성능이 높다.

---

## 스토리지: eMMC vs NVMe

```
스토리지 선택 기준:

eMMC 5.1:
  순차 읽기: ~300MB/s
  순차 쓰기: ~150MB/s
  랜덤 4K 읽기: ~20K IOPS
  크기: 32~256GB
  장점: 소형, 납땜 고정 (진동 강건), 저전력
  적합: Zone ECU, 소형 Domain Controller
  의료 로봇: 대부분의 Zone ECU에 적합

NVMe M.2 (PCIe Gen3):
  순차 읽기: ~3,500MB/s
  순차 쓰기: ~2,500MB/s
  랜덤 4K 읽기: ~500K IOPS
  크기: 256GB~2TB
  장점: 매우 빠른 읽기/쓰기, AI 모델 빠른 로드
  단점: 외부 커넥터 (진동 환경에서 커넥터 신뢰성 고려)
  적합: HPC (AI 모델 로딩, 수술 영상 기록)
  의료 로봇: AI 기능 HPC에는 NVMe 권장
```

AI 모델을 OTA 업데이트로 교체하는 경우, NVMe의 빠른 쓰기 속도가 업데이트 시간을 크게 단축한다. ResNet-50 모델 100MB를 eMMC에 쓰면 약 0.7초, NVMe에 쓰면 0.04초—수술 전 점검 시간에 영향을 준다.

---

## 하드웨어 선정 프로세스

의료 로봇 하드웨어를 최종 선정하기 전 거쳐야 할 단계:

**1단계: 요구사항 명세화.** 제어 루프 주기, 허용 지터, 필요 TOPS, 안전 등급(ASIL), 동작 온도, 인증 요구사항을 수치로 명세.

**2단계: 하드웨어 타임스탬핑 검증.** 후보 플랫폼에서 linuxptp(ptp4l)를 실행해 PTP 동기화 정밀도를 직접 측정. 목표: < 100ns 오차.

**3단계: TAS 오프로드 확인.** `ethtool -T` 또는 제조사 데이터시트로 하드웨어 TAS 지원 확인. `tc qdisc add ... taprio ... flags 0x2` 성공 여부 테스트.

**4단계: 실부하 WCRT 측정.** cyclictest를 최대 부하(CPU/IO/네트워크) 상태에서 24시간 이상 실행. 목표 최악 지연 이내 유지 확인.

**5단계: 열 해석.** TDP 조건에서 2시간 연속 동작 후 CPU 온도 로그 확인. 스로틀링 없음 확인.

**6단계: IEC 62304 적합성 검토.** 선택한 하드웨어의 SOUP 목록 작성, 공급망 보증(품질 계획, 변경 통지 절차) 확인.

---

## 참고 문헌

- Intel I225/I226 Datasheet — Intel Corporation
- NXP S32G2 Product Brief — NXP Semiconductors
- NXP TJA1103 Data Sheet — NXP Semiconductors
- Marvell 88Q5050 Automotive TSN Switch Datasheet — Marvell
- NVIDIA Jetson Orin Technical Reference Manual — NVIDIA
- IEEE 802.1AS-2020: Timing and Synchronization for Time-Sensitive Applications
- Linux ptp4l manual: https://linuxptp.sourceforge.net/
- IEC 62304:2006/AMD1:2015 §7.1: Hardware Safety Requirements

---

*관련: [2장 중앙집중형 컴퓨팅](./Centralized_Computing.md) | [1장 Ethernet 속도 표준](../1장_Ethernet_기초/Ethernet_Speed_Standards.md)*
