# 데이터 기반 비즈니스 모델 (Data-Driven Business Model)

## 개요
Ethernet과 클라우드 기술의 결합으로 의료 로봇이 수집하는 데이터의 양과 질이 비약적으로 향상되면서, 단순한 하드웨어 판매를 넘어 **데이터를 활용한 새로운 수익 모델**이 등장하고 있습니다. Intuitive Surgical(다빈치 로봇)의 경우 하드웨어 수익보다 소모품·서비스 수익이 더 크며, 이러한 **Recurring Revenue** 모델로의 전환이 의료 로봇 산업의 핵심 트렌드입니다.

---

## 비즈니스 모델 전환

```
전통적 모델 vs 데이터 기반 모델:

전통 (Capex):
  ┌────────────────────────────────────────┐
  │ 제조사 → 로봇 판매 → 병원              │
  │ 1회성 수익 ($1M~$2M/대)               │
  │ 유지보수 계약 (+$100K/년)             │
  └────────────────────────────────────────┘

데이터 기반 (Opex):
  ┌────────────────────────────────────────────────────┐
  │                                                    │
  │  ① RaaS (Robot as a Service)                      │
  │     수술 건수 × 단가 ($1,000~$3,000/건)           │
  │     또는 월 구독 ($20K~$50K/월)                   │
  │                                                    │
  │  ② 소모품 수익 (Instruments & Accessories)         │
  │     수술 기구 수명 관리 → 반복 구매                 │
  │     Intuitive: 소모품 수익 비중 35%+               │
  │                                                    │
  │  ③ 데이터 서비스 (Analytics Platform)             │
  │     수술 성과 분석, 외과의 교육                    │
  │     병원 운영 최적화 ($5K~$20K/월)                │
  │                                                    │
  │  ④ 데이터 판매 (익명화 후)                         │
  │     연구 기관, 제약사, 보험사                      │
  │     AI 학습 데이터셋                              │
  └────────────────────────────────────────────────────┘
```

---

## 데이터 수집 및 분석 파이프라인

```python
#!/usr/bin/env python3
"""
수술 로봇 데이터 수집 → 분석 → 비즈니스 인사이트 파이프라인
"""
import json
import time
import statistics
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime, timezone

@dataclass
class SurgerySession:
    """수술 세션 데이터 모델"""
    session_id: str
    robot_id: str
    surgeon_id: str         # 익명화된 ID
    surgery_type: str       # 예: "laparoscopic_cholecystectomy"
    start_time: datetime
    end_time: Optional[datetime] = None

    # 로봇 성능 데이터
    joint_movements: List[dict] = field(default_factory=list)
    force_data: List[dict] = field(default_factory=list)
    instrument_usage: Dict[str, int] = field(default_factory=dict)

    # 임상 결과 (익명화)
    surgery_duration_min: float = 0.0
    blood_loss_ml: Optional[float] = None
    complications: List[str] = field(default_factory=list)

    def duration_minutes(self) -> float:
        if self.end_time:
            return (self.end_time - self.start_time).seconds / 60
        return 0.0

    def to_analytics_payload(self) -> dict:
        """
        클라우드 전송용 분석 페이로드 (익명화 포함)
        PHI 제외, 성과 지표만 포함
        """
        return {
            'session_id': self.session_id,  # 익명 ID
            'robot_id': self.robot_id,
            'surgery_type': self.surgery_type,
            'timestamp': self.start_time.isoformat(),
            'duration_min': self.duration_minutes(),
            'metrics': {
                'total_joint_movement': sum(
                    j.get('distance_mm', 0)
                    for j in self.joint_movements
                ),
                'max_force_n': max(
                    (f.get('force_n', 0) for f in self.force_data),
                    default=0
                ),
                'instrument_count': len(self.instrument_usage),
                'complications': len(self.complications),
            }
        }


class SurgicalPerformanceAnalytics:
    """
    수술 성과 분석 엔진
    - 외과의 성과 추적
    - 로봇 성능 모니터링
    - 예측 유지보수
    """

    def __init__(self):
        self.sessions: List[SurgerySession] = []

    def add_session(self, session: SurgerySession):
        self.sessions.append(session)

    def surgeon_performance_report(self, surgeon_id: str) -> dict:
        """외과의 개인 성과 보고서"""
        surgeon_sessions = [
            s for s in self.sessions
            if s.surgeon_id == surgeon_id
        ]

        if not surgeon_sessions:
            return {}

        durations = [s.duration_minutes() for s in surgeon_sessions]
        complications = [len(s.complications) for s in surgeon_sessions]

        return {
            'surgeon_id': surgeon_id,
            'total_surgeries': len(surgeon_sessions),
            'avg_duration_min': statistics.mean(durations),
            'min_duration_min': min(durations),
            'complication_rate': sum(complications) / len(complications),
            'trend': 'improving' if durations[-1] < durations[0] else 'stable',
            # → 학습 곡선 분석: 수술 건수 증가에 따른 시간 단축
        }

    def predictive_maintenance(self, robot_id: str) -> dict:
        """예측 유지보수 - 기구 수명 분석"""
        robot_sessions = [
            s for s in self.sessions
            if s.robot_id == robot_id
        ]

        # 기구별 사용 횟수 집계
        total_usage: Dict[str, int] = {}
        for session in robot_sessions:
            for instrument, count in session.instrument_usage.items():
                total_usage[instrument] = total_usage.get(instrument, 0) + count

        # 수명 임계치 (예: 각 기구당 100회 사용)
        LIFETIME_THRESHOLD = 100
        alerts = []
        for instrument, count in total_usage.items():
            remaining = LIFETIME_THRESHOLD - count
            if remaining <= 20:  # 20회 미만이면 경고
                alerts.append({
                    'instrument': instrument,
                    'usage_count': count,
                    'remaining_uses': max(0, remaining),
                    'urgency': 'CRITICAL' if remaining <= 5 else 'WARNING'
                })

        return {
            'robot_id': robot_id,
            'instrument_usage': total_usage,
            'maintenance_alerts': alerts,
            'next_service_date': self._estimate_service_date(robot_sessions),
        }

    def _estimate_service_date(self, sessions: list) -> str:
        """다음 정기 점검일 추정"""
        total_ops = sum(s.duration_minutes() for s in sessions)
        # 예시: 200시간마다 점검
        SERVICE_INTERVAL_H = 200
        remaining_h = SERVICE_INTERVAL_H - (total_ops / 60)
        next_date = datetime.now()
        import datetime as dt
        return (next_date + dt.timedelta(hours=remaining_h)).strftime('%Y-%m-%d')
```

---

## RaaS (Robot as a Service) 가격 모델

```
RaaS 수익 모델 분석:

구독 모델:
  기본형: $20,000/월
    - 소프트웨어 업데이트 포함
    - 24시간 원격 지원
    - 기본 분석 대시보드

  프리미엄: $40,000/월
    - 고급 AI 분석
    - 외과의 교육 플랫폼
    - 개인화 수술 계획

사용량 기반 (Per-Use):
  - 복강경 수술: $1,000~$1,500/건
  - 관절 치환술: $2,000~$3,000/건
  - 신경외과 수술: $3,000~$5,000/건

소모품 수익 (Razor-Razorblade):
  - 수술 기구 세트: $200~$500/건
  - 전용 드레이프: $50~$100/건
  - 연간 소모품: $500K~$1M/대

비교: Intuitive Surgical (2023)
  기기 판매: $2.4B (35%)
  소모품/서비스: $4.5B (65%)
  → 데이터 기반 모델이 주 수익원
```

---

## 데이터 모네타이제이션 (Data Monetization)

```
의료 로봇 데이터 판매 전략:

1. 임상 연구 지원
   대상: 대학병원, 제약회사, 의료기기 연구소
   데이터: 익명화 수술 영상, 성과 데이터
   가격: $50K~$500K/데이터셋
   조건: IRB 승인, HIPAA De-identification

2. AI 학습 데이터셋
   대상: AI 스타트업, 빅테크 의료 AI
   데이터: 레이블된 수술 영상, 로봇 동작 데이터
   가격: 건당 $10~$100 (전문 레이블링 비용)

3. 보험사 협력
   대상: 의료보험사
   데이터: 수술 성공률, 재입원률 상관관계
   모델: 위험도 기반 보험료 산정 지원

4. 병원 벤치마킹 서비스
   대상: 병원 경영진
   서비스: 유사 병원 대비 성과 비교
   형태: SaaS 구독 ($5K~$20K/월)

데이터 거버넌스 요구사항:
  ① IRB (기관심사위원회) 승인
  ② HIPAA/GDPR 준수
  ③ 환자 동의서 (Informed Consent)
  ④ 데이터 익명화 (ISO 29101)
  ⑤ 계약서 (DPA: Data Processing Agreement)
```

---

## AI/ML 서비스화

```python
# 클라우드 AI 모델로 수술 지원
# 수술 계획 최적화 API 예시

import requests
import json

class SurgicalAIService:
    """
    클라우드 AI 수술 지원 서비스 클라이언트
    - CT/MRI 기반 수술 경로 계획
    - 실시간 이상 감지
    - 수술 성과 예측
    """

    API_BASE = "https://ai.surgical-platform.com/api/v1"
    API_KEY = "your-api-key"

    def plan_trajectory(self, ct_data_url: str,
                        surgery_type: str) -> dict:
        """
        CT 데이터 기반 로봇 수술 경로 계획
        Returns: 관절 궤적 (Joint trajectory)
        """
        response = requests.post(
            f"{self.API_BASE}/trajectory/plan",
            headers={"Authorization": f"Bearer {self.API_KEY}"},
            json={
                "ct_data_url": ct_data_url,
                "surgery_type": surgery_type,
                "robot_model": "surgical_v2",
                "constraints": {
                    "max_force_n": 10.0,
                    "max_speed_mm_s": 100.0,
                    "safe_zones": []
                }
            },
            timeout=30
        )
        return response.json()

    def detect_anomaly(self, sensor_data: list) -> dict:
        """
        실시간 이상 감지 (수술 중 위험 신호)
        - 과도한 힘 감지
        - 비정상 진동 패턴
        - 조직 저항 변화
        """
        response = requests.post(
            f"{self.API_BASE}/anomaly/detect",
            headers={"Authorization": f"Bearer {self.API_KEY}"},
            json={"sensor_data": sensor_data[-100:]},  # 최근 100개
            timeout=5
        )
        return response.json()
        # 반환: {'anomaly': True/False, 'confidence': 0.95,
        #         'type': 'excessive_force', 'recommendation': '속도 감소'}
```

---

## Reference
- [Intuitive Surgical - Annual Report 2023](https://investor.intuitivesurgical.com/financial-information/annual-reports)
- [HBR - How Smart, Connected Products Are Transforming Competition](https://hbr.org/2014/11/how-smart-connected-products-are-transforming-competition)
- [FDA - Digital Health Center of Excellence](https://www.fda.gov/medical-devices/digital-health-center-excellence)
- [McKinsey - The data-driven enterprise of 2025](https://www.mckinsey.com/capabilities/quantumblack/our-insights/the-data-driven-enterprise-of-2025)
- [HIPAA De-identification Methods](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/)
