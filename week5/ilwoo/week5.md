# Week 5. 네트워크 설정과 보안

## 1. 이번 주 학습 주제

- 네트워크 기본 개념과 인터페이스 구성 이해
- nmcli/nmtui를 활용한 IP/게이트웨이/DNS 설정
- 네트워크 상태 점검 도구 활용 (ping, ss, ip)
- 방화벽(firewalld) 정책 설정
- SELinux 동작 방식과 기본 보안 정책 이해

## 2. 실습 환경

- Host OS: Windows 11
- Virtualization: Hyper-V
- Guest OS: RHEL 8 (Red Hat Enterprise Linux 8) GUI 설치
- 사용자 계정: `ooo426`

---

# Part 1. 네트워크 기본 개념과 인터페이스 설정

## 3. 핵심 개념 정리

### 3-1. 네트워크 기본 용어

리눅스에서 네트워크를 다루기 전에 알아야 할 핵심 용어들을 정리한다.

#### IP 주소란?

**IP(Internet Protocol) 주소**는 네트워크에서 각 장치를 구분하는 **고유한 주소**이다. 우편물을 보낼 때 집 주소가 필요하듯, 컴퓨터끼리 통신할 때 IP 주소가 필요하다.

```
예시: 192.168.0.10
       └─ 이 숫자 조합이 네트워크에서 "이 컴퓨터"를 가리킨다
```

- **IPv4**: `192.168.0.10` 형태 (32비트, 약 43억 개)
- **IPv6**: `fe80::1` 형태 (128비트, 사실상 무한)
- **루프백(Loopback)**: `127.0.0.1` = 자기 자신을 가리키는 주소 (`localhost`)

#### 서브넷 마스크란?

**서브넷 마스크(Subnet Mask)**는 IP 주소에서 **네트워크 부분과 호스트(장치) 부분을 구분**하는 값이다.

```
IP 주소:       192.168.0.10
서브넷 마스크:  255.255.255.0  (또는 /24)
               └─────────┘└─┘
               네트워크 부분  호스트 부분

같은 네트워크: 192.168.0.x  (x가 다르면 같은 네트워크 내 다른 장치)
다른 네트워크: 192.168.1.x  (세 번째 숫자가 다르면 다른 네트워크)
```

> 쉽게 말해, 서브넷 마스크는 "같은 동네인지 아닌지"를 판별하는 기준이다. `/24`는 앞 24비트가 네트워크 주소라는 의미로, `255.255.255.0`과 같다.

#### 게이트웨이란?

**게이트웨이(Gateway)**는 내 네트워크 밖으로 나가는 **출구(문)**이다. 다른 네트워크(인터넷 등)와 통신하려면 게이트웨이를 거쳐야 한다.

```
내 PC (192.168.0.10)
   │
   └─→ 게이트웨이 (192.168.0.1) ─→ 인터넷
       (보통 공유기/라우터)
```

> 같은 네트워크(192.168.0.x) 안에서는 게이트웨이 없이 직접 통신 가능. 외부(네이버, 구글 등)로 나갈 때만 게이트웨이 필요.

#### DNS란?

**DNS(Domain Name System)**는 도메인 이름(google.com)을 IP 주소(142.250.196.110)로 변환해주는 **전화번호부** 같은 시스템이다.

```
사용자 입력: google.com
    ↓ DNS 조회
실제 통신:   142.250.196.110
```

> 사람은 이름(google.com)이 외우기 쉽고, 컴퓨터는 숫자(IP)로 통신한다. DNS가 이 둘을 연결해준다.

#### DHCP란?

**DHCP(Dynamic Host Configuration Protocol)**는 IP 주소를 **자동으로 할당**해주는 프로토콜이다. 공유기에 연결하면 IP를 자동으로 받는 것이 DHCP 덕분이다.

| 방식 | 설명 | 사용 예 |
|------|------|---------|
| **DHCP (자동)** | 서버가 IP를 자동 할당 | 일반 PC, 노트북 |
| **고정 IP (수동)** | 관리자가 직접 IP를 지정 | 서버, 네트워크 장비 |

### 3-2. 네트워크 인터페이스란?

**네트워크 인터페이스**는 컴퓨터가 네트워크에 연결되는 **접점(통로)**이다. 물리적 랜 카드일 수도 있고, 가상 인터페이스일 수도 있다.

```bash
# 현재 시스템의 네트워크 인터페이스 확인
ip link show
```

**인터페이스 이름 규칙** (RHEL 7+):

| 접두사 | 의미 | 예시 |
|--------|------|------|
| `eth` | 전통적 이더넷 이름 (구형) | `eth0`, `eth1` |
| `ens` | 핫플러그 슬롯 기반 이더넷 | `ens33`, `ens160` |
| `enp` | PCI 버스 위치 기반 이더넷 | `enp0s3`, `enp0s8` |
| `lo` | 루프백 (자기 자신) | `lo` (항상 존재) |
| `virbr` | 가상 브리지 | `virbr0` (libvirt 사용 시) |

> **왜 eth0이 아닌가?**: 과거에는 `eth0`, `eth1`처럼 단순한 이름을 사용했지만, 장치가 여러 개일 때 부팅마다 이름이 바뀌는 문제가 있었다. RHEL 7부터는 물리적 위치 기반 이름(ens, enp)을 사용하여 이름이 항상 일정하다.

### 3-3. NetworkManager란?

**NetworkManager**는 RHEL의 네트워크 설정을 관리하는 **서비스(데몬)**이다. IP 주소, DNS, 게이트웨이 설정을 담당하며, 이를 제어하는 도구가 `nmcli`와 `nmtui`이다.

```
NetworkManager (서비스)  ← 네트워크 설정을 실제로 관리하는 데몬
    ├─ nmcli             ← CLI 도구 (명령어로 제어)
    └─ nmtui             ← TUI 도구 (텍스트 UI, 메뉴 방식)
```

> week4에서 배운 systemd 개념과 연결: NetworkManager도 하나의 서비스(유닛)이다. `systemctl status NetworkManager`로 상태를 확인할 수 있다.

#### 커넥션(Connection)이란?

NetworkManager에서 **커넥션**은 특정 인터페이스에 적용할 **네트워크 설정 프로필**이다. 하나의 인터페이스에 여러 커넥션을 만들어두고 상황에 따라 전환할 수 있다.

```
인터페이스: eth0 (물리적 랜 카드)
    ├─ 커넥션 "회사" (고정 IP: 10.0.0.50, DNS: 10.0.0.1)
    └─ 커넥션 "집"   (DHCP 자동 할당)
```

## 4. 실습: nmcli/nmtui로 네트워크 설정

### 4-1. 현재 네트워크 상태 확인

```bash
# NetworkManager 서비스 상태 확인
systemctl status NetworkManager

# 현재 활성화된 커넥션 목록
nmcli connection show

# 인터페이스별 IP 주소 확인
nmcli device status
ip addr show
```

### 4-2. nmcli - CLI로 네트워크 설정

```bash
# 특정 커넥션의 상세 정보 확인
nmcli connection show "유선 연결 1"

# 고정 IP 설정 예시
sudo nmcli connection modify "유선 연결 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.100/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "8.8.8.8 8.8.4.4"

# 변경 사항 적용 (커넥션 재활성화)
sudo nmcli connection up "유선 연결 1"

# DHCP(자동)로 되돌리기
sudo nmcli connection modify "유선 연결 1" \
  ipv4.method auto
sudo nmcli connection up "유선 연결 1"
```

**nmcli 주요 속성**:

| 속성 | 설명 | 예시 값 |
|------|------|---------|
| `ipv4.method` | IP 할당 방식 | `auto`(DHCP) / `manual`(고정) |
| `ipv4.addresses` | IP 주소/서브넷 | `192.168.0.100/24` |
| `ipv4.gateway` | 게이트웨이 | `192.168.0.1` |
| `ipv4.dns` | DNS 서버 | `8.8.8.8` (Google DNS) |

### 4-3. nmtui - 텍스트 UI로 네트워크 설정

```bash
# 텍스트 기반 UI 실행
sudo nmtui
```

nmtui는 터미널에서 메뉴 방식으로 네트워크를 설정할 수 있는 도구이다. nmcli 명령어를 외우기 어려울 때 편리하다.

메뉴 구조:
1. **Edit a connection**: 커넥션의 IP/DNS/게이트웨이 수정
2. **Activate a connection**: 커넥션 활성화/비활성화
3. **Set system hostname**: 호스트 이름 변경

### 4-4. 설정 파일 직접 확인

NetworkManager의 커넥션 설정은 파일로도 저장된다:

```bash
# RHEL 8 기본 경로
ls /etc/sysconfig/network-scripts/
cat /etc/sysconfig/network-scripts/ifcfg-*
```

> `nmcli`나 `nmtui`로 변경한 설정은 이 파일에 자동 반영된다. 직접 파일을 수정할 수도 있지만, `nmcli`를 사용하는 것이 권장된다.

---

# Part 2. 네트워크 상태 점검 도구

## 5. 핵심 개념 정리

### 5-1. 네트워크 진단이 필요한 상황

네트워크 문제는 여러 계층에서 발생할 수 있다. 문제의 원인을 체계적으로 찾기 위해 각 계층을 점검하는 도구가 있다:

```
문제 진단 순서:
1. 물리 연결 확인      → ip link (인터페이스 상태)
2. IP 설정 확인        → ip addr (IP 할당 여부)
3. 게이트웨이 연결 확인 → ping 게이트웨이 (내부 통신)
4. 외부 연결 확인      → ping 8.8.8.8 (인터넷 연결)
5. DNS 확인           → ping google.com (이름 해석)
6. 포트/서비스 확인    → ss (열린 포트)
```

## 6. 실습: 네트워크 점검 도구

### 6-1. ip - 인터페이스 및 라우팅 확인

`ip` 명령어는 네트워크 인터페이스, IP 주소, 라우팅 테이블을 확인하는 **통합 도구**이다. 과거의 `ifconfig`, `route` 명령을 대체한다.

```bash
# 인터페이스 목록 및 상태 (UP/DOWN)
ip link show

# IP 주소 확인 (가장 많이 사용)
ip addr show
ip a              # 줄임말

# 특정 인터페이스만 확인
ip addr show eth0

# 라우팅 테이블 확인 (게이트웨이 확인)
ip route show
ip r              # 줄임말
```

**ip addr 출력 읽는 법**:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>    ← 인터페이스 상태 (UP = 활성)
    inet 192.168.0.10/24                       ← IPv4 주소/서브넷
    inet6 fe80::1/64                           ← IPv6 주소
```

> **ifconfig vs ip**: 과거에는 `ifconfig`로 네트워크를 확인했지만, RHEL 7부터는 `ip` 명령어가 표준이다. `ifconfig`는 `net-tools` 패키지를 별도 설치해야 사용 가능하다.

### 6-2. ping - 연결 상태 확인

`ping`은 대상 호스트에 ICMP 패킷을 보내 **응답이 돌아오는지** 확인하는 가장 기본적인 네트워크 진단 도구이다.

```bash
# 게이트웨이 연결 확인
ping -c 4 192.168.0.1       # 4번만 보내고 종료

# 외부 인터넷 연결 확인
ping -c 4 8.8.8.8           # Google DNS

# DNS 동작 확인 (도메인 → IP 변환)
ping -c 4 google.com
```

**ping 결과 읽는 법**:

```
PING 8.8.8.8: 64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=3.45 ms
                                                          └─ 응답 시간 (낮을수록 좋음)

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
                                    └─ 패킷 손실률 (0%가 정상)
```

| 결과 | 의미 | 다음 조치 |
|------|------|----------|
| 응답 정상 | 네트워크 연결 정상 | - |
| `Destination Host Unreachable` | 대상에 도달 불가 | 게이트웨이/라우팅 확인 |
| `Request timed out` | 응답 없음 | 방화벽/대상 서버 확인 |
| `Name or service not known` | DNS 해석 실패 | DNS 설정 확인 |

> **ping이 안 되면?** 위의 진단 순서대로 하나씩 확인한다. `ping 게이트웨이` → OK, `ping 8.8.8.8` → 실패이면, 게이트웨이가 외부로 나가는 경로에 문제가 있는 것이다.

### 6-3. ss - 포트 및 소켓 상태 확인

`ss`(Socket Statistics)는 현재 시스템에서 **열려 있는 포트와 네트워크 연결**을 확인하는 도구이다. 과거의 `netstat`을 대체한다.

```bash
# 리스닝(대기) 중인 TCP 포트 확인
ss -tlnp

# 리스닝 중인 UDP 포트 확인
ss -ulnp

# 모든 연결 상태 확인
ss -an

# 특정 포트 확인 (예: SSH 22번 포트)
ss -tlnp | grep :22
```

**ss 옵션 의미**:

| 옵션 | 의미 |
|------|------|
| `-t` | TCP 연결만 |
| `-u` | UDP 연결만 |
| `-l` | 리스닝(대기) 상태만 |
| `-n` | 포트 번호를 숫자로 표시 (이름 대신) |
| `-p` | 포트를 사용하는 프로세스 표시 |

**주요 포트 번호**:

| 포트 | 서비스 | 설명 |
|------|--------|------|
| 22 | SSH | 원격 접속 |
| 80 | HTTP | 웹 서버 |
| 443 | HTTPS | 웹 서버 (암호화) |
| 53 | DNS | 도메인 이름 해석 |
| 3306 | MySQL | 데이터베이스 |

> **포트(Port)란?** IP 주소가 "건물 주소"라면, 포트는 "호수(층수)"이다. 하나의 서버(IP)에서 여러 서비스가 각기 다른 포트 번호로 동작한다.

### 6-4. 기타 유용한 네트워크 도구

```bash
# DNS 조회 테스트
nslookup google.com
dig google.com          # 더 상세한 DNS 정보

# 경로 추적 (패킷이 어떤 경로로 가는지)
traceroute google.com

# 호스트 이름 확인/변경
hostname
hostnamectl
```

---

# Part 3. 방화벽과 SELinux

## 7. 핵심 개념 정리

### 7-1. 방화벽이란?

**방화벽(Firewall)**은 네트워크 트래픽을 **규칙에 따라 허용하거나 차단**하는 보안 시스템이다. 건물의 출입 통제 시스템처럼, 허가된 트래픽만 통과시킨다.

```
외부 네트워크 (인터넷)
       │
  [방화벽] ← 규칙에 따라 허용/차단
       │
  내부 서버 (sshd:22, httpd:80 등)
```

### 7-2. firewalld란?

**firewalld**는 RHEL의 기본 방화벽 관리 서비스이다. 내부적으로 `nftables`(또는 `iptables`)를 사용하며, 사용자는 `firewall-cmd` 명령어로 제어한다.

```
firewalld (서비스)       ← 방화벽 정책을 관리하는 데몬
    └─ firewall-cmd      ← CLI 도구 (정책 추가/삭제/조회)
```

> week4의 systemd 개념과 연결: firewalld도 systemd가 관리하는 서비스 유닛이다. `systemctl status firewalld`로 상태를 확인할 수 있다.

#### Zone(존)이란?

firewalld는 **존(Zone)** 개념으로 방화벽 정책을 관리한다. 존은 네트워크 상황에 따라 미리 정의된 **보안 수준**이다.

| 존 | 설명 | 기본 동작 |
|----|------|----------|
| **public** | 공공/외부 네트워크 (기본값) | 대부분 차단, 일부 허용 |
| **trusted** | 완전 신뢰 네트워크 | 모든 트래픽 허용 |
| **home** | 가정 네트워크 | 일부 서비스 허용 |
| **drop** | 최고 보안 | 모든 수신 차단, 응답 없음 |
| **block** | 높은 보안 | 모든 수신 차단, 거부 응답 |

> 쉽게 말해, 존은 "이 네트워크를 얼마나 신뢰하는가"에 따른 방화벽 프리셋이다. 대부분의 서버는 `public` 존을 사용한다.

### 7-3. SELinux란?

**SELinux(Security-Enhanced Linux)**는 리눅스 커널에 내장된 **강제 접근 제어(MAC)** 보안 시스템이다.

일반적인 리눅스 보안(파일 권한: rwx)은 **사용자 기반**이다. root 권한을 탈취당하면 모든 것이 노출된다. SELinux는 여기에 추가 보안 계층을 더한다:

```
일반 보안 (DAC):  사용자 → 파일 권한(rwx) → 접근 허용/거부
SELinux (MAC):    사용자 → 파일 권한(rwx) → SELinux 정책 → 접근 허용/거부
                                            └─ 추가 보안 계층
```

| 보안 모델 | 이름 | 설명 |
|----------|------|------|
| **DAC** | Discretionary Access Control | 파일 소유자가 권한 결정 (chmod, chown) |
| **MAC** | Mandatory Access Control | **시스템 정책**이 권한 결정 (SELinux) |

> **왜 필요한가?**: 웹 서버(httpd)가 해킹당했을 때, 일반 보안만 있으면 해커가 httpd 프로세스의 권한으로 시스템 전체를 탐색할 수 있다. SELinux가 있으면 httpd는 웹 콘텐츠 디렉터리만 접근 가능하고, 다른 영역은 정책에 의해 차단된다.

#### SELinux 동작 모드

| 모드 | 설명 | 용도 |
|------|------|------|
| **Enforcing** | 정책 위반 시 **차단 + 로그** | 운영 환경 (기본값) |
| **Permissive** | 정책 위반 시 **로그만** (차단 안 함) | 디버깅/테스트 |
| **Disabled** | SELinux 완전 비활성화 | 권장하지 않음 |

#### SELinux 컨텍스트란?

SELinux는 모든 파일, 프로세스, 포트에 **컨텍스트(Context)**라는 보안 라벨을 붙인다. 프로세스의 컨텍스트와 파일의 컨텍스트가 매칭되어야 접근이 허용된다.

```
파일 컨텍스트 예시:
/var/www/html/index.html → httpd_sys_content_t   (웹 콘텐츠)
/etc/shadow              → shadow_t              (패스워드 파일)

프로세스 컨텍스트 예시:
httpd (웹 서버)          → httpd_t

규칙: httpd_t 프로세스는 httpd_sys_content_t 파일만 접근 가능
      → /etc/shadow (shadow_t)에는 접근 불가!
```

## 8. 실습: firewalld 방화벽 정책 설정

### 8-1. 방화벽 상태 확인

```bash
# firewalld 서비스 상태 확인
systemctl status firewalld

# 현재 활성화된 존 확인
firewall-cmd --get-active-zones

# 기본 존 확인
firewall-cmd --get-default-zone

# 현재 존의 허용된 서비스 목록
firewall-cmd --list-all
```

### 8-2. 서비스/포트 허용 및 차단

```bash
# HTTP 서비스 허용 (임시 - 재부팅 시 사라짐)
sudo firewall-cmd --add-service=http
firewall-cmd --list-services     # 확인

# HTTP 서비스 허용 (영구)
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload       # 영구 설정 적용

# 특정 포트 직접 허용 (예: 8080/tcp)
sudo firewall-cmd --add-port=8080/tcp --permanent

# 서비스 제거 (차단)
sudo firewall-cmd --remove-service=http --permanent

# 변경 사항 적용
sudo firewall-cmd --reload
```

> **임시 vs 영구(--permanent)**: `--permanent` 없이 실행하면 즉시 적용되지만 재부팅 시 사라진다. `--permanent`를 붙이면 설정 파일에 저장되지만 즉시 적용은 안 되므로 `--reload`가 필요하다. 실무에서는 `--permanent` + `--reload`를 함께 사용한다.

### 8-3. 사용 가능한 서비스 목록

```bash
# firewalld가 알고 있는 모든 서비스 목록
firewall-cmd --get-services
```

주요 서비스:

| 서비스 | 포트 | 설명 |
|--------|------|------|
| ssh | 22/tcp | 원격 접속 |
| http | 80/tcp | 웹 서버 |
| https | 443/tcp | 웹 서버 (암호화) |
| dns | 53/tcp+udp | DNS 서버 |
| cockpit | 9090/tcp | 웹 관리 콘솔 |

## 9. 실습: SELinux 기본 운영

### 9-1. SELinux 상태 확인

```bash
# 현재 SELinux 모드 확인
getenforce

# 상세 상태 확인
sestatus
```

### 9-2. SELinux 모드 변경

```bash
# 임시로 Permissive 모드 전환 (재부팅 시 원래대로)
sudo setenforce 0          # 0 = Permissive
getenforce                 # Permissive 확인

# 다시 Enforcing 모드로
sudo setenforce 1          # 1 = Enforcing
getenforce                 # Enforcing 확인
```

영구 변경은 설정 파일을 수정해야 한다:

```bash
# SELinux 설정 파일 확인
cat /etc/selinux/config
# SELINUX=enforcing  ← 이 값을 변경 (재부팅 필요)
```

> **실무 팁**: SELinux를 끄지 말고(Disabled), 문제가 생기면 Permissive 모드에서 로그를 확인하여 원인을 파악하는 것이 권장된다. SELinux를 끄면 보안이 크게 약해진다.

### 9-3. SELinux 컨텍스트 확인

```bash
# 파일의 SELinux 컨텍스트 확인
ls -Z /var/www/html/
ls -Z /etc/shadow

# 프로세스의 SELinux 컨텍스트 확인
ps -eZ | grep sshd
```

### 9-4. SELinux 문제 해결

SELinux가 접근을 차단하면 `/var/log/audit/audit.log`에 로그가 남는다.

```bash
# SELinux 거부 로그 확인
sudo ausearch -m avc --ts recent

# 더 읽기 쉬운 형태로 보기 (sealert 도구)
sudo sealert -a /var/log/audit/audit.log
```

> **Boolean**: SELinux의 세부 정책을 켜고 끄는 스위치이다. 전체 SELinux를 끄지 않고도 특정 기능만 허용할 수 있다.

```bash
# 현재 Boolean 목록 확인
getsebool -a

# 특정 Boolean 확인 (예: httpd가 네트워크 연결을 할 수 있는지)
getsebool httpd_can_network_connect

# Boolean 변경 (영구 적용: -P)
sudo setsebool -P httpd_can_network_connect on
```

---

## 10. 전체 구조 요약

```
네트워크 설정 계층
├─ NetworkManager (서비스)
│    ├─ nmcli        → CLI로 IP/DNS/게이트웨이 설정
│    ├─ nmtui        → 텍스트 UI로 설정
│    └─ 설정 파일     → /etc/sysconfig/network-scripts/
│
├─ 네트워크 점검 도구
│    ├─ ip addr/link → 인터페이스/IP 확인
│    ├─ ping         → 연결 상태 확인
│    └─ ss           → 포트/소켓 상태 확인
│
├─ 방화벽 (firewalld)
│    ├─ Zone 기반 정책 관리
│    ├─ firewall-cmd → 서비스/포트 허용·차단
│    └─ --permanent + --reload → 영구 적용
│
└─ SELinux (강제 접근 제어)
     ├─ Enforcing / Permissive / Disabled
     ├─ 컨텍스트(Context) 기반 접근 제어
     └─ Boolean으로 세부 정책 제어
```

---

## 11. 참고 자료

- [Red Hat - Configuring and Managing Networking](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/)
- [Red Hat - Using and Configuring firewalld](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/securing_networks/using-and-configuring-firewalld_securing-networks)
- [Red Hat - Using SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_selinux/)

---

# Hyper-V Default Switch IP 변동 문제와 Internal Switch + NAT 해결

### 배경

회사에서 Fasoo DRM 제품(FED5, FUT 등)을 Rocky Linux 10.1 VM에 설치하는 작업을 진행하고 있었습니다. Hyper-V로 VM을 생성할 때 기본 제공되는 **Default Switch**를 사용했는데, **호스트 PC를 재부팅할 때마다 VM의 IP가 바뀌는 문제**가 발생했습니다.

서버 제품들의 config 파일에 IP가 하드코딩되어 있어서, 매번 재부팅 후 수십 개의 설정 파일을 `sed`로 일괄 치환해야 하는 상황이 반복되었습니다.

---

### 1. 왜 Default Switch에서 IP가 바뀌는가?

**Default Switch의 동작 원리:**

```
[호스트 PC 부팅]
    │
    ▼
Hyper-V가 Default ~~Switch~~ 초기화
    │
    ▼
내부적으로 ICS(Internet Connection Sharing) 기반의
DHCP 서버가 새로운 서브넷을 랜덤 할당
    │
    ▼
예: 이번엔 172.25.224.0/20, 다음엔 192.168.132.0/20 ...
    │
    ▼
VM이 부팅되면 DHCP로 해당 서브넷 내의 IP를 자동 수신
    │
    ▼
예: 172.25.233.46 → 재부팅 후 → 192.168.132.16
```

**핵심 원인:** Default Switch는 Hyper-V가 **자체 관리하는 NAT 스위치**로, 사용자가 서브넷이나 DHCP 범위를 제어할 수 없습니다. 호스트가 재부팅될 때마다 ICS 서비스가 재시작되면서 **서브넷 자체가 바뀝니다.**

따라서 VM 내부에서 고정 IP를 설정하더라도, 게이트웨이(Default Switch)의 서브넷이 바뀌면 **통신 자체가 불가능**해집니다.

---

### 2. Default Switch 사용 시 네트워크 구조

```
┌─────────────────────────────────────────────────┐
│  Windows 호스트 PC (사내망: 192.168.x.x)          │
│                                                   │
│  ┌──────────────────────┐                         │
│  │  Default Switch      │  ← Hyper-V 자체 관리    │
│  │  (ICS 기반 NAT)      │  ← 서브넷 재부팅마다 변동 │
│  │                      │                         │
│  │  GW: 172.25.224.1    │  ← 이번 부팅             │
│  │  GW: 192.168.132.1   │  ← 다음 부팅 (바뀜!)     │
│  └──────┬───────────────┘                         │
│         │ DHCP                                    │
│  ┌──────▼───────────────┐                         │
│  │  Rocky Linux VM      │                         │
│  │  IP: ???.???.???.??  │  ← 매번 바뀜             │
│  │  (DHCP 자동 할당)     │                         │
│  └──────────────────────┘                         │
└─────────────────────────────────────────────────┘

문제: 서브넷 자체가 바뀌므로 고정 IP 설정도 무의미
```

---

### 3. 해결: Internal Switch + NAT 구성

**핵심 아이디어:** Default Switch 대신 **직접 만든 Internal Switch**를 사용하고, NAT와 게이트웨이 IP를 **수동으로 고정**합니다.

### 3-1. Windows PowerShell(관리자)에서 실행

```powershell
# 1단계: Internal 타입의 가상 스위치 생성
New-VMSwitch -Name "FasooNAT" -SwitchType Internal

# 2단계: 호스트 측 가상 어댑터에 고정 IP 할당 (이것이 VM의 게이트웨이가 됨)
New-NetIPAddress -IPAddress 192.168.100.1 -PrefixLength 24 -InterfaceAlias "vEthernet (FasooNAT)"

# 3단계: NAT 규칙 생성 (VM이 인터넷 접속할 수 있도록)
New-NetNat -Name "FasooNATNetwork" -InternalIPInterfaceAddressPrefix 192.168.100.0/24
```

### 3-2. Hyper-V에서 VM 네트워크 어댑터 변경

VM 설정 → 네트워크 어댑터 → 가상 스위치를 **Default Switch → FasooNAT**으로 변경

### 3-3. VM 내부에서 고정 IP 설정

```bash
nmcli con mod "eth0" ipv4.addresses 192.168.100.10/24
nmcli con mod "eth0" ipv4.gateway 192.168.100.1
nmcli con mod "eth0" ipv4.dns "8.8.8.8"
nmcli con mod "eth0" ipv4.method manual
nmcli con up "eth0"
```

---

### 4. 해결 후 네트워크 구조

```
┌──────────────────────────────────────────────────────────┐
│  Windows 호스트 PC (사내망: 10.x.x.x)                     │
│                                                            │
│  ┌─────────────────────────────┐                           │
│  │  vEthernet (FasooNAT)       │  ← 사용자가 직접 생성      │
│  │  IP: 192.168.100.1/24       │  ← 고정! 재부팅해도 불변    │
│  │  역할: VM의 게이트웨이        │                           │
│  └──────────┬──────────────────┘                           │
│             │                                              │
│  ┌──────────▼──────────────────┐    ┌───────────────────┐  │
│  │  Rocky Linux VM             │    │  Windows NAT      │  │
│  │  IP: 192.168.100.10/24      │    │  192.168.100.0/24 │  │
│  │  GW: 192.168.100.1          │───▶│  → 호스트 외부망   │  │
│  │  DNS: 8.8.8.8               │    │    (인터넷/사내망)  │  │
│  │  (수동 고정)                 │    └───────────────────┘  │
│  └─────────────────────────────┘                           │
└──────────────────────────────────────────────────────────┘
```

---

### 5. 통신 흐름 비교

### Before (Default Switch)

```
[VM] 172.25.233.46 → [Default Switch GW] 172.25.224.1 → [호스트] → 인터넷
                      ↑ 재부팅하면 서브넷 자체가 바뀜
[VM] ???.???.???.?? → [Default Switch GW] ???.???.???.1 → [호스트] → 인터넷
```

| 구분 | 호스트 부팅 1회차 | 호스트 부팅 2회차 | 호스트 부팅 3회차 |
| --- | --- | --- | --- |
| 서브넷 | 172.25.224.0/20 | 192.168.132.0/20 | 172.30.16.0/20 |
| VM IP | 172.25.233.46 | 192.168.132.16 | 172.30.22.xxx |
| GW | 172.25.224.1 | 192.168.132.1 | 172.30.16.1 |

### After (Internal Switch + NAT)

```
[VM] 192.168.100.10 → [FasooNAT GW] 192.168.100.1 → [NAT] → [호스트] → 인터넷
     ↑ 항상 고정         ↑ 항상 고정
```

| 구분 | 호스트 부팅 1회차 | 호스트 부팅 2회차 | 호스트 부팅 N회차 |
| --- | --- | --- | --- |
| 서브넷 | 192.168.100.0/24 | 192.168.100.0/24 | 192.168.100.0/24 |
| VM IP | 192.168.100.10 | 192.168.100.10 | 192.168.100.10 |
| GW | 192.168.100.1 | 192.168.100.1 | 192.168.100.1 |

---

### 6. 왜 이 방법이 안전한가 (사내망 영향 없음)

- **Internal Switch**는 호스트 PC 내부에서만 존재하는 가상 네트워크
- 사내 물리 네트워크에 브릿지되지 않음
- VM의 외부 통신은 Windows NAT를 통해 호스트 PC의 IP로 나감 (SNAT)
- 사내 다른 PC에서 VM에 직접 접근 불가 (호스트 PC에서만 접근 가능)
- 기존 사내망 IP 대역과 충돌하지 않도록 `192.168.100.0/24`처럼 사용하지 않는 대역 선택

---

### 7. 실무에서 겪은 시행착오

| 문제 상황 | 원인 | 조치 |
| --- | --- | --- |
| 재부팅 후 FED5 접속 불가 | config 파일에 이전 IP가 하드코딩 | `sed -i 's/이전IP/새IP/g'`로 6개 config 파일 일괄 치환 |
| FED5 로그인 시 "Unverified redirects" | `appserver.config`의 `<domain>` 값이 이전 IP | domain IP 치환 후 재기동 |
| FUT→FED5 연동 실패 (Connection refused) | FED5 config에 FUT 포트가 잘못 설정 (8081→7081) | 포트 번호 치환 |
| IP 변경 작업을 3번 반복 후 NAT 구성 결정 | Default Switch의 근본적 한계 | Internal Switch + NAT로 영구 해결 |

---

### 요약

Default Switch는 편리하지만 **서버 제품처럼 IP 의존성이 있는 환경에서는 부적합**합니다. Internal Switch + NAT 구성은 한 번만 설정하면 영구적으로 IP가 고정되며, 사내망에 영향을 주지 않는 안전한 방법입니다.

---

### **1. ICS가 뭐야?**

**Internet Connection Sharing** — Windows에 내장된 인터넷 공유 기능입니다. 호스트 PC의 인터넷을 다른 네트워크 어댑터에 공유해주는 역할인데, Default Switch가 내부적으로 이걸 사용합니다. 문제는 ICS가 재시작될 때마다 **자기가 관리하는 서브넷을 새로 생성**한다는 점입니다.

### **2. DHCP 서버가 따로 있는 건가?**

별도 DHCP 서버가 있는 게 아니라, **Default Switch 자체가 간이 DHCP 서버 역할**을 합니다. Default Switch가 ICS를 통해 DHCP를 자동으로 돌리면서 VM에 IP를 할당해주는 구조입니다. 그래서 Default Switch의 서브넷이 바뀌면 → DHCP가 주는 IP도 바뀌는 것입니다.

우리가 만든 Internal Switch + NAT 구성에서는 **DHCP를 안 쓰고** VM 내부에서 `nmcli`로 직접 고정 IP를 수동 설정한 것이라 DHCP 의존성이 없습니다.

### **3. Internal Switch가 뭐야?**

Hyper-V 가상 스위치에는 3가지 타입이 있습니다:

| **타입** | **연결 범위** | **특징** |
| --- | --- | --- |
| **External** | 호스트 + VM + 외부 네트워크 | 물리 NIC에 브릿지, 사내망에 직접 노출 |
| **Internal** | 호스트 ↔ VM만 통신 | 외부 네트워크와 직접 연결 안 됨 |
| **Private** | VM ↔ VM만 통신 | 호스트도 접근 불가 |

우리가 만든 `FasooNAT`은 **Internal** 타입이라 호스트 PC와 VM 사이에서만 동작하는 **격리된 가상 네트워크**입니다. 여기에 Windows NAT를 추가해서 인터넷 접속도 가능하게 한 것입니다.

### **4. 로컬 PC에서 가상 스위치를 만들어서 VM을 고정하는 건가?**

네 맞습니다. 정확한 구조는:

`[VM: 192.168.100.10] ←──→ [가상 스위치 FasooNAT] ←──→ [호스트: 192.168.100.1]
                                                            │
                                                      [Windows NAT]
                                                            │
                                                      [사내망/인터넷]`

- 가상 스위치(`FasooNAT`)를 직접 만들고
- 호스트 측 IP를 `192.168.100.1`로 **고정**하고
- VM 측 IP를 `192.168.100.10`으로 **고정**하고
- NAT 규칙을 추가해서 인터넷도 되게 함

**모든 것을 우리가 직접 제어**하기 때문에 재부팅해도 바뀔 일이 없습니다.

### **5. 기존에는 Default Switch라서 바뀌었던 거지?**

맞습니다. 정리하면:

|  | **Default Switch** | **FasooNAT (Internal + NAT)** |
| --- | --- | --- |
| 서브넷 관리 | Hyper-V/ICS가 **자동** 관리 | **우리가 직접** 지정 |
| IP 할당 | 내장 DHCP가 **자동** 할당 | `nmcli`로 **수동** 고정 |
| 게이트웨이 | 재부팅마다 **변동** | `192.168.100.1` **고정** |
| 사용자 제어 | **불가** (설정 변경 안 됨) | **완전 제어** 가능 |

Default Switch는 "편하게 쓰라고" 만든 것이라 사용자가 설정을 건드릴 수 없고, 그래서 IP 고정이 근본적으로 불가능한 것입니다.
