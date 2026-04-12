# 패킷의 흐름 — 장치 바깥과 안쪽

## 전송 단위 (PDU, Protocol Data Unit)

네트워크 계층마다 데이터를 부르는 이름이 다르다. 각 계층은 이전 계층의 데이터를 감싸면서(캡슐화) 자기 계층의 헤더를 붙인다.

```
Application          Data
                      |
                      v  캡슐화
Transport        +----------+------+
                 | TCP Hdr  | Data |                    = Segment / Datagram
                 +----------+------+
                      |
                      v  캡슐화
Internet         +--------+----------+------+
                 | IP Hdr | TCP Hdr  | Data |           = Packet
                 +--------+----------+------+
                      |
                      v  캡슐화
Network Access   +----------+--------+----------+------+-----+
                 | Eth Hdr  | IP Hdr | TCP Hdr  | Data | FCS |  = Frame
                 +----------+--------+----------+------+-----+
```

| 계층      | 전송 단위                          | 헤더에 담긴 핵심 정보                              |
| ------- | ------------------------------ | ----------------------------------------- |
| 애플리케이션  | Data (메시지)                     | HTTP 요청/응답, DNS 질의 등                      |
| 전송      | Segment (TCP) / Datagram (UDP) | 출발지/목적지 **포트**, 시퀀스 번호, 체크섬               |
| 인터넷     | Packet                         | 출발지/목적지 **IP 주소**, TTL, 프로토콜 번호           |
| 네트워크 접근 | Frame                          | 출발지/목적지 **MAC 주소**, EtherType, FCS(에러 검출) |

수신 시에는 반대로 각 계층이 자기 헤더를 벗겨내면서(역캡슐화) 위로 올린다.

---

## DNS 질의 과정

`curl http://google.com`을 실행하면, HTTP 요청을 보내기 **전에** 먼저 `google.com`이 어떤 IP인지 알아내야 한다. 이 과정이 DNS(Domain Name System) 질의다.

### 클라이언트 측 흐름

```
[ curl 프로세스 ]
    |
    |  (1) getaddrinfo("google.com") 호출
    |      C 라이브러리(glibc)의 이름 해석 함수
    v
[ glibc 리졸버 ]
    |
    |  (2) /etc/nsswitch.conf 확인
    |      hosts: files dns myhostname
    |             -----  ---
    |               |     +-> 없으면 DNS 서버에 질의
    |               +-------> 먼저 /etc/hosts 파일 검색
    |
    |  (3) /etc/hosts 에서 google.com 검색
    |      -> 없음 (보통 localhost만 등록되어 있음)
    |
    |  (4) /etc/resolv.conf 에서 DNS 서버 주소 확인
    |      nameserver 8.8.8.8    <- nmcli로 설정한 값
    |
    |  (5) DNS 질의 패킷 생성
    |      UDP, 목적지 포트 53, 질의: "google.com의 A 레코드"
    v
[ 커널 네트워크 스택 ]
    |
    |  UDP 패킷 -> IP 패킷 -> Ethernet Frame
    |  dst IP = 8.8.8.8 (DNS 서버)
    v
  NIC -> 공유기 -> ISP -> Google Public DNS (8.8.8.8) 로 전송
```

---

### DNS 서버 측 — 재귀 질의 (Recursive Query)

DNS 서버(8.8.8.8)가 `google.com`의 IP를 모르면, 루트부터 단계별로 물어간다.

```
[ Google Public DNS  8.8.8.8 ]
    |
    |  캐시에 google.com 있는지 확인
    |  -> 캐시에 있으면 바로 응답 (대부분 이 경우)
    |  -> 캐시에 없으면 재귀 질의 시작:
    v

(1) 루트 네임서버에 질의
    "google.com 의 IP가 뭐야?"
    |
    v
[ 루트 네임서버  (.) ]
    |  "나는 모르지만, .com 담당 서버는 알려줄게"
    |  -> 응답: .com TLD 네임서버 주소
    v

(2) .com TLD 네임서버에 질의
    "google.com 의 IP가 뭐야?"
    |
    v
[ .com TLD 네임서버 ]
    |  "나는 모르지만, google.com 담당 서버는 알려줄게"
    |  -> 응답: google.com 권한 네임서버(ns1.google.com) 주소
    v

(3) google.com 권한 네임서버에 질의
    "google.com 의 A 레코드가 뭐야?"
    |
    v
[ ns1.google.com  (권한 네임서버) ]
    |  "google.com = 142.250.196.110 이야"  (TTL: 300초)
    |  -> 최종 응답
    v

[ Google Public DNS  8.8.8.8 ]
    |  결과를 캐시에 저장 (TTL 동안 유지)
    |  클라이언트에게 응답 반환
    v

내 서버 NIC <- 공유기 <- ISP <- DNS 응답 도착

[ glibc 리졸버 ]
    |  응답 수신: google.com = 142.250.196.110
    |  결과를 로컬 캐시에 저장
    v
[ curl 프로세스 ]
    "이제 142.250.196.110:80 으로 HTTP 요청을 보내면 된다"
```

---

### DNS 레코드 유형

| 레코드       | 의미                                  | 예시                                 |
| --------- | ----------------------------------- | ---------------------------------- |
| **A**     | 도메인 -> IPv4 주소                      | `google.com -> 142.250.196.110`    |
| **AAAA**  | 도메인 -> IPv6 주소                      | `google.com -> 2404:6800:4004:...` |
| **CNAME** | 도메인 -> 다른 도메인 (별칭)                  | `www.google.com -> google.com`     |
| **MX**    | 메일 서버 지정                            | `google.com -> smtp.google.com`    |
| **NS**    | 해당 도메인의 권한 네임서버                     | `google.com -> ns1.google.com`     |
| **PTR**   | IP -> 도메인 (역방향 조회)                  | `142.250.196.110 -> google.com`    |
| **SOA**   | 도메인의 관리 정보 (Primary NS, 관리자, 시리얼 등) | 존(zone)의 메타데이터                     |

> **왜 UDP 포트 53인가?** DNS 질의는 보통 매우 작은 패킷(수백 바이트)이므로 TCP 3-way Handshake의 오버헤드가 비효율적이다. 그래서 UDP를 사용한다. 단, 응답이 512바이트를 초과하거나 존 전송(zone transfer) 시에는 TCP를 사용한다.

---

## Part 1 — 장치 바깥: 네트워크를 통한 패킷의 여정

DNS 질의가 끝나 목적지 IP(142.250.196.110)를 알게 된 상태에서, 실제 HTTP 요청 패킷이 어떻게 이동하는지 따라간다.

```
==========================================================
  내 서버  (172.30.1.100)
==========================================================

  [ curl 프로세스 ]
       |
       v   "google.com 에 HTTP 요청 보내줘"
  [ 커널 네트워크 스택 ]
       |    DNS 질의: google.com -> 142.250.196.110
       |    목적지 IP가 내 서브넷(172.30.1.0/24)에 없음
       |    라우팅 테이블 -> default via 172.30.1.254 (게이트웨이)
       |    ARP -> 172.30.1.254 의 MAC 주소 조회 (게이트웨이의 MAC 주소) -> Frame 생성 위해)
       v
  [ NIC: wlp0s20f3 ]
       |    Frame 전송: dst MAC = 게이트웨이 MAC
       |                dst IP  = 142.250.196.110
       v
~~ Wi-Fi ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

==========================================================
  홉 1 — 공유기/라우터  (172.30.1.254)
==========================================================

  1. Frame 수신 -> dst MAC이 자기 것이므로 수신 확인
  2. Ethernet 헤더 벗김 -> IP Packet 확인
  3. 목적지 IP(142.250.196.110)가 자기 서브넷 아님 -> 라우팅
  4. NAT 수행: 출발지 IP 172.30.1.100 -> 공인 IP(211.xxx.xxx.xxx)로 변환
  5. 새 Frame 생성: dst MAC = ISP 라우터 MAC
       |
       v

==========================================================
  홉 2~6 — ISP(Internet Service Provider) 백본  (KT 네트워크)
==========================================================

  각 라우터가 반복:
    1. Frame 수신 -> Ethernet 헤더 벗김
    2. 목적지 IP 확인 -> 라우팅 테이블에서 다음 홉(next hop) 결정
    3. TTL 1 감소  (0이 되면 폐기 + ICMP Time Exceeded 반환)
    4. 새 Frame 생성 (dst MAC = 다음 홉 MAC) -> 전송

  112.189.118.181 -> 112.174.47.157 -> 112.174.84.133
       |
       v  ISP 피어링 포인트

==========================================================
  홉 7~ — Google 네트워크
==========================================================

  142.250.165.78 -> ... -> 목적지 서버 (142.250.196.110)

  최종 라우터가 목적지 서버와 같은 서브넷에 도달
  -> ARP로 서버 MAC 확인 -> Frame 직접 전달
       |
       v
  [ Google 웹 서버 ]
       |    HTTP 응답 생성
       |    역경로로 패킷 반환 (NAT 역변환 포함)
       v
  -> ISP -> 공유기 -> 내 서버 NIC -> curl 에 응답 도착

==========================================================
```

### 핵심 포인트

**라우터를 거칠 때마다 Frame은 새로 만들어진다.** IP Packet은 출발지~목적지까지 변하지 않지만(NAT 제외), Ethernet Frame의 MAC 주소는 매 홉마다 바뀐다. "다음 라우터의 MAC"으로 교체되는 것이다.

```
내 서버 -> 공유기:   src MAC = 내 NIC MAC      dst MAC = 공유기 MAC
공유기  -> ISP:      src MAC = 공유기 WAN MAC   dst MAC = ISP 라우터 MAC
ISP     -> Google:   src MAC = ISP 라우터 MAC   dst MAC = 다음 홉 MAC

(IP 주소는 출발지~목적지까지 동일, NAT 제외)
```

**NAT (Network Address Translation):** 가정용 공유기는 사설 IP(172.30.1.100)를 공인 IP(211.xxx)로 변환해서 외부로 보낸다. 응답이 돌아오면 NAT 테이블을 참조해 원래 사설 IP로 되돌린다. 이 덕분에 여러 장치가 하나의 공인 IP를 공유할 수 있다.

**TTL (Time To Live):** IP 헤더에 있는 필드로, 라우터를 거칠 때마다 1씩 감소한다. 0이 되면 패킷을 폐기하고 ICMP Time Exceeded를 보낸다. 라우팅 루프로 패킷이 영원히 떠도는 것을 방지한다. `tracepath`는 이 메커니즘을 이용해 경로를 추적한다.

---

## Part 2 — 장치 안쪽: 리눅스 커널의 패킷 처리 흐름

### 수신 경로 (Ingress) — 외부 → 애플리케이션

패킷이 NIC에 도착해서 애플리케이션까지 올라가는 전체 과정이다.

#### 1단계: 하드웨어 수신 — NIC → 커널 메모리

```
[ NIC 하드웨어 ]
    |
    |  (1) 전기/무선 신호 수신, Frame 재조립
    |  (2) DMA: Frame을 커널 메모리(Ring Buffer)에 복사
    |  (3) 하드웨어 인터럽트(IRQ) 발생 -> CPU에 "패킷 왔다" 알림
    v
[ 인터럽트 핸들러 — Top Half ]
    |
    |  (4) 최소한의 처리만 하고 softirq 스케줄링
    v
[ softirq / NAPI — Bottom Half ]
    |
    |  (5) Ring Buffer에서 Frame을 꺼내 sk_buff 구조체 생성
    v
    ... L2 처리로 계속
```

**(1) NIC가 전기/무선 신호를 수신**하면, 내부 하드웨어가 비트 스트림을 Ethernet Frame으로 복원한다. CRC(FCS)로 전송 중 에러 여부를 검증하고, 깨진 Frame은 여기서 바로 폐기한다.

**(2) DMA(Direct Memory Access)**는 NIC가 CPU를 거치지 않고 메인 메모리에 직접 데이터를 쓰는 방식이다. **Ring Buffer**는 NIC와 커널 사이의 원형 큐로, NIC가 한쪽에서 Frame을 넣고 커널이 반대쪽에서 꺼내간다. 큐가 가득 차면 새 Frame은 폐기된다(패킷 드롭).

**(3) NIC가 CPU에 하드웨어 인터럽트(IRQ)**를 보내 현재 작업을 중단시킨다. CPU는 즉시 인터럽트 핸들러로 점프한다.

**(4) 인터럽트 핸들러(Top Half)**가 실행되는 동안 해당 CPU의 다른 모든 작업이 멈춘다. 그래서 "패킷이 왔다"는 사실만 기록하고 즉시 빠져나온다. 실제 패킷 처리는 softirq(Bottom Half)에게 맡긴다.

| 구분 | 역할 | 상태 |
| --- | --- | --- |
| **Top Half** | IRQ 핸들러, 빠르게 끝내야 하는 긴급 처리 | IRQ 비활성화 (다른 작업 멈춤) |
| **Bottom Half** | softirq, 나중에 해도 되는 무거운 처리 | IRQ 활성화 (다른 작업 가능) |

**(5) NAPI(New API)**는 패킷이 적을 때는 인터럽트 방식으로 동작하다가, 패킷이 많아지면 폴링(polling) 방식으로 전환한다. 패킷 하나마다 인터럽트가 오면 CPU가 마비되므로(인터럽트 폭풍), 모아서 한꺼번에 처리하는 것이다.

Ring Buffer에서 Frame을 꺼내면 **sk_buff** 구조체를 생성한다. 이것은 커널에서 패킷을 표현하는 핵심 자료구조다:

| sk_buff 필드 | 내용 |
| --- | --- |
| data | 실제 패킷 바이트 (Ethernet 헤더부터 페이로드까지) |
| dev | 수신한 네트워크 인터페이스 (wlp0s20f3) |
| protocol | 프로토콜 (IPv4, ARP 등) |
| tstamp | 수신 타임스탬프 |
| len | 패킷 길이 |

이후 모든 계층이 이 sk_buff를 공유하면서 **헤더 포인터만 이동**시킨다. 복사 없이 포인터 조작으로 역캡슐화를 처리하므로 고성능이다.

#### 2단계: L2 처리 — Ethernet Frame 파싱

```
[ L2 처리 — 네트워크 접근 계층 ]
    |
    |  (6) Ethernet 헤더 파싱 (14바이트)
    |      dst MAC 필터링 -> 내 것이면 수신, 아니면 폐기
    |  (7) EtherType 확인 -> 상위 프로토콜 결정
    |      Ethernet 헤더 벗기고 상위 계층으로 전달
    v
    ... netfilter PREROUTING으로 계속
```

**(6)** 14바이트 Ethernet 헤더를 읽는다: `dst MAC (6B) + src MAC (6B) + EtherType (2B)`

dst MAC 필터링 규칙:

| dst MAC | 동작 | 예시 |
| --- | --- | --- |
| 내 MAC과 일치 | 수신 | 나에게 온 Frame |
| ff:ff:ff:ff:ff:ff | 수신 | 브로드캐스트 (ARP 요청 등) |
| 멀티캐스트 MAC | 수신 | 가입한 그룹이면 |
| 그 외 | 폐기 | 남의 Frame |

promiscuous 모드가 켜져 있으면 모든 Frame을 수신한다. `tcpdump`, `Wireshark` 같은 패킷 캡처 도구가 이 모드를 사용한다.

**(7)** EtherType 필드로 상위 프로토콜을 결정한다:

| EtherType | 프로토콜 |
| --- | --- |
| 0x0800 | IPv4 |
| 0x0806 | ARP |
| 0x86DD | IPv6 |

Ethernet 헤더를 벗기고(sk_buff 포인터를 14바이트 전진) 해당 프로토콜 핸들러로 전달한다.

#### 3단계: netfilter PREROUTING → 라우팅 결정

```
 ===========================================
 = netfilter: PREROUTING 체인              =
 ===========================================
    |
    |  (8) conntrack: 연결 상태 판별 (NEW/ESTABLISHED/RELATED/INVALID)
    |      DNAT: 목적지 IP/포트 변환 (필요 시)
    v
[ L3 처리 — 라우팅 결정 ]
    |
    |  (9)  IP 헤더 파싱 (20바이트)
    |  (10) 라우팅 결정 (FIB lookup)
    |
    +-------> dst IP가 내 IP -> INPUT 경로
    +-------> dst IP가 다른 IP -> FORWARD 경로
    v
    ... INPUT 경로를 따라감
```

**(8) PREROUTING**은 커널에 들어온 패킷이 가장 먼저 거치는 netfilter 훅(hook)이다. 여기서 두 가지가 일어난다:

**conntrack (연결 추적)** — 이 패킷이 기존 연결의 일부인지 판별한다:

| conntrack 상태 | 의미 | 예시 |
| --- | --- | --- |
| NEW | 새 연결의 첫 패킷 | TCP SYN |
| ESTABLISHED | 이미 수립된 연결의 패킷 | ACK, 데이터 |
| RELATED | 기존 연결과 관련된 새 연결 | FTP 데이터 채널 |
| INVALID | 어디에도 속하지 않는 비정상 패킷 | 순서 안 맞는 패킷 |

conntrack이 ESTABLISHED로 판별하면 INPUT 체인에서 규칙을 하나하나 검사하지 않고 바로 통과시킨다(성능 최적화).

**DNAT (목적지 NAT)** — 목적지 IP/포트를 다른 값으로 변환한다. 예: 공유기가 외부 포트 8080을 내부 서버 포트 80으로 포워딩. 라우팅 결정 전에 해야 하므로 PREROUTING에서 처리한다.

**(9) IP 헤더 파싱 (20바이트)** — 주요 필드:

| 필드 | 내용 |
| --- | --- |
| src IP | 보낸 사람 IP |
| dst IP | 받을 사람 IP |
| TTL | 남은 홉 수 |
| Protocol | 상위 프로토콜 (6=TCP, 17=UDP, 1=ICMP) |
| Header Checksum | IP 헤더 무결성 검증 |

체크섬이 맞지 않으면 여기서 폐기한다.

**(10) 라우팅 결정 (FIB lookup)** — 커널의 라우팅 테이블(FIB)에서 dst IP를 검색한다. 가장 구체적인 경로(longest prefix match)를 선택한다:

```
172.30.1.0/24 -> 직접 전달 (같은 서브넷)
0.0.0.0/0     -> 게이트웨이로 전달 (default route)
```

- dst IP가 내 IP → **INPUT 경로** (로컬 수신)
- dst IP가 다른 IP → **FORWARD 경로** (라우터 역할, `ip_forward` 활성화 시)

#### 4단계: netfilter INPUT — 방화벽 검사

```
 ===========================================
 = netfilter: INPUT 체인                   =
 ===========================================
    |
    |  (11) conntrack 상태 확인
    |       -> ESTABLISHED/RELATED: 즉시 ACCEPT
    |       -> NEW: firewalld 규칙 평가
    |       -> INVALID: DROP
    |
    |  결과: ACCEPT / REJECT / DROP
    v
    ... ACCEPT 시 L4 처리로 계속
```

**(11) INPUT 체인**에서 firewalld 규칙이 평가된다. 먼저 conntrack 상태를 확인한다:

**ESTABLISHED / RELATED** → 즉시 ACCEPT (규칙 검사 생략). 기존 연결의 응답 패킷은 매번 규칙을 확인할 필요가 없다. 이 덕분에 "나가는 건 허용, 들어오는 건 차단"이 가능하다. 내가 먼저 보낸 요청의 응답은 ESTABLISHED이므로 통과한다.

**NEW / INVALID** → firewalld 규칙 평가:

firewalld 평가 순서:
1. 출발지 IP 기반 zone 매칭 (`--add-source`로 설정한 것)
2. 수신 인터페이스 기반 zone 매칭
3. 기본 zone (보통 public)

해당 zone에서의 매칭:
- 허용된 서비스(ssh, http 등)의 포트와 일치? → ACCEPT
- 허용된 포트(`--add-port`)와 일치? → ACCEPT
- Rich Rule과 일치? → 해당 액션(accept/reject/drop/log)
- 위 어디에도 해당 없음 → zone의 target 적용

| zone target | 동작 |
| --- | --- |
| default | REJECT (거부 응답 반환, 클라이언트가 즉시 에러를 받음) |
| DROP | 묵살 (응답 없음, 클라이언트는 타임아웃까지 대기) |
| ACCEPT | 모두 허용 (trusted zone) |

> **loopback(127.0.0.1) 트래픽은 이 체인을 거치지 않는다.** `lo` 인터페이스는 trusted zone에 속해 무조건 통과한다. 그래서 `curl http://localhost/`로는 Rich Rule 로그가 안 남는다.

#### 5단계: L4 처리 — 소켓 검색 및 TCP 상태 머신

```
[ L4 처리 — 전송 계층 ]
    |
    |  (12) Protocol 필드 -> TCP/UDP/ICMP 결정
    |  (13) TCP/UDP 헤더 파싱
    |  (14) 소켓 검색: LISTEN 중인 소켓 찾기
    |  (15) TCP 상태 머신 처리
    v
[ 소켓 계층 ]
    |
    |  (16) 수신 버퍼(Recv-Q)에 데이터 대기
    v
[ 사용자 공간: 애플리케이션 ]
    (17) read() / recv() 시스템 콜로 데이터 수신
    (18) 애플리케이션 로직 처리 -> 응답 생성
```

**(12)** IP 헤더의 Protocol 필드로 전송 프로토콜을 결정한다: 6=TCP, 17=UDP, 1=ICMP

**(13)** 헤더 파싱:

| 프로토콜 | 헤더 크기 | 주요 필드 |
| --- | --- | --- |
| TCP | 20바이트 + 옵션 | src/dst port, seq number, ack number, flags(SYN/ACK/FIN/RST), window size, checksum |
| UDP | 8바이트 | src/dst port, length, checksum |

**(14) 소켓 검색 (socket lookup)** — dst IP + dst port 조합으로 커널의 소켓 해시 테이블을 검색해서 해당 포트에서 LISTEN 중인 소켓을 찾는다. `ss -tlnp`로 보이는 LISTEN 소켓들이다.

찾지 못하면:
- TCP → RST 패킷 반환 (클라이언트에 "Connection refused")
- UDP → ICMP Port Unreachable 반환

**(15) TCP 상태 머신 처리:**

**처음 연결 시 (3-way Handshake):**

```
클라이언트             서버
    |--- SYN -------->|  SYN Queue에 저장
    |<-- SYN-ACK -----|
    |--- ACK -------->|  Accept Queue로 이동, ESTABLISHED
```

SYN Queue가 가득 차면 **SYN Cookie**로 대응한다(DDoS 방어).

**데이터 수신 시:**
- 시퀀스 번호 확인 (순서 맞는지, 중복인지, 누락인지)
- 순서 맞으면 → 수신 버퍼에 적재
- 순서 틀리면 → Out-of-Order Queue에 보관, 재정렬 대기
- ACK를 반환해서 "받았다"고 알려줌

**TCP 흐름 제어:**
수신 버퍼가 가득 차면 Window Size를 0으로 알려서 "더 이상 보내지 마"라고 상대에게 통보한다(Zero Window). `ss` 명령어의 Recv-Q가 높으면 이 상태일 가능성이 있다.

**(16)** 소켓 수신 버퍼(Recv-Q)에 데이터가 도착하면 소켓에 연결된 프로세스를 깨운다. 프로세스가 `read()`에서 블로킹 대기 중이었다면 깨어나고, `epoll`/`select`를 쓰는 경우 이벤트를 발생시킨다.

**(17)** 프로세스가 `read()` / `recv()` 시스템 콜을 호출하면, 커널 수신 버퍼에서 사용자 공간 버퍼로 데이터를 복사한다. 이때 커널 공간 → 사용자 공간 전환(context switch)이 발생한다.

**(18)** 애플리케이션이 요청을 처리하고 응답을 생성한다.

---

### 송신 경로 (Egress) — 애플리케이션 → 외부

애플리케이션이 응답을 보낼 때의 과정이다. 수신의 역순이다.

#### 1단계: 애플리케이션 → 소켓 → L4

```
[ 사용자 공간: 애플리케이션 ]
    |
    |  (1) write() / send() 시스템 콜
    v
[ 소켓 계층 ]
    |
    |  (2) 송신 버퍼(Send-Q)에 적재
    v
[ L4 처리 — 전송 계층 ]
    |
    |  (3) TCP: 세그먼트 분할 + 시퀀스 번호 + 체크섬
    |  (4) TCP/UDP 헤더 추가
    v
    ... L3 처리로 계속
```

**(1)** `write()` / `send()` 시스템 콜로 데이터를 전송한다. 사용자 공간 버퍼의 데이터가 커널 소켓 버퍼로 복사된다. **이 시점에서 `write()`가 리턴되지만, 아직 네트워크로 나간 건 아니다.** 커널 버퍼에 들어갔을 뿐이다.

**(2)** 데이터를 소켓 송신 버퍼(Send-Q)에 적재한다. TCP의 경우, 송신 버퍼가 가득 차면 `write()`가 블로킹된다(상대가 ACK를 보내야 버퍼가 비워짐 = 흐름 제어). UDP는 버퍼링 없이 바로 전송 시도한다.

**(3) TCP 처리:**
- **세그먼트 분할**: 데이터가 MSS(Maximum Segment Size, 보통 1460바이트)보다 크면 여러 세그먼트로 나눈다
- 각 세그먼트에 **시퀀스 번호** 부여 (수신 측에서 순서 복원용)
- **체크섬** 계산 (전송 중 데이터 손상 감지용)

**UDP 처리:**
- 분할 없이 데이터그램 하나를 생성한다
- 체크섬 계산 (선택적이지만 보통 활성화)

**(4) TCP/UDP 헤더 추가:**

| 필드 | 내용 |
| --- | --- |
| src port | 자동 할당된 임시 포트 (ephemeral, 32768~60999) |
| dst port | 목적지 서비스 포트 (예: 80, 443) |

#### 2단계: L3 → netfilter OUTPUT/POSTROUTING

```
[ L3 처리 — 인터넷 계층 ]
    |
    |  (5) IP 헤더 추가 (20바이트)
    |  (6) 라우팅 결정 (FIB lookup)
    v
 ===========================================
 = netfilter: OUTPUT 체인                  =
 ===========================================
    |  (7) 아웃바운드 방화벽 검사 (기본 ACCEPT)
    v
 ===========================================
 = netfilter: POSTROUTING 체인             =
 ===========================================
    |  (8) SNAT / Masquerade (필요 시)
    v
    ... L2 처리로 계속
```

**(5) IP 헤더 추가 (20바이트):**

| 필드 | 값 |
| --- | --- |
| src IP | 출력 인터페이스의 IP (172.30.1.100) |
| dst IP | 목적지 IP (142.250.196.110) |
| TTL | 64 (리눅스 기본값) |
| Protocol | 6(TCP) 또는 17(UDP) |
| Total Length | IP 헤더 + TCP/UDP 헤더 + 데이터 |

**(6) 라우팅 결정 (FIB lookup)** — dst IP를 라우팅 테이블에서 검색해서 출력 인터페이스(`wlp0s20f3`)와 next hop(`172.30.1.254`)을 결정한다.

**(7) OUTPUT 체인** — 로컬에서 생성된 패킷에 대한 방화벽 검사. 외부에서 들어오는 패킷은 INPUT 체인이 담당하고, 내부에서 생성되어 나가는 패킷은 OUTPUT 체인이 담당한다. 기본 정책이 ACCEPT이므로 대부분 통과한다.

**(8) POSTROUTING 체인:**

| 기능 | 설명 |
| --- | --- |
| SNAT | 출발지 IP를 다른 IP로 변환 |
| Masquerade | SNAT의 변형, 동적 IP 환경에서 사용 (예: 공유기가 사설→공인 IP 변환) |

일반 서버에서는 보통 아무 것도 하지 않고 통과한다. NAT 게이트웨이 역할을 하는 경우에만 동작한다.

#### 3단계: L2 → 디바이스 드라이버 → NIC

```
[ L2 처리 — 네트워크 접근 계층 ]
    |
    |  (9)  next hop MAC 조회 (ARP)
    |  (10) Ethernet 헤더 추가 (14바이트)
    |  (11) FCS 추가 (4바이트)
    v
[ 디바이스 드라이버 ]
    |
    |  (12) sk_buff -> TX Ring Buffer 적재
    v
[ NIC 하드웨어 ]
    (13) DMA -> 전기/무선 신호 전송
         -> 네트워크로 출발
```

**(9) next hop IP의 MAC 주소 조회** — 커널의 ARP 캐시(Neighbor Table)에서 검색한다(`ip neigh show`로 확인 가능). 캐시에 없으면 ARP 요청 브로드캐스트를 보내고 응답을 대기한다. 이 동안 패킷은 대기열(qdisc)에서 대기한다.

**(10) Ethernet 헤더 추가 (14바이트):**

| 필드 | 값 |
| --- | --- |
| dst MAC | next hop(게이트웨이)의 MAC |
| src MAC | 내 NIC의 MAC |
| EtherType | 0x0800 (IPv4) |

**(11) FCS(Frame Check Sequence)** — 4바이트 CRC 체크섬을 Frame 끝에 붙인다. 수신 측 NIC가 이 값으로 전송 오류를 검증한다.

**(12)** sk_buff를 NIC의 TX Ring Buffer에 적재한다. **qdisc(큐잉 규칙)**를 거쳐 전송 순서가 결정된다(기본: pfifo_fast — 우선순위 큐 3개로 분류). TX Ring Buffer가 가득 차면 패킷은 대기하거나 폐기된다.

**(13)** NIC가 DMA로 Ring Buffer에서 Frame을 읽어 전기/무선 신호로 전송한다. 전송 완료 후 TX 인터럽트를 발생시켜 커널에 "전송 끝"을 알린다. 그러면 커널이 sk_buff를 해제(메모리 반환)한다.

---

### netfilter 체인 전체 구조

수신/송신/전달 경로에서 netfilter가 어디에 개입하는지 한눈에 보는 다이어그램이다.

```
                       패킷 도착
                          |
                          v
                    +--------------+
                    |  PREROUTING  |
                    +------+-------+
                           |
                           v
                      라우팅 결정
                      /        \
                     v          v
              (나에게 온 것)  (다른 곳으로 전달)
                  |               |
                  v               v
            +-----------+   +-----------+
            |   INPUT   |   |  FORWARD  |
            | (firewalld|   |           |
            |  규칙)    |   +-----+-----+
            +-----+-----+        |
                  |               v
                  v         +--------------+
            로컬 프로세스   |  POSTROUTING  |
                  |         +------+-------+
                  v                |
             응답 생성             v
                  |           패킷 송신
                  v
            +-----------+
            |  OUTPUT   |
            +-----+-----+
                  |
                  v
            +--------------+
            |  POSTROUTING  |
            +------+-------+
                   |
                   v
              패킷 송신
```

---

### netfilter 상세 — 테이블과 체인의 관계

위 다이어그램의 각 체인(PREROUTING, INPUT, ...)은 사실 내부에 **여러 테이블**을 가지고 있다. 테이블마다 역할이 다르다.

```
각 체인 안에서 테이블이 실행되는 순서:

PREROUTING 체인:
  [raw] -> [conntrack] -> [mangle] -> [nat(DNAT)]
    |          |              |            |
    |          |              |            +-> 목적지 IP/포트 변환
    |          |              +-> 패킷 헤더 수정 (TTL, TOS 등)
    |          +-> 연결 추적 시작 (NEW/ESTABLISHED/RELATED 판별)
    +-> conntrack 추적 제외 설정 (NOTRACK)

INPUT 체인:
  [mangle] -> [filter] -> [nat]
                 |
                 +-> 여기가 firewalld 규칙이 적용되는 곳
                     ACCEPT / REJECT / DROP 판정

OUTPUT 체인:
  [raw] -> [conntrack] -> [mangle] -> [nat(DNAT)] -> [filter]
                                                        |
                                                        +-> 아웃바운드 필터링

FORWARD 체인:
  [mangle] -> [filter]
                 |
                 +-> 전달(라우터 역할) 패킷 필터링

POSTROUTING 체인:
  [mangle] -> [nat(SNAT/Masquerade)]
                 |
                 +-> 출발지 IP 변환
```

| 테이블 | 역할 | 사용 예시 |
| --- | --- | --- |
| **raw** | conntrack 추적 여부 결정 | 대량 트래픽에서 추적 비용을 줄일 때 NOTRACK 설정 |
| **mangle** | 패킷 헤더 수정 | TTL 변경, TOS/DSCP 마킹 (QoS) |
| **nat** | 주소/포트 변환 | DNAT(포트 포워딩), SNAT(공유기 NAT), Masquerade |
| **filter** | 패킷 허용/차단 판정 | firewalld의 서비스/포트/Rich Rule이 여기에 등록됨 |

> **firewalld와의 관계:** `firewall-cmd`로 규칙을 추가하면, firewalld가 nftables 명령으로 변환해서 주로 **filter 테이블의 INPUT 체인**에 넣는다. 즉 `firewall-cmd --add-service=http`는 결국 nftables의 filter/INPUT에 "TCP 80 ACCEPT" 규칙을 추가하는 것이다.

---

### conntrack 상태와 방화벽의 관계

conntrack(연결 추적)은 netfilter의 핵심이다. "상태 기반 방화벽"이 가능한 이유가 바로 conntrack이다.

```
conntrack이 없다면 (무상태 방화벽):
  클라이언트 -> 서버: SYN (포트 80)      -> INPUT에서 규칙 검사 -> ACCEPT
  서버 -> 클라이언트: SYN-ACK            -> OUTPUT에서 규칙 검사 -> ACCEPT
  클라이언트 -> 서버: ACK + HTTP GET     -> INPUT에서 규칙 검사 -> ACCEPT
  매 패킷마다 규칙을 처음부터 전부 검사해야 한다.

conntrack이 있으면 (상태 기반 방화벽):
  클라이언트 -> 서버: SYN (포트 80)      -> conntrack: NEW -> 규칙 검사 -> ACCEPT
  서버 -> 클라이언트: SYN-ACK            -> conntrack: ESTABLISHED -> 즉시 ACCEPT
  클라이언트 -> 서버: ACK + HTTP GET     -> conntrack: ESTABLISHED -> 즉시 ACCEPT
  첫 패킷만 규칙을 검사하고, 이후는 conntrack이 바로 통과시킨다.
```

| conntrack 상태 | 의미 | 방화벽 처리 |
| --- | --- | --- |
| **NEW** | 새 연결의 첫 패킷 | 규칙을 전부 검사해서 ACCEPT/REJECT/DROP 결정 |
| **ESTABLISHED** | 이미 허용된 연결의 후속 패킷 | 규칙 검사 없이 즉시 ACCEPT |
| **RELATED** | 기존 연결과 관련된 새 연결 | 즉시 ACCEPT (예: FTP 데이터 채널, ICMP 에러) |
| **INVALID** | 어디에도 속하지 않는 비정상 패킷 | DROP (순서가 맞지 않거나 위조된 패킷) |

`firewall-cmd`로 서비스를 허용하면 **NEW 상태의 첫 패킷**만 해당 규칙에 매칭된다. 나머지는 conntrack이 알아서 처리한다.

---

### 핵심 용어 정리

| 용어 | 설명 |
| --- | --- |
| **DMA (Direct Memory Access)** | NIC가 CPU를 거치지 않고 직접 메모리에 데이터를 쓰는 방식. CPU 부하를 줄임 |
| **Ring Buffer** | NIC와 커널 사이의 원형 큐. NIC가 Frame을 넣고, 커널이 꺼내감. 가득 차면 패킷 드롭 |
| **IRQ (Interrupt Request)** | NIC가 CPU에 "새 패킷 도착"을 알리는 하드웨어 신호 |
| **Top Half / Bottom Half** | Top Half(IRQ 핸들러): 최소 처리 후 즉시 반환. Bottom Half(softirq): 실제 패킷 처리 |
| **softirq / NAPI** | 패킷을 하나씩 인터럽트 처리하면 느리므로, 모아서 한꺼번에 처리하는 메커니즘 |
| **sk_buff** | 커널이 패킷을 표현하는 핵심 구조체. 포인터 이동으로 캡슐화/역캡슐화 처리 |
| **conntrack** | netfilter의 연결 추적 모듈. 상태 기반 방화벽의 핵심 (NEW/ESTABLISHED/RELATED/INVALID) |
| **NAPI (New API)** | 대량 패킷 수신 시 인터럽트 대신 폴링으로 전환해 성능을 높이는 커널 인터페이스 |
| **FIB (Forwarding Information Base)** | 커널의 라우팅 테이블. longest prefix match로 경로를 결정 |
| **qdisc (Queuing Discipline)** | 송신 패킷의 큐잉/스케줄링 규칙. 기본: pfifo_fast |
| **MSS (Maximum Segment Size)** | TCP 세그먼트 하나에 담을 수 있는 최대 데이터 크기. 보통 1460바이트 |
| **SYN Cookie** | SYN Queue가 가득 찰 때 사용하는 DDoS 방어 메커니즘 |

---

## 요약: 패킷 하나의 전체 생애

`curl http://google.com` 한 줄이 실행될 때 일어나는 일을 처음부터 끝까지:

```
[ curl 프로세스 ]
  -> getaddrinfo("google.com")
    -> glibc 리졸버: /etc/hosts -> /etc/resolv.conf -> DNS 질의(UDP:53)
      -> DNS 응답: 142.250.196.110
  -> connect() + write() 시스템 콜
    -> 커널: TCP SYN (3-way Handshake 시작)
      -> 커널: IP 패킷 (dst=142.250.196.110, TTL=64)
        -> 커널: netfilter OUTPUT(filter) -> POSTROUTING(nat) 통과
          -> 커널: 라우팅 -> 게이트웨이 172.30.1.254
            -> 커널: ARP로 게이트웨이 MAC 확인
              -> 커널: Ethernet Frame 생성
                -> 디바이스 드라이버: TX Ring Buffer 적재
                  -> NIC: DMA로 Frame 전송
                    -> Wi-Fi 신호로 공유기에 도달
                      -> 공유기: NAT(사설 -> 공인 IP), 라우팅
                        -> ISP 백본 (홉마다 Frame 재생성, TTL--)
                          -> Google 네트워크 도착
                            -> Google NIC: DMA -> IRQ -> softirq -> sk_buff
                              -> Google 커널: L2 -> PREROUTING(conntrack: NEW)
                                -> 라우팅(INPUT) -> INPUT(filter: ACCEPT)
                                  -> L4: TCP SYN-ACK 반환 -> Handshake 완료
                                    -> Google httpd: HTTP 요청 처리, 응답 생성

[ 응답은 역경로로 되돌아옴 ]

Google httpd -> Google 커널 -> Google NIC
  -> ISP -> 공유기(NAT 역변환) -> 내 NIC
    -> 내 커널: DMA -> IRQ -> softirq -> sk_buff
      -> L2 파싱 -> PREROUTING(conntrack: ESTABLISHED) -> 즉시 ACCEPT
        -> L4: TCP, 소켓 수신 버퍼에 적재
          -> curl: read()로 HTTP 응답 수신
            -> 터미널에 출력
```

