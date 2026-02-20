# Wireshark PCAP 분석

## 개요
Ethernet 통신 문제를 해결하는 가장 강력한 도구는 패킷 스니퍼(Packet Sniffer)인 Wireshark입니다. 네트워크 상에서 오고 가는 모든 데이터(Packet)를 캡처하여 프로토콜별로 분석하고, 문제의 원인을 찾아내는 데 필수적입니다. PCAP(Packet Capture) 파일은 이러한 패킷 데이터를 저장하는 표준 형식입니다.

### Wireshark의 주요 기능
1.  **실시간 캡처**: 네트워크 인터페이스(NIC)를 통해 흐르는 패킷을 실시간으로 수집하고 보여줍니다. (Promiscuous Mode 사용)
2.  **프로토콜 디코딩**: 수집된 패킷의 헤더 정보를 분석하여 어떤 프로토콜인지(TCP, UDP, HTTP, SOME/IP 등) 식별하고, 내용을 사람이 읽을 수 있는 형태로 보여줍니다. (Dissector)
3.  **필터링**: 수많은 패킷 중에서 원하는 조건(IP 주소, 포트 번호, 프로토콜 종류 등)에 맞는 패킷만 골라낼 수 있습니다. (Display Filter / Capture Filter)
    -   `ip.addr == 192.168.0.1`: 특정 IP 주소와 통신한 패킷만 보기
    -   `tcp.port == 80`: 웹 트래픽(HTTP)만 보기
    -   `someip`: SOME/IP 프로토콜만 보기 (플러그인 필요)

### PCAP 파일 분석 방법
-   **Conversation Analysis**: 'Statistics -> Conversations' 메뉴를 통해 어떤 호스트끼리 통신했는지, 데이터 양은 얼마나 되는지 확인할 수 있습니다.
-   **Graph Analysis**: 'Statistics -> I/O Graphs' 메뉴를 통해 시간 흐름에 따른 트래픽 변화를 그래프로 볼 수 있습니다. (네트워크 부하, 패킷 손실 등 확인)
-   **Error Finding**: TCP 재전송(Retransmission), 윈도우 크기(Window Size) 문제, 체크섬 오류(Checksum Error) 등을 자동으로 감지하여 표시해 줍니다. ('Analyze -> Expert Info')

### 의료 로봇 분야 활용
로봇 시스템에서 통신 장애가 발생했을 때, Wireshark를 사용하여 다음과 같은 문제를 진단할 수 있습니다.
-   **지연 시간 측정**: 요청(Request) 패킷과 응답(Response) 패킷 사이의 시간 차이(Delta Time)를 측정하여 응답 지연이 발생하는지 확인할 수 있습니다.
-   **데이터 무결성 검증**: 전송된 데이터가 손상되지 않았는지(CRC Error), 순서가 뒤바뀌지 않았는지(Sequence Number Error) 확인할 수 있습니다.
-   **프로토콜 동작 확인**: SOME/IP나 DoIP와 같은 상위 프로토콜이 규격에 맞게 동작하는지(Service ID, Method ID 등) 검증할 수 있습니다.

## Reference
- [Wireshark Official Website](https://www.wireshark.org/)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
