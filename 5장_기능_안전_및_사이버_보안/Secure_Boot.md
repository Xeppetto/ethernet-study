# Secure Boot (보안 부팅)

## 개요

Secure Boot는 시스템 전원 인가부터 애플리케이션 실행까지 모든 소프트웨어의 무결성을 암호학적으로 검증하는 기술입니다. 공격자가 악성 펌웨어를 주입하거나 부트로더를 변조하여 시스템 제어권을 탈취하는 것을 원천 차단합니다. FDA, MDR, ISO/SAE 21434 모두 의료 기기/자동차 소프트웨어 무결성(Software Integrity)을 명시적으로 요구합니다.

Secure Boot가 없는 시스템은 아무리 강력한 통신 보안(TLS, MACsec)을 적용해도, 부팅 과정에서 악성 코드가 주입되면 모든 보안 장치를 우회할 수 있습니다. "신뢰는 하드웨어 ROM에서 시작된다"는 원칙이 핵심입니다.

> **의료 로봇 관점**: 수술 로봇 ECU가 수술 중 원격으로 악성 펌웨어를 주입받는다면, TLS 암호화가 되어있어도 소용없습니다. Secure Boot는 이런 공격을 부팅 단계에서 차단하는 첫 번째 방어선입니다. FDA 2023 가이드라인의 "Secure by Design" 요건에서 소프트웨어 무결성 검증은 핵심 항목이며, UNECE R155 Annex 5의 카테고리 3(업데이트 위협) 대응 방안으로도 Secure Boot가 명시됩니다. `UNECE_R155_R156.md`의 SUMS(R156)와 함께 읽으면 OTA 업데이트 보안 전체 흐름을 이해할 수 있습니다.

---

## Chain of Trust (신뢰 사슬)

```
Secure Boot 검증 체계:

전원 인가
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  ROM (변경 불가) - Root of Trust                      │
│  - SoC 제조 시 퓨즈(OTP Fuse)에 공개 키 해시 저장    │
│  - 첫 번째 부트로더(BL1)의 서명 검증                  │
└──────────────────────────┬───────────────────────────┘
                           │ 검증 성공
                           ▼
┌──────────────────────────────────────────────────────┐
│  BL1 (First Stage Bootloader)                        │
│  - 하드웨어 초기화 (Clock, DDR 기초 설정)             │
│  - BL2의 서명 검증                                    │
│  - 저장 위치: On-chip ROM or Secure SRAM             │
└──────────────────────────┬───────────────────────────┘
                           │ 검증 성공
                           ▼
┌──────────────────────────────────────────────────────┐
│  BL2 (Second Stage Bootloader, U-Boot/Trusted Firmware│
│  - DRAM 초기화 및 자가 테스트                         │
│  - BL31 (EL3 Runtime, TrustZone 초기화)              │
│  - Kernel Image 서명 검증                             │
└──────────────────────────┬───────────────────────────┘
                           │ 검증 성공
                           ▼
┌──────────────────────────────────────────────────────┐
│  Linux Kernel + Device Tree                          │
│  - dm-verity로 Root Filesystem 검증                  │
│  - 서명된 커널 모듈만 로드 (CONFIG_MODULE_SIG)        │
└──────────────────────────┬───────────────────────────┘
                           │ 검증 성공
                           ▼
┌──────────────────────────────────────────────────────┐
│  Application (ROS 2, 제어 소프트웨어)                 │
│  - IMA (Integrity Measurement Architecture)          │
│  - 실행 파일 해시 검증 후 실행                         │
└──────────────────────────────────────────────────────┘

검증 실패 시:
  → 즉시 부팅 중단
  → 오류 코드 HSM/UART 출력
  → Recovery Mode 진입 (제한된 복구 기능)
  → 안전 상태(Safe State) 유지
```

---

## 서명 알고리즘

```
Secure Boot 서명 알고리즘:

RSA-4096 + SHA-256:
  - 가장 널리 사용 (호환성 우수)
  - 단점: 키 크기 크고 느림 (임베디드에 부담)
  - 퓨즈 저장: 공개 키 해시 (SHA-256, 32 Byte)

ECDSA P-256 + SHA-256:
  - 컴팩트 (공개 키 32 Byte, 서명 64 Byte)
  - 빠른 검증 (저전력 MCU에 적합)
  - 권장: ARM TrustZone 기반 시스템

Ed25519:
  - 최신 EdDSA, 성능 우수
  - 32 Byte 키/서명 크기
  - Linux 커널 5.4+ 모듈 서명 지원

퓨즈 (OTP Fuse) 저장 예시:
  ┌──────────────────────────────────────────────────┐
  │ OTP Fuse Bank (NXP i.MX8M 예시):                │
  │  Bit 0~255: 공개 키 해시 (SHA-256)               │
  │  Bit 256: Secure Boot Enable                     │
  │  Bit 257: JTAG Disable                           │
  │  Bit 258: HAB (High Assurance Boot) Enable       │
  │  Bit 259~262: 보안 레벨                           │
  └──────────────────────────────────────────────────┘
```

---

## U-Boot Secure Boot (FIT Image)

```bash
# Flattened Image Tree (FIT) 이미지 생성
# - 커널, Device Tree, initramfs를 단일 서명 이미지로

# 1. FIT 이미지 기술 파일 (kernel.its)
cat > kernel.its << 'EOF'
/dts-v1/;
/ {
    description = "Robot Control System";
    #address-cells = <1>;

    images {
        kernel {
            description = "Linux Kernel";
            data = /incbin/("Image");
            type = kernel;
            arch = arm64;
            os = linux;
            compression = none;
            load = <0x40480000>;
            entry = <0x40480000>;
            hash-1 {
                algo = sha256;
            };
        };
        fdt {
            description = "Device Tree";
            data = /incbin/("robot_ecU.dtb");
            type = flat_dt;
            arch = arm64;
            compression = none;
            hash-1 {
                algo = sha256;
            };
        };
    };

    configurations {
        default = "conf-1";
        conf-1 {
            description = "Robot ECU Config";
            kernel = "kernel";
            fdt = "fdt";
            signature-1 {
                algo = sha256,rsa4096;
                key-name-hint = "robot-fw-key";
                sign-images = "kernel", "fdt";
            };
        };
    };
};
EOF

# 2. FIT 이미지 생성
mkimage -f kernel.its kernel.itb

# 3. 개인 키로 서명 (빌드 서버에서)
mkimage -F -k /secure/keys/ \
        -K u-boot.dtb \
        -r kernel.itb

# 4. 공개 키가 포함된 U-Boot 빌드
# (u-boot.dtb에 공개 키 임베드됨)

# U-Boot에서 검증 (자동 수행)
# => bootm $kernel_addr_r  ← FIT 이미지 로드 + 검증 + 부팅
```

---

## dm-verity (파일시스템 무결성)

```
dm-verity 동작 원리 (Linux Device Mapper):

파티션 구조:
  ┌─────────────────────────────────────────────────────┐
  │  Root Filesystem 파티션                             │
  │  [Data Block 0][Data Block 1]...[Data Block N]      │
  │                                                     │
  │  Hash Tree 파티션:                                  │
  │  Level 2: [Hash of hashes]...[Root Hash]           │
  │  Level 1: [Hash of data blocks]...                 │
  │  Level 0: [SHA-256 of each Data Block]             │
  │                                                     │
  │  Root Hash → 커널 파라미터에 임베드 (서명 검증됨)    │
  └─────────────────────────────────────────────────────┘

읽기 동작:
  1. 데이터 블록 읽기
  2. 해당 블록의 해시 계산
  3. Hash Tree에서 검증
  4. Root Hash까지 체인 검증
  5. 불일치 → I/O 오류 (읽기 실패)

설정 예시 (Android Verified Boot 방식):
  # Root 파티션을 읽기 전용으로 마운트 + 검증 활성화
  # dm-verity 테이블:
  # 0 2097152 verity 1 /dev/sda2 /dev/sda3 4096 4096 262144 1 sha256 \
  #   [root_hash_hex] [salt_hex]

  veritysetup format /dev/sda2 /dev/sda3
  # Root hash: abc123...
  # Salt: def456...

  veritysetup verify /dev/sda2 /dev/sda3 [root_hash]
```

---

## HSM / TPM (하드웨어 보안 모듈)

```
HSM 종류별 특성:

┌──────────────────┬────────────────────┬──────────────────────┐
│                  │ TPM 2.0            │ HSM (전용 모듈)       │
│                  │ (Trusted Platform) │ (외장/PCIe)          │
├──────────────────┼────────────────────┼──────────────────────┤
│ 표준             │ TCG TPM 2.0        │ FIPS 140-2 Level 3+  │
│ 용도             │ 키 저장, PCR 확장   │ CA 운용, HSM-as-a-Svc│
│ 성능             │ 중간               │ 매우 높음             │
│ 가격             │ $5~20/EA          │ $1,000~$50,000       │
│ 임베디드 적합성   │ 높음               │ 낮음 (대형 장비)      │
└──────────────────┴────────────────────┴──────────────────────┘

ARM TrustZone (SoC 내장):
  Secure World:
    - Trusted OS (OP-TEE, Trusty)
    - 암호화 키 저장
    - 보안 부팅 검증
    - 안전 I/O (터치, 지문)
  Normal World:
    - 일반 Linux OS
    - 로봇 제어 소프트웨어

SHE (Secure Hardware Extension, AUTOSAR):
  - 자동차 MCU용 (Infineon TC3xx, NXP S32K)
  - AES-128 하드웨어 가속
  - 키 16개 저장 (128비트)
  - SecOC 메시지 인증에 활용

TPM 2.0 활용 (Linux):
  # TPM 존재 확인
  ls /dev/tpm0

  # LUKS 디스크 암호화 키를 TPM에 봉인 (seal)
  systemd-cryptenroll --tpm2-device=auto /dev/sda5

  # TPM PCR 값 읽기 (Platform Configuration Register)
  # PCR 0: UEFI Firmware, PCR 4: Boot Manager
  # PCR 7: Secure Boot State, PCR 12: Kernel Command Line
  tpm2_pcrread sha256:0,4,7,12
```

---

## Anti-Rollback (다운그레이드 방지)

```
Anti-Rollback 메커니즘:

문제:
  공격자가 보안 취약점이 있는 구버전 펌웨어를 강제 설치
  → 구버전의 취약점 재활성화

해결책: 단조 증가 카운터 (Monotonic Counter)

방법 1: OTP Fuse 카운터
  - 퓨즈를 태움으로써 버전 카운터 증가
  - 불가역적 (절대 감소 불가)
  - NXP i.MX: OCOTP_SRK_REVOKE 퓨즈
  - 예: 퓨즈 0→1→3→7 (비트 OR로 단조 증가)

방법 2: TPM NV Counter
  - TPM NV (Non-Volatile) 저장소에 카운터 유지
  - 부팅 시 검증: 펌웨어 버전 ≥ NV Counter
  - 업데이트 성공 시: NV Counter 갱신

방법 3: Secure Storage 카운터 (ARMv8-M)
  - TrustZone 보안 스토리지에 카운터
  - 소프트웨어적이나 TZ로 보호

FIT Image에 버전 포함:
  # u-boot.its
  configuration {
    conf-1 {
      rollback-index = <5>;  # 이 이미지의 최소 허용 인덱스
    };
  };
  # U-Boot에서 저장된 Anti-Rollback Index < rollback-index 시 부팅 거부
```

---

## IMA (Integrity Measurement Architecture)

```
Linux IMA - 런타임 무결성 검사:

동작:
  1. 파일 열기 시 SHA-256 해시 계산
  2. TPM PCR 10에 측정값 확장
  3. 정책(Policy)에 따라 허용/거부

IMA 설정 (/etc/ima/ima-policy):
  # 특정 경로의 파일 실행 허용 (서명 필수)
  appraise func=BPRM_CHECK appraise_type=imasig

  # 의료 로봇 바이너리 무결성 검사
  appraise func=FILE_CHECK mask=MAY_READ \
    uid=0 fowner=0 \
    appraise_type=imasig|modsig

파일 서명 (개발 서버에서):
  # EVM 키 생성 및 파일에 서명
  evmctl sign --key /secure/ima-key.pem \
    /opt/robot/bin/joint_controller

  # 서명 확인
  evmctl ima_verify --key /etc/keys/ima-pubkey.pem \
    /opt/robot/bin/joint_controller

부팅 후 IMA 측정 로그 확인:
  cat /sys/kernel/security/ima/ascii_runtime_measurements
  # 10 [PCR] sha256:[hash] [filepath]
```

---

## 의료 로봇 Secure Boot 구현 체크리스트

```
FMEA 기반 Secure Boot 요구사항:

하드웨어:
  ☐ OTP Fuse로 Root 공개 키 해시 저장 (불가역)
  ☐ JTAG/Debug 포트 비활성화 (퓨즈)
  ☐ Secure Boot 강제 활성화 (퓨즈)
  ☐ HSM 또는 TPM 2.0 탑재
  ☐ Anti-Rollback 카운터 지원

소프트웨어:
  ☐ BL1~BL2~Kernel 서명 검증 체인
  ☐ FIT Image 서명 (RSA-4096 or ECDSA P-384)
  ☐ dm-verity 루트 파일시스템 보호
  ☐ 커널 모듈 서명 강제 (MODULE_SIG_FORCE)
  ☐ IMA/EVM 런타임 무결성 검사
  ☐ Anti-Rollback 버전 검증

프로세스:
  ☐ 개인 서명 키 HSM 보관 (오프라인 CA)
  ☐ 빌드 서버 접근 통제 (서명 서버 분리)
  ☐ OTA 업데이트 서명 파이프라인
  ☐ 키 폐기 및 갱신 절차 수립
  ☐ 보안 사고 대응 절차 (SBOM + 패치)
```

---

## 트러블슈팅

### 문제 1: Secure Boot 활성화 후 부팅 실패

```
증상: Secure Boot 퓨즈 활성화 후 부팅이 멈춤 (블랙 스크린)

원인 분석:
  1. 퓨즈에 저장된 공개 키 해시와 BL1 서명에 사용된 키 불일치
     - 가장 흔한 원인: 퓨즈 프로그래밍 시 잘못된 키 해시 사용
     - 확인 방법: UART 콘솔 메시지 (HAB failure 또는 Authentication Error)

  2. FIT Image 서명 키와 U-Boot에 임베드된 공개 키 불일치
     - 확인: U-Boot 콘솔에서 'iminfo $kernel_addr_r'

  3. 부팅 이미지 손상 (플래시 쓰기 오류)
     - 확인: sha256sum 또는 md5sum으로 이미지 체크섬 검증

해결 전략:
  # 단계별 Secure Boot 활성화 (권장 절차)

  # 1단계: HAB 설정 (NXP i.MX 예시)
  #   HAB 비활성 상태에서 서명된 이미지 부팅 확인
  #   → 서명 검증은 하되, 실패해도 부팅 계속 (Open Configuration)
  u-boot> hab_status  # HAB 이벤트 확인 (이벤트 없어야 정상)

  # 2단계: 서명 검증 성공 확인 후 퓨즈 프로그래밍
  u-boot> fuse prog 6 0 0x00000002  # HAB Enable (NXP i.MX8M)

  # 3단계: 활성화 후 재부팅 → 이제 서명 실패 시 부팅 중단

  UART 로그 정상:
  HAB Authentication: OK
  Starting kernel...

  UART 로그 오류:
  HAB Authentication: ERROR 0x33 0x22 (Authentication failed)
  → 키/서명 불일치 → 오프라인에서 키/이미지 재확인 필요
```

### 문제 2: dm-verity 루트 파일시스템 마운트 실패

```
증상: 커널 부팅 중 "dm-verity: ERROR" 또는 루트 파일시스템 마운트 실패

원인 분석:
  1. Root Hash 불일치
     - 루트 파티션이 수정되었거나 배드 블록 발생
     - Root Hash가 커널 파라미터의 값과 다름

  2. dm-verity 해시 트리 파티션이 루트 파티션과 다른 장치
     - 커널 파라미터의 장치 경로 오류

  3. salt 값 불일치
     - veritysetup 시 사용한 salt와 커널 파라미터의 salt 다름

진단:
  # 재부팅 후 Recovery 모드에서 dm-verity 검증
  veritysetup verify /dev/mmcblk0p2 /dev/mmcblk0p3 \
    [expected_root_hash]

  # Root Hash 재계산
  veritysetup format /dev/mmcblk0p2 /dev/mmcblk0p3
  # → Root hash: [새 값] 출력
  # → 커널 파라미터의 roothash=과 비교

  # 배드 블록 확인
  badblocks -v /dev/mmcblk0p2

해결:
  # OTA로 새 이미지 플래시 (A/B 파티션)
  # 또는 factory recovery partition 부팅
```

### 문제 3: IMA 정책에 의한 프로그램 실행 거부

```
증상: 시스템 부팅 후 일부 바이너리 실행 시 "Permission denied"
      dmesg에 "integrity: INTEGRITY_FAIL" 메시지

원인 분석:
  1. 서명되지 않은 바이너리 실행 시도 (IMA appraise 모드)
     - 개발 서버에서 새로 빌드한 바이너리 배포 시
     - IMA 서명 없이 바이너리 교체 시

  2. IMA 키가 시스템에 없거나 만료됨

해결:
  # 방법 1: 새 바이너리에 IMA 서명 (개발 서버에서)
  evmctl sign --key /secure/ima-key.pem \
    /opt/robot/bin/new_joint_controller

  # 방법 2: 서명 확인
  evmctl ima_verify --key /etc/keys/ima-pubkey.pem \
    /opt/robot/bin/new_joint_controller

  # 방법 3: 일시적 IMA 정책 완화 (개발 환경만!)
  echo "appraise func=BPRM_CHECK appraise_type=imasig" > /etc/ima/ima-policy
  # → imasig|modsig에서 imasig로만 변경 (서명 필수는 유지)

  # IMA 거부 로그 확인
  dmesg | grep "INTEGRITY_FAIL\|appraise"
  audit2allow -a  # SELinux + IMA 거부 분석 (SELinux 사용 시)
```

---

## Reference
- [UEFI Secure Boot Specification](https://uefi.org/sites/default/files/resources/UEFI_Spec_2_8_A_Feb14.pdf)
- [NIST SP 800-193 - Platform Firmware Resiliency Guidelines](https://csrc.nist.gov/publications/detail/sp/800-193/final)
- [ARM TrustZone Technology](https://developer.arm.com/ip-products/security-ip/trustzone)
- [OP-TEE Trusted OS](https://optee.readthedocs.io/)
- [U-Boot FIT Signature](https://u-boot.readthedocs.io/en/latest/usage/fit/signature.html)
- [Linux IMA (Integrity Measurement Architecture)](https://www.kernel.org/doc/html/latest/security/IMA-templates.html)
- [dm-verity Documentation](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html)
