# TLS 및 PKI 인증서 관리

## 개요

TLS(Transport Layer Security)는 TCP 통신에서 기밀성(Confidentiality), 무결성(Integrity), 인증(Authentication)을 제공하는 핵심 보안 프로토콜입니다. 의료 로봇은 내부망(Intranet)과 클라우드 서버 간 통신에서 TLS를 필수적으로 사용하며, 이를 안전하게 운영하기 위해 PKI(Public Key Infrastructure)와 인증서 관리 체계가 필요합니다.

단순히 "HTTPS를 쓰면 된다"는 인식은 불충분합니다. 의료 로봇은 다음을 모두 고려해야 합니다.

- **어떤 레이어를 암호화하는가**: 애플리케이션(TLS), Ethernet 프레임 수준(MACsec), UDP 기반(DTLS)
- **누구를 인증하는가**: 서버만(TLS) vs 양방향(mTLS) vs 장치 고유 ID(IEEE 802.1AR)
- **인증서를 어떻게 관리하는가**: 발급(EST), 갱신, 폐기(CRL/OCSP), 보관(HSM)

> **의료 로봇 관점**: 수술 로봇의 통신 보안은 단일 프로토콜로 해결되지 않습니다. 외부 클라우드 연결은 TLS 1.3 + mTLS, 병원 내부 네트워크의 스위치 포트 간 암호화는 MACsec, UDP 기반 실시간 제어 데이터(DDS/ROS2)는 DTLS, 장치 고유 인증은 IEEE 802.1AR DevID로 계층화하여 적용합니다. FDA 2023 가이드라인은 이러한 다층 통신 보안을 사이버 보안 아키텍처 다이어그램에 명시하도록 요구합니다. `ISO_SAE_21434.md`의 CAL 3/4 제어 항목으로 통신 보안이 포함됩니다.

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

MACsec은 Ethernet 프레임 수준(L2)에서 동작하는 링크 암호화 프로토콜입니다. TLS가 각 애플리케이션 세션마다 독립적으로 암호화를 협상하는 반면, MACsec은 Ethernet 링크 전체를 투명하게 암호화합니다. TLS와 MACsec은 상호 배타적이지 않으며, 함께 사용할 때 심층 방어(Defense in Depth)를 구현할 수 있습니다.

```
MACsec vs TLS 레이어 비교:

┌───────────────────────────────────────────────────────────┐
│ Application  │  SOME/IP, DDS/ROS2, MQTT, REST API        │
│ (L7)         │  ↑ TLS 1.3 암호화 (소켓 수준)             │
├──────────────┼────────────────────────────────────────────┤
│ Transport    │  TCP / UDP                                 │
│ (L4)         │  IP 헤더, 포트 번호 평문 (TLS 사용 시)     │
├──────────────┼────────────────────────────────────────────┤
│ Network      │  IP                                       │
│ (L3)         │  IP 헤더 평문                             │
├──────────────┼────────────────────────────────────────────┤
│ Data Link    │  Ethernet + MACsec                        │
│ (L2)         │  ↑ MACsec: MAC 주소 제외 전체 암호화       │
│              │  IP 헤더, 포트 번호도 암호화               │
├──────────────┼────────────────────────────────────────────┤
│ Physical     │  1GbE / 10GbE                             │
│ (L1)         │                                           │
└──────────────┴────────────────────────────────────────────┘

MACsec 장점:
  ✓ 애플리케이션 수정 없이 링크 전체 암호화
  ✓ IP 헤더, 포트, VLAN ID도 암호화 (트래픽 분석 방지)
  ✓ 하드웨어 가속 (라인 속도, < 1µs 오버헤드)
  ✓ 스위치 포트 간 점대점 암호화 (내부망 도청 방지)
  ✓ AES-GCM-128/256 인증 암호화 (무결성 + 기밀성)
  ✓ Packet Number로 재생 공격 방지

MACsec 단점:
  ✗ 동일 링크 (1홉)에서만 동작 (라우터 통과 시 재암호화 필요)
  ✗ 일부 임베디드 NIC는 하드웨어 지원 미흡
  ✗ TLS보다 설정/관리 복잡 (MKA 키 관리)
```

### MACsec 프레임 구조

```
MACsec 프레임 구조 (IEEE 802.1AE):

[Dst MAC 6B][Src MAC 6B][SecTAG 8~16B][암호화 페이로드][ICV 16B]
                            ↑ 802.1AE 태그

SecTAG (Security Tag):
  ┌──────────────────────────────────────────────────┐
  │ EtherType: 0x88E5 (2B)                           │
  │ TCI: Tag Control Information (1B)                │
  │   V: 버전 = 0                                    │
  │   ES: End Station (Explicit SCI 여부)            │
  │   SC: SCI Present                                │
  │   SCB: Single Copy Broadcast                     │
  │   E:  Encryption (1 = 암호화)                    │
  │   C:  Changed Text (1 = ICV 포함)               │
  │ AN: Association Number (2bit) → 키 세대 구분     │
  │ SL: Short Length (1B) → 짧은 프레임 처리         │
  │ PN: Packet Number (4B) → 재생 방지 카운터        │
  │ [SCI: Secure Channel Identifier, 8B 선택적]     │
  └──────────────────────────────────────────────────┘

ICV (Integrity Check Value, 16B):
  AES-GCM의 인증 태그 (변조 탐지)
  → 수신측에서 ICV 검증 실패 시 프레임 폐기
```

### MACsec 키 관리 (MKA 프로토콜)

MACsec 키는 수동 설정(정적 키) 또는 MKA(MACsec Key Agreement, IEEE 802.1X-2020)로 자동 협상할 수 있습니다. 의료 로봇 환경에서는 MKA + EAP-TLS 조합이 권장됩니다.

```bash
# 방법 1: 수동 키 설정 (개발/테스트용)
# 주의: 프로덕션에서는 MKA 사용 권장

# MACsec 인터페이스 생성
ip link add link eth0 macsec0 type macsec \
  sci 0x0123456789abcdef \
  encrypt on

# TX 보안 채널 + SA 추가
ip macsec add macsec0 tx sa 0 pn 1 on \
  key 00 0123456789abcdef0123456789abcdef01234567

# RX 보안 채널 + SA 추가 (상대방 SCI 사용)
ip macsec add macsec0 rx sci 0xfedcba9876543210 \
  on
ip macsec add macsec0 rx sci 0xfedcba9876543210 \
  sa 0 pn 1 on \
  key 00 0123456789abcdef0123456789abcdef01234567

# 인터페이스 활성화
ip link set macsec0 up
ip addr add 192.168.10.1/24 dev macsec0

# 방법 2: wpa_supplicant + MKA (권장, 자동 키 협상)
# /etc/wpa_supplicant/macsec.conf
cat > /etc/wpa_supplicant/macsec.conf << 'EOF'
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0

network={
    key_mgmt=NONE
    macsec_policy=1
    macsec_integ_only=0      # 0=암호화, 1=인증만
    eapol_flags=0
    mka_cak=0123456789abcdef0123456789abcdef01234567  # 32B CAK
    mka_ckn=0123456789abcdef0123456789abcdef          # 16B CKN
    mka_priority=255         # 높을수록 Key Server 우선
}
EOF

wpa_supplicant -D macsec_linux -i eth0 \
  -c /etc/wpa_supplicant/macsec.conf -B

# 상태 확인
ip -d link show macsec0
# macsec0: <...> mtu 1468  ← MTU 32B 감소 (SecTAG 8B + ICV 16B + 여유)
ip macsec show
```

---

## DTLS (Datagram TLS) - UDP 기반 보안

DTLS(Datagram Transport Layer Security)는 UDP 통신에서 TLS와 동등한 보안을 제공하는 프로토콜입니다. TCP 기반 TLS를 UDP에 적용할 수 없는 이유는 UDP의 비연결성(순서 보장 없음, 재전송 없음) 때문입니다. DTLS는 이런 UDP 특성을 고려하여 설계되었습니다.

```
DTLS vs TLS 비교:

┌─────────────────────┬───────────────────────┬────────────────────┐
│ 특성                │ TLS 1.3               │ DTLS 1.3           │
├─────────────────────┼───────────────────────┼────────────────────┤
│ 기반 프로토콜        │ TCP                   │ UDP                │
│ 표준               │ RFC 8446              │ RFC 9147           │
│ 패킷 순서           │ TCP가 보장            │ DTLS 레코드 번호   │
│ 패킷 손실           │ TCP가 재전송          │ DTLS 재전송 타이머 │
│ 연결 지향           │ 연결 기반             │ 세션 기반          │
│ 지연               │ TCP 오버헤드          │ UDP 지연 (낮음)    │
│ 실시간 적합성        │ 낮음                  │ 높음               │
└─────────────────────┴───────────────────────┴────────────────────┘

의료 로봇 DTLS 적용 시나리오:
  DDS (ROS 2 실시간 통신): FastDDS + DTLS 1.3
  SRTP (암호화 영상 스트림): DTLS-SRTP
  DICOM 영상 전송: 일부 구현에서 UDP + DTLS
  OpenVPN/WireGuard: 의료망 VPN (UDP 기반)

DTLS 핸드셰이크 특이사항:
  ① Cookie Exchange: UDP 기반이므로 IP 스푸핑 방지
     Client ──── ClientHello ────────────────► Server
     Client ◄─── HelloVerifyRequest (Cookie) ── Server
     Client ──── ClientHello + Cookie ──────► Server
     (이후 일반 TLS 1.3과 동일)

  ② Record Sequence Number: 패킷 순서 추적
  ③ 재전송 타이머: 손실된 핸드셰이크 메시지 재전송
```

```python
# DTLS 1.3 서버 예시 (PyDTLS 라이브러리)
# pip install dtls

from dtls import do_patch
do_patch()  # UDP 소켓을 DTLS로 패치

import ssl
import socket

def create_dtls_server(host: str = '0.0.0.0', port: int = 5684):
    """
    CoAP over DTLS 서버 (IoT 의료 기기 통신)
    포트 5684: CoAPS (CoAP Secure, DTLS)
    """
    # 일반 UDP 소켓 생성
    udp_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    udp_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    udp_sock.bind((host, port))

    # DTLS 래핑 (SSL 컨텍스트 재사용)
    ctx = ssl.SSLContext(ssl.PROTOCOL_DTLS_SERVER)
    ctx.load_cert_chain(
        '/etc/ssl/robot/server.crt',
        '/etc/ssl/robot/server.key'
    )
    ctx.load_verify_locations('/etc/ssl/robot/ca.crt')
    ctx.verify_mode = ssl.CERT_REQUIRED  # 클라이언트 인증 (mDTLS)
    ctx.minimum_version = ssl.DTLSVersion.DTLSv1_3

    dtls_sock = ctx.wrap_socket(udp_sock, server_side=True)
    print(f"DTLS 서버 시작: {host}:{port}")

    while True:
        data, addr = dtls_sock.recvfrom(4096)
        print(f"[{addr}] 수신: {len(data)} bytes")
        # 에코 응답 (실제: CoAP 처리)
        dtls_sock.sendto(data, addr)


# ROS 2 DDS DTLS 설정 (FastDDS DTLS 플러그인)
# /etc/ros2/fastdds_security.xml
FASTDDS_DTLS_CONFIG = """
<dds>
  <profiles>
    <transport_descriptors>
      <transport_descriptor>
        <transport_id>dtls_udp</transport_id>
        <type>DTLS</type>
        <TLSConfig>
          <password></password>
          <options>
            <option>NO_SSLV2</option>
            <option>NO_SSLV3</option>
            <option>NO_TLSV1</option>
            <option>NO_TLSV1_1</option>
            <option>NO_TLSV1_2</option>
          </options>
          <verify_mode>
            <verify>VERIFY_PEER</verify>
          </verify_mode>
          <cert_chain_file>/etc/ssl/robot/robot_ecU.crt</cert_chain_file>
          <private_key_file>/etc/ssl/robot/robot_ecU.key</private_key_file>
          <ca_file>/etc/ssl/robot/hospital-ca.crt</ca_file>
        </TLSConfig>
      </transport_descriptor>
    </transport_descriptors>
  </profiles>
</dds>
"""

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

## 통신 보안 레이어 선택 가이드

```
의료 로봇 통신 보안 기술 선택 매트릭스:

┌───────────────────┬──────────┬──────────┬──────────┬──────────┐
│ 시나리오          │ TLS 1.3  │ mTLS     │ MACsec   │ DTLS 1.3 │
├───────────────────┼──────────┼──────────┼──────────┼──────────┤
│ 클라우드 API      │   ✓      │   ✓      │   -      │   -      │
│ OTA 업데이트 서버  │   ✓      │   ✓      │   -      │   -      │
│ 진단(DoIP) 서버   │   ✓      │   ✓      │   -      │   -      │
│ 병원 내부망 암호화 │   -      │   -      │   ✓      │   -      │
│ 스위치-ECU 구간   │   -      │   -      │   ✓      │   -      │
│ ROS2 DDS 실시간   │   -      │   -      │   -      │   ✓      │
│ SRTP 영상 스트림  │   -      │   -      │   -      │   ✓      │
│ 장치 인증         │   IDevID │   LDevID │   CAK    │   인증서  │
└───────────────────┴──────────┴──────────┴──────────┴──────────┘

권장 조합:
  외부 통신: mTLS (IEEE 802.1AR LDevID) + TLS 1.3
  내부망:    MACsec (스위치 포트 간) + SecOC (ECU 간 메시지 인증)
  실시간:    DTLS 1.3 (DDS) + MACsec (링크 레벨)
```

---

## 트러블슈팅

### 문제 1: TLS 핸드셰이크 실패

```
증상: "SSL: CERTIFICATE_VERIFY_FAILED" 또는
      "certificate verify failed: self-signed certificate"

원인 분석:
  1. CA 인증서 미설치 / 잘못된 경로
     확인: openssl verify -CAfile /path/to/ca.crt server.crt
     해결: CA 인증서를 클라이언트 신뢰 저장소에 추가

  2. 인증서 만료
     확인: openssl x509 -in cert.pem -noout -dates
     해결: EST로 갱신 또는 새 인증서 발급

  3. 도메인 불일치 (Subject Alternative Name)
     확인: openssl x509 -in cert.pem -noout -text | grep -A2 "Subject Alt"
     → CN이 아닌 SAN이 실제 검증에 사용됨
     해결: 올바른 SAN(IP 또는 DNS)으로 인증서 재발급

  4. TLS 버전 불일치
     확인: openssl s_client -connect host:port -tls1_3
     해결: 서버/클라이언트 모두 TLS 1.3 지원 확인

빠른 진단:
  # 서버 인증서 체인 전체 확인
  openssl s_client -connect 192.168.10.100:443 \
    -showcerts -CAfile /etc/ssl/robot/ca.crt

  # 출력에서 확인:
  # Verify return code: 0 (ok)  → 정상
  # Verify return code: 20 (unable to get local issuer certificate)
  #   → CA 인증서 누락
```

### 문제 2: mTLS 클라이언트 인증 실패

```
증상: 서버가 클라이언트 인증서를 거부
      "SSL_ERROR_HANDSHAKE_FAILURE_ALERT"

원인 분석:
  1. 클라이언트 인증서가 서버의 클라이언트 CA와 다른 CA로 발급됨
     확인: openssl verify -CAfile client-ca.crt robot.crt
     해결: 올바른 Device CA로 재발급

  2. 인증서 폐기 (CRL에 포함)
     확인: openssl crl -in current.crl -noout -text | grep Serial
     해결: 새 인증서 발급 (새 시리얼)

  3. 클라이언트 인증서에 올바른 Extended Key Usage 없음
     확인: openssl x509 -in robot.crt -noout -text | grep "Extended Key"
     → clientAuth 필요
     해결: EKU=clientAuth로 인증서 재발급

  4. 키-인증서 쌍 불일치
     확인: openssl rsa -in robot.key -pubout | md5sum
             openssl x509 -in robot.crt -pubkey -noout | md5sum
     → 두 해시가 동일해야 함
     해결: 올바른 키 파일 사용 또는 키 재생성 후 CSR 재발급
```

### 문제 3: MACsec 설정 후 통신 불가

```
증상: MACsec 인터페이스 생성 후 ping 불응답

원인 분석:
  1. 양쪽 키 불일치
     확인: ip macsec show (양쪽 노드에서)
     → TX SA의 key와 상대방 RX SA의 key 동일해야 함

  2. SCI 설정 오류
     → RX SCI가 상대방 TX SCI와 다름
     확인: ip -d link show macsec0 | grep sci

  3. MTU 초과
     → MACsec 오버헤드 32B로 MTU 1468
     확인: ip link show macsec0 | grep mtu
     해결: 상위 계층 MTU 조정 (이더넷 MTU 1500 → 1468로 제한)

  4. VLAN 설정 충돌
     → MACsec은 VLAN 태그 내부에 위치해야 함
     올바른 스택: eth0 → macsec0 → macsec0.VLAN

진단 명령:
  # MACsec 통계 확인 (드롭 카운터)
  ip -s macsec show macsec0
  # → rx_sa_not_in_use: 수신 SA 없음 → RX 설정 오류
  # → rx_pkt_no_sa_found: RX SA 키 불일치
  # → rx_pkt_late: PN 역전 (재전송 또는 재생 공격 차단)
```

### 문제 4: 인증서 만료 자동화 실패

```
증상: 인증서 만료로 서비스 중단, EST 자동 갱신 미동작

원인 분석:
  1. EST 서버 접속 불가 (네트워크 문제)
     확인: curl -k https://pki.hospital.com:8443/.well-known/est/cacerts
     해결: EST 서버 상태 확인, 네트워크 경로 확인

  2. 현재 인증서로 mTLS 연결 실패 (기존 인증서 이미 만료)
     → 만료된 인증서로는 EST 갱신 불가
     해결: IDevID 또는 임시 인증서로 초기 등록 절차 수행

  3. 갱신 타이머 미설정
     → cron 또는 systemd timer 미실행
     확인: systemctl status cert-renew.timer
     해결: 만료 30일 전 트리거 타이머 설정

예방 자동화:
  # /etc/systemd/system/cert-renew.timer
  [Unit]
  Description=Certificate Renewal Check (Daily)

  [Timer]
  OnCalendar=daily
  Persistent=true

  [Install]
  WantedBy=timers.target

  # /etc/systemd/system/cert-renew.service
  [Unit]
  Description=Renew Robot Device Certificate

  [Service]
  ExecStart=/usr/local/bin/check_and_renew_cert.sh
  Type=oneshot
```

---

## Reference
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5280 - X.509 Certificate Profile](https://datatracker.ietf.org/doc/html/rfc5280)
- [RFC 7030 - EST (Enrollment over Secure Transport)](https://datatracker.ietf.org/doc/html/rfc7030)
- [IEEE 802.1AR - Secure Device Identity](https://standards.ieee.org/ieee/802.1AR/6995/)
- [IEEE 802.1AE - MACsec](https://standards.ieee.org/ieee/802.1AE/7029/)
- [NIST SP 800-57 - Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
