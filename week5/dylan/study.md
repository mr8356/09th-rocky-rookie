# Rocky Linux 스터디 — 5주차
## 네트워크 구성과 보안

> **기준 버전:** Rocky Linux 10 (RHEL 10 기반, 커널 6.12.0)

---

## 이번 주 전체 맥락

```
패킷이 서버에 도달하는 경로:

외부 클라이언트
    ↓ 네트워크
NIC (enp1s0)             ← 1. 물리 인터페이스 (NetworkManager 관리)
    ↓
커널 네트워크 스택
    ↓
nftables / firewalld     ← 2. 방화벽 (허용/차단 결정)
    ↓
프로세스 (httpd 등)
    ↓
파일 접근 시도
    ↓
DAC 검사 (rwxr-xr-x)    ← 3. 기존 Unix 권한
    ↓
SELinux MAC 검사         ← 4. SELinux (추가 보안층)
    ↓
허용 또는 거부
```

---

## 1. 네트워크 기본 개념

### 1-1. IP 주소와 서브넷

```
IPv4 주소: 192.168.1.100/24

192.168.1.100  ← 호스트 주소 (이 장비)
/24            ← 서브넷 마스크 = 255.255.255.0
               ← 앞 24비트가 네트워크, 뒤 8비트가 호스트

/24 서브넷에서:
  네트워크 주소:   192.168.1.0   (첫 번째, 사용 불가)
  브로드캐스트:    192.168.1.255 (마지막, 사용 불가)
  사용 가능 범위:  192.168.1.1 ~ 192.168.1.254 (254개)

자주 쓰는 서브넷:
  /24  = 255.255.255.0   → 254개 호스트  (소규모 LAN)
  /16  = 255.255.0.0     → 65534개 호스트
  /8   = 255.0.0.0       → 대규모 네트워크
```

사설 IP 대역 (인터넷에서 라우팅 안 됨):

```
10.0.0.0/8        → 대기업, 클라우드 VPC
172.16.0.0/12     → 중소규모
192.168.0.0/16    → 가정, 소규모 사무실
```

### 1-2. 게이트웨이와 DNS

```
게이트웨이 (기본 라우터):
  내 서버가 모르는 목적지로 패킷 보낼 때 거치는 첫 번째 라우터
  보통 서브넷의 첫 번째 주소: 192.168.1.1

  192.168.1.100 → 8.8.8.8 (Google DNS) 보내기:
  → "192.168.1.x 이외 = 모른다"
  → → 게이트웨이(192.168.1.1)에게 전달
  → → ISP → 인터넷

DNS (Domain Name System):
  호스트명 → IP 주소 변환
  google.com → 142.250.x.x

  /etc/resolv.conf (현재 DNS 설정 확인):
  nameserver 8.8.8.8
  nameserver 1.1.1.1

  /etc/hosts (로컬 정적 매핑, DNS보다 먼저 확인):
  192.168.1.10  db-server
  192.168.1.20  app-server
```

### 1-3. 인터페이스 명명 규칙

RHEL 10은 예측 가능한(predictable) 인터페이스 이름을 사용합니다.

```
전통 방식 (구형):
  eth0, eth1 → 부팅 순서에 따라 달라질 수 있어서 불안정

예측 가능한 이름 (현재):
  enp1s0     → en(이더넷) p1(PCI 버스 1번) s0(슬롯 0번)
  enp3s0f1   → PCI 버스 3번, 슬롯 0번, 함수 1번
  eno1       → en(이더넷) o1(온보드, BIOS 인덱스 1)
  wlp2s0     → wl(무선) p2(PCI 버스 2번) s0(슬롯 0)
  lo         → loopback (127.0.0.1, 자기 자신)

→ 재부팅해도 같은 하드웨어 = 같은 이름 보장
```

---

## 2. NetworkManager와 연결 프로파일

### 2-1. NetworkManager 개념

```
NetworkManager = 네트워크 연결을 관리하는 시스템 데몬

장치 (Device):    물리적/가상 NIC (enp1s0)
연결 (Connection): 장치에 적용하는 설정 프로파일

하나의 장치에 여러 프로파일을 만들 수 있음:
  enp1s0 장치
    ├── "office" 프로파일  (192.168.1.100/24)
    └── "lab"    프로파일  (10.0.0.50/24)

연결 프로파일 저장 위치 (RHEL 10 keyfile 형식):
  /etc/NetworkManager/system-connections/
```

```bash
# NetworkManager 상태 확인
systemctl status NetworkManager

# 장치 목록
nmcli device status
# DEVICE   TYPE      STATE      CONNECTION
# enp1s0   ethernet  connected  Wired-1
# lo       loopback  unmanaged  --

# 연결 목록
nmcli connection show
```

### 2-2. nmcli — 커맨드라인 설정

> 출처: [RHEL 10 Configuring and managing networking](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/configuring_and_managing_networking/index)

#### 현재 상태 확인

```bash
nmcli device status             # 장치 목록과 상태
nmcli connection show           # 연결 프로파일 목록
nmcli connection show enp1s0    # 특정 연결 상세 정보
nmcli device show enp1s0        # 장치 상세 정보 (IP, DNS 등)
```

#### 정적 IP 설정

```bash
# 새 연결 프로파일 생성 (정적 IP)
nmcli connection add \
  type ethernet \
  con-name "server-static" \
  ifname enp1s0 \
  ipv4.method manual \
  ipv4.addresses "192.168.1.100/24" \
  ipv4.gateway "192.168.1.1" \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  connection.autoconnect yes

# 연결 활성화
nmcli connection up "server-static"
```

#### 기존 연결 수정

```bash
# IP 주소 변경
nmcli connection modify "server-static" \
  ipv4.addresses "192.168.1.200/24"

# DNS 추가
nmcli connection modify "server-static" \
  ipv4.dns "8.8.8.8"

# 수정 후 반드시 재활성화
nmcli connection up "server-static"
```

#### DHCP로 전환

```bash
nmcli connection modify "server-static" \
  ipv4.method auto \
  ipv4.addresses "" \
  ipv4.gateway "" \
  ipv4.dns ""
nmcli connection up "server-static"
```

#### 연결 활성화/비활성화

```bash
nmcli connection up   "server-static"   # 연결 활성화
nmcli connection down "server-static"   # 연결 비활성화
nmcli connection delete "server-static" # 프로파일 삭제

# 장치 껐다 켜기
nmcli device disconnect enp1s0
nmcli device connect enp1s0
```

#### 호스트명 설정

```bash
hostnamectl set-hostname "webserver01.example.com"
hostnamectl status   # 확인
```

### 2-3. nmtui — TUI(텍스트 UI) 설정

GUI 없는 서버에서 메뉴 방식으로 설정할 때 씁니다.

```bash
nmtui
```

```
┌──────────────────────────────────────┐
│ NetworkManager TUI                   │
│                                      │
│ Please select an option              │
│                                      │
│ Edit a connection       ← IP 설정    │
│ Activate a connection   ← 연결 활성화│
│ Set system hostname     ← 호스트명   │
│                                      │
│              <OK>  <Cancel>          │
└──────────────────────────────────────┘
```

조작 방법:

```
방향키     이동
Enter      선택
Space      체크박스 토글
Tab        다음 필드
Esc        이전 화면
```

정적 IP 설정 경로:

```
Edit a connection
  → 연결 선택 (또는 <Add>)
  → IPv4 CONFIGURATION: <Automatic> → <Manual> 으로 변경
  → Addresses 옆 <Show>
  → IP/prefix 입력 (예: 192.168.1.100/24)
  → Gateway 입력
  → DNS servers 입력
  → <OK>
Activate a connection
  → 연결 선택 후 활성화
```

---

## 3. 네트워크 상태 점검 도구

### 3-1. ip — 인터페이스·라우팅 조회

`ifconfig`의 현대적 대체 도구입니다.

```bash
# IP 주소 확인
ip address show              # 전체 (줄여서: ip a)
ip address show enp1s0       # 특정 인터페이스만

# 출력 예시:
# 2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
#     link/ether 52:54:00:xx:xx:xx brd ff:ff:ff:ff:ff:ff
#     inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic enp1s0
#     inet6 fe80::...

# 라우팅 테이블 확인
ip route show                # 전체 (줄여서: ip r)
# default via 192.168.1.1 dev enp1s0 proto dhcp
# 192.168.1.0/24 dev enp1s0 proto kernel scope link src 192.168.1.100

# 특정 목적지의 경로
ip route get 8.8.8.8

# 링크(L2) 상태 확인
ip link show                 # 줄여서: ip l
ip link show enp1s0

# 인터페이스 올리기/내리기 (임시, NM 우선)
ip link set enp1s0 up
ip link set enp1s0 down

# ARP 테이블 (IP ↔ MAC 매핑)
ip neigh show
```

라우팅 테이블 읽는 법:

```
default via 192.168.1.1 dev enp1s0
  → 목적지 모르면 → 192.168.1.1(게이트웨이)로 → enp1s0 인터페이스로

192.168.1.0/24 dev enp1s0 proto kernel scope link src 192.168.1.100
  → 192.168.1.x → 같은 네트워크 → 직접 전달 (게이트웨이 불필요)
```

### 3-2. ping — 연결 테스트

```bash
# 기본 사용
ping 8.8.8.8              # 무한 반복 (Ctrl+C 종료)
ping -c 4 8.8.8.8         # 4번만
ping -c 4 google.com      # 도메인 (DNS 해석 포함)

# 출력 해석:
# PING google.com (142.250.x.x): 56 data bytes
# 64 bytes from 142.250.x.x: icmp_seq=1 ttl=118 time=5.23 ms
# 64 bytes from 142.250.x.x: icmp_seq=2 ttl=118 time=4.98 ms
#
# --- google.com ping statistics ---
# 4 packets transmitted, 4 received, 0% packet loss
# rtt min/avg/max/mdev = 4.98/5.10/5.23/0.10 ms

# TTL (Time To Live): 라우터를 몇 번 거칠 수 있는지
# 응답 못 받으면:
#   Request timeout → 방화벽이 ICMP 차단했거나 호스트 없음
#   Destination Host Unreachable → 라우팅 경로 없음
```

네트워크 트러블슈팅 순서:

```bash
ping 127.0.0.1          # 1. 루프백 — 네트워크 스택 자체
ping 192.168.1.100      # 2. 자기 IP — 인터페이스
ping 192.168.1.1        # 3. 게이트웨이 — 로컬 네트워크
ping 8.8.8.8            # 4. 외부 IP — 인터넷 연결
ping google.com         # 5. 도메인 — DNS 해석
```

### 3-3. ss — 소켓 상태 조회

`netstat`의 현대적 대체 도구입니다.

```bash
# 현재 열린 포트 (리스닝 소켓)
ss -tlnp
# -t  TCP
# -l  listening 상태만
# -n  포트 번호로 (서비스명 변환 안 함)
# -p  프로세스 정보 포함

# 출력 예시:
# State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# LISTEN  0       128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1234))
# LISTEN  0       128    0.0.0.0:80          0.0.0.0:*          users:(("httpd",pid=1235))

# UDP 소켓도 포함
ss -tlnpu

# 모든 연결 상태 (ESTABLISHED 포함)
ss -tnp

# 특정 포트 필터
ss -tlnp sport = :80
ss -tlnp sport = :22

# 특정 프로세스가 사용하는 포트
ss -tlnp | grep httpd
```

자주 쓰는 포트:

```
22    SSH
80    HTTP
443   HTTPS
3306  MySQL/MariaDB
5432  PostgreSQL
6379  Redis
8080  HTTP 대체
```

### 3-4. traceroute / tracepath

```bash
# 패킷이 거치는 경로 추적
traceroute google.com
tracepath google.com     # traceroute 불가 시 대안

# 각 홉(라우터)의 응답 시간 확인
# * * * 는 해당 라우터가 ICMP 응답 안 함 (방화벽)
```

---

## 4. firewalld — 방화벽

> 출처: [RHEL 10 Using and configuring firewalld](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld)

### 4-1. firewalld 개념

```
firewalld = 동적 방화벽 관리 데몬
백엔드:    nftables (RHEL 10 기본)

특징:
  재시작 없이 규칙 변경 가능 (동적)
  영구(permanent) / 런타임(runtime) 설정 분리
  존(zone) 기반으로 인터페이스별 정책 적용
```

firewalld vs nftables:

```
firewalld:
  추상화 레이어 — zone/service/port 개념으로 쉽게 설정
  대부분의 일반적인 케이스는 이걸로 충분

nftables:
  저수준 직접 제어 — 복잡한 규칙, 성능 중요 시
  firewalld가 내부적으로 nftables를 사용함

iptables: 구형 도구 → RHEL 10에서 제거될 예정, 사용 비권장
```

### 4-2. 존 (Zone) — 신뢰 수준

존 = 네트워크 인터페이스에 적용하는 신뢰 수준 정책

```
주요 존 (신뢰도 낮음 → 높음):

drop       모든 수신 패킷 차단(응답도 없음), 송신만 허용
block      모든 수신 거부(ICMP 거부 응답), 송신만 허용
public     ← 기본값. 신뢰하지 않는 공개 네트워크
             SSH, DHCPv6-client만 허용
external   NAT 환경 외부 인터페이스
dmz        제한적 외부 접근 허용, 내부 접근 제한
work       직장 네트워크 환경
home       신뢰하는 홈 네트워크
internal   내부 네트워크, 대부분 신뢰
trusted    모든 수신 허용 (가장 신뢰)
```

기본 존 확인 및 변경:

```bash
firewall-cmd --get-default-zone          # 현재 기본 존
firewall-cmd --set-default-zone=public   # 기본 존 변경

# 인터페이스가 속한 존 확인
firewall-cmd --get-zone-of-interface=enp1s0

# 인터페이스를 특정 존에 할당
firewall-cmd --zone=internal --change-interface=enp1s0 --permanent
```

### 4-3. 런타임 vs 영구 설정

```
런타임(runtime):
  즉시 적용되지만 재부팅하면 사라짐
  → 테스트용

영구(permanent):
  --permanent 옵션 추가
  재부팅해도 유지
  → 설정 후 --reload 해야 런타임에도 반영

올바른 영구 설정 방법:
  1. --permanent 옵션으로 설정 (파일에 기록)
  2. firewall-cmd --reload (파일 → 런타임에 반영)

  또는:
  1. --permanent 없이 바로 테스트
  2. 잘 되면 --permanent 다시 실행
  3. firewall-cmd --reload
```

### 4-4. 포트와 서비스 허용

```bash
# 서비스 이름으로 허용 (권장)
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-service=ssh --permanent

# 포트 번호로 허용
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=5000-5010/tcp --permanent  # 범위

# 적용
firewall-cmd --reload

# 확인
firewall-cmd --zone=public --list-all
# public (active)
#   target: default
#   interfaces: enp1s0
#   services: cockpit dhcpv6-client http https ssh
#   ports: 8080/tcp

# 미리 정의된 서비스 목록 확인
firewall-cmd --get-services

# 서비스 정의 파일 위치
ls /usr/lib/firewalld/services/    # 기본 정의 (수정 금지)
ls /etc/firewalld/services/        # 커스텀 정의
```

서비스 vs 포트:

```
서비스 방식 (권장):
  --add-service=http
  → /usr/lib/firewalld/services/http.xml 참조
  → 포트 80/tcp 자동 허용
  → 이름으로 의도 명확

포트 방식:
  --add-port=80/tcp
  → 비표준 포트나 직접 지정 필요할 때
```

### 4-5. 서비스 허용 제거

```bash
firewall-cmd --zone=public --remove-service=http --permanent
firewall-cmd --zone=public --remove-port=8080/tcp --permanent
firewall-cmd --reload
```

### 4-6. 특정 IP 소스 허용/차단

```bash
# 특정 IP만 허용 (서비스 + 소스 IP 조합)
firewall-cmd --zone=internal --add-source=192.168.1.0/24 --permanent

# 특정 IP 차단
firewall-cmd --zone=block --add-source=203.0.113.0/24 --permanent

# 리치 룰 (Rich Rule) — 복잡한 조건
# 특정 IP에서 SSH만 허용
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept' \
  --permanent

# 특정 IP 차단
firewall-cmd --zone=public \
  --add-rich-rule='rule family="ipv4" source address="10.0.0.5" drop' \
  --permanent

firewall-cmd --reload
```

### 4-7. 상태 확인 명령어 정리

```bash
firewall-cmd --state                      # 실행 중인지 확인
firewall-cmd --get-active-zones           # 활성 존과 인터페이스
firewall-cmd --zone=public --list-all     # 특정 존 전체 설정
firewall-cmd --list-all-zones             # 모든 존 설정
firewall-cmd --zone=public --list-services
firewall-cmd --zone=public --list-ports
firewall-cmd --get-services               # 사용 가능한 서비스 목록
```

### 4-8. firewalld와 Docker

참고: Docker가 자체적으로 iptables/nftables 규칙을 직접 삽입하기 때문에 firewalld 규칙이 Docker 컨테이너 포트를 막지 못하는 경우가 있습니다. Docker Compose나 컨테이너 환경에서는 이 동작을 알고 있어야 합니다.

---

## 5. SELinux — 강제 접근 제어

> 출처: [RHEL 10 Getting started with SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/getting-started-with-selinux)

### 5-1. DAC vs MAC

```
DAC (Discretionary Access Control) — 기존 Unix 권한
  -rwxr-xr-x root root /usr/sbin/httpd
  → 파일 소유자(root)가 권한을 재량껏 설정
  → "httpd가 어떤 파일에 접근 가능한가?" 는 알 수 없음
  → httpd가 해킹당하면 httpd 권한으로 뭐든 할 수 있음

MAC (Mandatory Access Control) — SELinux
  → 정책이 중앙에서 강제 적용
  → "httpd 프로세스는 httpd_sys_content_t 파일만 읽을 수 있다"
  → 해킹당해도 정책 밖의 행동은 커널이 차단

검사 순서:
  DAC 거부 → 종료 (SELinux 안 봄)
  DAC 허용 → SELinux 검사
  SELinux 거부 → 차단
  SELinux 허용 → 최종 허용

→ SELinux는 DAC 통과 후 추가 검사층
```

### 5-2. SELinux 동작 모드

```
enforcing  ← RHEL 10 기본값. 정책 위반 시 실제로 차단 + 로그
permissive   정책 위반해도 차단 안 함. 로그만 기록 (테스트용)
disabled     SELinux 완전 비활성화 (비권장, 재활성화 시 relabel 필요)
```

```bash
# 현재 모드 확인
getenforce
sestatus        # 상세 정보

# 임시로 모드 전환 (재부팅 시 원복)
setenforce 0    # enforcing → permissive (트러블슈팅 시)
setenforce 1    # permissive → enforcing

# 영구 설정 (재부팅 후에도 적용)
# /etc/selinux/config 수정:
SELINUX=enforcing    # 또는 permissive
```

> ⚠️ `disabled`로 설정 후 재활성화 시 파일 전체 relabeling 필요 → 부팅 오래 걸림. 문제 분석 시에는 `permissive` 권장.

### 5-3. SELinux 컨텍스트 (레이블)

SELinux는 모든 프로세스와 파일에 **컨텍스트(레이블)**를 붙입니다.

```
컨텍스트 형식:
  user:role:type:level

  system_u:object_r:httpd_sys_content_t:s0
  └──────┘ └──────┘ └─────────────────┘ └┘
   SELinux  역할      타입 (가장 중요)   보안레벨
   사용자
```

type 필드가 핵심:

```
타입은 항상 _t 로 끝남

프로세스 타입:
  httpd_t          → Apache 프로세스
  sshd_t           → SSH 데몬
  mysqld_t         → MySQL/MariaDB

파일 타입:
  httpd_sys_content_t   → Apache가 읽어야 할 웹 콘텐츠
  sshd_key_t            → SSH 키 파일
  mysqld_db_t           → DB 데이터 파일
  etc_t                 → /etc 아래 설정 파일
  user_home_t           → 사용자 홈 디렉토리
```

실제 정책 예시:

```
정책: allow httpd_t httpd_sys_content_t:file read;
  → httpd 프로세스는 httpd_sys_content_t 파일을 읽을 수 있다

정책 없음: httpd_t → mysqld_db_t (접근 불가)
  → Apache가 MySQL 데이터파일 직접 접근 불가
  → 해킹당한 Apache가 DB 탈취 시도해도 SELinux가 차단
```

### 5-4. 컨텍스트 확인

```bash
# 파일 컨텍스트 확인
ls -Z /var/www/html/
# -rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 index.html

ls -Z /etc/passwd
# -rw-r--r--. root root system_u:object_r:passwd_file_t:s0 /etc/passwd

# 프로세스 컨텍스트 확인
ps auxZ | grep httpd
# system_u:system_r:httpd_t:s0  root  1234  ...  httpd

# 현재 세션 컨텍스트
id -Z
```

### 5-5. 가장 흔한 문제 — 잘못된 컨텍스트

새로 만든 파일이나 잘못된 경로에 복사한 파일은 컨텍스트가 틀릴 수 있습니다.

```bash
# 예시: 웹 콘텐츠를 /tmp에서 복사 → 컨텍스트가 tmp_t로 됨
cp /tmp/index.html /var/www/html/

ls -Z /var/www/html/index.html
# unconfined_u:object_r:user_tmp_t:s0  ← 잘못됨!
# Apache(httpd_t)는 user_tmp_t 파일 못 읽음 → 403 Forbidden

# 해결: 컨텍스트 복원
restorecon -v /var/www/html/index.html
# Relabeled: user_tmp_t → httpd_sys_content_t

# 디렉토리 전체 재귀 복원
restorecon -Rv /var/www/html/
```

### 5-6. 비표준 경로 사용 시

Apache 기본 문서 루트(/var/www/html) 대신 다른 경로 사용 시:

```bash
# /data/web을 웹 루트로 쓰고 싶을 때

# 1. 컨텍스트 규칙 추가 (영구)
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"

# 2. 실제 파일에 적용
restorecon -Rv /data/web

# 확인
ls -Z /data/web
```

semanage fcontext vs chcon:

```
chcon:
  파일에 직접 컨텍스트 적용
  임시 — restorecon 실행하면 원래대로 돌아감
  테스트용

semanage fcontext:
  컨텍스트 매핑 규칙 자체를 변경
  영구 — restorecon 해도 유지됨
  운영환경용
```

### 5-7. SELinux 불린 (Boolean)

불린 = 정책 일부를 런타임에 켜고 끄는 스위치

```bash
# 불린 목록 확인
getsebool -a
getsebool -a | grep httpd
# httpd_can_network_connect --> off
# httpd_can_network_connect_db --> off
# httpd_enable_cgi --> on

# 자주 쓰는 불린 예시:
# httpd_can_network_connect   Apache가 외부 서버에 연결 허용
# httpd_can_network_connect_db  Apache가 DB에 직접 연결 허용
# httpd_use_nfs               Apache가 NFS 볼륨 사용 허용
# sshd_enable_x11_forwarding  SSH X11 포워딩 허용
# ftpd_anon_write             FTP 익명 쓰기 허용

# 불린 변경 (임시)
setsebool httpd_can_network_connect on

# 불린 변경 (영구 — -P 옵션)
setsebool -P httpd_can_network_connect on

# 확인
getsebool httpd_can_network_connect
```

### 5-8. SELinux 거부 로그 분석

SELinux가 차단하면 `/var/log/audit/audit.log`에 기록됩니다.

```bash
# 모든 AVC 거부 메시지
ausearch -m avc -ts recent
grep "avc:  denied" /var/log/audit/audit.log

# AVC 로그 예시:
# type=AVC msg=audit(1234567890.123:456): avc:  denied  { read } for
#   pid=1234 comm="httpd" name="index.html"
#   scontext=system_u:system_r:httpd_t:s0
#   tcontext=unconfined_u:object_r:user_tmp_t:s0
#   tclass=file permissive=0
#
# 해석:
#   httpd 프로세스(httpd_t)가
#   user_tmp_t 타입 파일 index.html을
#   읽으려다 거부됨

# audit2why — 왜 거부됐는지 설명
ausearch -m avc -ts recent | audit2why

# audit2allow — 허용하는 정책 모듈 자동 생성 (주의해서 사용)
ausearch -m avc -ts recent | audit2allow -M mypolicy
semodule -i mypolicy.pp
```

audit2allow 주의:

```
audit2allow는 편리하지만:
  → 거부된 것을 무조건 허용하는 정책 생성
  → 보안 의도와 맞지 않을 수 있음
  → 먼저 restorecon, 불린 변경으로 해결 시도
  → 그래도 안 되면 audit2allow 검토
```

### 5-9. SELinux 포트 레이블

서비스를 비표준 포트로 실행 시:

```bash
# 현재 포트 레이블 확인
semanage port -l | grep http
# http_port_t  tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000

# Apache를 8888 포트로 설정했을 때 SELinux 오류 발생
# → 8888이 http_port_t에 없기 때문

# 해결: 포트 레이블 추가
semanage port -a -t http_port_t -p tcp 8888

# 확인
semanage port -l | grep http
```

---

## 6. 전체 실습 시나리오

새 서버에서 Apache 웹서버 설정 + 방화벽 + SELinux 완전 설정:

```bash
# 1. Apache 설치 및 시작
dnf install httpd
systemctl enable --now httpd

# 2. 방화벽에서 HTTP/HTTPS 허용
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all   # 확인

# 3. 웹 콘텐츠 배치
mkdir -p /var/www/html
echo "Hello World" > /var/www/html/index.html
restorecon -Rv /var/www/html/           # SELinux 컨텍스트 복원

# 4. 비표준 포트(8080) 사용 시
# httpd.conf: Listen 8080
firewall-cmd --zone=public --add-port=8080/tcp --permanent
semanage port -a -t http_port_t -p tcp 8080
firewall-cmd --reload

# 5. DB 연결 허용 (PHP가 MySQL에 연결)
setsebool -P httpd_can_network_connect_db on

# 6. 상태 확인
systemctl status httpd
firewall-cmd --zone=public --list-all
sestatus
getsebool httpd_can_network_connect_db
```

---


## 7. 패킷이 앱에서 NIC까지 — 커널 네트워크 스택 상세

아래 Python 코드 한 줄이 실제로 커널을 어떻게 통과하는지 추적합니다.

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("142.250.x.x", 80))
s.send(b"GET / HTTP/1.1\r\nHost: google.com\r\n\r\n")
```

### 7-1. 전체 송신 경로 개요

```
───────────────────────────────────────────
사용자 공간 (User Space)
───────────────────────────────────────────
Python → libc → syscall (write / sendto)
                   ↓ 소프트웨어 인터럽트
───────────────────────────────────────────
커널 공간 (Kernel Space)
───────────────────────────────────────────
① 소켓 레이어          fd → sock 구조체
② TCP 레이어           세그먼트 분할 + TCP 헤더
③ IP 레이어            라우팅 + IP 헤더
④ nftables OUTPUT      방화벽 OUTPUT 체인 검사
⑤ 이웃 서브시스템       ARP → 게이트웨이 MAC 획득
⑥ 이더넷 레이어        이더넷 프레임 구성
⑦ NIC 드라이버         TX ring에 복사, DMA
───────────────────────────────────────────
하드웨어
───────────────────────────────────────────
⑧ NIC                  전기/광 신호로 변환 → 케이블
```

### 7-2. ① 소켓 레이어

#### fd 테이블

```
fd (file descriptor) = 프로세스가 열어놓은 자원을 가리키는 정수 번호

프로세스마다 커널이 fd 테이블을 하나씩 유지:

  fd 0  → stdin  (키보드)
  fd 1  → stdout (터미널)
  fd 2  → stderr (터미널)
  fd 3  → /var/log/app.log   (열어놓은 파일)
  fd 4  → TCP 소켓           (google.com:80 연결)

Linux "모든 것은 파일" 철학:
  파일이든 소켓이든 파이프든 → 전부 fd 하나로 추상화
  앱은 숫자(fd)에 read/write만 하면 됨
  커널이 fd 보고 "이건 소켓이네" → tcp_sendmsg() 로 디스패치

커널 내부:
  fd 4 → struct file → f_op->write() → tcp_sendmsg()
                    └→ private_data  → struct socket
                                          └→ struct sock (TCP 상태, 버퍼)
```

#### 송신 흐름

```
Python: s.send(data)
  → libc: write(sockfd, data, len)
  → syscall: sys_write()

커널:
  fd 테이블에서 sockfd → struct socket 찾기
  socket.type == SOCK_STREAM → TCP 처리 함수로 위임
  데이터를 소켓 송신 버퍼(sk_sndbuf)에 복사

소켓 송신 버퍼:
  앱이 쓴 데이터를 일단 여기 쌓아둠
  TCP가 상황에 맞게 꺼내서 씀
  버퍼가 꽉 차면 write()가 블로킹됨 (blocking I/O)

  /proc/sys/net/core/wmem_default   기본 버퍼 크기
  /proc/sys/net/core/wmem_max       최대 버퍼 크기
```

### 7-3. ② TCP 레이어

```
TCP가 하는 일:
  1. 데이터를 MSS 단위로 분할
     MSS (Maximum Segment Size) = MTU - IP헤더(20) - TCP헤더(20)
     이더넷 MTU 1500 → MSS ≈ 1460 bytes

  2. 각 세그먼트에 TCP 헤더 추가:

     ┌──────────────────────────────────────────┐
     │ 출발 포트 (2B) │ 목적 포트 (2B)          │
     │ SEQ 번호 (4B)                            │ ← 순서 보장
     │ ACK 번호 (4B)                            │ ← 수신 확인
     │ 헤더 길이 │ 플래그 │ 윈도우 크기          │
     │ 체크섬 (2B) │ 긴급 포인터 (2B)           │
     └──────────────────────────────────────────┘

  플래그:
    SYN  연결 시작
    ACK  수신 확인
    FIN  연결 종료
    PSH  즉시 전달 요청 (버퍼 비우기)
    RST  연결 강제 리셋

  3-way handshake (connect() 시):
    클라 → 서버: SYN (SEQ=x)
    서버 → 클라: SYN+ACK (SEQ=y, ACK=x+1)
    클라 → 서버: ACK (ACK=y+1)
    → 이후 send() 가 실제 데이터 전송

  흐름 제어:
    수신측 윈도우 크기만큼만 전송 가능
    ACK 안 오면 재전송 타이머 → 재전송
```

### 7-4. ③ IP 레이어

```
IP가 하는 일:

1. 라우팅 결정:
   목적지: 142.250.x.x

   라우팅 테이블 조회 (ip route show):
   default via 192.168.1.1 dev enp1s0
   192.168.1.0/24 dev enp1s0 scope link

   → 142.250.x.x 는 192.168.1.0/24 아님
   → default 경로 적용
   → 다음 홉: 192.168.1.1 (게이트웨이), 인터페이스: enp1s0

2. IP 헤더 추가:
   ┌──────────────────────────────────────────┐
   │ 버전(4) │ IHL │ TOS │ 전체 길이          │
   │ 식별자 (2B)    │ 플래그 │ 단편화 오프셋  │
   │ TTL (1B) │ 프로토콜(TCP=6) │ 체크섬     │
   │ 출발 IP: 192.168.1.100                   │
   │ 목적 IP: 142.250.x.x                     │
   └──────────────────────────────────────────┘

   TTL (Time To Live):
     라우터 거칠 때마다 1씩 감소
     0이 되면 패킷 폐기 + ICMP Time Exceeded 반환
     기본값 64 (Linux), 128 (Windows)
     traceroute가 이 원리를 이용 (TTL 1→2→3... 늘려가며 경로 추적)

   단편화 (Fragmentation):
     패킷이 MTU(1500) 초과 시 IP 레이어에서 쪼갬
     수신측에서 재조합
     현대에는 Path MTU Discovery로 미리 크기 협상해서 단편화 회피
```

### 7-5. ④ nftables / firewalld — hook 구조

```
nftables는 커널 네트워크 경로의 특정 지점에 hook을 걸어서 패킷을 검사합니다.

                   [로컬 앱에서 생성된 패킷]
                             ↓
  NIC 수신              OUTPUT hook ←── 송신 전 검사
      ↓                      ↓
  PREROUTING hook       라우팅 결정
      ↓                      ↓
  라우팅 결정          POSTROUTING hook
      ↓    ↘ 포워딩          ↓
  INPUT  FORWARD hook    NIC 송신
  hook       ↓
      ↓  POSTROUTING
  로컬 프로세스

각 hook에서 firewalld 규칙이 검사됨:
  INPUT:      외부 → 이 서버  (포트 허용/차단)
  OUTPUT:     이 서버 → 외부  (기본 허용)
  FORWARD:    이 서버 통과    (Docker NAT, 라우팅)
  PREROUTING: DNAT (포트포워딩)
  POSTROUTING: SNAT (IP 변환, Masquerade)

실제 nftables 규칙 확인:
  nft list ruleset
```

### 7-6. ⑤ 이웃 서브시스템 — ARP

```
IP 레이어는 다음 홉 IP(192.168.1.1)를 알았음
이더넷 프레임을 만들려면 MAC 주소가 필요함

ARP (Address Resolution Protocol):
  "192.168.1.1 MAC 주소 누구야?" 브로드캐스트
  게이트웨이 응답: "나야, aa:bb:cc:dd:ee:ff"

ARP 캐시 확인:
  ip neigh show
  192.168.1.1 dev enp1s0 lladdr aa:bb:cc:dd:ee:ff REACHABLE

  REACHABLE: 최근에 확인됨
  STALE:     오래됨, 다음 사용 시 재확인
  FAILED:    응답 없음

핵심:
  프레임의 목적 MAC = 게이트웨이 MAC  (매 홉마다 바뀜)
  패킷의 목적 IP   = 최종 목적지 IP   (끝까지 유지)
  → 라우터는 MAC 보고 수신, IP 보고 다음 홉 결정
```

### 7-7. ⑥ 이더넷 프레임 구성

```
지금까지 만들어진 것들을 최종으로 감쌈:

┌────────────────────────────────────────────────────────────┐
│ 이더넷 헤더 (14B)                                          │
│  목적 MAC: aa:bb:cc:dd:ee:ff  (게이트웨이)                 │
│  출발 MAC: 52:54:00:xx:xx:xx  (내 NIC)                    │
│  EtherType: 0x0800 (IPv4)                                  │
├────────────────────────────────────────────────────────────┤
│ IP 헤더 (20B)                                              │
│  출발 IP: 192.168.1.100  /  목적 IP: 142.250.x.x          │
├────────────────────────────────────────────────────────────┤
│ TCP 헤더 (20B)                                             │
│  출발 포트: 54321  /  목적 포트: 80                        │
├────────────────────────────────────────────────────────────┤
│ 데이터 (최대 1460B)                                        │
│  GET / HTTP/1.1\r\nHost: google.com\r\n\r\n               │
├────────────────────────────────────────────────────────────┤
│ FCS (4B) — 이더넷 오류 검출 체크섬                         │
└────────────────────────────────────────────────────────────┘
총 최대 1514 bytes = MTU 1500 + 이더넷 헤더 14
```

### 7-8. ⑦ NIC 드라이버 — TX Ring과 DMA

```
커널 드라이버가 하는 일:

1. sk_buff (소켓 버퍼 구조체) 에 프레임 완성됨
   sk_buff = 커널이 패킷 하나를 표현하는 핵심 자료구조
   각 레이어가 sk_buff 앞에 헤더를 붙여가며 내려옴 (포인터 이동만으로)

2. TX ring buffer (송신 큐) 에 sk_buff 포인터 기록
   TX ring = NIC와 드라이버가 공유하는 원형 큐

3. NIC에 "보낼 거 있다" 신호 (doorbell register 쓰기)

4. NIC가 DMA로 메모리에서 프레임 읽어 전송
   DMA (Direct Memory Access): CPU 개입 없이 NIC가 메모리 직접 읽음
   전송 완료 후 인터럽트 발생 → 드라이버가 sk_buff 해제

sk_buff 메모리 구조 (레이어 내려갈수록 data 포인터가 앞으로 당겨짐):

  초기 (데이터만):
  (head)[헤드룸](data)[데이터](tail)[테일룸](end)

  TCP 헤더 추가 후 (skb_push):
  (head)[헤드룸](data)[TCP 헤더][데이터](tail)[테일룸](end)

  IP 헤더 추가 후:
  (head)[헤드룸](data)[IP 헤더][TCP 헤더][데이터](tail)[테일룸](end)

  ETH 헤더 추가 후:
  (head)(data)[ETH 헤더][IP 헤더][TCP 헤더][데이터](tail)[테일룸](end)

  헤더 추가(skb_push): data 포인터를 앞으로 당김 → 복사 없음
  헤더 제거(skb_pull): data 포인터를 뒤로 밈    → 복사 없음
  → 레이어 이동 = O(1) 연산
```

### 7-9. 수신 경로 — NIC에서 앱까지

```
⑧ NIC 하드웨어
   전기 신호 수신 → 프레임 버퍼에 저장
   DMA로 RAM에 복사
   인터럽트 발생 (또는 NAPI polling)
     ↓
⑦ NIC 드라이버
   RX ring에서 sk_buff 생성
   NAPI (New API): 고부하 시 인터럽트 폭풍 방지를 위해
                   인터럽트 → polling 모드 전환
     ↓
⑥ 이더넷 레이어
   FCS 체크섬 검증 (오류 시 폐기)
   목적 MAC 확인 (내 MAC 또는 브로드캐스트?)
   이더넷 헤더 제거 → IP 레이어로
     ↓
④ nftables PREROUTING + INPUT 체인
   PREROUTING: 라우팅 전 (DNAT 포트포워딩 여기서)
   라우팅: 목적 IP가 내 IP → INPUT / 아니면 → FORWARD
   INPUT: 포트 22(SSH) 허용? → 통과
          포트 3306 차단? → DROP
     ↓
③ IP 레이어
   IP 헤더 검증 (체크섬, TTL 확인)
   단편화 재조합 (필요 시)
   IP 헤더 제거 → TCP 레이어로
     ↓
② TCP 레이어
   포트 번호로 소켓 찾기 (port 22 → sshd의 소켓)
   SEQ/ACK 번호 검증 → 순서 재조합
   ACK 전송 (수신 확인)
   수신 버퍼(sk_rcvbuf)에 데이터 쌓음
     ↓
① 소켓 레이어
   수신 버퍼에 데이터 있으면 앱의 read() / recv() 깨움
     ↓
사용자 공간
   앱의 recv() 반환 → 데이터 획득
```

### 7-10. FORWARD 경로 — Docker NAT

Docker 컨테이너가 외부와 통신하는 경우:

```
외부 → 호스트:8080 → 컨테이너:80

수신:
  NIC → PREROUTING
  → DNAT: 호스트:8080 → 172.17.0.2:80 (컨테이너 IP로 변경)
  → 라우팅: 목적 IP가 172.17.0.2 → 내 IP 아님 → FORWARD
  → FORWARD 체인 검사 (Docker가 자동으로 허용 규칙 추가)
  → POSTROUTING
  → docker0 인터페이스 → 컨테이너

컨테이너 → 외부:
  컨테이너 → docker0 → FORWARD → POSTROUTING
  → SNAT/Masquerade: 172.17.0.2 → 호스트 IP로 변경
  → enp1s0 → 인터넷

→ Docker가 nftables를 직접 조작해서 이 규칙들을 자동 생성
→ firewalld가 모르는 규칙 → firewalld로 Docker 포트를 막을 수 없는 이유
```

### 7-11. 전체 캡슐화 정리

```
각 레이어가 데이터를 감싸는 구조:

앱 데이터:       [  HTTP 요청 데이터  ]

TCP 캡슐화:      [ TCP 헤더 | HTTP 데이터 ]
                 └─── TCP 세그먼트 ────────┘

IP 캡슐화:       [ IP 헤더 | TCP 헤더 | HTTP 데이터 ]
                 └────────── IP 패킷 ────────────────┘

이더넷 캡슐화:   [ ETH 헤더 | IP 헤더 | TCP 헤더 | HTTP 데이터 | FCS ]
                 └──────────────── 이더넷 프레임 ────────────────────┘

수신 측에서는 반대 순서로 헤더를 벗겨가며 위 레이어로 전달:
  이더넷 헤더 제거 → IP 레이어
  IP 헤더 제거    → TCP 레이어
  TCP 헤더 제거   → 소켓 버퍼 → 앱의 recv()
```

---

## 8. 전체 실습 시나리오

새 서버에서 Apache 웹서버 설정 + 방화벽 + SELinux 완전 설정:

```bash
# 1. Apache 설치 및 시작
dnf install httpd
systemctl enable --now httpd

# 2. 방화벽에서 HTTP/HTTPS 허용
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all

# 3. 웹 콘텐츠 배치
mkdir -p /var/www/html
echo "Hello World" > /var/www/html/index.html
restorecon -Rv /var/www/html/

# 4. 비표준 포트(8080) 사용 시
firewall-cmd --zone=public --add-port=8080/tcp --permanent
semanage port -a -t http_port_t -p tcp 8080
firewall-cmd --reload

# 5. DB 연결 허용 (PHP → MySQL)
setsebool -P httpd_can_network_connect_db on

# 6. 상태 확인
systemctl status httpd
firewall-cmd --zone=public --list-all
sestatus
ss -tlnp | grep httpd
```

---

## 📚 주요 출처

| 문서 | URL |
|------|-----|
| RHEL 10 Configuring and managing networking | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/configuring_and_managing_networking/index |
| RHEL 10 Configuring firewalls and packet filters | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/configuring_firewalls_and_packet_filters/index |
| RHEL 10 Using and configuring firewalld | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld |
| RHEL 10 Using SELinux | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/index |
| RHEL 10 Getting started with SELinux | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/getting-started-with-selinux |
