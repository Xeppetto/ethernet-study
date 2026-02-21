# DoIP (Diagnostics over IP) - ISO 13400

## 개요
DoIP(Diagnostics over IP)는 Ethernet 기반 진단 통신 표준(ISO 13400)입니다. 기존 CAN 기반 진단(K-Line, ISO 15765)을 대체하여 고속 Ethernet으로 UDS(ISO 14229) 진단 서비스를 제공합니다. 의료 로봇을 포함한 임베디드 시스템의 펌웨어 업데이트, 고장 진단, 원격 유지보수에 필수적입니다.

---

## DoIP 아키텍처

```
진단 테스터 (PC/진단 장비)
        │
        │ Ethernet (TCP/UDP 13400)
        │
   DoIP Gateway (차량/로봇 진입점)
        │
   내부 버스 (CAN, LIN, Ethernet)
        │
   ┌────┴─────┐
   │  ECU #1  │  ECU #2  │  ECU #3  │
   │ (주 제어 )│ (관절)   │ (센서)   │
```

### DoIP 포트 및 프로토콜
| 용도 | 프로토콜 | 포트 |
|------|---------|------|
| Vehicle Discovery | UDP | 13400 |
| DoIP 데이터 전송 | TCP | 13400 |
| DoIP Announce | UDP Multicast | 13400 |
| TLS 암호화 DoIP | TCP (TLS) | 3496 |

---

## DoIP 연결 수립 과정

```
Phase 1: Vehicle Discovery (UDP)

Tester                              DoIP Gateway
  │─── Vehicle Identification Request ──────────►│
  │    UDP Broadcast/Multicast                    │
  │                                              │
  │◄── Vehicle Announcement ────────────────────  │
  │    포함 정보: VIN, EID(Entity ID), GID        │

Phase 2: TCP 연결 수립

  │─── TCP SYN ──────────────────────────────────►│ (포트 13400)
  │◄── TCP SYN-ACK ─────────────────────────────  │
  │─── TCP ACK ──────────────────────────────────►│

Phase 3: Routing Activation (인증)

  │─── Routing Activation Request ───────────────►│
  │    Source Address: 0x0E00 (Tester)            │
  │    Activation Type: 0x00 (Default)            │
  │                                              │
  │◄── Routing Activation Response ─────────────  │
  │    응답 코드: 0x10 (Routing successfully)      │

Phase 4: UDS 진단 메시지 전송

  │─── Diagnostic Message ───────────────────────►│
  │    Source: 0x0E00, Target: 0x0101 (ECU #1)   │
  │    UDS Request: 22 F1 90 (ReadDataById VIN)  │
  │                                              │
  │◄── Diagnostic Message Positive ACK ─────────  │
  │                                              │
  │◄── Diagnostic Message (UDS Response) ───────  │
  │    62 F1 90 [VIN 데이터 17 Byte]              │
```

---

## DoIP 메시지 구조

```
DoIP Generic Header (8 Byte):
┌────────────────┬────────────────┬────────────────────────────┐
│ Protocol Ver.  │  Inverse PV    │  Payload Type  │  Length   │
│    1 Byte      │   1 Byte       │    2 Byte      │  4 Byte   │
└────────────────┴────────────────┴────────────────────────────┘

Protocol Version: 0x01 (ISO 13400-2:2010), 0x02 (ISO 13400-2:2012+)
Inverse PV: ~Protocol Version (XOR 검증)

Payload Type 주요 값:
  0x0001: Vehicle Identification Request
  0x0004: Vehicle Announcement / Identification Response
  0x0005: Routing Activation Request
  0x0006: Routing Activation Response
  0x8001: Diagnostic Message
  0x8002: Diagnostic Message Positive ACK
  0x8003: Diagnostic Message Negative ACK
```

---

## UDS (Unified Diagnostic Services) 주요 서비스

DoIP는 전송 계층이고, 실제 진단 내용은 UDS(ISO 14229)가 담당합니다.

```
UDS 서비스 분류:

진단 세션 관리:
  0x10: DiagnosticSessionControl
        00: 기본 세션, 01: 확장 세션, 02: 프로그래밍 세션
  0x11: ECUReset (소프트/하드 리셋)
  0x27: SecurityAccess (암호화 인증)

DTC (Diagnostic Trouble Code) 관리:
  0x19: ReadDTCInformation
  0x14: ClearDiagnosticInformation

데이터 읽기/쓰기:
  0x22: ReadDataByIdentifier (DID)
  0x2E: WriteDataByIdentifier (DID)
  0x2C: DynamicallyDefineDataIdentifier

ECU 프로그래밍 (OTA 핵심):
  0x34: RequestDownload (다운로드 요청)
  0x35: RequestUpload (업로드 요청)
  0x36: TransferData (데이터 전송)
  0x37: RequestTransferExit (전송 종료)

루틴 제어:
  0x31: RoutineControl (자가진단, 초기화 등)
  0x85: ControlDTCSetting (DTC 기록 ON/OFF)
  0x87: LinkControl (통신 속도 변경)
```

---

## DoIP 실습 예시

### Python으로 DoIP 연결 (개념 코드)
```python
import socket
import struct

DOIP_PORT = 13400
PROTOCOL_VERSION = 0x02

def build_doip_header(payload_type, payload):
    """DoIP Generic Header 생성"""
    length = len(payload)
    header = struct.pack(
        '>BBHI',
        PROTOCOL_VERSION,       # Protocol Version
        ~PROTOCOL_VERSION & 0xFF,  # Inverse PV
        payload_type,           # Payload Type
        length                  # Length
    )
    return header + payload

def routing_activation_request(source_addr=0x0E00):
    """Routing Activation Request 페이로드"""
    payload = struct.pack('>HBL', source_addr, 0x00, 0x00000000)
    return build_doip_header(0x0005, payload)

def uds_read_did(source=0x0E00, target=0x0101, did=0xF190):
    """UDS ReadDataByIdentifier (0x22) Request"""
    uds_msg = bytes([0x22]) + struct.pack('>H', did)
    payload = struct.pack('>HH', source, target) + bytes([0x00]) + uds_msg
    return build_doip_header(0x8001, payload)

# TCP 연결 및 진단
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('192.168.1.100', DOIP_PORT))

# Routing Activation
sock.send(routing_activation_request())
response = sock.recv(1024)
print(f"Routing ACK: {response.hex()}")

# VIN 읽기 (DID 0xF190)
sock.send(uds_read_did(did=0xF190))
response = sock.recv(1024)
print(f"VIN Response: {response.hex()}")

sock.close()
```

### Wireshark 필터
```bash
# DoIP 전체 트래픽
doip

# Routing Activation
doip.type == 0x0005 or doip.type == 0x0006

# 진단 메시지만
doip.type == 0x8001

# 특정 ECU (Target Address) 진단
doip.target_addr == 0x0101

# UDS 서비스 0x22 (ReadDataByIdentifier)
doip && data[0:1] == 22:00

# UDS DTC 관련
doip && data[0:1] == 19:00
```

---

## OTA 업데이트 흐름 (DoIP + UDS)

```
1단계: 진단 세션 전환
  Tester → ECU: 10 02 (Programming Session 전환)
  ECU → Tester: 50 02 (긍정 응답)

2단계: 보안 해제 (SecurityAccess)
  Tester → ECU: 27 01 (Seed 요청)
  ECU → Tester: 67 01 [Seed 4Byte] (랜덤 챌린지)
  Tester → ECU: 27 02 [Key 4Byte]  (HMAC 계산한 키)
  ECU → Tester: 67 02 (보안 해제 성공)

3단계: 소프트웨어 다운로드 시작
  Tester → ECU: 34 00 44 [주소 4B] [크기 4B] (RequestDownload)
  ECU → Tester: 74 20 [MaxBlockLen] (Block당 최대 크기)

4단계: 데이터 전송 (반복)
  Tester → ECU: 36 01 [데이터 블록 1] (TransferData Block#1)
  ECU → Tester: 76 01 (ACK)
  Tester → ECU: 36 02 [데이터 블록 2] (TransferData Block#2)
  ECU → Tester: 76 02 (ACK)
  ...
  (1Gbps Ethernet으로 1GB 데이터 ≈ 8초)

5단계: 완료 및 검증
  Tester → ECU: 37 (RequestTransferExit)
  ECU → Tester: 77 (ACK)
  Tester → ECU: 31 01 FF 01 (Routine: 다운로드 검증 체크섬)
  ECU → Tester: 71 01 FF 01 00 (체크섬 OK)

6단계: ECU 재시작
  Tester → ECU: 11 01 (HardReset)
```

---

## DoIP 보안

```
DoIP + TLS (TCP 3496):
  - 진단 세션 암호화 (AES-256-GCM)
  - 클라이언트 인증서 기반 인증
  - 원격 진단 시 필수

SecurityAccess (UDS 0x27):
  - Seed-Key 알고리즘 (HMAC, AES 기반)
  - 프로그래밍 세션 접근 보호
  - 권한 레벨 분리 (L1: 읽기, L2: 쓰기, L3: 프로그래밍)
```

---

## Reference
- [ISO 13400-2:2019 - DoIP Standard](https://www.iso.org/standard/69527.html)
- [ISO 14229-1 - UDS (Unified Diagnostic Services)](https://www.iso.org/standard/72439.html)
- [Vector DoIP](https://www.vector.com/kr/ko/know-how/protocols/doip/)
- [python-doip 오픈소스](https://github.com/mukel/python-doip)
