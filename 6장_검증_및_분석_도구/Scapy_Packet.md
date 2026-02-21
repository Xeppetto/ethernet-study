# Scapy - 패킷 생성 및 분석

## 개요
Scapy는 Python으로 작성된 강력한 패킷 조작 라이브러리입니다. Wireshark가 패킷을 *읽는* 도구라면, Scapy는 패킷을 *만들고*, *보내고*, *분석하는* 도구입니다. 의료 로봇 Ethernet 통신 테스트에서 커스텀 패킷 생성, 프로토콜 적합성 검증, 보안 취약점 분석(Fault Injection)에 활용됩니다.

> **주의**: Scapy는 강력한 네트워크 도구이므로 반드시 허가된 테스트 환경에서만 사용해야 합니다.

---

## 설치 및 기본 사용

```bash
# 설치
pip install scapy

# 루트 권한 필요 (raw socket)
sudo python3 scapy_test.py
# 또는
sudo scapy  # 인터랙티브 셸
```

---

## 기본 패킷 구성

```python
from scapy.all import *

# 계층 구조로 패킷 구성 (/ 연산자로 스택)
# Ethernet / IP / UDP / Payload

# 기본 패킷 생성
pkt = Ether() / IP(dst="192.168.1.100") / TCP(dport=80)

# 패킷 내용 확인
pkt.show()
# ###[ Ethernet ]###
#   dst       = ff:ff:ff:ff:ff:ff
#   src       = xx:xx:xx:xx:xx:xx
#   type      = IPv4
# ###[ IP ]###
#   dst       = 192.168.1.100
#   ttl       = 64
# ###[ TCP ]###
#   dport     = http

# 패킷 요약
pkt.summary()
# Ether / IP / TCP 192.168.1.100:http

# 원시 바이트 보기
bytes(pkt).hex()

# 패킷 전송 (L3: IP 레이어부터)
send(IP(dst="192.168.1.100") / ICMP())

# 패킷 전송 (L2: Ethernet 레이어부터)
sendp(Ether(dst="00:11:22:33:44:55") / IP(dst="192.168.1.100") / ICMP(),
      iface="eth0")

# 전송 후 응답 받기
reply = sr1(IP(dst="192.168.1.100") / ICMP(), timeout=2)
```

---

## ARP 스캔 / 분석

```python
from scapy.all import *

def arp_scan(network: str) -> list:
    """
    ARP 스캔으로 네트워크 활성 호스트 탐지
    의료 로봇 네트워크 인벤토리 확인
    """
    # ARP 요청 브로드캐스트
    answered, unanswered = srp(
        Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=network),
        timeout=2,
        iface="eth0",
        verbose=False
    )

    hosts = []
    for sent, received in answered:
        hosts.append({
            'ip': received[ARP].psrc,
            'mac': received[Ether].src,
        })
        print(f"  {received[ARP].psrc:<20} {received[Ether].src}")

    return hosts

print("=== 네트워크 스캔: 192.168.10.0/24 ===")
hosts = arp_scan("192.168.10.0/24")
print(f"발견된 호스트: {len(hosts)}개")

# ARP 스푸핑 탐지 (보안 검증)
def detect_arp_spoofing(interface: str = "eth0", count: int = 100):
    """ARP Reply 패킷 모니터링 → 동일 IP의 다른 MAC 탐지"""
    arp_table = {}
    anomalies = []

    def process_packet(pkt):
        if ARP in pkt and pkt[ARP].op == 2:  # ARP Reply
            ip = pkt[ARP].psrc
            mac = pkt[ARP].hwsrc
            if ip in arp_table and arp_table[ip] != mac:
                anomalies.append({
                    'ip': ip,
                    'known_mac': arp_table[ip],
                    'suspicious_mac': mac
                })
                print(f"[경고] ARP 스푸핑 탐지! IP: {ip}, "
                      f"기존 MAC: {arp_table[ip]}, 새 MAC: {mac}")
            arp_table[ip] = mac

    sniff(filter="arp", prn=process_packet, count=count, iface=interface)
    return anomalies
```

---

## SOME/IP 패킷 분석 (커스텀 레이어)

```python
from scapy.all import *
from scapy.packet import Packet
from scapy.fields import *

# SOME/IP 커스텀 Scapy 레이어 정의
class SOMEIP(Packet):
    """
    AUTOSAR SOME/IP 헤더 (RFC-like, AUTOSAR PRS_SOMEIP)
    Header: 16 Bytes
    """
    name = "SOME/IP"
    fields_desc = [
        XShortField("service_id", 0x0101),     # 서비스 ID
        XShortField("method_id", 0x0001),       # 메서드/이벤트 ID
        IntField("length", None),               # 나머지 길이 (자동 계산)
        XShortField("client_id", 0xF001),       # 클라이언트 ID
        XShortField("session_id", 0x0001),      # 세션 ID
        ByteField("protocol_version", 0x01),   # 항상 1
        ByteField("interface_version", 0x01),  # 인터페이스 버전
        ByteEnumField("message_type", 0x00, { # 메시지 유형
            0x00: "REQUEST",
            0x01: "REQUEST_NO_RETURN",
            0x02: "NOTIFICATION",
            0x80: "RESPONSE",
            0x81: "ERROR"
        }),
        ByteEnumField("return_code", 0x00, {   # 반환 코드
            0x00: "E_OK",
            0x01: "E_NOT_OK",
            0x02: "E_UNKNOWN_SERVICE",
            0x03: "E_UNKNOWN_METHOD",
        }),
    ]

    def post_build(self, pkt, payload):
        # length 자동 계산 (payload 포함)
        if self.length is None:
            length = len(payload) + 8  # 헤더 나머지 + 페이로드
            pkt = pkt[:4] + struct.pack(">I", length) + pkt[8:]
        return pkt + payload

# SOME/IP 패킷 바인딩 (UDP 30490 → SOME/IP 자동 파싱)
bind_layers(UDP, SOMEIP, dport=30490)
bind_layers(UDP, SOMEIP, sport=30490)

# SOME/IP 패킷 생성 및 전송
someip_request = (
    Ether(dst="00:11:22:33:44:55") /
    IP(dst="192.168.1.100") /
    UDP(dport=30490, sport=30500) /
    SOMEIP(
        service_id=0x0101,
        method_id=0x0001,
        client_id=0xF001,
        session_id=0x0001,
        message_type=0x00  # REQUEST
    ) /
    Raw(b"\x00\x00\x00\x00")  # 페이로드 (4 Byte)
)

sendp(someip_request, iface="eth0", verbose=False)
print("SOME/IP Request 전송 완료")

# SOME/IP 스니핑 및 분석
def analyze_someip_traffic(interface: str = "eth0", count: int = 50):
    """SOME/IP 트래픽 캡처 및 분석"""
    pkts = sniff(filter="udp port 30490", count=count, iface=interface)

    stats = {}
    for pkt in pkts:
        if SOMEIP in pkt:
            si = pkt[SOMEIP]
            key = f"0x{si.service_id:04x}/0x{si.method_id:04x}"
            if key not in stats:
                stats[key] = {'count': 0, 'types': {}}
            stats[key]['count'] += 1
            msg_type = si.message_type
            stats[key]['types'][msg_type] = \
                stats[key]['types'].get(msg_type, 0) + 1

    print("=== SOME/IP 트래픽 분석 ===")
    for service, data in stats.items():
        print(f"  {service}: {data['count']} 패킷, "
              f"타입 분포: {data['types']}")

    return stats
```

---

## DoIP 패킷 생성 (진단 시뮬레이터)

```python
from scapy.all import *
import struct

# DoIP 패킷 수동 생성 (커스텀 레이어)
def build_doip_packet(payload_type: int, payload: bytes) -> bytes:
    """
    DoIP Generic Header 생성 (ISO 13400-2)
    Protocol Version: 0x02 (DoIP Version 2)
    """
    proto_version = 0x02
    inverse = proto_version ^ 0xFF
    length = len(payload)

    header = struct.pack(">BBHI",
                         proto_version,    # 1 Byte: 버전
                         inverse,          # 1 Byte: 역버전
                         payload_type,     # 2 Byte: 페이로드 타입
                         length)           # 4 Byte: 길이

    return header + payload

def doip_routing_activation(source_address: int = 0x0E80) -> bytes:
    """
    DoIP Routing Activation Request (0x0005)
    source_address: 테스터 논리 주소 (0x0E80 = 외부 진단 툴)
    """
    payload = struct.pack(">HBQ",
                          source_address,  # 2 Byte
                          0x00,            # 활성화 타입 (Default)
                          0x0000000000000000)  # 예약
    return build_doip_packet(0x0005, payload[:3])

def doip_uds_request(target_addr: int, uds_service: bytes) -> bytes:
    """
    DoIP Diagnostic Message (0x8001)
    UDS 서비스 요청을 DoIP로 래핑
    """
    payload = struct.pack(">HH", 0x0E80, target_addr) + uds_service
    return build_doip_packet(0x8001, payload)

# DoIP로 UDS ReadDataByIdentifier 전송 (예: DID 0xF190 = VIN)
def send_doip_read_vin(robot_ip: str = "192.168.1.100"):
    """로봇 ECU에서 VIN/시리얼 번호 읽기"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((robot_ip, 13400))  # DoIP 포트

    # 1. Routing Activation
    activation = doip_routing_activation(0x0E80)
    sock.send(activation)
    resp = sock.recv(1024)
    print(f"Routing Activation 응답: {resp.hex()}")

    # 2. UDS 0x22 ReadDataByIdentifier - DID 0xF190
    uds_req = bytes([0x22, 0xF1, 0x90])  # 서비스 0x22, DID 0xF190
    doip_msg = doip_uds_request(0x0001, uds_req)
    sock.send(doip_msg)
    resp = sock.recv(1024)
    print(f"UDS ReadDID 응답: {resp.hex()}")

    # DoIP 헤더 파싱 (8 Byte)
    if len(resp) >= 12:
        payload_type = struct.unpack(">H", resp[2:4])[0]
        payload_len = struct.unpack(">I", resp[4:8])[0]
        if payload_type == 0x8001:  # Diagnostic Message
            uds_resp = resp[12:]
            if uds_resp[0] == 0x62:  # 긍정 응답
                vin_bytes = uds_resp[3:]
                print(f"VIN/Serial: {vin_bytes.decode('ascii', errors='replace')}")
            elif uds_resp[0] == 0x7F:  # 부정 응답
                nrc = uds_resp[2]
                print(f"NRC (부정 응답): 0x{nrc:02X}")

    sock.close()
```

---

## Fault Injection (보안 / 견고성 테스트)

```python
from scapy.all import *
import time
import random

def inject_malformed_packet(target_ip: str, interface: str = "eth0"):
    """
    비정상 패킷 주입 → ECU Fault Handling 검증
    IEC 62304 / ISO 26262 Fault Injection 요구사항
    """
    tests = []

    # 테스트 1: 잘못된 DoIP 버전
    pkt1 = (IP(dst=target_ip) /
            TCP(dport=13400) /
            Raw(b"\xFF\x00\x00\x05\x00\x00\x00\x03\x0E\x80\x00"))  # 버전 0xFF
    tests.append(("DoIP 잘못된 버전", pkt1))

    # 테스트 2: 초과 길이 DoIP
    pkt2 = (IP(dst=target_ip) /
            TCP(dport=13400) /
            Raw(b"\x02\xFD\x80\x01\xFF\xFF\xFF\xFF\x0E\x80\x00\x01"))
    tests.append(("DoIP 길이 초과", pkt2))

    # 테스트 3: UDP 플러딩 (Babbling Idiot 시뮬)
    def flood_test(target_ip, count=1000, rate=0.0001):
        print(f"UDP 플러딩 ({count}패킷)...")
        for i in range(count):
            pkt = IP(dst=target_ip) / UDP(dport=30490) / Raw(b"\x00" * 64)
            send(pkt, verbose=False)
            time.sleep(rate)
        print("플러딩 완료")

    # 테스트 4: E2E CRC 오류 패킷 (SOME/IP)
    def send_corrupted_e2e(target_ip, service_id=0x0101):
        """E2E CRC 오류로 ECU Safe State 전환 확인"""
        # 정상 페이로드에 CRC 오류 주입
        payload = bytes([
            0x01,           # Counter
            0xFF,           # 잘못된 CRC (정상값이 아님)
            (service_id >> 8) & 0xFF,
            service_id & 0xFF,
            0x00, 0x00, 0x00, 0x01  # 데이터
        ])
        pkt = (IP(dst=target_ip) /
               UDP(dport=30490) /
               Raw(bytes([
                   (service_id >> 8) & 0xFF, service_id & 0xFF,  # Service ID
                   0x00, 0x01,  # Method ID
                   0x00, 0x00, 0x00, len(payload) + 8,  # Length
                   0xF0, 0x01,  # Client ID
                   0x00, 0x01,  # Session ID
                   0x01,        # Protocol Version
                   0x01,        # Interface Version
                   0x00,        # REQUEST
                   0x00,        # E_OK
               ]) + payload))
        sendp(pkt, iface=interface, verbose=False)
        print("E2E CRC 오류 패킷 전송")

    # 각 테스트 실행
    for name, pkt in tests:
        print(f"\n[테스트] {name}")
        send(pkt, verbose=False)
        time.sleep(0.1)  # ECU 반응 대기

    send_corrupted_e2e(target_ip)
    print("\n모든 Fault Injection 테스트 완료")


def capture_and_analyze(interface: str = "eth0",
                         duration: int = 10,
                         output_file: str = "/tmp/capture.pcap"):
    """
    트래픽 캡처 및 기본 분석
    """
    print(f"캡처 시작 ({duration}초)...")
    pkts = sniff(iface=interface, timeout=duration)

    # PCAP 저장
    wrpcap(output_file, pkts)
    print(f"PCAP 저장: {output_file} ({len(pkts)} 패킷)")

    # 통계
    proto_count = {}
    for pkt in pkts:
        proto = pkt.name
        proto_count[proto] = proto_count.get(proto, 0) + 1

    print("\n=== 프로토콜 분포 ===")
    for proto, count in sorted(proto_count.items(),
                                key=lambda x: x[1], reverse=True):
        print(f"  {proto:<20}: {count:>6} 패킷")

    return pkts
```

---

## PCAP 로딩 및 오프라인 분석

```python
from scapy.all import *
import statistics

def analyze_pcap(pcap_file: str, bpf_filter: str = None):
    """
    기존 PCAP 파일 로딩 및 분석
    Wireshark 캡처 파일을 Python으로 처리
    """
    # 필터 없이 전체 로딩
    if bpf_filter:
        pkts = rdpcap(pcap_file)
        pkts = [p for p in pkts if p.haslayer(eval(bpf_filter))]
    else:
        pkts = rdpcap(pcap_file)

    print(f"총 패킷 수: {len(pkts)}")

    # TCP 재전송 탐지
    seq_seen = {}
    retx_count = 0
    for pkt in pkts:
        if TCP in pkt:
            key = (pkt[IP].src, pkt[IP].dst,
                   pkt[TCP].sport, pkt[TCP].dport)
            seq = pkt[TCP].seq
            if key in seq_seen and seq <= seq_seen[key]:
                retx_count += 1
            seq_seen[key] = seq

    print(f"TCP 재전송: {retx_count}회")

    # UDP 패킷 간격 분석 (Jitter)
    udp_timestamps = {}
    for pkt in pkts:
        if UDP in pkt:
            dport = pkt[UDP].dport
            if dport not in udp_timestamps:
                udp_timestamps[dport] = []
            udp_timestamps[dport].append(float(pkt.time))

    print("\n=== UDP 포트별 Jitter ===")
    for port, timestamps in udp_timestamps.items():
        if len(timestamps) > 2:
            intervals = [(timestamps[i+1] - timestamps[i]) * 1000
                         for i in range(len(timestamps) - 1)]
            print(f"  Port {port}: "
                  f"avg={statistics.mean(intervals):.3f}ms, "
                  f"max={max(intervals):.3f}ms, "
                  f"stdev={statistics.stdev(intervals):.3f}ms")

    return pkts

# 실행 예시
# pkts = analyze_pcap('/tmp/robot_capture.pcap')
```

---

## Scapy 활용 체크리스트

```
의료 로봇 네트워크 검증에 Scapy 활용:

프로토콜 적합성 테스트:
  ☐ SOME/IP 서비스/메서드 ID 체계 검증
  ☐ DoIP 헤더 형식 검증
  ☐ PTP/gPTP 메시지 타이밍 검증
  ☐ VLAN 태깅 및 우선순위 검증

Fault Injection (IEC 62304/ISO 26262):
  ☐ 잘못된 패킷 헤더 → ECU 거부 확인
  ☐ E2E CRC 오류 → Safe State 전환
  ☐ Sequence Number 오류 → 오류 처리
  ☐ UDP 플러딩 → PSFP/IDS 차단 확인

보안 검증 (ISO/SAE 21434):
  ☐ ARP 스푸핑 탐지
  ☐ VLAN 호핑 방지
  ☐ DoS 내성 (플러딩 시 정상 서비스 유지)

PCAP 분석:
  ☐ Wireshark 캡처 파일 Python 처리
  ☐ 지연/Jitter 자동 측정
  ☐ 이상 패킷 자동 탐지
```

---

## Reference
- [Scapy Official Documentation](https://scapy.readthedocs.io/)
- [Scapy GitHub](https://github.com/secdev/scapy)
- [Scapy - Building Network Tools](https://0xbharath.github.io/art-of-packet-crafting-with-scapy/)
- [AUTOSAR SOME/IP Protocol Specification (PRS_SOMEIP)](https://www.autosar.org/standards/foundation)
