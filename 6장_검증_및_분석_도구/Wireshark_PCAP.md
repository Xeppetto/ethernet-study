# Wireshark PCAP 분석

## 개요
Wireshark는 세계 표준 네트워크 패킷 분석 도구(오픈소스, GPL)입니다. 의료 로봇의 Ethernet 통신 디버깅, TSN 동작 검증, 프로토콜 적합성 테스트에 필수적입니다. PCAP(Packet Capture) 파일 형식으로 저장하여 사후 분석(Post-mortem Analysis)도 가능합니다.

---

## Wireshark 설치 및 권한 설정

```bash
# Ubuntu/Debian 설치
sudo apt-get install wireshark tshark

# 비root 사용자 패킷 캡처 권한 (Linux)
sudo usermod -aG wireshark $USER
sudo setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
# → 재로그인 후 적용

# 원격 서버에서 캡처하여 로컬에서 분석
ssh robot@192.168.1.100 "tcpdump -i eth0 -w - -U" | wireshark -k -i -

# 하드웨어 타임스탬핑 활성화 (정밀 측정)
ethtool -T eth0
# Capabilities:
#   hardware-transmit  ← TX 하드웨어 타임스탬핑
#   hardware-receive   ← RX 하드웨어 타임스탬핑
#   hardware-raw-clock ← PHC 기준
```

---

## 캡처 필터 (Capture Filter, BPF 문법)

```bash
# BPF(Berkeley Packet Filter) - 캡처 시 적용, 성능 효율적

# 특정 호스트
host 192.168.1.100

# 특정 서브넷
net 192.168.10.0/24

# 특정 포트
port 13400                    # DoIP
port 30490                    # SOME/IP
port 8883                     # MQTT TLS

# 특정 프로토콜
arp
icmp
udp

# 복합 조건
host 192.168.1.100 and port 13400
not port 22 and not port 443   # SSH/HTTPS 제외

# Ethernet 프레임 타입
ether proto 0x88F7            # PTP (IEEE 1588)
ether proto 0x88E5            # MACsec
ether proto 0x8100            # 802.1Q VLAN
ether proto 0x88A8            # 802.1ad QinQ
```

---

## 디스플레이 필터 (Display Filter)

```bash
# 캡처 후 필터링 - Wireshark 전용 문법

# ── TCP 관련 ──
tcp.flags.syn == 1 and tcp.flags.ack == 0    # TCP SYN (연결 시작)
tcp.analysis.retransmission                  # 재전송 패킷
tcp.analysis.zero_window                     # 윈도우 크기 0 (수신 버퍼 꽉 참)
tcp.analysis.duplicate_ack                   # 중복 ACK (패킷 손실 의심)
tcp.time_delta > 0.1                         # 100ms 이상 응답 지연

# ── PTP / IEEE 802.1AS ──
ptp                                           # 모든 PTP 패킷
ptp.v2.messagetype == 0x0                    # Sync 메시지
ptp.v2.messagetype == 0x8                    # Follow_Up
ptp.v2.messagetype == 0x9                    # Delay_Req
ptp.v2.messagetype == 0xA                    # Delay_Resp
ptp.v2.messagetype == 0x3                    # Pdelay_Req (P2P)
ptp.v2.correction != 0                       # 보정 적용된 패킷 (TC)

# ── SOME/IP ──
someip                                        # SOME/IP 전체
someip.serviceid == 0x0101                   # 특정 서비스
someip.methodid == 0x0001                    # 특정 메서드
someip.msgtype == 0x02                       # Notification (이벤트)
someip.returncode != 0x00                    # 오류 응답

# ── DoIP / UDS ──
doip                                          # DoIP 전체
doip.type == 0x8001                          # 진단 메시지
doip.type == 0x0005                          # Routing Activation
# UDS 서비스 필터 (payload 기준)
doip && data[4:1] == 22                      # ReadDataByIdentifier
doip && data[4:1] == 7f                      # 부정 응답 (NRC)
doip && data[4:1] == 27                      # SecurityAccess
doip && data[4:1] == 36                      # TransferData (Flash)

# ── MQTT ──
mqtt                                          # MQTT 전체
mqtt.msgtype == 3                            # PUBLISH
mqtt.topic contains "robot/R001"             # 특정 토픽
mqtt.msgtype == 4                            # PUBACK (QoS 1 ACK)

# ── ARP / VLAN ──
arp                                           # ARP 전체
arp.opcode == 2                              # ARP Reply
vlan.id == 10                               # VLAN 10
vlan.priority >= 5                           # CoS 5 이상

# ── 복합 조건 ──
ip.addr == 192.168.1.100 && tcp.port == 13400
!(ip.addr == 10.0.0.1) && tcp.flags.reset == 1  # 특정 호스트 제외 TCP RST
```

---

## tshark (CLI 기반 Wireshark)

```bash
# 실시간 캡처 + 필터
tshark -i eth0 -f "port 13400" -Y "doip" \
  -T fields \
  -e frame.time_epoch \
  -e ip.src \
  -e ip.dst \
  -e doip.type \
  -E header=y \
  -E separator=,

# 파일로 저장 (링 버퍼: 100MB × 5개)
tshark -i eth0 -b filesize:102400 -b files:5 \
  -w /tmp/robot_capture.pcap

# PCAP 분석
tshark -r capture.pcap -Y "tcp.analysis.retransmission" \
  -T fields -e frame.number -e ip.src -e ip.dst

# 패킷 통계 (프로토콜 분포)
tshark -r capture.pcap -q -z io,phs

# 대화 통계 (Top 10 IP 쌍)
tshark -r capture.pcap -q -z conv,ip | head -20

# 특정 필드 추출 (CSV)
tshark -r capture.pcap -Y "ptp" \
  -T fields \
  -e frame.time_relative \
  -e ptp.v2.messagetype \
  -e ptp.v2.clockidentity \
  -e ptp.v2.correction \
  -E header=y -E separator=, \
  > ptp_analysis.csv
```

---

## 지연 시간 측정 (Delta Time 분석)

```bash
# 요청-응답 지연 측정 (DoIP 진단)
tshark -r capture.pcap \
  -Y "doip.type == 0x8001 or doip.type == 0x8002" \
  -T fields \
  -e frame.number \
  -e frame.time_relative \
  -e frame.time_delta \
  -e ip.src \
  -e doip.type

# I/O 그래프 데이터 추출 (패킷 간격 분석)
tshark -r capture.pcap \
  -Y "someip.serviceid == 0x0101" \
  -T fields \
  -e frame.time_epoch \
  -e frame.len \
  > someip_timing.csv

# Python으로 Jitter 분석
python3 << 'EOF'
import csv
import statistics

timestamps = []
with open('someip_timing.csv') as f:
    for row in csv.reader(f, delimiter='\t'):
        if row:
            timestamps.append(float(row[0]))

intervals = [t2 - t1 for t1, t2 in zip(timestamps, timestamps[1:])]
intervals_ms = [i * 1000 for i in intervals]

print(f"패킷 수: {len(timestamps)}")
print(f"평균 간격: {statistics.mean(intervals_ms):.3f} ms")
print(f"최소 간격: {min(intervals_ms):.3f} ms")
print(f"최대 간격: {max(intervals_ms):.3f} ms")
print(f"표준편차(Jitter): {statistics.stdev(intervals_ms):.3f} ms")
EOF
```

---

## Expert Info 활용 (오류 자동 탐지)

```
Wireshark Expert Info 레벨:
  Error   (빨강): TCP RST, Malformed Packet, Bad Checksum
  Warning (노랑): TCP 재전송, 중복 ACK, Zero Window
  Note    (하늘): TCP 연결 종료, Keep-Alive
  Chat    (파랑): 일반 TCP 이벤트

확인 방법:
  메뉴 → Analyze → Expert Information
  또는 상태 바 좌측 색상 아이콘 클릭

의료 로봇 체크리스트:
  ☐ TCP Retransmission 없음 → 통신 신뢰성
  ☐ TCP Zero Window 없음   → 수신 버퍼 충분
  ☐ PTP 메시지 주기 정상    → 타임 동기화
  ☐ SOME/IP 오류 응답 없음  → 서비스 정상
  ☐ DoIP ACK 지연 없음      → 진단 응답 정상
```

---

## 프로토콜별 Wireshark 플러그인

```bash
# SOME/IP Dissector 설치 (Vector SOME/IP Plugin)
# Wireshark → Help → About Wireshark → Plugins 폴더에 복사
# someip.dll (Windows) / someip.so (Linux)

# Preferences 설정 (SOME/IP)
# Edit → Preferences → Protocols → SOMEIP
# SOMEIP Port Range: 30490-30500

# DoIP는 기본 내장 (Wireshark 3.0+)
# UDP/TCP 13400 자동 해석

# PTP/IEEE 1588 기본 내장
# 멀티캐스트 224.0.1.129 자동 인식

# AUTOSAR PDU (BSW) - Vector CANape/CANoe 연동
# LIN/CAN over Ethernet 분석도 가능
```

---

## 의료 로봇 통신 검증 시나리오

```bash
# 시나리오 1: TSN 타임 동기화 검증
# gPTP 메시지 주기 측정 (정상: 125ms ± 1ms)
tshark -r capture.pcap -Y "ptp.v2.messagetype == 0" \
  -T fields -e frame.time_relative -e ptp.v2.clockidentity | \
  awk 'NR>1{print $1-prev; prev=$1} NR==1{prev=$1}' | \
  sort -n | awk '{sum+=$1; n++; if($1>max)max=$1} END{
    printf "Avg:%.3fms Max:%.3fms\n",sum/n*1000,max*1000}'

# 시나리오 2: OTA 업데이트 무결성 검증
# Flash 전송 속도 측정 (TransferData 패킷)
tshark -r ota_capture.pcap -Y "doip.type == 0x8001" \
  -T fields -e frame.time_epoch -e frame.len | \
  awk '{bytes+=$2; time=$1} END{
    printf "전송 속도: %.2f Mbps\n", bytes*8/(time-start)/1e6}'

# 시나리오 3: SOME/IP 서비스 발견 검증
tshark -r capture.pcap -Y "someip" \
  -T fields -e frame.time_relative \
  -e someip.serviceid -e someip.methodid \
  -e someip.msgtype -e someip.returncode | \
  grep "0x8100"  # SD (Service Discovery) 메시지
```

---

## Reference
- [Wireshark Official Website](https://www.wireshark.org/)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [tshark Man Page](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Wireshark Display Filter Reference](https://www.wireshark.org/docs/dfref/)
- [SOME/IP Wireshark Dissector](https://github.com/COVESA/someip-dissector)
