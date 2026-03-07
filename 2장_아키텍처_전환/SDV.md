# Software Defined Vehicle (SDV) / Software Defined Robot (SDR)

## 개요
SDV(Software Defined Vehicle)는 소프트웨어가 자동차의 핵심 기능과 가치를 결정한다는 개념입니다. 하드웨어를 구매한 후에도 OTA 업데이트를 통해 기능이 추가되고 성능이 개선됩니다. 의료 로봇 분야에서는 동일 개념이 **SDR(Software Defined Robot)**으로 확장됩니다.

---

## SDV vs 기존 차량/로봇 비교

| 특성 | 기존 방식 | SDV / SDR |
|------|---------|-----------|
| 기능 확정 시점 | 출시 시 고정 | 소프트웨어 업데이트로 지속 확장 |
| 기능 추가 방법 | 새 ECU/하드웨어 교체 | OTA 무선 업데이트 |
| 유사 사례 | 피처폰 (기능 고정) | 스마트폰 (앱/OS 업데이트) |
| E/E 아키텍처 | 분산 ECU (기능별) | HPC + Zone ECU (중앙집중) |
| 소프트웨어 구조 | 하드웨어 의존적 | 하드웨어 추상화 (HAL) |
| 수익 모델 | 판매 시 확정 | Feature on Demand, 구독 서비스 |
| 진단 방식 | OBD2 포트 + 스캐너 | DoIP 원격 진단 |
| 업데이트 주기 | 5~7년 (차량 교체) | 수주~수개월 |

---

## SDV/SDR 기술 스택

```
SDV/SDR 기술 스택 (아래서 위로):

┌─────────────────────────────────────────────────┐
│  비즈니스 레이어                                  │
│  Feature on Demand, 구독 서비스, 원격 진단        │
│  Fleet Management, 사용 데이터 분석               │
├─────────────────────────────────────────────────┤
│  클라우드/OTA 플랫폼                              │
│  CI/CD 파이프라인, UPTANE 보안 OTA               │
│  A/B 테스트, 점진적 롤아웃, 롤백                  │
├─────────────────────────────────────────────────┤
│  애플리케이션 레이어                               │
│  AI 모델, 수술/주행 서비스, HMI                   │
├─────────────────────────────────────────────────┤
│  미들웨어 레이어                                  │
│  AUTOSAR Adaptive, ROS 2, DDS, SOME/IP          │
├─────────────────────────────────────────────────┤
│  가상화 레이어                                    │
│  Type-1 Hypervisor (QNX / ACRN), Container      │
├─────────────────────────────────────────────────┤
│  OS 레이어                                        │
│  RTOS (QNX Neutrino, Zephyr) + Linux (Yocto)    │
├─────────────────────────────────────────────────┤
│  하드웨어 레이어                                  │
│  HPC (GPU/NPU), Zone ECU, TSN 스위치             │
└─────────────────────────────────────────────────┘
```

---

## OTA (Over-The-Air) 업데이트

### OTA 아키텍처
```
클라우드 OTA 서버 (Backend)              차량 / 의료 로봇
┌────────────────────┐              ┌──────────────────────────┐
│  패키지 저장소      │              │  OTA Client (HPC)         │
│  서명/검증 서버     │◄─ HTTPS/TLS ─►│                          │
│  롤아웃 관리        │              │  ① 업데이트 확인 (폴링)   │
│  A/B 테스트         │              │  ② 다운로드 (백그라운드)  │
│  취약점 모니터링    │              │  ③ UPTANE 서명 검증      │
└────────────────────┘              │  ④ 설치 (비활성 파티션)  │
                                    │  ⑤ 재부팅 후 활성화      │
                                    │  ⑥ 헬스체크 (30초)       │
                                    │  ⑦ 확정 or 자동 롤백     │
                                    └──────────────────────────┘
```

### A/B 파티션 업데이트
```
eMMC/NVMe 스토리지 레이아웃:
  ┌────────────────────────────────────────────────┐
  │  Bootloader (U-Boot)    │  Secure Boot 검증    │
  ├────────────────────────┬─────────────────────  │
  │  파티션 A (Active)      │  파티션 B (Inactive) │
  │  OS v2.1.0 (현재 실행) │  OS v2.2.0 (기록 중) │
  │  App v1.5.0             │  App v1.6.0           │
  ├────────────────────────┴─────────────────────  │
  │  Data 파티션 (설정, 로그 - 공유)                │
  └────────────────────────────────────────────────┘

업데이트 절차:
  1. 파티션 B에 v2.2.0 기록 (서비스 중단 없음, ~30분)
  2. Bootloader에 B 파티션 부팅 요청 플래그 설정
  3. 재부팅 → 파티션 B로 전환
  4. 30초 헬스체크 (핵심 서비스 응답 확인)
  5. 정상: 파티션 B "활성" 확정, A는 롤백 대기
  6. 이상: 자동 롤백 → 파티션 A로 복귀 (재부팅)

장점: 업데이트 실패 시 무중단 이전 버전 복귀
```

### UPTANE 보안 OTA 프레임워크
```
UPTANE 이중 저장소 구조:

Image Repository (이미지 무결성 보장)
  ├─ Targets Metadata (소프트웨어 목록 + 해시)
  ├─ Snapshots Metadata (일관성 보장)
  └─ Timestamp Metadata (최신성 보장)

Director Repository (ECU별 업데이트 지시)
  ├─ Targets Metadata (특정 차량/로봇 ECU 지정)
  └─ ECU별 개별 서명 (타겟 공격 방지)

보안 위협 대응:
┌──────────────────────────┬─────────────────────────────┐
│ 공격 유형                 │ UPTANE 대응 방법             │
├──────────────────────────┼─────────────────────────────┤
│ 가짜 이미지 배포           │ Image + Director 이중 서명   │
│ 구버전 다운그레이드 공격   │ 버전 번호 + Timestamp 검증   │
│ ECU 타겟 공격             │ ECU별 개별 서명 메타데이터  │
│ 중간자 공격 (MITM)        │ TLS + 다층 서명 검증         │
│ 저장소 키 탈취            │ Online/Offline 키 분리       │
└──────────────────────────┴─────────────────────────────┘
```

---

## CI/CD 파이프라인

### GitLab CI/CD 파이프라인 정의
```yaml
# .gitlab-ci.yml: 의료 로봇 SDR 소프트웨어 파이프라인
stages:
  - build
  - test
  - analyze
  - package
  - staging
  - release

variables:
  TARGET_ARCH: aarch64
  TOOLCHAIN: aarch64-linux-gnu

# 1단계: 크로스 컴파일 빌드
build:arm64:
  stage: build
  image: registry.hub.docker.com/arm64v8/ubuntu:22.04
  script:
    - cmake -B build -DCMAKE_TOOLCHAIN_FILE=arm64.cmake -DCMAKE_BUILD_TYPE=Release
    - cmake --build build --parallel 8
  artifacts:
    paths: [build/]

# 2단계: 단위 테스트 + 커버리지
test:unit:
  stage: test
  script:
    - cd build && ctest --output-on-failure -j4
    - gcovr --xml coverage.xml --fail-under-line 80  # 라인 커버리지 80% 이상
  coverage: '/Total.*?(\d+\.?\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# 3단계: 정적 분석 (MISRA C / CERT C++)
analyze:misra:
  stage: analyze
  script:
    - cppcheck --addon=misra.py --suppress=misra-c2012-15.5 --error-exitcode=1 src/
    - clang-tidy src/**/*.cpp -- -I include/
  allow_failure: false

# 3단계: 보안 취약점 스캔
analyze:security:
  stage: analyze
  script:
    - flawfinder --error-level=2 src/
    - semgrep --config=auto --error src/

# 4단계: OTA 패키지 생성 및 서명
package:ota:
  stage: package
  script:
    - python3 scripts/create_ota_package.py
        --version $CI_COMMIT_TAG
        --target aarch64
        --sign-key $OTA_SIGNING_KEY
    - sha256sum dist/robot-sw-${CI_COMMIT_TAG}.tar.gz > dist/checksum.txt
  artifacts:
    paths: [dist/]

# 5단계: 스테이징 환경 배포 및 HIL 검증
staging:deploy:
  stage: staging
  environment: staging
  script:
    - ansible-playbook deploy-staging.yml -e "version=$CI_COMMIT_TAG"
    - python3 tests/hil/run_hil_tests.py --env staging --timeout 3600
  when: manual  # 수동 승인 후 실행

# 6단계: 점진적 프로덕션 롤아웃
release:production:
  stage: release
  environment: production
  script:
    - python3 scripts/ota_rollout.py
        --version $CI_COMMIT_TAG
        --strategy canary
        --phases "1%:10min,10%:30min,50%:1h,100%"
  when: manual
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/'
```

### Quality Gate 요구사항
```
배포 전 필수 통과 조건 (IEC 62304 Class C):
  ✓ 단위 테스트 라인 커버리지 ≥ 80% (MC/DC: 100%)
  ✓ MISRA C:2012 위반 0건 (Category 1, 2)
  ✓ 정적 분석 Critical 결함 0건
  ✓ 보안 취약점 High/Critical 0건
  ✓ RTM (Requirements Traceability Matrix) 100% 연결
  ✓ HIL 테스트 Pass Rate 100%
  ✓ OTA 패키지 서명 검증 통과
  ✓ 롤백 테스트 시나리오 통과
```

---

## 가상화: Hypervisor와 컨테이너

### Hypervisor 비교
| 타입 | 방식 | 예시 | 실시간성 | ASIL |
|------|------|------|---------|------|
| Type-1 Bare-metal | 하드웨어 직접 | QNX Hypervisor, ACRN, Xen | 우수 | ASIL-B/D |
| Type-2 Hosted | Host OS 위 실행 | KVM, VirtualBox | 제한적 | 불가 |
| 컨테이너 | OS 커널 공유 | Docker, Podman | OS 의존 | 불가 |

```
Type-1 Hypervisor 구성 (의료 로봇 HPC):

  HPC 하드웨어 (ARM Cortex-A78AE 12코어)
  ──────────────────────────────────────────
  QNX Hypervisor (Type-1, ARM Stage-2 MMU)
  ──────────────────┬───────────────────────
  Guest OS A        │  Guest OS B
  QNX Neutrino RTOS │  Ubuntu 22.04 LTS
  Core 0,1,2 전용   │  Core 3~11 할당
  ──────────────────│───────────────────────
  안전 제어          │  비안전 서비스
  관절 제어 (1ms)    │  AI 추론 (TensorRT)
  E-Stop 감시       │  OTA 관리 (ara::ucm)
  ASIL-B 격리        │  클라우드 연동 (MQTT)
  ──────────────────┴───────────────────────
  Shared Memory IPC: 512MB (격리된 영역)
  virtio-serial 채널: 제어 명령 교환
```

### 컨테이너 기반 서비스 배포
```bash
# 의료 로봇 AI 서비스 컨테이너화
docker run -d \
  --name surgical-ai-v2 \
  --runtime=nvidia \           # GPU 가속 (TensorRT)
  --cpuset-cpus="3,4,5,6" \   # CPU 코어 고정 (비실시간 코어)
  --memory="12g" \              # 메모리 제한
  --network=host \              # Ethernet DDS 직접 접근
  -v /opt/models:/models:ro \  # AI 모델 마운트 (읽기 전용)
  -v /dev/video0:/dev/video0 \ # 내시경 카메라 접근
  --security-opt no-new-privileges \
  registry.hospital.com/surgical-ai:v2.2.0

# OTA 업데이트: 새 버전으로 무중단 전환
docker pull registry.hospital.com/surgical-ai:v2.3.0
docker stop surgical-ai-v2 && docker rm surgical-ai-v2
docker run -d ... registry.hospital.com/surgical-ai:v2.3.0
```

---

## 의료 로봇 SDR 활용 사례

### Feature on Demand (FoD)
```
기본 구성 (구매 시 포함):
  ✓ 7자유도 기본 조종 (마스터-슬레이브)
  ✓ 내시경 4K 영상 스트리밍
  ✓ 기본 안전 기능 (E-Stop, 충돌 감지)

추가 구매 가능 소프트웨어 기능 (FoD):
  └─ AI 보조 수술 모듈      ($1,500/월 구독)
     └─ 실시간 조직 분류 (AI)
     └─ 자동 카메라 추적
  └─ 원격 수술 지원 모드    (규제 인증 후 활성화)
     └─ 전문의 원격 가이던스
  └─ 고급 영상 처리          ($300/월)
     └─ 4K HDR 색상 보정
     └─ 3D 재구성 실시간
  └─ 특수 수술 패키지        (기기별 라이선스)
     └─ 복강경 수술 최적화
     └─ 신경외과 정밀 모드

구현: 소프트웨어는 HPC에 암호화 상태로 존재
       라이선스 키 (HSM 검증) → 기능 활성화
```

### 원격 유지보수 및 모니터링
```
병원 수술 로봇                     제조사 서비스 센터 / 클라우드
      │                                      │
      │── DoIP: DTC 목록 자동 전송 ──────────►│
      │   (수술 종료 후 매일 자동 보고)        │
      │── MQTT: 운영 지표 (KPI) 전송 ────────►│
      │   (관절 마모, 모터 전류, 온도)         │  AI 분석
      │                                      │  예지 정비 예측
      │◄── 진단 처방 및 소프트웨어 패치 ────── │
      │    (설정 조정, 알고리즘 개선)          │
      │                                      │
      │── OTA: 검증된 업데이트 수신 ──────────►│
      │   (UPTANE 서명 → A/B 적용)            │

모니터링 지표 (MQTT Topic: hospital/robot/{serial}/telemetry):
  {
    "joint_wear": [0.12, 0.08, 0.15, ...],  // 관절 마모율
    "motor_current_ma": [450, 420, 480, ...], // 모터 전류
    "temperature_c": [38.5, 37.2, 39.1, ...], // 온도
    "surgery_count": 1247,                    // 총 수술 횟수
    "uptime_hours": 2340                      // 가동 시간
  }
```

---

## 규제 대응 (UNECE R155/R156 + FDA)

```
SDV/SDR 관련 규제 매핑:

UNECE R155 (사이버보안):
  ✓ CSMS (사이버보안 관리 시스템) 수립
  ✓ OTA 업데이트 보안 채널 (TLS 1.3 + UPTANE)
  ✓ 원격 진단 접근 제어 (인증/인가)
  ✓ 보안 사고 모니터링 및 대응 절차

UNECE R156 (소프트웨어 업데이트):
  ✓ SUMS (소프트웨어 업데이트 관리 시스템) 구축
  ✓ RxSWIN (규정 소프트웨어 식별 번호) 관리
  ✓ 업데이트 전 안전성 영향 평가
  ✓ 롤백 기능 (업데이트 실패 시 복원)

FDA 2023 사이버보안 지침 (의료기기):
  ✓ SBOM (Software Bill of Materials) 제출
  ✓ 취약점 공개 정책 수립
  ✓ 패치 배포 기간 보장 (제품 수명 전 기간)
  ✓ 보안 업데이트 메커니즘 설계

IEC 62304 (의료기기 소프트웨어):
  ✓ 소프트웨어 수명주기 프로세스 준수
  ✓ 변경 관리 (OTA 업데이트 = 소프트웨어 변경)
  ✓ 회귀 테스트 수행 (영향받는 기능 전체)
  ✓ 릴리스 노트 문서화 요구
```

---

## Reference
- [Eclipse Software Defined Vehicle](https://sdv.eclipse.org/)
- [SOAFEE - Scalable Open Architecture for Embedded Edge](https://soafee.io/)
- [UPTANE - Automotive OTA Security Standard](https://uptane.github.io/)
- [ACRN Hypervisor for Automotive](https://projectacrn.org/)
- [Linux Foundation ELISA - Safety Critical Linux](https://elisa.tech/)
- [UNECE R156 - Software Update Management](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software-update)
- [FDA 2023 Cybersecurity Guidance for Medical Devices](https://www.fda.gov/regulatory-information/search-fda-guidance-documents/cybersecurity-medical-devices-quality-system-considerations-and-content-premarket-submissions)
