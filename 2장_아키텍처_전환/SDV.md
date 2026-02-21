# Software Defined Vehicle (SDV) / Software Defined Robot (SDR)

## 개요
SDV(Software Defined Vehicle)는 소프트웨어가 자동차의 핵심 기능과 가치를 결정한다는 개념입니다. 하드웨어를 구매한 후에도 OTA 업데이트를 통해 기능이 추가되고 성능이 개선됩니다. 의료 로봇 분야에서는 동일 개념이 **SDR(Software Defined Robot)**으로 확장됩니다.

---

## SDV의 핵심 가치

```
기존 하드웨어 중심:
  출시 시 기능 = 최종 기능
  기능 추가 = 새 장비 구매

SDV:
  출시 시 기능 < 사용 중 기능 (지속 업데이트)
  기능 추가 = OTA 소프트웨어 업데이트

비교: 스마트폰 패러다임
  iPhone 출시 → iOS 업데이트 → 새 기능 추가 (카메라 개선, AI 기능 등)
  Tesla 출시  → OTA 업데이트 → Autopilot 성능 개선, 새 모드 추가
```

---

## SDV/SDR을 위한 기술 스택

```
SDV 기술 스택 (아래서 위로):

┌─────────────────────────────────────────────────┐
│  비즈니스 레이어                                  │
│  Feature on Demand, 구독 서비스, 원격 진단        │
├─────────────────────────────────────────────────┤
│  클라우드/OTA 플랫폼                              │
│  CI/CD 파이프라인, 패키지 관리, A/B 테스트        │
├─────────────────────────────────────────────────┤
│  애플리케이션 레이어                               │
│  AI 모델, 서비스 로직, HMI                        │
├─────────────────────────────────────────────────┤
│  미들웨어 레이어                                  │
│  AUTOSAR Adaptive, ROS 2, DDS, SOME/IP          │
├─────────────────────────────────────────────────┤
│  가상화 레이어                                    │
│  Hypervisor, Container Runtime                   │
├─────────────────────────────────────────────────┤
│  OS 레이어                                        │
│  RTOS (QNX, Zephyr) + Linux (Yocto)             │
├─────────────────────────────────────────────────┤
│  하드웨어 레이어                                  │
│  HPC, Zone ECU, Ethernet 백본, TSN 스위치        │
└─────────────────────────────────────────────────┘
```

---

## OTA (Over-The-Air) 업데이트

### OTA 아키텍처
```
클라우드 OTA 서버                          차량/로봇
(Backend)
┌──────────────────┐                ┌──────────────────────┐
│  패키지 저장소    │                │  OTA Client           │
│  서명/검증 서버   │◄── HTTPS ─────►│  (HPC에 탑재)         │
│  롤아웃 관리      │                │                       │
│  A/B 테스트       │                │  ① 업데이트 확인      │
└──────────────────┘                │  ② 다운로드 (백그라운드)│
                                    │  ③ 서명 검증           │
                                    │  ④ 설치 (비활성 파티션) │
                                    │  ⑤ 재부팅 후 활성화    │
                                    │  ⑥ 헬스체크 후 확정    │
                                    └──────────────────────┘
```

### A/B 파티션 업데이트 (Android/Linux OTA 방식)
```
스토리지:
  파티션 A (현재 실행 중): v1.2.0  ← 현재 부팅
  파티션 B (업데이트 대상): v1.3.0  ← 백그라운드 기록

업데이트 절차:
  1. B 파티션에 v1.3.0 기록 (서비스 중단 없음)
  2. 재부팅 후 B로 전환
  3. 정상 동작 확인 후 B를 "활성" 확정
  4. 문제 발생 시 A로 자동 롤백 (Rollback)

장점: 업데이트 실패 시 이전 버전으로 안전 복귀
```

### UPTANE 보안 표준
차량/의료 로봇 OTA의 보안 위협을 대응하는 프레임워크입니다.

```
공격 유형                  UPTANE 대응 방법
────────────────────────   ─────────────────────────────────
가짜 이미지 배포           Director + Image Repository 이중 서명
구버전 다운그레이드 공격   버전 번호 + Timestamp 검증
ECU 특정 공격              ECU별 개별 서명 (Per-ECU Metadata)
중간자 공격                TLS + 다층 서명 검증
```

---

## 가상화: Hypervisor와 컨테이너

### Hypervisor 비교
| 타입 | 방식 | 예시 | 실시간성 |
|------|------|------|---------|
| Type-1 (Bare-metal) | 하드웨어 직접 접근 | Xen, QNX Hypervisor, ACRN | 우수 |
| Type-2 (Hosted) | Host OS 위에서 실행 | VirtualBox, KVM | 제한적 |

```
Type-1 Hypervisor 구성 (의료 로봇 HPC):

  HPC 하드웨어 (ARM Cortex-A / x86)
  ────────────────────────────────────
  QNX Hypervisor (Type-1)
  ────────────────┬───────────────────
  Guest OS A      │  Guest OS B
  QNX Neutrino    │  Ubuntu 22.04
  RTOS            │  Linux
  ────────────────│───────────────────
  안전 제어        │  비안전 서비스
  관절 제어        │  AI 추론
  실시간 < 1ms    │  OTA 관리
  ASIL-B          │  일반 앱
  ────────────────┴───────────────────
  공유 메모리 통신 (격리된 영역)
```

### 컨테이너 (Docker/Podman)
```bash
# 로봇 AI 서비스 컨테이너화 예시
docker run -d \
  --name robot-ai-service \
  --runtime=nvidia \           # GPU 가속
  -v /dev/bus:/dev/bus \       # 하드웨어 접근
  --network=host \             # Ethernet 직접 접근
  robot-ai:v1.3.0

# 컨테이너 이미지 OTA 업데이트
docker pull registry.hospital.com/robot-ai:v1.4.0
docker stop robot-ai-service
docker run ... robot-ai:v1.4.0
```

---

## CI/CD 파이프라인

```
소스 코드 변경 (Git Push)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│  CI (Continuous Integration) Pipeline                │
│                                                     │
│  ① 코드 정적 분석 (MISRA-C, AUTOSAR Coding)        │
│  ② 단위 테스트 (Google Test, Unity)                │
│  ③ 빌드 (크로스 컴파일: x86 → ARM)                 │
│  ④ 통합 테스트 (SIL: Software-in-the-Loop)         │
│  ⑤ HIL 테스트 (Hardware-in-the-Loop)              │
│  ⑥ 보안 취약점 스캔                                │
└─────────────────────────────────────────────────────┘
         │ (모든 테스트 통과)
         ▼
┌─────────────────────────────────────────────────────┐
│  CD (Continuous Delivery) Pipeline                  │
│                                                     │
│  ① OTA 패키지 서명 (코드 서명 키)                   │
│  ② 스테이징 환경 배포 → 검증                        │
│  ③ 점진적 롤아웃 (1% → 10% → 100%)                │
│  ④ 모니터링 및 자동 롤백                            │
└─────────────────────────────────────────────────────┘
```

---

## 의료 로봇 SDR 활용 사례

### Feature on Demand (FoD)
```
기본 구성:          추가 구매 가능 기능:
  기본 수술 제어      └─ AI 보조 수술 (구독 $500/월)
                     └─ 원격 수술 모드 (인증 후 활성화)
                     └─ 고급 영상 처리 (4K HDR)
                     └─ 특수 수술 도구 지원 패키지

구현: 소프트웨어는 HPC에 이미 존재, 라이선스 키로 활성화
```

### 원격 유지보수
```
병원 로봇                    제조사 서비스 센터
      │                             │
      │── 진단 데이터 전송 (DoIP) ──►│
      │   (DTC, 센서 로그, AI 모델 성능)
      │                             │ 분석
      │◄── 원격 진단 및 처방 ──────  │
      │    (설정 조정, 펌웨어 패치)   │
```

---

## Reference
- [Eclipse Software Defined Vehicle](https://sdv.eclipse.org/)
- [SOAFEE - Scalable Open Architecture for Embedded Edge](https://soafee.io/)
- [UPTANE - Automotive OTA Security](https://uptane.github.io/)
- [ACRN Hypervisor for Automotive](https://projectacrn.org/)
- [Linux Foundation ELISA - Safety Linux](https://elisa.tech/)
