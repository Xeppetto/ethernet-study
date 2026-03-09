# Wireshark PCAP 분석

## 개요
Wireshark는 세계 표준 네트워크 패킷 분석 도구(오픈소스, GPL)로, 30년 이상의 역사를 가진 업계 표준입니다. 의료 로봇의 Ethernet 통신 디버깅, TSN 동작 검증, 프로토콜 적합성 테스트에 필수적입니다. PCAP(Packet Capture) 파일 형식으로 저장하여 사후 분석(Post-mortem Analysis)도 가능하므로, 수술 중 발생한 네트워크 이상을 사후에 재현하고 원인을 규명하는 데도 활용됩니다.

Wireshark의 핵심 가치는 **눈에 보이지 않는 네트워크 계층의 동작을 가시화**하는 데 있습니다. 개발자가 작성한 코드가 실제 어떤 패킷을 생성하는지, TSN 스케줄러가 올바른 순서로 트래픽을 전송하는지, PTP 동기화가 실제로 수렴하는지를 모두 Wireshark 하나로 검증할 수 있습니다.

> **의료 로봇 관점**: 수술 로봇 시스템에서 Wireshark는 세 가지 중요한 역할을 합니다. ① **개발 단계**: SOME/IP 서비스 발견이 올바르게 동작하는지, E2E 헤더가 정상 삽입되는지 확인. ② **검증 단계**: TSN GCL 스케줄에 따라 트래픽이 정해진 슬롯에 전송되는지, PTP 동기화 정밀도가 < 1µs인지 측정. ③ **사고 조사**: 수술 중 통신 이상 발생 시 캡처된 PCAP로 원인 재현. IEC 62304에서는 이 PCAP 증거가 소프트웨어 검증 문서의 일부가 됩니다.

---

## Wireshark 설치 및 권한 설정

```bash
# Ubuntu/Debian 설치
sudo apt-get install wireshark tshark

# 설치 중 "Non-superusers..." 질문: Yes 선택
# → wireshark 그룹이 생성되어 비root 캡처 가능

# 비root 사용자 패킷 캡처 권한 추가
sudo usermod -aG wireshark $USER
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
# → 재로그인 후 적용 (newgrp wireshark로 즉시 적용 가능)

# 권한 확인
groups $USER      # wireshark 포함 여부 확인
getcap /usr/bin/dumpcap
# /usr/bin/dumpcap = cap_net_admin,cap_net_raw+eip

# 원격 서버에서 캡처 → 로컬 Wireshark로 실시간 분석
ssh robot@192.168.1.100 "tcpdump -i eth0 -w - -U" | wireshark -k -i -
# -w - : 표준출력으로 PCAP 스트림
# -U   : 패킷 단위 플러시 (버퍼링 없이)
# -k   : Wireshark 즉시 캡처 시작
# -i - : 표준입력을 인터페이스로 사용

# 하드웨어 타임스탬핑 활성화 (정밀 측정 필수)
ethtool -T eth0
# Capabilities:
#   hardware-transmit     ← TX 하드웨어 타임스탬핑 지원
#   hardware-receive      ← RX 하드웨어 타임스탬핑 지원
#   hardware-raw-clock    ← PHC 기준 (PTP 연동 필수)
# → 위 세 항목이 모두 있어야 ns 수준 타임스탬핑 가능
```

---

## Wireshark 프로파일(Profile) 설정

의료 로봇 전용 프로파일을 설정하면 매번 필터와 컬럼을 재설정하지 않아도 됩니다.

```
프로파일 설정 방법:
  Edit → Configuration Profiles → + (New Profile)
  프로파일 이름: "Medical-Robot-TSN"

컬럼 추가 (Edit → Preferences → Columns):
  No., Time, Delta, Source, Destination, Protocol, Length, Info

  + Delta Time: Type = "Delta time displayed" ← 패킷 간격 측정 핵심

컬럼 순서 추천:
  [No.] [Time(Rel)] [Delta] [Src IP] [Dst IP] [Protocol] [Length] [Info]
        (상대시간)  (간격)

Time 표시 형식 (View → Time Display Format):
  UTC Date and Time of Day → 절대 시각
  또는 Seconds Since Capture Start → 상대 시각 (디버깅 편리)

Time 정밀도 설정:
  Automatic (파일 정밀도에 따름) 또는
  나노초 표시: View → Time Display Format → Nanosecond precision
```

---

## 캡처 필터 (Capture Filter, BPF 문법)

BPF(Berkeley Packet Filter)는 캡처 시점에 적용되어 불필요한 패킷을 저장하지 않습니다. 고부하 환경에서 디스크 절약과 성능 향상에 필수적입니다.

```bash
# ── 기본 호스트/네트워크 필터 ──
host 192.168.1.100               # 특정 호스트 양방향
src host 192.168.1.100           # 특정 호스트 송신만
dst host 192.168.1.100           # 특정 호스트 수신만
net 192.168.10.0/24              # 서브넷 전체

# ── 포트 기반 필터 ──
port 13400                       # DoIP (UDP + TCP)
port 30490                       # SOME/IP Service Discovery
port 8883                        # MQTT over TLS

# ── L4 프로토콜 ──
tcp                              # TCP 전체
udp                              # UDP 전체
arp                              # ARP만
icmp                             # ICMP만

# ── EtherType 기반 필터 (L2 프로토콜) ──
ether proto 0x88F7               # PTP (IEEE 1588/802.1AS)
ether proto 0x88E5               # MACsec (IEEE 802.1AE)
ether proto 0x8100               # 802.1Q VLAN 태그
ether proto 0x88A8               # 802.1ad QinQ (이중 VLAN)
ether proto 0x8892               # PROFINET

# ── 복합 조건 (AND/OR/NOT) ──
host 192.168.1.100 and port 13400
not port 22 and not port 443     # SSH/HTTPS 제외
(port 30490 or port 30500) and udp  # SOME/IP 포트 범위

# ── 실전 패턴: 안전 VLAN만 캡처 ──
# VLAN BPF 필터: vlan [id]
vlan 10                          # VLAN ID 10 패킷만
vlan 10 and port 30490           # 안전 VLAN의 SOME/IP

# 링 버퍼 캡처 (장기 모니터링: 100MB × 5개 파일)
tshark -i eth0 \
  -f "not port 22" \
  -b filesize:102400 \
  -b files:5 \
  -w /tmp/robot_capture.pcap
# → robot_capture_00001_20240115120000.pcap 형식으로 순환 저장
```

---

## 디스플레이 필터 (Display Filter)

캡처 후 원하는 패킷만 필터링합니다. Wireshark 고유 문법으로 BPF보다 표현력이 훨씬 강력합니다.

```bash
# ── TCP 연결/오류 분석 ──
tcp.flags.syn == 1 and tcp.flags.ack == 0    # TCP SYN (새 연결 시도)
tcp.flags.reset == 1                         # TCP RST (강제 종료)
tcp.analysis.retransmission                  # 재전송 패킷 (신뢰성 문제)
tcp.analysis.fast_retransmission             # 빠른 재전송 (패킷 손실)
tcp.analysis.zero_window                     # 수신 버퍼 꽉 참
tcp.analysis.duplicate_ack                   # 중복 ACK (패킷 손실 의심)
tcp.analysis.lost_segment                    # 손실 세그먼트 감지
tcp.time_delta > 0.1                         # 100ms 이상 응답 지연

# ── PTP / IEEE 802.1AS (gPTP) ──
ptp                                           # 모든 PTP 메시지
ptp.v2.messagetype == 0x0                    # Sync
ptp.v2.messagetype == 0x8                    # Follow_Up
ptp.v2.messagetype == 0x2                    # Pdelay_Req
ptp.v2.messagetype == 0x3                    # Pdelay_Resp
ptp.v2.messagetype == 0xb                    # Announce
ptp.v2.correction != 0                       # Transparent Clock 보정 있는 패킷
ptp.v2.domainnumber == 0                     # 도메인 0 패킷만

# ── SOME/IP 서비스 분석 ──
someip                                        # SOME/IP 전체
someip.serviceid == 0x0101                   # 조인트 제어 서비스
someip.methodid == 0x0001                    # SetPosition 메서드
someip.msgtype == 0x00                       # Request
someip.msgtype == 0x80                       # Response
someip.msgtype == 0x02                       # Notification (이벤트 발행)
someip.returncode != 0x00                    # 오류 응답 (0x00 = OK)
someip.serviceid == 0xFFFF                   # SOME/IP-SD (서비스 발견)

# ── DoIP / UDS 진단 ──
doip                                          # DoIP 전체
doip.type == 0x0005                          # Routing Activation Request
doip.type == 0x0006                          # Routing Activation Response
doip.type == 0x8001                          # Diagnostic Message (UDS 요청)
doip.type == 0x8002                          # Diagnostic Message ACK
doip.type == 0x8003                          # Diagnostic Message NACK

# UDS 서비스 ID 필터 (DoIP payload 직접 접근)
doip.type == 0x8001 and data[8:1] == 22      # ReadDataByIdentifier (0x22)
doip.type == 0x8001 and data[8:1] == 19      # ReadDTCInformation (0x19)
doip.type == 0x8001 and data[8:1] == 36      # TransferData (OTA 플래싱)
doip.type == 0x8001 and data[8:1] == 7f      # NRC (부정 응답)
doip.type == 0x8001 and data[8:1] == 27      # SecurityAccess (0x27)

# ── MQTT ──
mqtt                                          # MQTT 전체
mqtt.msgtype == 1                            # CONNECT
mqtt.msgtype == 3                            # PUBLISH
mqtt.msgtype == 4                            # PUBACK (QoS 1 확인)
mqtt.msgtype == 12                           # PINGREQ (연결 유지)
mqtt.topic contains "robot/R001"             # 특정 로봇 토픽
mqtt.topic matches "robot/.*/joint"          # 정규식 토픽 필터

# ── ARP / VLAN / L2 ──
arp                                           # ARP 전체
arp.opcode == 1                              # ARP Request (주소 질의)
arp.opcode == 2                              # ARP Reply (응답)
vlan.id == 10                               # VLAN 10 (안전 네트워크)
vlan.id == 20                               # VLAN 20 (제어 네트워크)
vlan.priority >= 5                           # PCP ≥ 5 (높은 우선순위)
eth.dst == ff:ff:ff:ff:ff:ff                # 브로드캐스트

# ── TLS 보안 ──
tls                                           # TLS 전체
tls.handshake.type == 1                      # ClientHello
tls.handshake.type == 2                      # ServerHello
tls.handshake.type == 11                     # Certificate
tls.record.version == 0x0304                 # TLS 1.3 레코드
ssl.alert_message.level == 2                 # Fatal Alert (보안 오류)

# ── 복합 조건 ──
ip.addr == 192.168.1.100 && tcp.port == 13400
!(ip.addr == 10.0.0.1) && tcp.flags.reset == 1  # 특정 호스트 제외 RST
frame.time_relative > 10 && frame.time_relative < 20  # 10~20초 구간
```

---

## tshark (CLI 기반 Wireshark)

GUI 없이 커맨드라인에서 패킷을 분석합니다. 자동화 스크립트, CI/CD 파이프라인 통합에 필수적입니다.

```bash
# ── 실시간 캡처 + CSV 출력 ──
tshark -i eth0 \
  -f "port 13400" \
  -Y "doip" \
  -T fields \
  -e frame.number \
  -e frame.time_epoch \
  -e ip.src \
  -e ip.dst \
  -e doip.type \
  -E header=y \
  -E separator=,

# ── PCAP 오프라인 분석 ──
# 1. 재전송 목록
tshark -r capture.pcap \
  -Y "tcp.analysis.retransmission" \
  -T fields \
  -e frame.number \
  -e frame.time_relative \
  -e ip.src \
  -e ip.dst \
  -e tcp.seq

# 2. 프로토콜 계층 통계 (io,phs)
tshark -r capture.pcap -q -z io,phs
# Protocol Hierarchy Statistics
# ===================================================================
# Filter: <No Filter>
# eth                                    frames:5234 bytes:4521231
#   ip                                   frames:5100 bytes:4400000
#     udp                                frames:3000 bytes:2500000
#       someip                           frames:2000 bytes:2000000
#     tcp                                frames:2100 bytes:1900000
#       doip                             frames:100  bytes:150000

# 3. 대화 통계 (상위 10 IP 쌍)
tshark -r capture.pcap -q -z conv,ip | head -20

# 4. PTP 타임스탬프 CSV 추출
tshark -r capture.pcap \
  -Y "ptp" \
  -T fields \
  -e frame.time_relative \
  -e ptp.v2.messagetype \
  -e ptp.v2.clockidentity \
  -e ptp.v2.correction \
  -E header=y -E separator=, \
  > ptp_analysis.csv

# 5. SOME/IP 서비스 발견 흐름 분석
tshark -r capture.pcap \
  -Y "someip.serviceid == 0xFFFF" \
  -T fields \
  -e frame.time_relative \
  -e ip.src \
  -e someip.methodid \
  -E header=y -E separator=,

# 6. 에러 응답만 추출
tshark -r capture.pcap \
  -Y "someip.returncode != 0x00" \
  -V | grep -A 5 "Return Code"
```

---

## 지연 시간 측정 (Delta Time 분석)

```bash
# ── DoIP 요청-응답 RTT 측정 ──
# 진단 메시지(0x8001) → ACK(0x8002) 간격 측정
tshark -r capture.pcap \
  -Y "doip.type == 0x8001 or doip.type == 0x8002" \
  -T fields \
  -e frame.number \
  -e frame.time_relative \
  -e frame.time_delta \
  -e ip.src \
  -e doip.type \
  -E separator="\t"

# ── SOME/IP 이벤트 주기 측정 ──
tshark -r capture.pcap \
  -Y "someip.serviceid == 0x0101 and someip.msgtype == 0x02" \
  -T fields \
  -e frame.time_epoch \
  -e frame.len \
  > someip_timing.csv

# Jitter 분석 (Python)
python3 << 'EOF'
import csv
import statistics

timestamps = []
with open('someip_timing.csv') as f:
    for row in csv.reader(f, delimiter='\t'):
        if row and row[0]:
            try:
                timestamps.append(float(row[0]))
            except ValueError:
                pass

intervals = [(timestamps[i+1] - timestamps[i]) * 1000
             for i in range(len(timestamps) - 1)]

print(f"이벤트 수:      {len(timestamps)}")
print(f"평균 간격:      {statistics.mean(intervals):.3f} ms")
print(f"최소 간격:      {min(intervals):.3f} ms")
print(f"최대 간격:      {max(intervals):.3f} ms")
print(f"표준편차(Jitter): {statistics.stdev(intervals):.3f} ms")
p99 = sorted(intervals)[int(len(intervals)*0.99)]
print(f"P99 간격:       {p99:.3f} ms")
print(f"목표(1ms ±50µs): {'PASS' if statistics.stdev(intervals) < 0.05 else 'FAIL'}")
EOF
```

---

## Statistics 도구 활용

Wireshark GUI의 Statistics 메뉴는 PCAP를 시각적으로 분석하는 강력한 도구입니다.

```
주요 Statistics 메뉴:

1. Statistics → I/O Graph (입출력 그래프)
   - 시간축 x, 패킷/바이트 수 y 그래프
   - 여러 필터를 레이어로 겹쳐 비교 가능

   의료 로봇 활용:
     Filter 1: "vlan.priority == 7"  → TC7 안전 트래픽 양
     Filter 2: "vlan.priority == 0"  → BE 트래픽 양
     → GCL 슬롯에 따라 TC7이 먼저 집중되고 이후 BE 전송 확인

2. Statistics → TCP Stream Graphs → Round Trip Time
   - TCP 세션별 RTT 시계열 그래프
   - 지연 급증 구간 시각적 탐지

3. Statistics → Flow Graph (Sequence Diagram)
   - 프로토콜 시퀀스 다이어그램 자동 생성
   - TCP: SYN → SYN/ACK → ... → FIN 흐름 표시
   - SOME/IP: SD → Subscribe → Notify 흐름 표시

   사용법:
     Statistics → Flow Graph
     Flow type: TCP flows 또는 All packets
     Address type: Network address

4. Statistics → Protocol Hierarchy
   - 전체 캡처에서 프로토콜별 비율
   - "이 파일에서 SOME/IP가 몇 %인가?" 즉시 확인

5. Statistics → Conversations
   - IP/TCP/UDP 별 통신 쌍 목록
   - Bytes, Packets, Duration 정렬 가능
   - 예상치 못한 통신 쌍 탐지 (보안 관점)

6. Statistics → Endpoints
   - 참여한 IP/MAC 주소 목록
   - 알 수 없는 MAC이 있는지 확인 (침입 탐지)
```

---

## Color Rules (색상 규칙) 설정

중요한 패킷을 즉시 눈에 띄게 만들어 분석 속도를 높입니다.

```
Color Rules 설정 방법:
  View → Coloring Rules → + (새 규칙 추가)

의료 로봇 권장 색상 규칙 (우선순위 순):

1. 빨강 (Critical): TCP RST, E2E 오류
   Filter: tcp.flags.reset == 1
   Background: Red, Foreground: White

2. 주황 (Warning): TCP 재전송
   Filter: tcp.analysis.retransmission
   Background: Orange

3. 노랑 (주의): 100ms 이상 지연
   Filter: frame.time_delta > 0.1
   Background: Yellow

4. 연두 (PTP): gPTP 메시지
   Filter: ether.type == 0x88F7
   Background: Light Green

5. 파랑 (SOME/IP): SOME/IP 서비스
   Filter: someip
   Background: Light Blue

6. 보라 (DoIP): UDS 진단
   Filter: doip
   Background: Lavender

7. 진청 (VLAN 10): 안전 네트워크
   Filter: vlan.id == 10
   Background: Light Cyan
```

---

## Expert Info 활용 (오류 자동 탐지)

```
Wireshark Expert Info 레벨 및 의미:
  Error   (빨강): 심각한 네트워크 문제
    - TCP RST (연결 강제 종료)
    - Malformed Packet (잘못된 패킷 구조)
    - Bad Checksum (체크섬 오류)

  Warning (노랑): 성능 저하 경고
    - TCP 재전송 (Retransmission)
    - 중복 ACK (Duplicate ACK)
    - Zero Window (수신 버퍼 꽉 참)
    - Out of Order 패킷

  Note    (하늘): 정상이지만 주목할 이벤트
    - TCP Keep-Alive
    - TCP 연결 종료 (FIN)
    - PTP 클록 변경

  Chat    (파랑): 일반 프로토콜 이벤트

확인 방법:
  메뉴: Analyze → Expert Information
  또는: 화면 하단 상태 바 좌측 색상 원형 아이콘 클릭

의료 로봇 체크리스트:
  ☐ TCP Retransmission 0건     → 통신 신뢰성 OK
  ☐ TCP Zero Window 0건        → 수신 버퍼 충분
  ☐ Malformed Packet 0건       → 프로토콜 구현 정상
  ☐ PTP 메시지 주기 125ms 유지  → 타임 동기화 정상
  ☐ SOME/IP returncode 0x00    → 서비스 정상
  ☐ DoIP ACK 지연 < 50ms       → 진단 응답 정상
```

---

## 프로토콜별 Wireshark 플러그인 설정

```bash
# ── SOME/IP Dissector ──
# Wireshark 3.x부터 기본 내장 (someip.dll/so 별도 설치 불필요)
# 단, 포트 설정 필요:
# Edit → Preferences → Protocols → SOMEIP
#   SOMEIP Port Range: 30490-30500 (기본)
#   SOMEIP-SD Port: 30490 (기본)

# ── DoIP ──
# Wireshark 3.0+에서 기본 내장 (UDP/TCP 13400 자동 인식)

# ── PTP/IEEE 1588/802.1AS ──
# 기본 내장: 멀티캐스트 01:1B:19:00:00:00 및 01:80:C2:00:00:0E 자동 인식
# Preferences → Protocols → PTP:
#   Force PTP version: v2 (gPTP는 항상 v2)
#   Transport: Ethernet (L2)

# ── AUTOSAR PDU / FIBEX 연동 ──
# Vector DBC/FIBEX 파일 있을 때 CAN over Ethernet 역직렬화 가능
# Edit → Preferences → Protocols → SOMEIP
#   Import FIBEX File: vehicle_model.xml

# ── MACsec (IEEE 802.1AE) ──
# 암호화된 MACsec는 복호화 키 없이 헤더만 표시됨
# Wireshark → Edit → Preferences → Protocols → MACsec
#   MACsec Key: (16진수 키 직접 입력, 개발 환경만)
```

---

## 의료 로봇 통신 검증 시나리오

### 시나리오 1: TSN 타임 동기화 검증

```bash
# gPTP Sync 메시지 주기 측정 (정상: 125ms ± 1ms)
tshark -r capture.pcap \
  -Y "ptp.v2.messagetype == 0x0" \
  -T fields -e frame.time_relative | \
awk 'NR>1 {
    interval = ($1 - prev) * 1000
    sum += interval; n++
    if(interval > max) max = interval
    if(min == 0 || interval < min) min = interval
    prev = $1
} NR==1 { prev = $1 }
END {
    printf "Sync 간격: 평균 %.3fms, 최소 %.3fms, 최대 %.3fms (n=%d)\n",
           sum/n, min, max, n
}'

# Transparent Clock 보정 확인 (다중 홉 경유 시 CF 누적)
tshark -r capture.pcap \
  -Y "ptp.v2.messagetype == 0x8" \
  -T fields \
  -e frame.time_relative \
  -e ptp.v2.correction
# → Follow_Up의 CorrectionField 값이 증가하면 TC가 동작 중
```

### 시나리오 2: SOME/IP 서비스 발견 검증

```bash
# SD(Service Discovery) 메시지 추적
tshark -r capture.pcap \
  -Y "someip.serviceid == 0xFFFF" \
  -T fields \
  -e frame.time_relative \
  -e ip.src \
  -e someip.methodid
# methodid:
#   0x0100 = OfferService
#   0x0200 = FindService (Find)
#   0x0400 = Subscribe
#   0x0800 = SubscribeAck

# 서비스 발견 지연 측정 (FindService → OfferService 간격)
# → 정상: < 100ms (TTL 내 응답)
```

### 시나리오 3: OTA 업데이트 무결성 검증

```bash
# TransferData 패킷 속도 측정
tshark -r ota_capture.pcap \
  -Y "doip.type == 0x8001" \
  -T fields \
  -e frame.time_epoch \
  -e frame.len | \
awk '
BEGIN { start = 0; bytes = 0 }
{
    if (start == 0) start = $1
    bytes += $2
    last = $1
}
END {
    duration = last - start
    rate_mbps = bytes * 8 / duration / 1e6
    printf "전송 크기: %d bytes, 시간: %.2fs, 속도: %.2f Mbps\n",
           bytes, duration, rate_mbps
}'

# SecurityAccess 시퀀스 확인
tshark -r ota_capture.pcap \
  -Y "doip.type == 0x8001" \
  -T fields \
  -e frame.time_relative \
  -e data \
  | grep -E "27|67"  # SecurityAccess 요청(27)/응답(67)
```

### 시나리오 4: E-Stop 신호 전파 지연 측정

```bash
# VLAN 10 (Safety), PCP 7 패킷의 시간 순서 및 간격
tshark -r capture.pcap \
  -Y "vlan.id == 10 and vlan.priority == 7" \
  -T fields \
  -e frame.time_relative \
  -e ip.src \
  -e ip.dst \
  -e frame.len | \
awk 'NR>1 {
    delta = ($1 - prev) * 1000
    if(delta > 5) printf "비정상 간격: %.3fms at t=%.3fs\n", delta, $1
    prev = $1
} NR==1 { prev = $1 }'
# → E-Stop 패킷 간격이 설계값(1ms)을 크게 초과하면 TSN 설정 문제
```

---

## PCAP 파일 관리 및 자동화

```bash
# ── 장기 캡처 자동화 스크립트 ──
#!/bin/bash
CAPTURE_DIR=/var/log/robot/pcap
IFACE=eth0
MAX_FILES=24      # 최대 파일 수 (24시간 × 1시간)
FILE_SIZE=512000  # 512MB 단위 롤오버

mkdir -p $CAPTURE_DIR

tshark -i $IFACE \
  -f "not port 22 and not port 443" \   # SSH/HTTPS 제외
  -b filesize:$FILE_SIZE \
  -b files:$MAX_FILES \
  -b duration:3600 \                     # 1시간마다 새 파일
  -w $CAPTURE_DIR/robot.pcap \
  -q                                     # 조용한 모드 (백그라운드)

# ── PCAP 병합 (여러 파일 → 하나) ──
mergecap -w merged.pcap \
  capture_00001.pcap \
  capture_00002.pcap \
  capture_00003.pcap

# ── PCAP 자르기 (특정 시간 구간 추출) ──
editcap -A "2024-01-15 10:30:00" \
        -B "2024-01-15 10:35:00" \
        full_capture.pcap \
        incident_5min.pcap

# ── 특정 IP/포트 패킷만 추출 ──
tshark -r full.pcap \
  -Y "ip.addr == 192.168.1.100" \
  -w filtered.pcap
```

---

## 자주 발생하는 문제와 해결

```
문제: "No packets captured" (캡처 0개)
  원인 1: 권한 부족
    확인: id | grep wireshark
    해결: sudo usermod -aG wireshark $USER && newgrp wireshark

  원인 2: 잘못된 인터페이스 이름
    확인: ip link show 또는 tshark -D
    해결: tshark -i enp3s0 (실제 인터페이스명 사용)

  원인 3: BPF 필터 문법 오류
    확인: tshark -i eth0 -f "잘못된 필터" → 오류 메시지 확인
    해결: tcpdump -i eth0 -d "필터" (BPF 문법 검증)

문제: PTP 메시지가 보이지 않음
  원인: 멀티캐스트 주소 필터링 또는 허브 아닌 스위치 사용
  확인: SPAN/미러링 포트 설정 확인
  해결: 스위치의 포트 미러링(SPAN)으로 캡처 포트에 트래픽 복사

문제: SOME/IP 패킷이 "UDP" 로만 표시됨
  원인: Wireshark가 SOME/IP 포트를 모름
  해결:
    Analyze → Decode As → 포트 30490 → SOME/IP 선택
    또는 Edit → Preferences → Protocols → SOMEIP → 포트 설정

문제: 타임스탬프 정밀도가 ms 수준
  원인: 소프트웨어 타임스탬핑 사용
  해결:
    ethtool -T eth0 확인 (hardware-receive 필요)
    tshark -i eth0 --time-stamp-type=adapter_unsynced
```

---

## Reference
- [Wireshark Official Website](https://www.wireshark.org/)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [tshark Man Page](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Wireshark Display Filter Reference](https://www.wireshark.org/docs/dfref/)
- [BPF (Berkeley Packet Filter) - tcpdump Documentation](https://www.tcpdump.org/manpages/pcap-filter.7.html)
- [SOME/IP Wireshark Dissector (COVESA)](https://github.com/COVESA/someip-dissector)
- [Wireshark - PTP Dissector](https://wiki.wireshark.org/PTP)
- [editcap / mergecap Manual](https://www.wireshark.org/docs/man-pages/editcap.html)
