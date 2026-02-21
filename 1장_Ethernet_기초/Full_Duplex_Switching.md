# Full Duplex Switching (전이중 스위칭)

## 개요
초기 Ethernet은 Half Duplex(반이중) 방식으로 충돌(Collision)이 빈번했고, CSMA/CD 알고리즘으로 이를 해결했습니다. 현대 Ethernet은 Full Duplex + Switching 기반으로 충돌 자체가 발생하지 않으며, 결정론적 지연(Deterministic Latency)을 위한 TSN 스케줄링까지 발전했습니다.

이 문서에서는 Full Duplex 통신 원리, 스위칭 방식, QoS, 그리고 실무에서 자주 접하는 설정/확인 방법을 다룹니다.

---

## Half Duplex vs Full Duplex

```
Half Duplex (반이중)
───────────────────
노드 A ────송신────►
           ◄────수신──── 노드 B
※ 동시 불가. 충돌 발생 시 CSMA/CD로 재전송

Full Duplex (전이중)
────────────────────
노드 A ────TX────► 노드 B
노드 A ◄───RX───── 노드 B
※ 물리적으로 송신/수신 쌍 분리 → 충돌 없음
```

| 특성 | Half Duplex | Full Duplex |
|------|-------------|-------------|
| 동시 송수신 | 불가 | 가능 |
| 충돌(Collision) | 발생 가능 (CSMA/CD 필요) | 없음 |
| 실효 대역폭 | 이론의 50% 미만 | 거의 100% 활용 |
| 지연(Latency) | 불규칙 (충돌 재전송) | 안정적 |
| 허브 연결 | Half duplex 동작 | Full duplex 불가 |
| 스위치 연결 | Full duplex 가능 | Full duplex 사용 |

> **현실**: 1000Mbps 이상 Gigabit Ethernet은 스위치 환경에서만 사용되므로 사실상 항상 Full Duplex입니다.

---

## Ethernet 스위치 동작 원리

### MAC 주소 학습 (MAC Address Learning)
```
초기 상태: MAC 테이블 비어있음

Step 1: 포트 1에서 프레임 수신 (Src: AA:BB, Dst: CC:DD)
  → MAC 테이블에 AA:BB = 포트1 학습

Step 2: CC:DD를 모름 → Flooding (알 수 없는 목적지)
  → 포트 1 제외 모든 포트로 전송

Step 3: CC:DD 장치가 응답하면 포트 3에서 프레임 수신
  → MAC 테이블에 CC:DD = 포트3 학습

이후: AA:BB ↔ CC:DD 통신 시 포트 1 ↔ 포트 3으로만 전달
```

### MAC 테이블 확인 (Linux 브릿지)
```bash
# Linux bridge 사용 시
bridge fdb show

# 출력 예시
aa:bb:cc:dd:ee:01 dev eth0 vlan 10 master br0
aa:bb:cc:dd:ee:02 dev eth1 vlan 10 master br0
```

---

## 스위칭 방식 비교

### Store-and-Forward
```
프레임 수신 ──► 전체 버퍼 저장 ──► CRC 검사 ──► 오류 없으면 전달
                                    └─► 오류 있으면 폐기

지연 = 전체 프레임 수신 시간 (= 프레임 크기 / 링크 속도)
예: 1518 Byte 프레임, 1Gbps 링크
  → 1518 × 8 bit / 1,000,000,000 bps ≈ 12.1 µs 수신 지연 추가
```

### Cut-Through
```
프레임 수신 ──► 목적지 MAC(6Byte) 확인 ──► 즉시 전달 시작
                                 ↑
                     최소 지연: 6~8 Byte 수신 후 전달 시작
                     1Gbps 기준: ~64ns

장점: 매우 낮은 전달 지연 (Store-and-Forward 대비 수십 µs 절약)
단점: 오류 프레임도 전달될 수 있음 (Runt frame, CRC 오류 포함)
```

### Fragment-Free (Modified Cut-Through)
- 최소 64Byte 수신 후 전달 (충돌 감지 완료 후)
- Cut-Through보다 약간 높은 지연, 최소 프레임 오류 방지

**TSN 환경 권장**: Store-and-Forward + TAS(802.1Qbv) 조합으로 결정론적 지연 보장

---

## Auto-Negotiation (자동 협상)

Ethernet 연결 시 속도(10/100/1000Mbps)와 이중화(Half/Full Duplex)를 자동으로 맞추는 기능입니다.

```bash
# ethtool로 현재 협상 상태 확인
ethtool eth0

# 출력 예시
Settings for eth0:
    Speed: 1000Mb/s
    Duplex: Full
    Auto-negotiation: on
    Link detected: yes
```

### Auto-Negotiation 실패 시 흔한 문제
```
한 쪽: Auto-negotiation ON → 1000Mbps Full Duplex
반대쪽: Auto-negotiation OFF, 강제 100Mbps Full Duplex

결과: Duplex Mismatch!
→ 한 쪽은 Half duplex로 인식 → CSMA/CD 동작 시도
→ 충돌 카운터 급증 (ethtool -S eth0 | grep collision)
→ 성능 크게 저하 (이론 대역폭의 10% 미만)
```

```bash
# 강제 설정 (Auto-negotiation 끄기, 비권장)
ethtool -s eth0 speed 1000 duplex full autoneg off

# Auto-negotiation 통계 확인
ethtool -S eth0 | grep -E "collision|error|drop"
```

---

## QoS와 우선순위 큐 (Priority Queues)

Full Duplex 스위칭 환경에서도 트래픽이 몰리면 큐잉 지연이 발생합니다. QoS로 중요 트래픽을 우선 처리합니다.

### IEEE 802.1p 우선순위 (CoS: Class of Service)
Ethernet 프레임 내 VLAN Tag의 PCP(Priority Code Point) 3비트를 활용합니다.

| PCP 값 | 우선순위 | 명칭 | 사용 사례 |
|--------|---------|------|-----------|
| 7 | 최고 | Network Control | BPDU, PTP |
| 6 | 높음 | Internetwork Control | - |
| 5 | 높음 | Critical | TSN Scheduled Traffic |
| 4 | 중간 | Video | 영상 스트리밍 |
| 3 | 중간 | Voice | 음성 |
| 2 | 낮음 | Excellent Effort | - |
| 1 | 매우 낮음 | Background | 배경 작업 |
| 0 | 기본 | Best Effort | 일반 트래픽 |

### Linux에서 QoS 설정
```bash
# 트래픽 클래스 확인
tc qdisc show dev eth0

# 기본 pfifo_fast 설정 확인 (3개 밴드)
tc qdisc show dev eth0
# qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap 1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```

---

## 스위치 포트 상태 진단

### Linux 명령어를 이용한 포트 통계
```bash
# 인터페이스 통계 확인
ip -s link show eth0

# 상세 에러 통계 (드라이버 제공)
ethtool -S eth0

# 실시간 트래픽 모니터링
watch -n 1 "cat /proc/net/dev | grep eth0"

# 주요 확인 항목
ethtool -S eth0 | grep -E "rx_errors|tx_errors|rx_dropped|tx_dropped|collisions"
```

### 흔한 스위치 포트 문제
| 증상 | 원인 | 조치 |
|------|------|------|
| rx_errors 증가 | CRC 오류 (물리적 케이블 문제) | 케이블/커넥터 교체 |
| tx_dropped 증가 | 출력 큐 포화 (혼잡) | QoS 설정 또는 대역폭 업그레이드 |
| collisions 발생 | Duplex Mismatch | Auto-Negotiation 설정 통일 |
| rx_missed_errors | NIC 버퍼 오버플로우 | RX 링 버퍼 크기 증가 (`ethtool -G`) |

---

## 의료/산업 로봇 네트워크 설계 권장 사항

```
제어 ECU ──► TSN 스위치 (Cut-Through + 802.1Qbv) ──► 관절 ECU
              │
              ├── VLAN 10 (제어: PCP=5, ST 트래픽)  → 지연 < 100µs
              ├── VLAN 20 (센서 영상: PCP=4)         → 지연 < 5ms
              └── VLAN 30 (진단/관리: PCP=0)         → Best Effort

권장 설정:
• 모든 링크: 1Gbps Full Duplex, Auto-Negotiation ON
• 스위치: Store-and-Forward + TAS(802.1Qbv) 활성화
• PTP (IEEE 802.1AS): 마이크로초 수준 시간 동기화
• RSTP/MSTP: 루프 방지 (단, TSN 환경에서는 토폴로지 사전 고정 권장)
```

---

## Reference
- [IEEE 802.3-2022 - Ethernet Standard (Full Duplex)](https://standards.ieee.org/ieee/802.3/10422/)
- [IEEE 802.1p - Traffic Class Expediting](https://standards.ieee.org/ieee/802.1Q/6844/)
- [Linux ethtool man page](https://man7.org/linux/man-pages/man8/ethtool.8.html)
- [Linux tc-pfifo_fast](https://man7.org/linux/man-pages/man8/tc-pfifo_fast.8.html)
