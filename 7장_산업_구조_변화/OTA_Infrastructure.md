# OTA 인프라 (Over-The-Air Infrastructure)

## 개요
OTA(Over-The-Air)는 무선 통신을 통해 소프트웨어, 펌웨어, 설정값 등을 업데이트하는 기술입니다. 스마트폰의 OS 업데이트처럼, 의료 로봇 시스템도 지속적으로 기능을 개선하고, 버그를 수정하며, 새로운 보안 위협에 대응하기 위해 OTA 인프라 구축이 필수적입니다. 이는 단순히 파일을 다운로드하는 것을 넘어, 업데이트의 전체 생명주기(Lifecycle)를 안전하고 효율적으로 관리하는 것을 의미합니다.

### OTA의 유형
-   **SOTA (Software Over-The-Air)**: 인포테인먼트 시스템, 내비게이션, 사용자 인터페이스(UI) 등 애플리케이션 레벨의 소프트웨어 업데이트를 의미합니다.
-   **FOTA (Firmware Over-The-Air)**: ECU(Electronic Control Unit)의 제어 펌웨어, 모터 드라이버, 센서 로직 등 하위 레벨의 펌웨어를 업데이트합니다. 리부팅이 필요하거나, 안전 기능(Safe State) 확보가 중요합니다.

### OTA 인프라의 구성 요소
1.  **Repository Server**: 펌웨어 패키지(바이너리, 메타데이터, 서명 등)를 저장하고 관리하는 중앙 서버입니다. (Cloud Storage)
2.  **Campaign Management**: 어떤 로봇들에게, 언제, 어떤 버전의 업데이트를 배포할지 계획하고 실행하는 관리 도구입니다. (Targeting, Scheduling)
3.  **Delivery Network (CDN)**: 전 세계에 분산된 로봇들에게 빠르고 안정적으로 업데이트 파일을 전송하기 위한 콘텐츠 전송 네트워크입니다.
4.  **Client Agent**: 로봇 내부에서 서버와 통신하며 업데이트 패키지를 다운로드하고, 검증(Verification)하고, 설치(Installation)하는 소프트웨어 모듈입니다. (Update Client)
5.  **Security Module**: 업데이트 파일의 무결성과 기밀성을 보장하기 위한 암호화 키 관리(KMS), 서명 검증(Secure Boot 연동) 기능을 제공합니다.

### 의료 로봇 분야 적용 시 고려사항
의료 로봇은 사람의 생명을 다루는 기기이므로, OTA 수행 시 안전성 확보가 최우선입니다.
-   **Safety Check**: 로봇이 수술 중이거나, 전원이 불안정하거나, 안전하지 않은 상태일 때는 업데이트를 수행하지 않도록 엄격한 조건 검사(Pre-condition Check)가 필요합니다.
-   **A/B Partitioning**: 펌웨어를 저장할 공간을 두 개(Active/Inactive)로 나누어, 업데이트 중 오류가 발생하더라도 이전 버전으로 즉시 복구(Rollback)할 수 있는 이중화 구조를 갖춰야 합니다.
-   **Delta Update**: 전체 펌웨어를 다운로드하는 대신, 변경된 부분(Delta)만 전송하여 업데이트 시간과 데이터 사용량을 줄여야 합니다. (BSDIFF 등)
-   **Uptane Framework**: 자동차 보안 OTA 표준인 Uptane 프레임워크를 적용하여, 중간자 공격이나 키 탈취 공격으로부터 보호해야 합니다.

## Reference
- [Uptane - The Standard for Secure Software Updates for Automobiles](https://uptane.github.io/)
- [Red Hat - OTA Updates for IoT](https://www.redhat.com/en/topics/edge-computing/what-is-ota)
