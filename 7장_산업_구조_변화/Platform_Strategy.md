# 플랫폼화 전략 (Platform Strategy)

## 개요
제조업의 패러다임이 '개별 제품 중심'에서 '공통 플랫폼 중심'으로 변화하고 있습니다. 플랫폼화 전략(Platform Strategy)은 다양한 제품 라인업을 하나의 기본 구조(플랫폼) 위에서 개발하여, 비용을 절감하고 개발 기간을 단축하며 품질을 확보하는 핵심 전략입니다. 자동차 산업(예: VW MEB, Hyundai E-GMP)과 마찬가지로, 의료 로봇 분야에서도 공통의 하드웨어/소프트웨어 플랫폼을 구축하고 이를 기반으로 여러 파생 로봇을 개발하는 방식이 대세가 되고 있습니다.

### 플랫폼화의 이점
-   **규모의 경제**: 공통 부품(모듈)을 대량 생산하여 단가를 낮출 수 있습니다.
-   **개발 기간 단축 (Time-to-Market)**: 이미 검증된 플랫폼 위에서 차체(Shell)와 애플리케이션만 변경하여 신제품을 빠르게 출시할 수 있습니다.
-   **품질 안정화**: 한 번 검증된 플랫폼의 안정성을 계속 활용할 수 있어, 신제품 개발 시 발생할 수 있는 초기 결함(Initial Quality Defect)을 최소화할 수 있습니다.
-   **유지보수 효율**: 부품 호환성이 높아져 재고 관리 및 수리가 용이합니다.

### 하드웨어 플랫폼
-   **공통 섀시(Chassis)**: 이동형 로봇(AMR)의 경우, 주행 모듈(바퀴, 모터, 배터리)을 공통화하고 상단에 어떤 장치(로봇 팔, 디스플레이, 적재함)를 올리느냐에 따라 용도를 변경합니다.
-   **모듈형 관절**: 수술 로봇 팔의 관절 모듈(액추에이터 + 감속기 + 센서 + 제어기)을 표준화하여, 자유도(Degree of Freedom)나 작업 반경에 따라 조립식으로 구성합니다.

### 소프트웨어 플랫폼
-   **표준화된 OS 및 미들웨어**: ROS 2, DDS, Linux/RTOS 등 표준 소프트웨어 스택을 모든 로봇 제품군에 공통으로 적용합니다.
-   **SDK 및 API**: 외부 개발자나 의료진이 로봇의 기능을 쉽게 확장하거나 새로운 애플리케이션을 개발할 수 있도록 개방형 인터페이스를 제공합니다. (예: Intuitive da Vinci Research Kit)
-   **OTA (Over-The-Air)**: 모든 로봇에 동일한 OTA 시스템을 적용하여 펌웨어 관리 및 업데이트를 일원화합니다.

### 의료 로봇 분야의 적용 사례
-   **Intuitive Surgical**: 다빈치(da Vinci) 수술 로봇 시리즈는 공통된 제어 콘솔과 비전 시스템 기술을 기반으로 다양한 모델(Si, Xi, SP 등)을 출시했습니다.
-   **Medtronic**: Hugo RAS 시스템은 모듈형 설계를 채택하여 수술실 환경에 맞춰 유연하게 배치할 수 있습니다.
-   **Stryker**: Mako 로봇 팔 기술을 기반으로 무릎 관절, 엉덩이 관절 등 다양한 정형외과 수술 애플리케이션을 확장하고 있습니다.

## Reference
- [Harvard Business Review - The Age of Platforms](https://hbr.org/2016/04/pipelines-platforms-and-the-new-rules-of-strategy)
- [Deloitte - The future of platforms](https://www2.deloitte.com/content/dam/Deloitte/nl/Documents/technology-media-telecommunications/deloitte-nl-tmt-the-future-of-platforms.pdf)
