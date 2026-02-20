# ISO/SAE 21434 (Cybersecurity)

## 개요
ISO 26262가 '기능 안전'을 다루는 표준이라면, ISO/SAE 21434는 '사이버 보안'을 다루는 국제 표준입니다. 연결된(Connected) 시스템이 늘어나면서 해킹 위협이 증가하고, 이로 인해 기능 안전까지 위협받는 상황이 발생하자 이를 체계적으로 관리하기 위해 제정되었습니다. 의료 로봇 역시 인터넷망에 연결되거나 원격 제어 기능을 포함하면서 사이버 보안의 중요성이 커지고 있습니다.

### 핵심 개념
ISO/SAE 21434는 제품 수명 주기(Lifecycle) 전반에 걸쳐 사이버 보안 위험을 식별하고 대응하는 프로세스를 정의합니다.

1.  **TARA (Threat Analysis and Risk Assessment)**:
    -   **Asset Identification**: 보호해야 할 자산(ECU, 데이터, 통신 채널 등)을 식별합니다.
    -   **Threat Modeling**: 발생 가능한 위협 시나리오(예: CAN 메시지 스푸핑, 펌웨어 변조 등)를 도출합니다.
    -   **Impact Rating**: 위협이 실현되었을 때의 피해 규모(안전, 재정, 운영, 개인정보)를 평가합니다.
    -   **Attack Feasibility Rating**: 공격에 필요한 시간, 전문성, 장비 등을 고려하여 공격 가능성을 평가합니다.
    -   **Risk Treatment Decision**: 위험 수준(Risk Level)을 산출하고, 이를 수용할지(Accept), 회피할지(Avoid), 완화할지(Mitigate), 전가할지(Transfer) 결정합니다.

2.  **Cybersecurity Goal**: TARA 결과를 바탕으로 달성해야 할 보안 목표를 수립합니다. (예: "제동 제어 메시지의 기밀성과 무결성을 보장한다.")

3.  **V-Model**: 시스템 요구사항 정의부터 설계, 구현, 검증에 이르는 V 모델 프로세스에 보안 활동을 통합합니다.

### 의료 로봇 분야의 적용
수술 로봇이나 환자 데이터 처리 시스템이 해킹당하면 환자의 생명이 위협받거나 민감한 의료 정보가 유출될 수 있습니다. ISO/SAE 21434를 적용하여 다음과 같은 보안 대책을 수립할 수 있습니다.

-   **Secure Communication**: 로봇 내부의 제어기와 외부 서버 간의 통신을 암호화(Encryption)하고 인증(Authentication)하여 도청과 변조를 방지합니다. (예: TLS, MACsec)
-   **Intrusion Detection System (IDS)**: 네트워크 트래픽을 실시간으로 감시하여 이상 징후(Anomaly)를 탐지하고 대응합니다.
-   **Vulnerability Management**: 소프트웨어의 취약점(CVE)을 지속적으로 모니터링하고 패치합니다.
-   **Supply Chain Security**: 로봇에 사용되는 타사 부품이나 소프트웨어(SOUP: Software of Unknown Pedigree)의 보안성을 검증하고 관리합니다.

## Reference
- [ISO/SAE 21434:2021 - Road vehicles — Cybersecurity engineering](https://www.iso.org/standard/70918.html)
- [NIST - Cybersecurity Framework](https://www.nist.gov/cyberframework)
