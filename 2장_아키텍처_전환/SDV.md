# Software Defined Vehicle (SDV)

## 개요
SDV(Software Defined Vehicle)는 '소프트웨어가 정의하는 자동차'라는 뜻으로, 하드웨어(차체, 엔진 등)가 아닌 소프트웨어가 자동차의 핵심 기능과 가치를 결정한다는 개념입니다. 이는 스마트폰처럼, 하드웨어를 한 번 구매한 뒤에도 지속적인 소프트웨어 업데이트(OTA)를 통해 성능이 향상되고 새로운 기능이 추가될 수 있음을 의미합니다. 의료 로봇 분야에서도 'Software Defined Robot(SDR)'이라는 개념으로 확장되어 적용될 수 있습니다.

### SDV의 핵심 가치
- **OTA (Over-The-Air) 업데이트**: 로봇이나 차량을 서비스센터에 입고하지 않고도 원격으로 펌웨어를 업그레이드하여 버그를 수정하거나 보안 패치를 적용하고, 새로운 기능을 추가할 수 있습니다.
- **개인화된 경험**: 사용자의 운전 습관이나 선호도에 맞춰 차량 설정을 최적화하거나 구독형 서비스(FOD, Feature On Demand)를 제공할 수 있습니다.
- **데이터 기반 개선**: 실제 주행 데이터나 사용 데이터를 수집하여 클라우드에서 분석하고, 이를 바탕으로 AI 모델을 개선하여 다시 배포하는 선순환 구조를 만듭니다.

### SDV를 위한 기술 요소
SDV를 실현하기 위해서는 기존의 하드웨어 중심 아키텍처에서 탈피해야 합니다.
1.  **E/E 아키텍처 혁신**: Zonal Architecture와 중앙집중형 컴퓨팅(HPC)을 도입하여 하드웨어 복잡도를 줄이고 소프트웨어 유연성을 높여야 합니다.
2.  **가상화 및 컨테이너**: Hypervisor나 Container(Docker 등) 기술을 사용하여 하드웨어와 소프트웨어를 분리하고(Decoupling), 애플리케이션 이식성을 확보해야 합니다.
3.  **표준화된 미들웨어**: AUTOSAR Adaptive, ROS 2, DDS 등 표준화된 통신 미들웨어를 사용하여 서로 다른 벤더의 소프트웨어 모듈 간 호환성을 보장해야 합니다.
4.  **클라우드 네이티브 개발**: CI/CD(Continuous Integration/Continuous Deployment) 파이프라인을 구축하여 소프트웨어 개발, 테스트, 배포 과정을 자동화해야 합니다.

### 의료 로봇 분야의 SDR (Software Defined Robot)
의료 로봇 또한 고가의 장비이므로 한 번 도입하면 장기간 사용됩니다. SDV의 개념을 적용하면 다음과 같은 혁신이 가능합니다.
- **수술 기법 업데이트**: 새로운 수술 도구가 개발되거나 AI 보조 기능이 향상되면 소프트웨어 업데이트만으로 로봇에 적용할 수 있습니다.
- **원격 유지보수**: 로봇의 상태 데이터를 실시간으로 모니터링하여 고장을 예측하고(Predictive Maintenance), 원격으로 문제를 진단하거나 해결할 수 있습니다.
- **규제 대응**: 의료 규제가 변경되거나 새로운 보안 표준이 적용될 때, 신속하게 소프트웨어 패치를 배포하여 규제 준수 상태를 유지할 수 있습니다.

## Reference
- [Eclipse Foundation - Software Defined Vehicle](https://sdv.eclipse.org/)
- [SOAFEE - Scalable Open Architecture for Embedded Edge](https://soafee.io/)
