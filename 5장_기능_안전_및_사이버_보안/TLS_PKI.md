# TLS 및 PKI 인증서 관리

## 개요
Ethernet 네트워크를 사용하는 의료 로봇은 수많은 데이터를 주고받습니다. 이 데이터가 외부로 유출되거나 변조되는 것을 막기 위해 전송 계층 보안(Transport Layer Security, TLS)을 적용합니다. TLS는 인터넷 뱅킹이나 HTTPS 웹사이트에서 사용되는 것과 동일한 강력한 암호화 기술이며, 이를 안전하게 운영하기 위해서는 공개 키 기반 구조(Public Key Infrastructure, PKI)와 인증서(Certificate) 관리가 필수적입니다.

### TLS (Transport Layer Security)
TLS는 TCP/IP 통신의 기밀성(Confidentiality), 무결성(Integrity), 인증(Authentication)을 제공하는 프로토콜입니다.
1.  **Handshake**: 클라이언트와 서버가 서로의 신원을 확인하고(인증서 교환), 암호화 통신에 사용할 대칭 키(Session Key)를 생성하는 과정입니다.
2.  **Encryption**: 생성된 세션 키를 사용하여 데이터를 암호화하여 전송합니다. (AES, ChaCha20 등)
3.  **Integrity Check**: 메시지 인증 코드(HMAC)를 사용하여 데이터가 전송 중에 변경되지 않았는지 확인합니다.

### PKI (Public Key Infrastructure)
PKI는 디지털 인증서를 생성, 관리, 배포, 사용, 저장, 폐기하는 일련의 하드웨어, 소프트웨어, 정책, 절차의 집합입니다.
-   **CA (Certificate Authority)**: 신뢰할 수 있는 최상위 인증 기관으로, 하위 인증서에 전자 서명을 하여 신뢰성을 부여합니다. (Root CA -> Intermediate CA -> Entity Certificate)
-   **Certificate**: 사용자의 공개 키(Public Key)와 신원 정보(Subject Name)가 포함된 디지털 문서입니다. (X.509 표준)
-   **CRL (Certificate Revocation List)** / **OCSP (Online Certificate Status Protocol)**: 유효하지 않거나 폐기된 인증서 목록을 관리하여, 보안 사고 발생 시 즉시 대응할 수 있도록 합니다.

### 의료 로봇 분야 적용
의료 로봇은 병원 내부망(Intranet)뿐만 아니라 클라우드 서버와도 통신하며, 때로는 원격 수술을 위해 인터넷망을 사용하기도 합니다.
-   **Mutual TLS (mTLS)**: 서버만 클라이언트를 인증하는 것이 아니라, 클라이언트(로봇)도 서버가 진짜인지 인증하는 양방향 인증 방식을 사용하여, 중간자 공격(MITM)을 원천 차단해야 합니다.
-   **Device Identity**: 각 로봇 기기마다 고유한 인증서를 발급하여(Production Certificate), 허가되지 않은 기기가 네트워크에 접속하는 것을 막습니다. (IEEE 802.1AR Secure Device Identity)
-   **Certificate Lifecycle Management**: 인증서에는 유효 기간이 있으므로, 만료되기 전에 자동으로 갱신(Renewal)하거나 교체(Rotation)하는 시스템을 구축해야 합니다. (EST: Enrollment over Secure Transport, ACME 프로토콜 등)

## Reference
- [RFC 8446 - The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 5280 - Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://datatracker.ietf.org/doc/html/rfc5280)
