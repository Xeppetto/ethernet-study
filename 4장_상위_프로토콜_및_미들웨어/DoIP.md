# DoIP (Diagnostics over IP) - ISO 13400

## 개요
DoIP(Diagnostics over IP)는 Ethernet 기반 진단 통신 표준입니다(ISO 13400-2). 기존 CAN 기반 진단 프로토콜(K-Line, ISO 15765 CAN Transport Layer)을 대체하여 고속 Ethernet으로 UDS(ISO 14229) 진단 서비스를 제공합니다. 1Gbps Ethernet을 사용하면 1GB 펌웨어 업데이트를 약 8초 만에 완료할 수 있어, CAN(500Kbps)에서 수십 분 걸리던 OTA 업데이트가 혁신적으로 단축됩니다.

의료 로봇에서는 DoIP를 통해 **원격 유지보수, 펌웨어 OTA 업데이트, 고장 진단, 규제 기관 감사** 기록 접근이 가능합니다. ISO 13400과 함께 UDS(ISO 14229), OBD-II(ISO 15031)가 진단 프로토콜 스택을 구성합니다.

---

## DoIP 엔티티 유형 및 아키텍처

### 엔티티 유형 비교

| 엔티티 | 역할 | 논리 주소 범위 | 예시 |
|--------|------|--------------|------|
| DoIP Tester | 진단 테스트 시작점 | 0x0E00~0x0EFF | PC 진단 툴, OTA 서버 |
| DoIP Gateway | 외부→내부 라우팅 | 0x0001~0x0DFF | 차량 게이트웨이, 로봇 게이트웨이 |
| DoIP Node | ECU 직접 통신 | 0x0001~0x0DFF | 독립 Ethernet ECU |
| External Test Equipment | 테스트 장비 | 0x0F00~0x0FFF | 공장 자동화 테스터 |

### 시스템 아키텍처 다이어그램

```
[진단 테스터 PC / OTA 서버]
         │
         │ Ethernet (TCP/UDP 13400)
         │
  ┌──────┴──────────────────────────────────────────────┐
  │  DoIP Gateway (로봇 메인 컨트롤러)                    │
  │  논리 주소: 0x0001                                    │
  │  ├── Routing Table                                   │
  │  │   0x0101 → Joint ECU #1 (CAN/Eth)                │
  │  │   0x0102 → Joint ECU #2 (CAN/Eth)                │
  │  │   0x0200 → Vision ECU  (Ethernet)                │
  │  │   0x0300 → Safety ECU  (CAN)                     │
  │  └── NAT/Bridge 기능                                 │
  └─────┬─────────────────────────────────────────────────┘
        │
   내부 버스 (CAN 2.0B / Ethernet)
        │
   ┌────┴──────┬──────────────────┬─────────────────┐
   │           │                  │                 │
Joint ECU #1  Joint ECU #2    Vision ECU       Safety ECU
0x0101        0x0102          0x0200           0x0300
```

---

## DoIP 포트 및 프로토콜

| 용도 | 프로토콜 | 포트 | 설명 |
|------|---------|------|------|
| Vehicle Discovery | UDP (Broadcast) | 13400 | 네트워크 내 DoIP 기기 탐색 |
| Vehicle Announcement | UDP (Multicast) | 13400 | DoIP 기기 자발적 알림 |
| DoIP 데이터 전송 | TCP | 13400 | UDS 진단 메시지 (신뢰성 보장) |
| TLS 암호화 DoIP | TCP + TLS | 3496 | 원격/OTA 진단 (암호화 필수) |

---

## DoIP 연결 수립 과정 (4단계)

```
Phase 1: Vehicle Discovery (UDP Broadcast/Multicast)

Tester (0x0E00)                         DoIP Gateway
    │─── Vehicle Identification Request ──────────────►│
    │    UDP Broadcast to 255.255.255.255:13400        │
    │    Payload: 없음 (0x0001 타입)                    │
    │                                                  │
    │◄── Vehicle Announcement / ID Response ───────────│
    │    UDP Unicast to Tester IP                      │
    │    포함: VIN(17B), EID(6B MAC), GID(6B 그룹)     │
    │         Logic Address: 0x0001                    │

Phase 2: TCP 3-Way Handshake

    │─── TCP SYN ─────────────────────────────────────►│ :13400
    │◄── TCP SYN-ACK ──────────────────────────────────│
    │─── TCP ACK ─────────────────────────────────────►│

Phase 3: Routing Activation (DoIP 계층 인증)

    │─── Routing Activation Request ──────────────────►│
    │    Source Address: 0x0E00 (Tester ID)            │
    │    Activation Type: 0x00 (Default)               │
    │    Reserved: 0x00000000                          │
    │                                                  │
    │◄── Routing Activation Response ──────────────────│
    │    Logical Address of Tester: 0x0E00             │
    │    Logical Address of GW: 0x0001                 │
    │    Response Code: 0x10 (Routing successfully)    │

Phase 4: UDS 진단 메시지 교환

    │─── Diagnostic Message ──────────────────────────►│
    │    Source: 0x0E00, Target: 0x0101 (ECU #1)       │
    │    UDS: 10 03 (Extended Diagnostic Session)      │
    │                                                  │
    │◄── Diagnostic Message Positive ACK ──────────────│
    │    ACK Code: 0x00 (확인 수신)                    │
    │                                                  │
    │◄── Diagnostic Message (UDS Positive Response) ───│
    │    50 03 00 32 01 F4 (ExtSession OK + Timing)    │
```

---

## DoIP 메시지 헤더 구조

```
DoIP Generic Header (8 Byte):
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────────────┬───────────────┬───────────────────────────────┐
│ Protocol Ver. │  Inverse PV   │        Payload Type           │
│    (1 Byte)   │  (~ProtVer)   │          (2 Byte)             │
├───────────────┴───────────────┴───────────────────────────────┤
│                      Payload Length                            │
│                        (4 Byte)                               │
└───────────────────────────────────────────────────────────────┘
│                      Payload (가변)                            │
└───────────────────────────────────────────────────────────────┘

Protocol Version:
  0x01: ISO 13400-2:2010
  0x02: ISO 13400-2:2012 이후 (현재 표준)
  0xFF: Wildcard (Vehicle Identification 요청 시)

Inverse PV: ~Protocol_Version & 0xFF (오류 감지용)

Payload Type 주요 값:
  0x0001: Vehicle Identification Request
  0x0003: Vehicle Identification Request with EID
  0x0004: Vehicle Announcement / ID Response
  0x0005: Routing Activation Request
  0x0006: Routing Activation Response
  0x0007: Alive Check Request
  0x0008: Alive Check Response
  0x4001: DoIP Entity Status Request
  0x4002: DoIP Entity Status Response
  0x8001: Diagnostic Message
  0x8002: Diagnostic Message Positive ACK
  0x8003: Diagnostic Message Negative ACK

Routing Activation Response Code:
  0x00: Denied (알 수 없는 소스 주소)
  0x01: Denied (소켓 포화)
  0x04: Denied (보안 거부)
  0x10: Successfully activated
  0x11: Successfully activated (이미 활성화된 소스)
```

---

## Python DoIP 구현 예시

```python
import socket
import struct
import time

DOIP_PORT = 13400
PROTOCOL_VERSION = 0x02

def build_doip_header(payload_type: int, payload: bytes) -> bytes:
    """DoIP Generic Header 생성"""
    return struct.pack(
        '>BBHI',
        PROTOCOL_VERSION,
        (~PROTOCOL_VERSION) & 0xFF,
        payload_type,
        len(payload)
    ) + payload

def routing_activation_request(source_addr: int = 0x0E00) -> bytes:
    """Routing Activation Request (Type 0x0005)"""
    payload = struct.pack('>HBL', source_addr, 0x00, 0x00000000)
    return build_doip_header(0x0005, payload)

def diagnostic_message(src: int, tgt: int, uds_data: bytes) -> bytes:
    """Diagnostic Message (Type 0x8001)"""
    payload = struct.pack('>HH', src, tgt) + uds_data
    return build_doip_header(0x8001, payload)

def parse_doip_response(data: bytes) -> dict:
    """DoIP 응답 파싱"""
    if len(data) < 8:
        return {"error": "short header"}
    proto_ver, inv_ver, ptype, plen = struct.unpack('>BBHI', data[:8])
    return {
        "protocol_version": proto_ver,
        "payload_type": hex(ptype),
        "payload_length": plen,
        "payload": data[8:8 + plen].hex()
    }

class DoIPClient:
    def __init__(self, gateway_ip: str, port: int = DOIP_PORT):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.settimeout(5.0)
        self.sock.connect((gateway_ip, port))
        self._activate_routing()

    def _activate_routing(self):
        self.sock.send(routing_activation_request(0x0E00))
        resp = self.sock.recv(1024)
        parsed = parse_doip_response(resp)
        if parsed["payload_type"] != "0x6":
            raise ConnectionError("Routing activation failed")
        print("Routing activated")

    def send_uds(self, target_addr: int, uds_request: bytes) -> bytes:
        """UDS 서비스 요청 전송 및 응답 수신"""
        msg = diagnostic_message(0x0E00, target_addr, uds_request)
        self.sock.send(msg)

        # Positive ACK 수신 (0x8002)
        ack = self.sock.recv(1024)

        # UDS 응답 수신 (0x8001)
        resp = self.sock.recv(4096)
        parsed = parse_doip_response(resp)
        # 페이로드: Source(2B) + Target(2B) + UDS응답
        return bytes.fromhex(parsed["payload"])[4:]

    def read_did(self, target: int, did: int) -> bytes:
        """UDS 0x22 ReadDataByIdentifier"""
        request = bytes([0x22]) + struct.pack('>H', did)
        return self.send_uds(target, request)

    def close(self):
        self.sock.close()

# 사용 예시
client = DoIPClient('192.168.1.1')
vin_data = client.read_did(0x0001, 0xF190)
print(f"VIN: {vin_data.decode('ascii', errors='replace')}")
sw_ver = client.read_did(0x0101, 0xF189)
print(f"Joint ECU SW Version: {sw_ver.hex()}")
client.close()
```

---

## Wireshark 분석

```bash
# DoIP 전체 트래픽
doip

# Routing Activation Request/Response
doip.type == 0x0005 or doip.type == 0x0006

# 진단 메시지만 (UDS 포함)
doip.type == 0x8001

# 특정 ECU (Target Address) 진단
doip.target_addr == 0x0101

# UDS 서비스별 필터
doip && data[4:5] == 10:00   # DiagnosticSessionControl
doip && data[4:5] == 22:00   # ReadDataByIdentifier
doip && data[4:5] == 27:00   # SecurityAccess
doip && data[4:5] == 34:00   # RequestDownload
doip && data[4:5] == 36:00   # TransferData
doip && data[4:5] == 7f:00   # Negative Response

# VIN 읽기 (DID 0xF190)
doip && data[5:7] == f1:90

# TLS 암호화 DoIP (포트 3496)
tcp.port == 3496
```

---

## OTA 업데이트 흐름 (DoIP + UDS)

```
전체 OTA Flash 프로그래밍 타임라인:

① 세션 전환 (Programming Session)
   → 10 02             ECU 응답: 50 02 00 32 01 F4
   → P2*max = 0x01F4 = 500ms (OTA 중 응답 타임아웃)

② 보안 해제 (SecurityAccess Level 0x05/0x06)
   → 27 05             ECU: 67 05 [Seed 4B]  (랜덤 챌린지)
   → 27 06 [Key 4B]    ECU: 67 06            (해제 성공)
   Key = HMAC_SHA256(Seed, Secret)[:4]

③ 메모리 삭제 (RoutineControl 0xFF00)
   → 31 01 FF 00 [StartAddr:4B] [Size:4B]
   → ECU: 7F 31 78              (처리 중, 기다려라)
   → 31 03 FF 00                (삭제 완료 요청)
   → ECU: 71 03 FF 00 00        (완료 확인)
   소요 시간: 플래시 종류에 따라 2~30초

④ 다운로드 시작 (RequestDownload)
   → 34 00 44 [주소:4B] [압축해제크기:4B]
   → ECU: 74 20 [MaxBlockSize:2B]
   MaxBlockSize 예: 0x0400 = 1024 Byte

⑤ 데이터 전송 (TransferData 반복)
   → 36 01 [Block #1 데이터, MaxBlockSize]  ECU: 76 01
   → 36 02 [Block #2 데이터]                ECU: 76 02
   ...
   1Gbps Ethernet: 1GB 데이터 ≈ 8초
   CAN 500Kbps: 1GB 데이터 ≈ 4.5시간

⑥ 전송 종료 (RequestTransferExit)
   → 37                ECU: 77

⑦ 체크섬 검증 (RoutineControl 0x0202)
   → 31 01 02 02 [CRC32:4B]
   → ECU: 71 01 02 02 00  (체크섬 일치)
   → ECU: 71 01 02 02 01  (체크섬 불일치 → 재전송 필요)

⑧ ECU 재시작 (ECUReset)
   → 11 01  (HardReset)   ECU: 11 01 후 즉시 재시작
   재시작 후 Secure Boot 검증 → 정상 부팅 확인
```

---

## DoIP 보안

```
DoIP + TLS (포트 3496):
  TLS 1.3 + 상호 인증 (mTLS)
  ├── AES-256-GCM 세션 암호화
  ├── 클라이언트(테스터) X.509 인증서 검증
  └── 원격 진단/OTA 업데이트 시 필수

SecurityAccess (UDS 0x27) 알고리즘:
  ┌─────────────────────────────────────────────────┐
  │  Level 01/02 (읽기/기본 쓰기):                   │
  │    Key = AES-128-CMAC(Seed, DeviceKey_L1)[:4]  │
  │                                                 │
  │  Level 05/06 (플래시 프로그래밍):                │
  │    Key = HMAC-SHA256(Seed || ECU_SN, MasterKey) │
  │    → ECU 시리얼 번호 바인딩으로 키 재사용 방지   │
  └─────────────────────────────────────────────────┘

접근 제어 레벨:
  Level 01/02: 파라미터 읽기/쓰기
  Level 03/04: 보정 데이터 변경
  Level 05/06: Flash 프로그래밍 (가장 높은 권한)
  Level 63/64: 개발/생산 전용 (출하 전 비활성화)
```

---

## 의료 로봇 DoIP 활용

```
원격 유지보수 시나리오:

현장 엔지니어 PC (TLS 터널)
    │
    │ DoIP over TLS (포트 3496)
    ├─► 원격 진단 세션 수립
    ├─► DTC 읽기/삭제 (0x19 / 0x14)
    ├─► 캘리브레이션 데이터 업데이트 (0x2E)
    └─► 펌웨어 OTA 업데이트 (34/36/37)

의료 로봇 DID 설계 (제조사 정의 범위 0x0100~0xEFFF):
  0xF190: 로봇 시리얼 번호 (17자리 ASCII)
  0xF189: SW 버전 (Major.Minor.Patch)
  0xF196: HW 버전
  0x0101: 관절 캘리브레이션 데이터 (6관절 × 8B = 48B)
  0x0102: 현재 관절 각도 (6 × float32 = 24B)
  0x0103: 모터 전류 (6 × float32 = 24B)
  0x0104: 온도 센서 (6 × float32 = 24B)
  0x0200: 수술 카운터 (uint32, 4B)
  0x0201: 총 동작 시간 (uint64 ms, 8B)
  0x0202: 에러 히스토리 (최근 100개, 가변)
  0x0300: 안전 파라미터 (SecAccess L5/L6 필요)
  0x0301: 물리적 제한값 (관절 각도/속도 한계)

규제 대응:
  FDA 21 CFR Part 11: DoIP 로그 접근 감사 추적
  MDR Annex II: 기술 문서에 진단 인터페이스 명시
  IEC 62304: 펌웨어 업데이트 검증 절차 문서화
```

---

## Reference
- [ISO 13400-2:2019 - DoIP Application Layer](https://www.iso.org/standard/69527.html)
- [ISO 14229-1:2020 - UDS Application Layer Services](https://www.iso.org/standard/72439.html)
- [Vector DoIP Introduction](https://www.vector.com/kr/ko/know-how/protocols/doip/)
- [python-doip Open Source](https://github.com/mukel/python-doip)
- [automotive-doip-client (PyPI)](https://pypi.org/project/doipclient/)
- [Wireshark DoIP Dissector](https://wiki.wireshark.org/DoIP)
