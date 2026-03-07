# UDS (Unified Diagnostic Services) - ISO 14229

## 개요
UDS(Unified Diagnostic Services)는 ECU 진단을 위한 국제 표준 프로토콜입니다(ISO 14229). 고장 코드(DTC) 읽기·삭제, ECU 정보 조회, 파라미터 설정, 펌웨어 업데이트(Flash Programming)가 모두 UDS 서비스 기반으로 수행됩니다. 전통적으로 CAN(ISO 15765-4 CAN Transport Layer)을 통해 전송되었으나, 현재는 Ethernet 기반 **DoIP(ISO 13400)**가 표준 전송 계층입니다.

의료 로봇에서 UDS는 **현장 유지보수, 품질 감사, 규제 대응** 시 핵심 진단 인터페이스입니다. FDA 21 CFR Part 820 소프트웨어 검증 요건, ISO 13485 품질관리시스템 준수를 위해 진단 이력(Freeze Frame, Snapshot Data) 관리가 중요합니다.

---

## UDS 요청/응답 구조

```
UDS 요청 (Request):
  [Service ID (1B)] + [Sub-function (1B, 선택)] + [파라미터/데이터]

UDS 긍정 응답 (Positive Response):
  [Service ID + 0x40 (1B)] + [Echo 필드] + [응답 데이터]
  규칙: 응답 서비스 ID = 요청 서비스 ID + 0x40
  예: 0x22 ReadDataByIdentifier → 0x62 (긍정 응답)
      0x27 SecurityAccess      → 0x67

UDS 부정 응답 (Negative Response, NRC):
  [0x7F] + [요청 Service ID (1B)] + [NRC 코드 (1B)]
  예: 7F 22 31 = ReadDataByIdentifier에 대해 requestOutOfRange

NRC (Negative Response Code) 상세:
  0x10: generalReject                         일반 거부
  0x11: serviceNotSupported                   서비스 미지원
  0x12: subFunctionNotSupported               서브펑션 미지원
  0x13: incorrectMessageLengthOrInvalidFormat 형식 오류
  0x14: responseTooLong                       응답이 너무 큼
  0x21: busyRepeatRequest                     재요청 필요 (바쁨)
  0x22: conditionsNotCorrect                  조건 불충족 (세션 오류)
  0x24: requestSequenceError                  순서 오류 (예: Security 먼저)
  0x25: noResponseFromSubnetComponent         하위 ECU 응답 없음
  0x26: failurePreventsExecutionOfRequestedAction 안전 이유로 실행 거부
  0x31: requestOutOfRange                     범위 초과
  0x33: securityAccessDenied                  보안 거부
  0x35: invalidKey                            SecurityAccess 키 불일치
  0x36: exceededNumberOfAttempts              시도 횟수 초과 (잠금)
  0x37: requiredTimeDelayNotExpired           재시도 딜레이 미경과
  0x70: uploadDownloadNotAccepted             업/다운로드 거부
  0x71: transferDataSuspended                 전송 중단
  0x72: generalProgrammingFailure             프로그래밍 실패
  0x73: wrongBlockSequenceCounter             블록 번호 오류
  0x78: requestCorrectlyReceived-ResponsePending  처리 중 (기다려라)
  0x7E: subFunctionNotSupportedInActiveSession    현재 세션 미지원
  0x7F: serviceNotSupportedInActiveSession        현재 세션 미지원
```

---

## 진단 세션 관리 (DiagnosticSessionControl - 0x10)

### 세션 전환 및 타이밍

```
세션 종류:
  0x01: defaultSession     ← 기본 (항상 가능, 제한된 서비스)
  0x02: programmingSession ← 펌웨어 Flash (SecurityAccess 필수)
  0x03: extendedSession    ← 확장 진단 (DTC 조작, 파라미터)
  0x40: safetySession      ← 안전 파라미터 (일부 제조사 전용)

세션 전환 예시:
  Tester → ECU: 10 03
    (ExtendedDiagnosticSession 전환 요청)
  ECU → Tester: 50 03 00 19 01 F4
                    └──┘└────┘
                    P2=25ms    P2*=500ms

타이밍 파라미터:
  P2Server:     ECU 응답 최대 대기 시간 (기본: 50ms)
  P2*Server:    NRC 0x78 이후 최대 대기 시간 (기본: 5000ms)
  S3Server:     세션 유지 타이머 (기본: 5000ms, TesterPresent 갱신 필요)

세션 유지 (TesterPresent - 0x3E):
  Tester → ECU: 3E 00       (suppressPosRspMsgIndicationBit=0)
  ECU → Tester: 7E 00       (긍정 응답)
  또는
  Tester → ECU: 3E 80       (suppression=1, 응답 요청 안 함)
  → S3Server 타이머 초기화 (5초 이내 전송 필요)
```

---

## SecurityAccess (0x27) - 보안 해제

### Seed-Key 메커니즘

```
보안 레벨 (Level Pair):
  Level 01/02: 파라미터 읽기/쓰기 (낮은 권한)
  Level 03/04: 보정 데이터 변경
  Level 05/06: 플래시 프로그래밍 (최고 권한)
  Level 63/64: 개발/제조 전용 (출하 전 비활성화)

요청-응답 흐름:
  Step 1: Seed 요청
    Tester → ECU: 27 05          (Level 5 Seed 요청)
    ECU → Tester: 67 05 A3B2C1D0 (4-Byte 랜덤 Seed)

  Step 2: Key 계산 (Tester 로컬에서)
    Key = AES-128-CMAC(
            key   = DeviceMasterKey,
            msg   = Seed || ECU_SerialNumber
          )[:4]    ← 상위 4 Byte 추출

  Step 3: Key 전송
    Tester → ECU: 27 06 4F8E2D91  (계산한 Key)
    ECU → Tester: 67 06           (보안 해제 성공)
    또는
    ECU → Tester: 7F 27 35        (키 불일치 → SecurityAccessDenied)

잠금 정책:
  연속 실패 3회 → 딜레이 30분 (NRC 0x37)
  잠금은 ECU 재시작으로 초기화 (제조사마다 다름)
```

---

## DTC 관리

### DTC 구조

```
DTC (Diagnostic Trouble Code) - 3 Byte:

ISO 15031-6 OBD DTC 형식 (SAE J2012):
  Byte 1 [7:6]: 시스템 분류
    00: Powertrain (P0xxx~P3xxx)
    01: Chassis    (C0xxx~C3xxx)
    10: Body       (B0xxx~B3xxx)
    11: Network    (U0xxx~U3xxx)
  Byte 1 [5:4]: 표준/제조사 코드 구분
  Byte 2, 3: 고장 번호 (BCD 또는 HEX)

DTC Status Byte (1 Byte, 8개 비트 플래그):
  Bit 0: testFailed                 현재 고장 발생 중
  Bit 1: testFailedThisMonitoringCycle
  Bit 2: pendingDTC                 간헐적 고장 (미확정)
  Bit 3: confirmedDTC               확정 DTC
  Bit 4: testNotCompletedSinceLastClear
  Bit 5: testFailedSinceLastClear
  Bit 6: testNotCompletedThisMonitoringCycle
  Bit 7: warningIndicatorRequested  경고등 점등 요청
```

### DTC 읽기/삭제

```
Sub-function 0x02: reportDTCByStatusMask
  Tester → ECU: 19 02 08    (Confirmed DTC, Mask=0x08)
  ECU → Tester: 59 02 08
                C0 12 34 2F ← DTC #1: 0xC01234, Status=0x2F
                C0 56 78 08 ← DTC #2: 0xC05678, Status=0x08

Status 0x2F 해석 (0b00101111):
  Bit 0=1: testFailed (현재 고장)
  Bit 1=1: thisMonitoringCycle
  Bit 2=1: pendingDTC (간헐적)
  Bit 3=1: confirmedDTC (확정)
  Bit 5=1: failedSinceLastClear

Sub-function 0x06: reportDTCExtendedDataByDTCNumber
  → Freeze Frame 데이터 (고장 발생 시 환경 데이터 스냅샷)
  → 관절 각도, 온도, 전류 등 DTC 발생 당시 상황 기록

Sub-function 0x0A: reportSupportedDTC
  → ECU가 지원하는 모든 DTC 목록 조회

DTC 삭제 (0x14: ClearDiagnosticInformation):
  Tester → ECU: 14 FF FF FF  (모든 DTC 삭제)
  ECU → Tester: 54
  또는:
  Tester → ECU: 14 C0 12 34  (특정 DTC만 삭제)
```

---

## 데이터 읽기/쓰기 (ReadDataByIdentifier - 0x22)

### DID (Data Identifier) 예약 범위

```
DID 범위 및 용도:
  0x0000~0x00FF: ISO 15031 표준 (OBD II 진단 파라미터)
  0x0100~0xEFFF: 제조사 정의 (자유 사용)
  0xF000~0xF0FF: ISO 14229 서비스 파라미터
  0xF100~0xF1FF: 차량/기기 정보 (VIN, 버전 등)
  0xF200~0xF2FF: 통신 설정
  0xF300~0xFEFF: 예약
  0xFF00~0xFFFF: ISO 14229 예약

ISO 14229 표준 DID (F1xx 범위):
  0xF186: ActiveDiagnosticSession    현재 세션 번호
  0xF187: SparePartNumber            부품 번호
  0xF188: ECUSoftwareNumber          SW 부품 번호
  0xF189: ECUSoftwareVersionNumber   SW 버전
  0xF18A: SystemSupplierID           공급업체 ID
  0xF18B: ECUManufacturingDate       제조일 (BCD: YY-MM-DD)
  0xF18C: ECUSerialNumber            ECU 시리얼
  0xF190: VIN                        차량/기기 식별번호 (17자리 ASCII)
  0xF191: VehicleManufacturerECUHardwareNumber
  0xF192: SystemSupplierECUHardwareNumber
  0xF193: SystemSupplierECUHardwareVersionNumber
  0xF194: SystemSupplierECUSoftwareNumber
  0xF195: SystemSupplierECUSoftwareVersionNumber

읽기 예시 (VIN):
  Tester → ECU: 22 F1 90
  ECU → Tester: 62 F1 90 57 42 41 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30
                              └─────────────────────────────────────────────────┘
                              ASCII "WBA000000000000000" (17 Byte)

쓰기 예시 (0x2E WriteDataByIdentifier):
  Tester → ECU: 2E 01 01 3F C9 0F DB ...  (캘리브레이션 데이터)
  ECU → Tester: 6E 01 01                  (긍정 응답)
```

---

## ECU Flash 프로그래밍 (RequestDownload/TransferData)

```
전체 Flash 프로그래밍 절차:

① Extended 또는 Programming Session 전환
   Tester → ECU: 10 02      ECU: 50 02 00 32 01 F4
   ↑ P2*max=0x01F4=500ms (대용량 삭제/쓰기 응답 대기)

② SecurityAccess Level 05/06 해제
   27 05 → 67 05 [Seed]
   27 06 [Key] → 67 06

③ 메모리 삭제 (RoutineControl 0xFF00)
   31 01 FF 00 [StartAddr:4B] [Size:4B]
   → ECU: 7F 31 78  (NRC 0x78: 처리 중)
   → 폴링: 31 03 FF 00
   → ECU: 71 03 FF 00 00  (완료, 소요 2~30초)

④ RequestDownload (0x34)
   Tester → ECU: 34 00 44 00100000 00080000
                            └──────┘└──────┘
                            주소    크기(512KB)
   ECU → Tester: 74 20 0400
                       └──┘
                       MaxBlockSize=1024 Byte

⑤ TransferData 반복 (0x36)
   Tester → ECU: 36 01 [Block#1, 1024 Byte]  → 76 01
   Tester → ECU: 36 02 [Block#2, 1024 Byte]  → 76 02
   ...
   Tester → ECU: 36 00 [LastBlock]            → 76 00
   (BlockSequenceCounter: 0x01~0xFF, 이후 0x00부터 재시작)

⑥ RequestTransferExit (0x37)
   37 → 77

⑦ 검증 (RoutineControl 0x0202)
   31 01 02 02 [CRC32:4B] → 71 01 02 02 00 (OK)

⑧ ECUReset (0x11)
   11 01 → ECU 재시작 (응답 없이)
   재시작 후 Secure Boot 검증 → 정상 부팅 대기
```

---

## Wireshark UDS 분석

```bash
# DoIP 상의 UDS 전체
doip

# 서비스별 필터 (DoIP UDS payload 오프셋 4B)
doip && data[4:5] == 10:00   # DiagnosticSessionControl
doip && data[4:5] == 22:00   # ReadDataByIdentifier
doip && data[4:5] == 2e:00   # WriteDataByIdentifier
doip && data[4:5] == 27:00   # SecurityAccess
doip && data[4:5] == 19:00   # ReadDTCInformation
doip && data[4:5] == 14:00   # ClearDiagnosticInformation
doip && data[4:5] == 34:00   # RequestDownload
doip && data[4:5] == 36:00   # TransferData
doip && data[4:5] == 7f:00   # Negative Response (NRC)

# 특정 DID 읽기 (VIN: 0xF190)
doip && data[5:7] == f1:90

# NRC 0x78 (ResponsePending) 만 보기 → OTA 진행 시 빈번 발생
doip && data[4:1] == 7f:00 && data[6:1] == 78:00
```

---

## 의료 로봇 UDS DID 설계

```
의료 로봇 ECU DID 정의 (제조사 정의 범위: 0x0100~0xEFFF):

기기 식별:
  0xF190: 로봇 시리얼 번호 (17 Byte ASCII, ISO 표준 준수)
  0xF189: SW 버전 (8 Byte: Major.Minor.Patch.Build)
  0xF196: HW 버전 (4 Byte)
  0xF18B: 제조 날짜 (BCD 4 Byte, YYYY-MM-DD)

관절 데이터:
  0x0101: 관절 캘리브레이션 (6관절 × 8 Byte = 48 Byte)
  0x0102: 현재 관절 각도 (6 × float32 = 24 Byte, 읽기 전용)
  0x0103: 모터 전류 (6 × float32 = 24 Byte, 읽기 전용)
  0x0104: 온도 센서 (6 × float32 = 24 Byte, 읽기 전용)

운용 통계:
  0x0200: 수술 카운터 (uint32, 총 수행 건수)
  0x0201: 총 동작 시간 (uint64 ms, 누적 파워온 시간)
  0x0202: 에러 히스토리 (최근 100개, 가변 길이, 읽기 전용)
  0x0203: 마지막 유지보수 날짜 (BCD 4 Byte)

안전 파라미터 (SecurityAccess L5/L6 필요):
  0x0300: 안전 제한값 (Security Limits)
  0x0301: 관절 각도 한계 (Angle Limits, 6관절 × 8 Byte)
  0x0302: 최대 속도 한계 (Speed Limits, 6관절 × 4 Byte)
  0x0303: 최대 토크 한계 (Torque Limits, 6관절 × 4 Byte)

규제 요건 대응:
  FDA 21 CFR Part 820: DID 0x0202 에러 이력 추적
  ISO 13485 §8.3: 부적합 제품 식별 → DTC + DID 활용
  IEC 62304 §5.8: 소프트웨어 문제 해결 → UDS 진단 로그
  MDR Annex XIV: 임상 평가 데이터 → 수술 카운터 DID
```

---

## Reference
- [ISO 14229-1:2020 - UDS Application Layer Services](https://www.iso.org/standard/72439.html)
- [ISO 14229-2:2013 - UDS Session Layer Services](https://www.iso.org/standard/72473.html)
- [ISO 14229-3:2012 - UDS on CAN (UDSonCAN)](https://www.iso.org/standard/72474.html)
- [ISO 13400-2:2019 - DoIP](https://www.iso.org/standard/69527.html)
- [ISO 15031-6 - OBD DTC Standard (SAE J2012)](https://www.sae.org/standards/content/j2012_201210/)
- [Vector UDS Protocol Reference](https://www.vector.com/kr/ko/know-how/protocols/uds/)
- [python-udsoncan (PyPI)](https://pypi.org/project/udsoncan/)
