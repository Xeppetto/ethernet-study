# Secure Boot

## 개요
Secure Boot(보안 부팅)은 시스템이 전원을 켜고 부팅되는 과정에서, 신뢰할 수 있는 소프트웨어(Bootloader, Kernel, Application)만 실행되도록 보장하는 기술입니다. 이는 해커가 악성 펌웨어를 주입하거나 시스템을 탈취하여 로봇의 제어권을 가져가는 것을 방지하는 가장 기본적인 보안 메커니즘입니다.

### 동작 원리
Secure Boot는 'Root of Trust(신뢰점)'에서 시작하여 부팅 단계마다 실행될 소프트웨어의 무결성을 검증하는 'Chain of Trust(신뢰 사슬)' 방식으로 동작합니다.

1.  **Hardware Root of Trust**: 프로세서(SoC) 제조 단계에서 변경 불가능한 ROM(Read Only Memory)에 저장된 공개 키(Public Key) 또는 해시(Hash) 값을 기반으로 검증을 시작합니다. (퓨즈(Fuse) 설정 등)
2.  **Bootloader Verification**: ROM 코드가 첫 번째 부트로더(First Stage Bootloader)의 전자 서명(Digital Signature)을 검증합니다. 서명이 유효하면 실행하고, 그렇지 않으면 부팅을 중단합니다.
3.  **Kernel Verification**: 부트로더가 OS 커널 이미지의 서명을 검증합니다.
4.  **Application Verification**: 커널이 파일 시스템이나 애플리케이션의 무결성을 검증합니다. (예: dm-verity)

### 주요 구성 요소
-   **Public Key Infrastructure (PKI)**: 펌웨어 이미지에 서명하기 위한 개인 키(Private Key)와 이를 검증하기 위한 공개 키(Public Key) 쌍을 관리합니다. 개인 키는 제조사의 보안 서버(HSM)에 안전하게 보관되어야 합니다.
-   **Hardware Security Module (HSM)**: 암호화 연산과 키 저장을 안전하게 수행하는 전용 하드웨어 모듈입니다. (TPM, TrustZone, SHE 등)

### 의료 로봇 분야의 중요성
수술 로봇과 같이 높은 신뢰성이 요구되는 시스템에서, 누군가 악의적으로 제어 소프트웨어를 변조한다면 환자에게 심각한 위험을 초래할 수 있습니다.
-   **무단 개조 방지**: 사용자가 임의로 로봇의 기능을 변경하거나 제한을 해제(Jailbreak)하는 것을 막을 수 있습니다.
-   **지적 재산권 보호**: 제조사의 소프트웨어 알고리즘이 유출되거나 복제되는 것을 방지합니다.
-   **안전 규제 준수**: FDA나 MDR과 같은 규제 기관에서는 의료 기기의 사이버 보안 요구사항으로 'Software Integrity'를 명시하고 있으며, Secure Boot는 이를 만족시키는 핵심 기술입니다.

## Reference
- [UEFI - Secure Boot](https://uefi.org/sites/default/files/resources/UEFI_Spec_2_8_A_Feb14.pdf)
- [NIST SP 800-193 - Platform Firmware Resiliency Guidelines](https://csrc.nist.gov/publications/detail/sp/800-193/final)
