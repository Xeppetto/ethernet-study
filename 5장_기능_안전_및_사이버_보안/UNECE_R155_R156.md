# UNECE R155 / R156 (Vehicle Cyber Security)

## 개요
UNECE(United Nations Economic Commission for Europe) R155와 R156은 자동차의 사이버 보안(Cyber Security)과 소프트웨어 업데이트(Software Update)에 대한 법적 규제(Regulation)입니다. 이는 단순한 권고 사항이 아니라, 유럽을 포함한 여러 국가에서 차량을 판매하기 위해 반드시 준수해야 하는 의무 사항입니다. 의료 기기 산업에서도 유사한 규제가 강화되고 있으므로, 이러한 선행 규제를 이해하는 것은 중요합니다.

### UNECE R155 (Cyber Security Management System, CSMS)
자동차 제조사가 사이버 보안을 체계적으로 관리하고 있음을 입증하도록 요구하는 규정입니다.
- **조직적 역량**: 제조사 내에 사이버 보안을 관리할 수 있는 조직과 프로세스가 갖추어져 있어야 합니다. (인력, 도구, 정책 등)
- **위험 관리**: 차량의 개발 단계뿐만 아니라 양산 이후의 운행 단계까지 지속적으로 사이버 위협을 모니터링하고 대응해야 합니다.
- **입증 요구**: 형식 승인(Type Approval) 시, TARA(Threat Analysis and Risk Assessment) 결과와 보안 조치가 적절함을 문서로 제출해야 합니다.

### UNECE R156 (Software Update Management System, SUMS)
차량 소프트웨어의 무선 업데이트(OTA) 과정에서 발생할 수 있는 안전 및 보안 문제를 예방하기 위한 규정입니다.
- **식별성**: 차량에 설치된 소프트웨어의 버전(RxSWIN: Regulation x Software Identification Number)을 명확하게 식별하고 관리해야 합니다.
- **안전성**: 업데이트 전, 중, 후에 차량의 기능 안전이 유지되도록 보장해야 합니다. (예: 주행 중 업데이트 금지, 업데이트 실패 시 롤백 기능)
- **사용자 고지**: 업데이트 내용과 절차를 사용자에게 명확하게 알리고 동의를 받아야 합니다.
- **보안성**: 업데이트 패키지의 무결성과 기밀성을 보호하고, 인증된 서버와 차량만 통신하도록 해야 합니다.

### 의료 로봇 분야의 시사점
의료 로봇 역시 FDA(미국 식품의약국)나 MDR(유럽 의료기기법)의 규제 하에 있습니다. 최근 FDA는 의료 기기의 사이버 보안 관리를 강화하는 가이드라인을 발표했으며, 이는 UNECE R155/R156의 내용과 유사한 방향으로 가고 있습니다.
- **Post-Market Surveillance**: 제품 판매 이후에도 지속적으로 취약점을 모니터링하고 패치를 제공해야 합니다. (SBOM: Software Bill of Materials 관리 필수)
- **Secure Update**: 로봇 펌웨어 업데이트 시 위변조를 방지하고, 업데이트 과정 중 환자의 안전을 위협하지 않도록 설계해야 합니다.

## Reference
- [UNECE R155 - Cyber security and cyber security management system](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-155-cyber-security-and-cyber-security)
- [UNECE R156 - Software update and software update management system](https://unece.org/transport/documents/2021/03/standards/un-regulation-no-156-software-update-and-software-update)
