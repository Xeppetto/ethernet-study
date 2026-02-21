# TLS 및 PKI 인증서 관리

## 개요
TLS(Transport Layer Security)는 TCP 통신에서 기밀성(Confidentiality), 무결성(Integrity), 인증(Authentication)을 제공하는 프로토콜입니다. 의료 로봇은 내부망(Intranet)과 클라우드 서버 간 통신에서 TLS를 필수적으로 사용하며, 이를 안전하게 운영하기 위해 PKI(Public Key Infrastructure)와 인증서 관리 체계가 필요합니다.

---

## TLS 1.3 핸드셰이크 과정

```
TLS 1.3 핸드셰이크 (RFC 8446, 1-RTT):

Client (Robot)                          Server (Cloud/Diag)
     │                                        │
     │──── ClientHello ──────────────────────►│
     │     supported_versions: TLS 1.3        │
     │     supported_groups: x25519, P-256    │
     │     key_share: [x25519 공개 키]         │
     │     signature_algs: ecdsa_secp256r1    │
     │                                        │
     │◄─── ServerHello ───────────────────────│
     │     key_share: [x25519 공개 키]         │
     │     ← 이 시점에 공유 비밀 생성           │
     │◄─── {EncryptedExtensions} ─────────────│ ← 이후 암호화
     │◄─── {CertificateRequest} ──────────────│ (mTLS 시)
     │◄─── {Certificate} ─────────────────────│
     │◄─── {CertificateVerify} ───────────────│
     │◄─── {Finished} ────────────────────────│
     │                                        │
     │──── {Certificate} ─────────────────────►│ (mTLS: 클라이언트 인증서)
     │──── {CertificateVerify} ───────────────►│
     │──── {Finished} ────────────────────────►│
     │                                        │
     │◄═══ Application Data (암호화) ══════════│
     │                                        │
     (총 1 RTT - TLS 1.2 대비 0.5 RTT 절약)

TLS 1.3 핵심 개선 (vs TLS 1.2):
  ① 핸드셰이크 1-RTT (1.2: 2-RTT)
  ② 0-RTT 세션 재개 (연결 이력 있을 때)
  ③ 취약 알고리즘 제거 (RSA key exchange, RC4, DES, MD5)
  ④ Perfect Forward Secrecy 강제 (ECDHE 필수)
  ⑤ 서버 인증서도 암호화 전송 (스니핑 방지)
```

---

## 암호화 알고리즘 (의료 로봇 권장)

```
TLS 1.3 Cipher Suite (의료 로봇 권장):

최고 보안:
  TLS_AES_256_GCM_SHA384
    ├── AES-256-GCM: 256비트 대칭 암호화 (인증 암호화)
    └── SHA-384: 메시지 인증 (HMAC)

일반 보안:
  TLS_AES_128_GCM_SHA256     (성능/보안 균형)
  TLS_CHACHA20_POLY1305_SHA256 (저사양 MCU에 유리)

키 교환:
  ECDHE P-256: 128비트 보안 강도 (표준)
  ECDHE X25519: 성능 우수 (구글 권장)
  ECDHE P-384: 192비트 보안 강도 (정부/의료 권장)

인증서 서명:
  ECDSA P-256/P-384: ECU 인증서 (컴팩트, 빠름)
  RSA-3072 이상: 레거시 호환 시

양자 내성 (Post-Quantum, 향후 대비):
  ML-KEM (Kyber) + ECDHE 하이브리드
  ML-DSA (Dilithium) 서명
```

---

## PKI 인증서 계층

```
의료 로봇 PKI 계층 구조:

Level 0: Root CA (오프라인, HSM 보관)
  ├── 자체 서명 (Self-signed)
  ├── 유효 기간: 20년
  ├── 키: RSA-4096 or ECDSA P-384
  └── 보관: 에어갭(Air-gap) 환경

Level 1: Intermediate CA (온라인)
  ├── Root CA가 서명
  ├── 유효 기간: 5~10년
  ├── 목적별 분리:
  │   ├── Device CA (ECU 인증서 발급)
  │   ├── Server CA (클라우드 서버)
  │   └── OTA CA (업데이트 서명)
  └── 보관: HSM (FIPS 140-2 Level 3+)

Level 2: End Entity Certificate
  ├── 로봇 ECU 인증서 (Device Certificate)
  │   유효 기간: 1~3년
  │   Subject: CN=robot-R001-jointECU, O=HospitalRobotCo
  ├── 서버 인증서 (TLS Server)
  │   유효 기간: 1년
  └── OTA 코드 서명 인증서
      유효 기간: 2년

X.509 인증서 구조:
┌─────────────────────────────────────────────┐
│ Version: 3                                  │
│ Serial Number: 0x1A2B3C...                  │
│ Signature Algorithm: ecdsa-with-SHA256      │
│ Issuer: CN=Device-CA, O=RobotCo             │
│ Validity: Not Before / Not After            │
│ Subject: CN=robot-R001, O=HospitalRobotCo   │
│ Subject Public Key: EC P-256 (65 bytes)     │
│ Extensions:                                 │
│   Key Usage: digitalSignature, keyAgreement │
│   Extended Key Usage: clientAuth            │
│   Subject Alt Name: robot-R001.hospital.com │
│   Basic Constraints: CA:FALSE               │
│   CRL Distribution Points: URL             │
│   OCSP Access Method: URL                  │
│ Signature: [CA의 서명]                       │
└─────────────────────────────────────────────┘
```

---

## Mutual TLS (mTLS) - 양방향 인증

```
mTLS 구성 (의료 로봇 필수):

일반 TLS (단방향):
  Client ──── 서버 인증서 검증 ────► Server
  클라이언트 인증 없음

mTLS (양방향):
  Client ──── 서버 인증서 검증 ────► Server
  Client ◄─── 클라이언트 인증서 검증 ── Server
  양방향 인증

mTLS 적용 시나리오:
  로봇 ECU ←→ 클라우드 백엔드 (MQTT/REST API)
  로봇 ECU ←→ DoIP 진단 서버
  로봇 ECU ←→ OTA 업데이트 서버

nginx mTLS 서버 설정 예시:
  ssl_certificate     /etc/ssl/certs/server.crt;
  ssl_certificate_key /etc/ssl/private/server.key;
  ssl_client_certificate /etc/ssl/certs/device-ca.crt;
  ssl_verify_client   on;  ← 클라이언트 인증서 검증 강제
  ssl_verify_depth    2;   ← CA 체인 깊이
```

---

## IEEE 802.1AR - Secure Device Identity (DevID)

```
IEEE 802.1AR 인증서 종류:

IDevID (Initial DevID):
  - 제조 공장에서 칩/모듈에 영구 설치
  - HSM 또는 Secure Element에 저장
  - 제조사 Root CA로 서명
  - 변경 불가
  - 목적: 기기 출처 증명

LDevID (Locally Significant DevID):
  - 운용 단계에서 발급 (예: 병원에서 로봇 등록 시)
  - IDevID로 인증 후 LDevID 발급
  - 갱신 가능
  - 목적: 실제 운용 인증

DevID 발급 흐름:
  공장 ──── IDevID 설치 (제조사 CA)
              │
         병원 납품
              │
         ┌────▼───────────┐
         │ 병원 IT 등록    │
         │ EST 서버에 요청 │
         │ (IDevID로 인증) │
         └────┬───────────┘
              │
         LDevID 발급
         (병원 Device CA)
              │
         이후 TLS/MQTT 접속 시 LDevID 사용
```

---

## 인증서 발급/갱신 자동화

```python
# EST (Enrollment over Secure Transport, RFC 7030) 클라이언트
# Python 예시 - 인증서 자동 갱신

import ssl
import http.client
import base64
from pathlib import Path

EST_SERVER = 'pki.hospital.com'
EST_PORT = 8443

def renew_certificate(current_cert_path: str,
                       current_key_path: str,
                       csr_path: str) -> bytes:
    """
    EST를 통한 인증서 갱신 (Re-enrollment)
    RFC 7030: /.well-known/est/simplereenroll
    """
    # 현재 인증서로 mTLS 연결
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    ctx.load_cert_chain(current_cert_path, current_key_path)
    ctx.load_verify_locations('/etc/ssl/certs/hospital-ca.crt')
    ctx.minimum_version = ssl.TLSVersion.TLSv1_3

    # CSR (Certificate Signing Request) 읽기
    csr_pem = Path(csr_path).read_bytes()
    csr_b64 = base64.b64encode(csr_pem)

    # EST simplereenroll 요청
    conn = http.client.HTTPSConnection(
        EST_SERVER, EST_PORT, context=ctx
    )
    conn.request(
        'POST',
        '/.well-known/est/simplereenroll',
        body=csr_b64,
        headers={
            'Content-Type': 'application/pkcs10',
            'Content-Transfer-Encoding': 'base64'
        }
    )

    resp = conn.getresponse()
    if resp.status == 200:
        new_cert_der = base64.b64decode(resp.read())
        print(f"인증서 갱신 성공: {len(new_cert_der)} bytes")
        return new_cert_der
    else:
        raise Exception(f"EST 갱신 실패: {resp.status} {resp.reason}")


def check_cert_expiry(cert_path: str, warn_days: int = 30) -> int:
    """인증서 만료일 확인 및 경고"""
    import datetime
    ctx = ssl.create_default_context()
    with open(cert_path, 'rb') as f:
        cert_data = f.read()

    cert = ssl.PEM_cert_to_DER_cert(cert_data.decode())
    # 실제로는 cryptography 라이브러리 사용
    # from cryptography import x509
    # cert_obj = x509.load_der_x509_certificate(cert)
    # days_left = (cert_obj.not_valid_after - datetime.datetime.utcnow()).days

    # 예시 반환
    days_left = 25
    if days_left < warn_days:
        print(f"[WARNING] 인증서 {warn_days}일 내 만료: {days_left}일 남음")
        # 갱신 트리거
        renew_certificate(
            '/etc/ssl/robot/current.crt',
            '/etc/ssl/robot/current.key',
            '/etc/ssl/robot/renew.csr'
        )
    return days_left
```

---

## 인증서 폐기 관리

```
CRL vs OCSP 비교:

CRL (Certificate Revocation List):
  - CA가 주기적으로 발행하는 폐기 목록
  - 파일 형식 (.crl, DER or PEM)
  - 단점: 주기적 업데이트, 파일 크기 증가
  - 적합: 오프라인 환경, 내부망

OCSP (Online Certificate Status Protocol):
  - 실시간 인증서 상태 조회
  - OCSP Responder (온라인 서버)
  - OCSP Stapling: 서버가 OCSP 응답을 캐시하여 첨부
  - 단점: 인터넷 연결 필요
  - 적합: 실시간 검증 필요 환경

의료 로봇 권장:
  내부망: CRL + 캐시 (24시간 갱신)
  클라우드 통신: OCSP Stapling
  비상 시: Pinned Certificate (인증서 고정)

인증서 폐기 시나리오:
  로봇 도난 → 즉시 LDevID 폐기 →
  CRL 갱신 + OCSP 업데이트 →
  도난 로봇은 서버 접속 불가
```

---

## MACsec (IEEE 802.1AE) - L2 암호화

```
MACsec vs TLS:

TLS:
  L4~L7 암호화 (Application, Transport)
  IP 헤더/포트는 평문
  각 애플리케이션에서 개별 처리

MACsec (IEEE 802.1AE):
  L2 암호화 (Ethernet 프레임 전체)
  MAC 주소 외 전체 페이로드 암호화
  스위치 포트 간 점대점 암호화
  하드웨어 가속 (라인 속도 암호화)

MACsec 프레임 구조:
  [Dst MAC][Src MAC][802.1Q][SecTAG][암호화 페이로드][ICV]
                              ↑ MACsec 태그 (8B)

SecTAG (Security Tag, 8 Byte):
  Ethertype: 0x88E5
  TCI/AN: 암호화 알고리즘 + 연결 번호
  SL: 짧은 길이 (Short Length)
  PN: Packet Number (재생 방지)
  SCI: Secure Channel Identifier (16B, 옵션)

의료 로봇 MACsec 적용:
  Safety Network (VLAN 10):
    MACsec 필수 (AES-GCM-256)
    암호화 + 인증 + 재생 방지

Linux MACsec 설정:
  # 키 설정 (실제로는 MKA 프로토콜 자동 협상)
  ip macsec add macsec0 dev eth0 sci 1
  ip macsec add macsec0 tx sa 0 pn 1 on key 00 \
    0123456789abcdef0123456789abcdef
  ip link set macsec0 up
  ip addr add 192.168.10.1/24 dev macsec0
```

---

## Wireshark로 TLS 분석

```bash
# TLS 핸드셰이크 분석
tshark -r capture.pcap -Y "tls.handshake" \
  -T fields \
  -e ip.src -e ip.dst \
  -e tls.handshake.type \
  -e tls.handshake.ciphersuite

# 핸드셰이크 타입:
# 1: ClientHello, 2: ServerHello
# 11: Certificate, 15: CertificateVerify
# 20: Finished

# TLS 버전 확인
tshark -r capture.pcap -Y "tls" \
  -T fields -e tls.record.version

# 인증서 정보 추출
tshark -r capture.pcap -Y "tls.handshake.type == 11" \
  -T fields \
  -e tls.handshake.certificate

# TLS 복호화 (SSLKEYLOGFILE 사용)
# 개발 환경에서만 사용!
export SSLKEYLOGFILE=/tmp/tls_keylog.txt
# 애플리케이션 실행 후...
tshark -r capture.pcap \
  -o "tls.keylog_file:/tmp/tls_keylog.txt" \
  -Y "http2 or mqtt"
```

---

## Reference
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5280 - X.509 Certificate Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [RFC 7030 - EST (Enrollment over Secure Transport)](https://datatracker.ietf.org/doc/html/rfc7030)
- [IEEE 802.1AR - Secure Device Identity](https://standards.ieee.org/ieee/802.1AR/6995/)
- [IEEE 802.1AE - MACsec](https://standards.ieee.org/ieee/802.1AE/7029/)
- [NIST SP 800-57 - Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
