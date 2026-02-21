# Lab 02: Wireshark로 의료 로봇 프로토콜 분석

## 개요
이 실습에서는 Wireshark와 tshark를 사용하여 실제 또는 시뮬레이션된 의료 로봇 통신 트래픽을 분석합니다. SOME/IP, DoIP, MQTT, PTP 패킷을 캡처하고 해석하는 방법을 익힙니다.

**필요 환경:**
- Wireshark 4.0+ (또는 tshark)
- Python 3.8+ with Scapy, paho-mqtt
- 실습 PCAP 파일 또는 직접 생성

---

## Step 1: 트래픽 생성 (Scapy)

```python
#!/usr/bin/env python3
# lab02_traffic_gen.py
# 의료 로봇 시뮬레이션 트래픽 생성

from scapy.all import *
import struct
import time
import threading

TARGET_IP = "127.0.0.1"  # 루프백 테스트

def gen_ptp_sync():
    """PTP Sync 메시지 생성 (L2 멀티캐스트)"""
    # IEEE 1588 PTP Sync (멀티캐스트 MAC: 01:1B:19:00:00:00)
    ptp_payload = bytes([
        0x00,  # Message Type: Sync (0x00) | Transport (0x0)
        0x02,  # Version: 2
        0x00, 0x2C,  # Message Length: 44
        0x00,  # Domain: 0
        0x00,  # SdoId
        0x00, 0x00,  # Flags
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,  # Correction Field
        0x00, 0x00, 0x00, 0x00,  # Message type specific
        0x00, 0x11, 0x22, 0xFF, 0xFE, 0x33, 0x44, 0x55, 0x00, 0x01,  # Clock ID + Port
        0x00, 0x01,  # Sequence ID
        0x02,  # Control: Sync
        0xFD,  # Log Message Interval: -3 (125ms)
        # Origin Timestamp (10 bytes)
        0x00, 0x00, 0x67, 0x8A, 0xE5, 0x00,  # Seconds
        0x00, 0x00, 0x00, 0x00,              # Nanoseconds
    ])

    pkt = Ether(
        dst="01:1B:19:00:00:00",
        src="00:11:22:33:44:55",
        type=0x88F7  # EtherType: PTP
    ) / Raw(ptp_payload)
    return pkt


def gen_someip_notification():
    """SOME/IP 이벤트 알림 패킷 생성"""
    # SOME/IP 헤더 (16 Bytes) + 페이로드
    service_id = 0x0101
    method_id = 0x8001  # Event ID
    client_id = 0xF001
    session_id = 0x0001
    payload = struct.pack(">fff", 1.5, -0.5, 2.0)  # 관절 3개 위치

    header = struct.pack(">HHIHHBBBB",
        service_id, method_id,
        8 + len(payload),   # Length
        client_id, session_id,
        0x01,  # Protocol Version
        0x01,  # Interface Version
        0x02,  # Notification
        0x00   # E_OK
    )

    pkt = (
        Ether(dst="ff:ff:ff:ff:ff:ff", src="00:11:22:33:44:55") /
        IP(dst="239.0.0.1") /
        UDP(sport=30490, dport=30490) /
        Raw(header + payload)
    )
    return pkt


def send_robot_traffic(duration: int = 30):
    """
    의료 로봇 트래픽 시뮬레이션
    - PTP Sync: 8Hz (125ms)
    - SOME/IP 제어 데이터: 100Hz (10ms)
    - MQTT 텔레메트리: 1Hz (1000ms)
    """
    import paho.mqtt.client as mqtt

    # MQTT 클라이언트 (텔레메트리)
    mqtt_client = mqtt.Client()
    try:
        mqtt_client.connect("localhost", 1883, 60)
        mqtt_client.loop_start()
        mqtt_connected = True
    except Exception:
        mqtt_connected = False
        print("MQTT 브로커 없음 - MQTT 스킵")

    start = time.time()
    seq = 0
    ptp_timer = 0
    someip_timer = 0
    mqtt_timer = 0

    print(f"트래픽 생성 시작 ({duration}초)...")
    while time.time() - start < duration:
        now = time.time() - start

        # PTP Sync (125ms)
        if now - ptp_timer >= 0.125:
            ptp_timer = now
            try:
                sendp(gen_ptp_sync(), iface="lo", verbose=False)
            except Exception:
                pass

        # SOME/IP (10ms)
        if now - someip_timer >= 0.01:
            someip_timer = now
            seq = (seq + 1) % 65535
            try:
                sendp(gen_someip_notification(), iface="lo", verbose=False)
            except Exception:
                pass

        # MQTT 텔레메트리 (1s)
        if mqtt_connected and now - mqtt_timer >= 1.0:
            mqtt_timer = now
            import json
            payload = json.dumps({
                'robot_id': 'R001',
                'timestamp': time.time(),
                'joint_positions': [1.5, -0.5, 2.0, 0.1, -1.2, 0.8],
                'battery': 85.0
            })
            mqtt_client.publish("robot/R001/telemetry", payload, qos=1)

        time.sleep(0.001)  # 1ms 슬립

    if mqtt_connected:
        mqtt_client.loop_stop()
        mqtt_client.disconnect()

    print("트래픽 생성 완료")


if __name__ == '__main__':
    send_robot_traffic(duration=30)
```

---

## Step 2: 트래픽 캡처

```bash
# 터미널 1: 트래픽 생성
sudo python3 lab02_traffic_gen.py &

# 터미널 2: 캡처
sudo tshark -i lo \
    -f "ether proto 0x88F7 or udp port 30490 or tcp port 1883" \
    -a duration:35 \
    -w /tmp/lab02_capture.pcap

# 캡처 완료 후 확인
tshark -r /tmp/lab02_capture.pcap -q -z io,phs
```

---

## Step 3: 프로토콜별 분석

### PTP 분석

```bash
# PTP 메시지 추출
tshark -r /tmp/lab02_capture.pcap -Y "ptp" \
    -T fields \
    -e frame.number \
    -e frame.time_relative \
    -e ptp.v2.messagetype \
    -e eth.src \
    -E header=y

# PTP 메시지 타입 분포 확인
tshark -r /tmp/lab02_capture.pcap -Y "ptp" \
    -T fields -e ptp.v2.messagetype | sort | uniq -c

# PTP Sync 주기 확인 (125ms?)
tshark -r /tmp/lab02_capture.pcap -Y "ptp.v2.messagetype == 0" \
    -T fields -e frame.time_epoch | \
    awk 'NR>1{printf "간격: %.3fms\n", ($1-prev)*1000} {prev=$1}'
```

### SOME/IP 분석

```bash
# SOME/IP 패킷 (UDP 30490)
tshark -r /tmp/lab02_capture.pcap -Y "udp.port == 30490" \
    -T fields \
    -e frame.time_relative \
    -e udp.length \
    -e ip.src -e ip.dst \
    -E header=y

# SOME/IP 페이로드 16진수 보기
tshark -r /tmp/lab02_capture.pcap -Y "udp.port == 30490" \
    -T fields -e data | head -5

# SOME/IP 헤더 파싱 (Python)
python3 << 'EOF'
import struct
from scapy.all import rdpcap, UDP, Raw

pkts = rdpcap('/tmp/lab02_capture.pcap')
someip_pkts = [p for p in pkts if UDP in p and p[UDP].dport == 30490]

print(f"SOME/IP 패킷 수: {len(someip_pkts)}")

for pkt in someip_pkts[:3]:
    raw = bytes(pkt[Raw])
    if len(raw) >= 16:
        service_id, method_id, length, client_id, session_id, \
            proto_ver, iface_ver, msg_type, return_code = \
            struct.unpack(">HHIHHBBBB", raw[:16])

        print(f"\nSOME/IP 패킷:")
        print(f"  Service ID:  0x{service_id:04X}")
        print(f"  Method ID:   0x{method_id:04X}")
        print(f"  Length:      {length}")
        print(f"  Message Type:{['REQUEST','REQ_NO_RESP','NOTIFICATION'][msg_type] if msg_type < 3 else hex(msg_type)}")
        print(f"  Return Code: {'E_OK' if return_code == 0 else f'0x{return_code:02X}'}")
        print(f"  Payload:     {raw[16:].hex()}")
EOF
```

### MQTT 분석

```bash
# MQTT 패킷 확인 (TCP 1883)
tshark -r /tmp/lab02_capture.pcap -Y "mqtt" \
    -T fields \
    -e frame.time_relative \
    -e mqtt.msgtype \
    -e mqtt.topic \
    -e mqtt.msg

# MQTT 패킷 타입 분포
tshark -r /tmp/lab02_capture.pcap -Y "mqtt" \
    -T fields -e mqtt.msgtype | sort | uniq -c
# 1=CONNECT, 2=CONNACK, 3=PUBLISH, 4=PUBACK
```

---

## Step 4: 지연 측정 실습

```python
#!/usr/bin/env python3
# lab02_latency_measure.py
# PCAP에서 요청-응답 지연 측정

from scapy.all import *
import struct
import statistics

def measure_latency_from_pcap(pcap_file: str):
    """
    PCAP에서 SOME/IP 요청-응답 지연 측정
    같은 Service/Method/Session ID를 가진 REQUEST → RESPONSE 쌍 탐색
    """
    pkts = rdpcap(pcap_file)

    requests = {}   # (service, method, session) → timestamp
    latencies = []  # [ms]

    for pkt in pkts:
        if UDP not in pkt:
            continue
        raw = bytes(pkt.payload.payload.payload)  # UDP payload
        if len(raw) < 16:
            continue

        try:
            service_id, method_id, length, client_id, session_id, \
                proto_ver, iface_ver, msg_type, return_code = \
                struct.unpack(">HHIHHBBBB", raw[:16])
        except Exception:
            continue

        key = (service_id, method_id, session_id)

        if msg_type == 0x00:  # REQUEST
            requests[key] = float(pkt.time)
        elif msg_type == 0x80:  # RESPONSE
            if key in requests:
                latency_ms = (float(pkt.time) - requests[key]) * 1000
                latencies.append(latency_ms)
                del requests[key]

    if latencies:
        print("\n=== SOME/IP 요청-응답 지연 ===")
        print(f"  측정 횟수: {len(latencies)}")
        print(f"  평균:     {statistics.mean(latencies):.3f} ms")
        print(f"  최소:     {min(latencies):.3f} ms")
        print(f"  최대:     {max(latencies):.3f} ms")
        print(f"  표준편차: {statistics.stdev(latencies):.3f} ms")
    else:
        print("지연 측정 데이터 없음 (요청-응답 쌍 없음)")

    return latencies


def analyze_packet_interval(pcap_file: str, filter_func=None):
    """
    패킷 간격 분석 (Jitter 계산)
    """
    pkts = rdpcap(pcap_file)
    if filter_func:
        pkts = [p for p in pkts if filter_func(p)]

    timestamps = [float(p.time) for p in pkts]
    if len(timestamps) < 2:
        print("데이터 부족")
        return

    intervals_ms = [(t2-t1)*1000 for t1, t2 in
                    zip(timestamps, timestamps[1:])]

    print(f"\n=== 패킷 간격 분석 ({len(timestamps)} 패킷) ===")
    print(f"  평균 간격: {statistics.mean(intervals_ms):.3f} ms")
    print(f"  최소 간격: {min(intervals_ms):.3f} ms")
    print(f"  최대 간격: {max(intervals_ms):.3f} ms")
    print(f"  Jitter:   {statistics.stdev(intervals_ms):.3f} ms")

    # Jitter 허용치 판단
    target_interval = statistics.mean(intervals_ms)
    jitter_pct = statistics.stdev(intervals_ms) / target_interval * 100
    print(f"  Jitter (%): {jitter_pct:.1f}%")
    print(f"  판정: {'GOOD (<5%)' if jitter_pct < 5 else 'BAD (>5%)'}")


# 실행
measure_latency_from_pcap('/tmp/lab02_capture.pcap')

# SOME/IP 패킷 간격 분석
analyze_packet_interval(
    '/tmp/lab02_capture.pcap',
    filter_func=lambda p: UDP in p and p[UDP].dport == 30490
)
```

---

## 실습 과제

```
Lab 02 과제:

기본 과제:
  ☐ PTP Sync 주기를 측정하여 125ms ± 허용 범위 확인
  ☐ SOME/IP 패킷 헤더를 수동으로 파싱하여 필드값 확인
  ☐ MQTT PUBLISH 패킷에서 토픽과 페이로드 추출

심화 과제:
  ☐ tshark 필터로 오류 패킷(TCP 재전송, UDP 손실) 찾기
  ☐ Python으로 PCAP 분석하여 패킷 통계 보고서 작성
  ☐ SOME/IP Notification 간격 Jitter 계산

토론 주제:
  Q1. SOME/IP와 DDS(ROS 2)의 패킷 구조 차이는?
  Q2. PTP 메시지가 유니캐스트가 아닌 멀티캐스트를 사용하는 이유는?
  Q3. 의료 로봇에서 Wireshark를 사용한 실시간 모니터링의 한계는?
```
