# OTA 인프라 (Over-The-Air Update Infrastructure)

## 개요
OTA(Over-The-Air)는 무선 통신을 통해 소프트웨어, 펌웨어, 설정값을 안전하게 업데이트하는 기술입니다. 의료 로봇은 사람의 생명과 직결되므로, OTA 인프라에는 **보안**, **안전성**, **원자성(Atomicity)**이 핵심 요구사항입니다. UNECE R156(SUMS) 및 FDA 가이드라인에서 의무적으로 요구합니다.

---

## OTA 시스템 아키텍처

```
OTA 인프라 전체 구조:

┌──────────────────────────────────────────────────────────────┐
│                    OTA Backend (Cloud)                        │
│                                                              │
│  ┌────────────────┐     ┌──────────────────────────────┐    │
│  │  Image/Package │     │      Campaign Manager        │    │
│  │  Repository    │     │  - 타겟 선택 (로봇 ID 기반)  │    │
│  │  - 서명된 패키지│     │  - 배포 일정                 │    │
│  │  - 버전 메타데이터│    │  - 롤아웃 전략 (Canary)     │    │
│  │  - Delta 생성  │     │  - 결과 모니터링              │    │
│  └────────────────┘     └──────────────────────────────┘    │
│           │                           │                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              Key Management Service (KMS)              │  │
│  │  - 펌웨어 서명 키 (HSM 보관)                           │  │
│  │  - UPTANE Root/Targets/Snapshot/Timestamp 키          │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────────┘
                           │ TLS 1.3 + mTLS + UPTANE 메타데이터
                           │
┌──────────────────────────▼───────────────────────────────────┐
│                    Robot OTA Client                           │
│  ┌──────────────────┐     ┌──────────────────────────────┐  │
│  │  Pre-condition   │     │   Download Manager            │  │
│  │  Checker         │     │  - Chunked Download          │  │
│  │  - 수술 중 여부  │     │  - 재시도 (지수 백오프)       │  │
│  │  - 배터리 상태   │     │  - Delta 적용 (bsdiff)       │  │
│  │  - 스토리지 여유 │     │  - 해시 검증                 │  │
│  └──────────────────┘     └──────────────────────────────┘  │
│  ┌──────────────────┐     ┌──────────────────────────────┐  │
│  │  Verifier        │     │   Installer (A/B 파티션)     │  │
│  │  - 서명 검증      │     │  - Inactive 파티션에 설치    │  │
│  │  - Anti-Rollback │     │  - 검증 후 Active 전환       │  │
│  │  - 호환성 확인   │     │  - 실패 시 자동 Rollback     │  │
│  └──────────────────┘     └──────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## SOTA vs FOTA

```
업데이트 유형 비교:

SOTA (Software OTA):
  대상: Application, 설정, 인증서, AI 모델
  예시: ROS 2 패키지, 의료 영상 처리 SW
  특징: ECU 리부팅 불필요 (런타임 교체 가능)
  위험도: 중간
  방법: Container 이미지 업데이트 (Docker pull)

FOTA (Firmware OTA):
  대상: 부트로더, 커널, ECU 펌웨어
  예시: Safety ECU 제어 로직, 모터 드라이버
  특징: 리부팅 필수, A/B 파티션 필요
  위험도: 높음 (실패 시 벽돌화 위험)
  방법: UDS 0x34~0x37 + DoIP + UPTANE

FOFU (Full Flash Over the Air Update):
  대상: 전체 이미지 (OS + 커널 + 앱)
  특징: 최대 안전, 최소 지원 필요
  시간: 수십 분 (대역폭에 따라)
  적용: 대형 업데이트, 보안 패치
```

---

## A/B 파티션 설계

```
A/B 파티션 구조:

eMMC/NVMe 레이아웃:
┌────────────────────────────────────────────────────┐
│  파티션 테이블 (GPT)                                │
├─────────────────┬──────────────────────────────────┤
│ Bootloader A    │ Bootloader B                     │
│ (Primary)       │ (Backup)                        │
├─────────────────┴──────────────────────────────────┤
│ Boot A (Active) │ Boot B (Inactive)                │
│  kernel + dtb   │  kernel + dtb (새 버전)          │
├─────────────────┴──────────────────────────────────┤
│ System A (Active) │ System B (Inactive)            │
│  rootfs 현재     │  rootfs 업데이트 대상             │
├───────────────────────────────────────────────────┤
│ Data (공통: 사용자 설정, 로그)                       │
├───────────────────────────────────────────────────┤
│ Metadata (현재 Active 파티션, 부팅 시도 횟수 등)     │
└───────────────────────────────────────────────────┘

업데이트 흐름:
  1. 현재: System A Active
  2. 업데이트 수신 → System B에 설치
  3. 서명 검증 성공 → Boot B로 전환 설정
  4. 재부팅 → System B로 부팅
  5. 검증 성공 → B를 Active로 확정
  6. 실패 시 → 자동으로 A로 롤백

Linux A/B 파티션 관리 도구:
  u-boot: bootcount, bootlimit 변수
  grub: grub-editenv
  systemd-boot: EFI 변수
  OSTree: Atomic OS 업데이트 (Flatcar Linux)
```

---

## UPTANE 보안 프레임워크 구현

```python
#!/usr/bin/env python3
"""
UPTANE 기반 OTA 클라이언트 (개념 구현)
실제 구현: https://github.com/uptane/aktualizr
"""
import hashlib
import json
import ssl
import time
from datetime import datetime, timezone
from typing import Optional

class UptaneMetadata:
    """UPTANE 메타데이터 검증"""

    def __init__(self, root_pub_key: bytes):
        self.root_pub_key = root_pub_key
        self.current_versions = {
            'root': 0,
            'snapshot': 0,
            'targets': 0,
            'timestamp': 0
        }

    def verify_timestamp(self, timestamp_meta: dict) -> bool:
        """
        1단계: Timestamp 메타데이터 검증
        - 서명 검증
        - 만료 시간 확인 (24시간 유효)
        - 버전 번호 단조 증가 확인
        """
        # 만료 시간 확인
        expires = datetime.fromisoformat(timestamp_meta['signed']['expires'])
        if expires < datetime.now(timezone.utc):
            print("[UPTANE] Timestamp 만료됨 → 거부")
            return False

        # 버전 번호 확인 (Rollback 방지)
        version = timestamp_meta['signed']['version']
        if version <= self.current_versions['timestamp']:
            print(f"[UPTANE] Timestamp 버전 다운그레이드 시도: {version}")
            return False

        self.current_versions['timestamp'] = version
        return True

    def verify_targets(self, targets_meta: dict,
                       target_name: str) -> Optional[dict]:
        """
        3단계: Targets 메타데이터에서 타겟 이미지 정보 추출
        - 파일 해시 (sha256)
        - 파일 크기
        - 커스텀 메타데이터 (hwid, version)
        """
        targets = targets_meta['signed']['targets']
        if target_name not in targets:
            print(f"[UPTANE] 타겟 없음: {target_name}")
            return None

        target = targets[target_name]
        return {
            'sha256': target['hashes']['sha256'],
            'length': target['length'],
            'custom': target.get('custom', {}),
        }

    def verify_image(self, image_data: bytes,
                     expected_sha256: str,
                     expected_length: int) -> bool:
        """
        4단계: 다운로드된 이미지 검증
        - 크기 확인
        - SHA-256 해시 확인
        """
        if len(image_data) != expected_length:
            print(f"[UPTANE] 이미지 크기 불일치: {len(image_data)} != {expected_length}")
            return False

        actual_hash = hashlib.sha256(image_data).hexdigest()
        if actual_hash != expected_sha256:
            print(f"[UPTANE] 해시 불일치: {actual_hash}")
            return False

        print(f"[UPTANE] 이미지 검증 성공: sha256={actual_hash[:16]}...")
        return True


class OTAClient:
    """
    의료 로봇 OTA 클라이언트
    UPTANE + A/B 파티션 + 안전 사전 검사
    """

    REPO_URL = "https://ota-server.hospital.com"
    ROBOT_ID = "R001"
    CERT_DIR = "/etc/robot/ota"

    def __init__(self):
        self.uptane = UptaneMetadata(root_pub_key=b"root_key_placeholder")
        self.current_version = self._read_current_version()

    def _read_current_version(self) -> str:
        """현재 Active 파티션 버전 읽기"""
        try:
            with open("/etc/robot/version") as f:
                return f.read().strip()
        except FileNotFoundError:
            return "1.0.0"

    def pre_condition_check(self) -> tuple[bool, str]:
        """
        업데이트 시작 전 안전 조건 확인
        IEC 62304 / UNECE R156 요구사항
        """
        checks = [
            ('수술 중 아님', not self._is_surgery_active()),
            ('배터리 > 50%', self._battery_level() > 50),
            ('스토리지 > 2GB', self._free_storage_gb() > 2.0),
            ('네트워크 안정', self._network_stable()),
            ('안전 상태', self._is_safe_state()),
        ]

        for name, result in checks:
            if not result:
                return False, f"사전 조건 실패: {name}"

        return True, "모든 조건 충족"

    def _is_surgery_active(self) -> bool:
        """수술 중 여부 확인 (실제: 안전 ECU 신호)"""
        return False  # 실제: DDS topic 또는 GPIO

    def _battery_level(self) -> float:
        return 85.0  # 실제: 배터리 관리 시스템 API

    def _free_storage_gb(self) -> float:
        import shutil
        usage = shutil.disk_usage("/data")
        return usage.free / 1e9

    def _network_stable(self) -> bool:
        return True  # 실제: ping 또는 TCP 연결 테스트

    def _is_safe_state(self) -> bool:
        return True  # 실제: 안전 ECU 상태 신호

    def check_for_update(self) -> Optional[dict]:
        """서버에서 업데이트 확인"""
        import urllib.request
        try:
            url = f"{self.REPO_URL}/api/v1/updates/{self.ROBOT_ID}"
            req = urllib.request.Request(url)
            with urllib.request.urlopen(req, timeout=30) as resp:
                data = json.loads(resp.read())
                if data.get('available'):
                    print(f"[OTA] 업데이트 가용: {data['version']}")
                    return data
                else:
                    print("[OTA] 최신 버전입니다.")
                    return None
        except Exception as e:
            print(f"[OTA] 서버 통신 실패: {e}")
            return None

    def perform_update(self, update_info: dict) -> bool:
        """업데이트 수행"""
        # 1. 사전 조건 확인
        ok, reason = self.pre_condition_check()
        if not ok:
            print(f"[OTA] 업데이트 거부: {reason}")
            return False

        # 2. UPTANE 메타데이터 검증 (생략 - 실제: aktualizr 사용)
        print("[OTA] UPTANE 메타데이터 검증...")

        # 3. 이미지 다운로드 (Inactive 파티션)
        print(f"[OTA] 다운로드 중: {update_info['version']}")
        # download_with_retry(update_info['url'], '/dev/mmcblk0p4')

        # 4. 이미지 검증
        # image_data = read_partition('/dev/mmcblk0p4')
        # self.uptane.verify_image(image_data, update_info['sha256'], ...)

        # 5. 부팅 설정 변경 (B 파티션으로)
        print("[OTA] 재부팅 설정 변경 (A→B)")
        # set_boot_partition('B')

        # 6. 재부팅 (안전한 시점에)
        print("[OTA] 업데이트 완료. 재부팅 예약.")
        # schedule_reboot()

        return True
```

---

## Delta 업데이트

```bash
# Delta 업데이트 생성 (bsdiff 사용)
# 전체 업데이트 대비 80~90% 크기 절감

# 패치 생성 (빌드 서버에서)
bsdiff firmware_v1.0.bin firmware_v2.0.bin patch_v1_to_v2.bspatch

# 크기 비교
ls -lh firmware_v2.0.bin patch_v1_to_v2.bspatch
# -rw-r--r-- 1 build 128M firmware_v2.0.bin
# -rw-r--r-- 1 build  14M patch_v1_to_v2.bspatch  ← 89% 절감

# 패치 적용 (로봇에서)
bspatch firmware_v1.0.bin firmware_v2.0_patched.bin patch_v1_to_v2.bspatch

# 검증
sha256sum firmware_v2.0.bin firmware_v2.0_patched.bin
# → 동일한 해시값 확인

# xdelta3 (대안, 더 빠름)
xdelta3 -s firmware_v1.0.bin firmware_v2.0.bin patch.xdelta
xdelta3 -d -s firmware_v1.0.bin patch.xdelta firmware_v2.0_patched.bin
```

---

## Reference
- [UPTANE Standard](https://uptane.github.io/docs/standard/uptane-standard)
- [Uptane - aktualizr Client](https://github.com/uptane/aktualizr)
- [UNECE R156 - SUMS](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software-update)
- [Android A/B System Updates](https://source.android.com/docs/core/ota/ab)
- [OSTree - Atomic OS Updates](https://ostreedev.github.io/ostree/)
- [bsdiff](http://www.daemonology.net/bsdiff/)
