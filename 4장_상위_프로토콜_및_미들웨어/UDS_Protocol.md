# UDS (Unified Diagnostic Services) - ISO 14229

## 개요
UDS(Unified Diagnostic Services)는 ECU 진단을 위한 표준 프로토콜입니다(ISO 14229). 고장 코드(DTC) 읽기, ECU 정보 조회, 파라미터 설정, 펌웨어 업데이트(Flash)가 모두 UDS 서비스 기반으로 수행됩니다. 전통적으로 CAN(ISO 15765-4)을 통해 전송되었으나, 현재는 Ethernet 기반 **DoIP(ISO 13400)**를 통해 전송하는 것이 표준이 되었습니다.

---

## UDS 서비스 구조

```
UDS 요청 (Request):
  [Service ID (1B)] + [Sub-function (1B, 선택)] + [데이터]

UDS 긍정 응답 (Positive Response):
  [Service ID + 0x40 (1B)] + [Echo 데이터] + [응답 데이터]
  예: ReadDataById 요청 0x22 → 응답 0x62

UDS 부정 응답 (Negative Response):
  [0x7F (1B)] + [요청 Service ID (1B)] + [NRC (1B)]
  NRC(Negative Response Code) 주요 값:
    0x11: serviceNotSupported
    0x12: subFunctionNotSupported
    0x13: incorrectMessageLengthOrInvalidFormat
    0x22: conditionsNotCorrect (세션 조건 미충족)
    0x31: requestOutOfRange
    0x33: securityAccessDenied
    0x35: invalidKey (SecurityAccess 실패)
    0x78: requestCorrectlyReceived-ResponsePending (처리 중, 기다려라)
```

---

## 진단 세션 관리 (DiagnosticSessionControl - 0x10)

```
세션 종류:
  0x01: defaultSession     ← 기본 (항상 가능)
  0x02: programmingSession ← 펌웨어 업데이트 (보안 해제 후)
  0x03: extendedSession    ← 확장 진단 (DTC 조작 등)
  0x60: safetySession      ← 안전 관련 파라미터 접근 (일부 제조사)

세션 전환:
  Tester → ECU: 10 03           (Extended Session 전환 요청)
  ECU → Tester: 50 03 00 32 01 F4  (긍정 응답 + P2/P2* 타이밍)
                         └──┘└────┘
                         P2=50ms   P2*=500ms

세션 유지 (TesterPresent - 0x3E):
  Tester → ECU: 3E 00   (주기적으로 전송, 세션 만료 방지)
  ECU → Tester: 7E 00   (응답)
  세션 타임아웃 전 S3Server 타이머(일반적으로 5초) 초기화
```

---

## SecurityAccess (0x27)

프로그래밍 세션 접근 및 보안 기능 사용 전 인증 과정입니다.

```
Step 1: Seed 요청
  Tester → ECU: 27 01        (Level 1 Seed 요청)
  ECU → Tester: 67 01 A3 B2 C1 D0  (4-Byte Seed)

Step 2: Key 계산 (로컬에서)
  Key = Security_Algorithm(Seed, Secret_Key)
  일반적인 알고리즘: HMAC-SHA256, AES-128, 또는 제조사 고유 알고리즘

Step 3: Key 전송
  Tester → ECU: 27 02 4F 8E 2D 91  (계산된 Key)
  ECU → Tester: 67 02              (보안 해제 성공)
  또는
  ECU → Tester: 7F 27 35          (키 불일치 → SecurityAccessDenied)

보안 레벨:
  Level 01/02: 일반 파라미터 쓰기
  Level 03/04: 보정 데이터 변경
  Level 05/06 이상: 플래시 프로그래밍 (가장 높은 권한)
```

---

## DTC 관리 (ReadDTCInformation - 0x19)

```
DTC 구조 (3 Byte):
  Byte 1 (High): 시스템 분류
    0x00: Powertrain (P 코드)
    0x40: Chassis (C 코드)
    0x80: Body (B 코드)
    0xC0: Network (U 코드)
  Byte 2, 3: 고장 번호

DTC Status Byte (1 Byte, 비트 필드):
  Bit 0: testFailed                (현재 고장 발생 중)
  Bit 1: testFailedThisMonitoringCycle
  Bit 2: pendingDTC                (간헐적 고장)
  Bit 3: confirmedDTC              (확정 DTC)
  Bit 4: testNotCompletedSinceLastClear
  Bit 5: testFailedSinceLastClear
  Bit 6: testNotCompletedThisMonitoringCycle
  Bit 7: warningIndicatorRequested (경고등 점등)

주요 Sub-function:
  0x01: reportNumberOfDTCByStatusMask  (DTC 개수 확인)
  0x02: reportDTCByStatusMask          (DTC 목록 + 상태)
  0x06: reportDTCExtendedDataRecordByDTCNumber (Freeze Frame)
  0x0A: reportSupportedDTC             (지원 DTC 전체 목록)
  0x14: ClearDiagnosticInformation     (DTC 삭제)
```

### DTC 읽기 예시
```
Tester → ECU: 19 02 08    (Status Mask 0x08 = Confirmed DTC만)
ECU → Tester: 59 02 08
              01 96 55 2F  ← DTC #1: 0x019655, Status=0x2F
              02 74 32 08  ← DTC #2: 0x027432, Status=0x08

DTC 0x019655 해석:
  01: Powertrain 계열
  96 55: 고장 번호 → 제조사 DTC 테이블 참조
  Status 0x2F = 0010 1111:
    Bit 0: testFailed = 1 (현재 고장)
    Bit 1: thisMonitoringCycle = 1
    Bit 2: pendingDTC = 1
    Bit 3: confirmedDTC = 1
    Bit 5: failedSinceLastClear = 1
```

---

## 데이터 읽기/쓰기 (ReadDataByIdentifier - 0x22)

```
DID (Data Identifier) 예약 범위:
  0x0000~0x00FF: ISO 15031 표준 (OBD)
  0x0100~0xEFFF: 제조사 정의
  0xF000~0xFFFF: ISO 14229 예약

주요 표준 DID:
  0xF186: ActiveDiagnosticSession   (현재 세션)
  0xF187: VehicleManufacturerSparePartNumber
  0xF188: VehicleManufacturerECUSoftwareNumber
  0xF189: VehicleManufacturerECUSoftwareVersionNumber
  0xF18A: SystemSupplierIdentifier
  0xF18B: ECUManufacturingDate      (날짜)
  0xF18C: ECUSerialNumber
  0xF190: VIN (차량 식별 번호, 17 Byte ASCII)
  0xF197: SystemNameOrEngineType

읽기 예시:
  Tester → ECU: 22 F1 90           (VIN 읽기)
  ECU → Tester: 62 F1 90 57 42 41 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30
                            └───────────────────────────────────────────────────┘
                            VIN: "WBA000000000000000" (17자리)
```

---

## ECU Flash 프로그래밍 (OTA 핵심 절차)

```
전체 UDS Flash 프로그래밍 절차:

① 세션 전환
   10 02 → 50 02 (Programming Session)

② 보안 해제
   27 01 → 67 01 [Seed]
   27 02 [Key] → 67 02

③ 기존 메모리 삭제 (RoutineControl)
   31 01 FF 00 [Address][Size] → 71 01 FF 00 00 (삭제 중)
   31 03 FF 00 → 71 03 FF 00  (삭제 완료 확인)

④ 다운로드 시작 (RequestDownload)
   34 00 44 [Address:4B] [UncompressedSize:4B] → 74 20 [MaxBlockSize]

⑤ 데이터 전송 (TransferData, 반복)
   36 01 [BlockData(MaxBlockSize)] → 76 01 (Block 1 완료)
   36 02 [BlockData] → 76 02
   ...

⑥ 전송 종료 (RequestTransferExit)
   37 → 77

⑦ 체크섬 검증
   31 01 02 02 [CRC32:4B] → 71 01 02 02 00 (검증 OK)

⑧ ECU 재시작
   11 01 → (응답 없이 재시작)
```

---

## Wireshark로 UDS 분석

```bash
# DoIP 상의 UDS 분석
doip

# UDS 서비스 ID 필터 (0x22: ReadDataByIdentifier)
doip && data[4:5] == 22:00

# UDS 부정 응답만 보기
doip && data[4:5] == 7f:00

# 특정 DID 조회
doip && data[5:7] == f1:90  # VIN

# Flash 프로그래밍 (RequestDownload)
doip && data[4:5] == 34:00
```

---

## 의료 로봇 UDS DID 설계 예시

```
의료 로봇 ECU DID 정의 (제조사 정의 범위: 0x0100~0xEFFF):

0xF190: 시스템 시리얼 번호 (Robot S/N)
0xF195: 소프트웨어 버전 (SW Version)
0xF196: 하드웨어 버전 (HW Version)
0x0101: 관절 캘리브레이션 데이터 (Joint Calibration)
0x0102: 현재 관절 각도 (Current Joint Positions)
0x0103: 모터 전류 값 (Motor Currents)
0x0104: 열 센서 데이터 (Temperature Sensors)
0x0200: 수술 카운터 (Procedure Count)
0x0201: 총 동작 시간 (Operating Hours)
0x0202: 에러 히스토리 (Error Log, 최근 100개)
0x0300: 안전 파라미터 (Safety Limits - 쓰기 보호)
0x0301: 물리적 제한값 (Joint Limits)
```

---

## Reference
- [ISO 14229-1 - UDS Part 1: Application Layer Services](https://www.iso.org/standard/72439.html)
- [ISO 14229-2 - UDS Part 2: Session Layer Services](https://www.iso.org/standard/72473.html)
- [ISO 14229-3 - UDS on CAN (UDSonCAN)](https://www.iso.org/standard/72474.html)
- [ISO 13400-2 - DoIP](https://www.iso.org/standard/69527.html)
- [Vector - UDS Protocol](https://www.vector.com/kr/ko/know-how/protocols/uds/)
