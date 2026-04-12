# 방화벽과 SELinux

## 사전 지식

**패킷 필터링:** 네트워크를 통해 들어오고 나가는 패킷을 정해진 규칙에 따라 허용(ACCEPT)하거나 차단(DROP/REJECT)하는 메커니즘이다. 서버에 도달하기 전 마지막 문지기 역할을 한다.

---

**ACCEPT / REJECT / DROP의 차이:**

| 동작       | 패킷 처리 | 클라이언트에게 응답                   | 보안 수준     |
| -------- | ----- | ---------------------------- | --------- |
| `ACCEPT` | 통과 허용 | 정상 응답                        | —         |
| `REJECT` | 차단    | ICMP "connection refused" 반환 | 서버 존재를 노출 |
| `DROP`   | 차단    | 응답 없음 (묵살)                   | 서버 존재를 숨김 |

`REJECT`는 클라이언트가 즉시 포기하므로 사용자 경험이 낫고, `DROP`은 클라이언트가 타임아웃까지 대기하지만 서버 존재를 알 수 없어 보안상 강력하다.

---

**netfilter 체인:** 리눅스 커널에는 `netfilter`라는 패킷 필터링 프레임워크가 내장되어 있다. 커널은 패킷의 이동 방향에 따라 다른 체인에서 규칙을 평가한다.

- `INPUT` — 이 서버로 들어오는 패킷
- `OUTPUT` — 이 서버에서 나가는 패킷
- `FORWARD` — 이 서버를 통과하는 패킷 (라우터 역할 시)

---

**주요 서비스 포트:**

| 서비스   | 포트  | 프로토콜    |
| ----- | --- | ------- |
| SSH   | 22  | TCP     |
| HTTP  | 80  | TCP     |
| HTTPS | 443 | TCP     |
| DNS   | 53  | TCP/UDP |
| NTP   | 123 | UDP     |
| SMTP  | 25  | TCP     |

---

**DAC (Discretionary Access Control, 임의접근제어):** 리눅스의 기본 권한 모델이다. 파일의 소유자(user), 그룹(group), 기타(other)에 대해 읽기(r), 쓰기(w), 실행(x) 권한을 부여한다. "임의(Discretionary)"라는 표현처럼, 파일 소유자가 자기 파일의 권한을 자유롭게 설정할 수 있다.

```bash
# DAC 권한 확인
ls -l /etc/passwd
-rw-r--r--. 1 root root 1234 Jan 1 00:00 /etc/passwd
# 소유자(root): rw-, 그룹(root): r--, 기타(other): r--
```

DAC에서 커널이 판단하는 기준은 단 하나: **"이 프로세스를 실행하는 사용자(UID/GID)가 이 파일에 대해 권한이 있는가?"** 즉 프로세스의 정체(어떤 프로그램인지)는 전혀 따지지 않는다.

---

**DAC의 한계:**

```
상황: httpd가 apache 사용자(UID=48)로 실행 중

DAC가 보는 것:    "apache(UID=48)가 이 파일을 읽을 수 있나?"
DAC가 안 보는 것:  "이 프로세스가 웹 서버인지 아닌지"

결과:
  /etc/passwd         → -rw-r--r-- → other에 r 있음 → apache도 읽기 가능 ✓
  /home/user1/.bashrc → -rw-r--r-- → other에 r 있음 → apache도 읽기 가능 ✓
  /var/log/messages   → -rw------- → other에 권한 없음 → apache 읽기 불가 ✗
```

웹 서버가 `/etc/passwd`나 사용자 홈의 파일을 읽을 이유는 전혀 없지만, DAC는 "apache 사용자가 other로서 읽기 권한이 있으니 허용"이라고 판단한다. 만약 httpd가 취약점으로 공격자에게 탈취되면, 공격자는 이 권한 범위 안에서 의도하지 않은 파일들까지 읽을 수 있다.

root 계정의 경우 DAC 규칙 자체를 완전히 무시하므로, root로 실행되는 프로세스가 탈취되면 시스템 전체가 노출된다.

---

**MAC (Mandatory Access Control, 강제접근제어):** DAC의 한계를 보완한다. "**어떤 프로세스가** 어떤 파일에 접근할 수 있는가"를 관리자가 정의한 정책으로 강제한다. 파일 소유자도, root도 이 정책을 바꿀 수 없다("강제"의 의미).

```
같은 상황을 MAC(SELinux)에서 보면:

httpd (도메인: httpd_t) → /etc/passwd (타입: etc_t)
  → 정책에 "httpd_t가 etc_t를 읽어도 된다"는 규칙이 있는가? → 없음 → 거부

httpd (도메인: httpd_t) → /var/www/html/index.html (타입: httpd_sys_content_t)
  → 정책에 "httpd_t가 httpd_sys_content_t를 읽어도 된다"는 규칙이 있는가? → 있음 → 허용
```

MAC은 프로세스의 **정체(도메인)**를 따진다. httpd는 웹 콘텐츠만 읽으면 되므로, 정책은 `httpd_t → httpd_sys_content_t` 접근만 허용하고 나머지는 전부 거부한다. DAC에서 other 읽기 권한이 있어도 MAC이 막는다.

| | DAC | MAC (SELinux) |
| --- | --- | --- |
| 판단 기준 | 사용자(UID/GID) | 프로세스 도메인 + 파일 타입 |
| 권한 설정 주체 | 파일 소유자 | 시스템 관리자 (정책) |
| root 예외 | 모든 규칙 무시 | 정책에 없으면 거부 |
| 목적 | "누가 이 파일을 소유하는가" | "이 프로세스가 이 자원에 접근해도 되는가" |

---

**LSM (Linux Security Module):** 리눅스 커널이 보안 모듈을 플러그인 방식으로 붙일 수 있도록 제공하는 프레임워크다. 커널 코드 곳곳에 "보안 검사 지점(hook)"이 삽입되어 있고, LSM에 등록된 보안 모듈이 각 hook에서 허용/거부를 판단한다. SELinux는 이 LSM을 통해 커널에 내장되어 있다. (AppArmor도 LSM 기반이지만, RHEL은 SELinux를 사용한다.)

---

**AVC (Access Vector Cache):** SELinux가 접근 허용/거부 판단을 매번 정책 전체에서 조회하면 느려진다. 그래서 판단 결과를 AVC라는 캐시에 저장해 같은 요청이 반복될 때 빠르게 응답한다. 거부가 발생하면 이 캐시 이름을 따서 **AVC 거부 메시지**가 로그에 남는다.

---

**DAC → MAC 검사 흐름:** 접근 요청이 들어오면 커널은 DAC를 먼저 평가하고, DAC가 허용한 경우에만 SELinux(MAC)를 추가로 평가한다. 두 단계를 모두 통과해야 접근이 허용된다.

```
접근 요청 → [DAC 검사] → 허용 → [SELinux(MAC) 검사] → 허용 → 접근 성공
                ↓                         ↓
              거부                       거부
         (Permission denied)        (audit.log에 AVC 기록)

DAC 거부: ls -l 로 권한 확인 → chmod/chown으로 해결
MAC 거부: ls -Z 로 컨텍스트 확인 → restorecon/setsebool로 해결
```

이 순서 때문에 트러블슈팅 시 "DAC 문제인지 SELinux 문제인지"를 먼저 구분해야 한다. `setenforce 0`으로 SELinux를 Permissive로 바꿔서 접근이 되면 SELinux 문제, 여전히 안 되면 DAC(또는 다른) 문제다.

---

# Part 1 — firewalld 방화벽

## firewalld 개념

### iptables → nftables → firewalld

리눅스 방화벽 도구는 세 세대를 거쳐 발전했다.

- **iptables:** netfilter를 사용자 공간에서 제어하는 CLI 도구. 오랫동안 표준이었으나, 규칙 변경 시 전체 테이블을 교체해야 해서 규칙이 많아질수록 성능이 저하되고 관리가 복잡해지는 단점이 있었다.
- **nftables:** iptables의 후계자로 RHEL 8부터 기본 채택. 규칙을 원자적(atomic)으로 교체할 수 있어 성능이 좋고, IPv4/IPv6/ARP를 하나의 문법으로 통합 관리한다.
- **firewalld:** nftables를 **백엔드**로 사용하면서, zone/service 추상화를 제공하는 **프론트엔드 관리 데몬**이다.

---

### 백엔드란?

여기서 "백엔드"는 **실제로 커널에 패킷 필터링 규칙을 넣는 엔진**을 뜻한다. firewalld 자체가 패킷을 필터링하는 것이 아니라, 관리자의 명령을 받아서 nftables 규칙으로 변환한 뒤 커널의 netfilter에 전달한다.

```
관리자가 하는 것:
  firewall-cmd --add-service=http --permanent

내부에서 일어나는 일:
  ┌────────────────────────────────────────────────────────┐
  │  사용자 공간 (User Space)                                │
  │                                                        │
  │  firewall-cmd (CLI)                                    │
  │       ↓ D-Bus                                          │
  │  firewalld (데몬) ─ "http = TCP 80이니까..."             │
  │       ↓ nftables 규칙 생성                               │
  │  nft (nftables 엔진)                                    │
  ├────────────────────────────────────────────────────────┤
  │  커널 공간 (Kernel Space)                                │
  │                                                        │
  │  netfilter ─ 실제 패킷 필터링 수행                         │
  │  (INPUT/OUTPUT/FORWARD 체인에서 규칙 평가)                 │
  └────────────────────────────────────────────────────────┘
```

정리하면:

| 계층 | 역할 | 비유 |
| --- | --- | --- |
| **firewalld** (프론트엔드) | 관리자와 대화. zone/service 같은 추상화 제공 | 리모컨 |
| **nftables** (백엔드) | firewalld의 명령을 커널이 이해하는 규칙으로 변환 | TV 수신 칩 |
| **netfilter** (커널) | 커널 안에서 실제 패킷을 검사하고 ACCEPT/DROP | TV 화면 |

관리자가 `firewall-cmd`만 쓰면 되고, nftables를 직접 만질 일은 거의 없다. 다만 firewalld가 해결하지 못하는 고급 필터링(예: 특정 패킷 헤더 매칭, 커넥션 트래킹 세부 제어)이 필요할 때만 `nft` 명령어로 nftables를 직접 다룬다.

> **RHEL 10 주의:** RHEL 10에서 firewalld는 `nftables`를 백엔드로 사용한다. `iptables` 백엔드는 더 이상 지원되지 않는다. firewalld와 nftables(또는 iptables)를 동시에 직접 실행하면 규칙이 충돌하므로 하나만 사용해야 한다.

---

### 기본 정책 모델 — 화이트리스트 방식

firewalld는 **기본적으로 모든 인바운드 트래픽을 거부**하고, 관리자가 **명시적으로 허용한 서비스·포트만 통과**시키는 화이트리스트(whitelist) 방식으로 동작한다. 즉 "열어주지 않으면 막혀 있다"가 기본이다.

```
외부 → 서버

  허용 목록에 있음?  ──  YES → 통과 (ACCEPT)
         │
         NO → 차단 (zone에 따라 REJECT 또는 DROP)
```

이 모델 때문에 서비스를 설치하고 실행해도, 방화벽에서 해당 포트를 열지 않으면 외부 접근이 불가능하다. 예를 들어 `httpd`를 시작해도 `firewall-cmd --add-service=http`를 하지 않으면 외부에서 80 포트로 접속할 수 없다.

반면 **아웃바운드 트래픽(서버 → 외부)**은 기본적으로 허용된다. 서버에서 `dnf install`이나 `curl` 같은 외부 요청은 별도 설정 없이 동작한다.

---

### zone이란

firewalld의 핵심 추상화 단위다. zone은 "이 네트워크 인터페이스(또는 트래픽 출처)를 얼마나 신뢰하는가"에 대한 신뢰 수준을 정의하고, 그에 맞는 허용 규칙의 묶음이다.

네트워크 인터페이스나 특정 출발지 IP를 zone에 할당하면, 해당 zone에 정의된 규칙이 그 트래픽에 적용된다.

> **RHEL 10 기본값:** 새로 설치된 RHEL 10 시스템의 기본 zone은 `public`이며, SSH와 DHCPv6-client 서비스만 허용되어 있다. 즉 웹 서버(httpd)를 설치해도 방화벽에서 HTTP/HTTPS를 열어주지 않으면 외부에서 접근할 수 없다.

RHEL 10 기본 제공 zone:

| zone       | 신뢰 수준 | 기본 동작                            |
| ---------- | ----- | -------------------------------- |
| `trusted`  | 최고    | 모든 트래픽 허용                        |
| `home`     | 높음    | 지정된 서비스만 허용                      |
| `internal` | 높음    | home과 유사                         |
| `work`     | 중간    | 지정된 서비스만 허용                      |
| `public`   | 낮음    | ssh, dhcpv6-client만 허용 (기본 zone) |
| `external` | 낮음    | Masquerade 활성화, ssh만 허용          |
| `dmz`      | 낮음    | ssh만 허용                          |
| `block`    | 거부    | 모든 인바운드 차단 (ICMP로 거부 응답)         |
| `drop`     | 거부    | 모든 인바운드 묵살 (응답 없음)               |

`block`과 `drop`의 차이: `block`은 ICMP "connection refused"를 돌려보내고, `drop`은 아무 응답도 없이 패킷을 버린다. 보안 측면에서는 `drop`이 서버 존재 자체를 숨기므로 더 강력하다.

---

### zone의 target

`--list-all` 출력에서 `target: default`라는 항목이 보인다. target은 zone의 **최종 판단** — 해당 zone에 명시된 서비스/포트/Rich Rule 어디에도 매치되지 않은 패킷을 어떻게 처리할지를 결정한다.

| target | 동작 |
| --- | --- |
| `default` | zone별 기본 동작 적용 (대부분의 zone에서 REJECT) |
| `ACCEPT` | 규칙에 없어도 전부 허용 (`trusted` zone이 이 target) |
| `REJECT` | 규칙에 없으면 ICMP 거부 응답과 함께 차단 |
| `DROP` | 규칙에 없으면 응답 없이 묵살 |

`public` zone의 `target: default`는 실질적으로 REJECT와 같다. 허용 목록에 없는 패킷은 차단되고 ICMP 거부 응답을 돌려보낸다.

```bash
# zone의 target 변경 (예: public zone을 DROP으로)
firewall-cmd --zone=public --set-target=DROP --permanent
firewall-cmd --reload
```

---

### zone 할당 우선순위

패킷이 들어올 때 firewalld는 다음 순서로 zone을 결정한다.

1. **출발지 IP/서브넷 기반 zone** — `--add-source`로 등록된 경우 우선 적용
2. **인터페이스 기반 zone** — 패킷이 들어온 NIC에 할당된 zone
3. **기본 zone** — 위 두 경우에 해당하지 않으면 default zone 적용

---

## firewall-cmd 기본 조작

### 상태 확인

```bash
# firewalld 서비스 상태
systemctl status firewalld

# 현재 기본 zone 확인
firewall-cmd --get-default-zone

# 모든 zone 목록
firewall-cmd --get-zones

# 특정 zone의 전체 설정 확인
firewall-cmd --zone=public --list-all

# 기본 zone의 전체 설정 확인
firewall-cmd --list-all
```

`--list-all` 출력 예시:

```
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp1s0
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

---

### 인터페이스와 zone 연결

```bash
# 인터페이스가 어느 zone에 속하는지 확인
firewall-cmd --get-zone-of-interface=enp1s0

# 인터페이스를 특정 zone으로 변경
firewall-cmd --zone=internal --change-interface=enp1s0 --permanent

# 기본 zone 변경
firewall-cmd --set-default-zone=public
```

`--set-default-zone`은 `--permanent` 없이도 영구 적용된다.

---

## 서비스 단위 허용 / 차단

서비스 파일(`/usr/lib/firewalld/services/*.xml`)에는 서비스 이름과 해당 포트/프로토콜 매핑이 미리 정의되어 있다. 포트 번호를 외울 필요 없이 서비스 이름으로 관리할 수 있다.

```bash
# 사용 가능한 모든 서비스 목록
firewall-cmd --get-services

# 특정 서비스 정의 파일 확인
cat /usr/lib/firewalld/services/http.xml
```

서비스 XML 파일 구조 예시 (`/usr/lib/firewalld/services/http.xml`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>WWW (HTTP)</short>
  <description>HTTP is the protocol used to serve Web pages.</description>
  <port protocol="tcp" port="80"/>
</service>
```

하나의 서비스가 여러 포트를 사용할 수 있다 (예: Samba 서비스는 TCP 139, 445 등). 커스텀 서비스를 만들려면 `/etc/firewalld/services/`에 XML 파일을 작성한다.

> **RHEL 10 참고:** `/usr/lib/firewalld/`의 파일은 시스템 기본값이므로 직접 수정하면 안 된다. 커스텀 서비스나 zone 변경은 `/etc/firewalld/`에 작성하며, 이 디렉토리의 파일이 `/usr/lib/firewalld/`보다 우선 적용된다.

### 서비스 허용

```bash
# 런타임에 http 서비스 허용 (재부팅/reload 시 사라짐)
firewall-cmd --zone=public --add-service=http

# 영구 설정에 http 서비스 허용
firewall-cmd --zone=public --add-service=http --permanent

# 변경 사항 즉시 반영 (영구 설정 → 런타임 동기화)
firewall-cmd --reload
```

---

### 서비스 차단

```bash
# 런타임에서 제거
firewall-cmd --zone=public --remove-service=http

# 영구 설정에서 제거
firewall-cmd --zone=public --remove-service=http --permanent
firewall-cmd --reload
```

---

### 여러 서비스 한 번에

```bash
firewall-cmd --zone=public \
    --add-service=http \
    --add-service=https \
    --permanent
firewall-cmd --reload
```

---

## 포트 단위 허용 / 차단

미리 정의된 서비스 파일이 없는 경우, 또는 비표준 포트를 사용하는 경우 직접 포트를 지정한다.

```bash
# 8080/tcp 포트 허용
firewall-cmd --zone=public --add-port=8080/tcp --permanent

# 포트 범위 허용 (60000~61000 UDP)
firewall-cmd --zone=public --add-port=60000-61000/udp --permanent

# 포트 제거
firewall-cmd --zone=public --remove-port=8080/tcp --permanent

# 적용
firewall-cmd --reload

# 열린 포트 목록 확인
firewall-cmd --zone=public --list-ports
```

---

## 런타임 vs 영구 적용

firewalld의 설정에는 두 가지 레이어가 있다.

| 레이어               | 저장 위치             | 유지 시점                                              |
| ----------------- | ----------------- | -------------------------------------------------- |
| **런타임(Runtime)**  | 메모리               | 현재 실행 중에만 유효. `firewall-cmd --reload` 또는 재부팅 시 사라짐 |
| **영구(Permanent)** | `/etc/firewalld/` | 재부팅 후에도 유지. 반드시 `--reload`로 런타임에 반영해야 함            |

```
런타임 설정 ─────────────── 즉시 적용, 재부팅 시 소멸
영구 설정   ─ --reload ──── 재부팅 후 유지
```

### 권장 워크플로우

```bash
# 방법 A: 영구 설정 후 reload (권장)
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload

# 방법 B: 런타임 적용 후 테스트, 이상 없으면 영구화
firewall-cmd --zone=public --add-service=http           # 런타임 적용
# ... 서비스 동작 확인 ...
firewall-cmd --runtime-to-permanent                      # 현재 런타임 상태 전체를 영구 저장
```

방법 B는 먼저 동작을 확인하고 영구 저장하므로, 잘못된 설정으로 재접속이 막히는 상황을 방지할 수 있다. 특히 SSH 세션으로 원격 작업 중일 때 유용하다.

> **RHEL 10 원격 작업 팁:** SSH로 원격 서버를 관리할 때 방화벽 실수로 SSH가 차단되면 접속이 끊긴다. `--timeout` 옵션으로 SSH 서비스를 제거 테스트하거나, 런타임만 변경 후 검증 → `--runtime-to-permanent`으로 확정하는 습관을 들이면 사고를 방지할 수 있다.

---

## Rich Rule — 세밀한 접근 제어

서비스/포트 단위 허용만으로는 부족할 때, Rich Rule로 출발지 IP, 목적지 포트, 프로토콜을 조합한 정밀 규칙을 작성할 수 있다.

```bash
# 특정 IP(10.0.0.50)에서만 SSH 접근 허용
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.50" service name="ssh" accept' --permanent

# 특정 서브넷에서 HTTP 차단
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="http" reject' --permanent

# Rich Rule 목록 확인
firewall-cmd --zone=public --list-rich-rules

# Rich Rule 삭제
firewall-cmd --zone=public --remove-rich-rule='rule family="ipv4" source address="10.0.0.50" service name="ssh" accept' --permanent
```

Rich Rule 구조:

```
rule [family="ipv4|ipv6"]
     [source address="..."]
     [destination address="..."]
     [service name="..." | port port="..." protocol="..."]
     [accept | reject | drop | log]
```

### --add-source — IP 기반 zone 할당

인터페이스가 아닌 **출발지 IP/서브넷**을 기준으로 zone을 적용할 수 있다. 관리자 IP만 `trusted` zone으로 분류하는 식으로 쓴다.

```bash
# 관리 서브넷을 trusted zone에 할당
firewall-cmd --zone=trusted --add-source=10.0.0.0/24 --permanent
firewall-cmd --reload

# 확인
firewall-cmd --zone=trusted --list-sources
```

---

### --timeout — 일시적 허용

테스트 목적으로 일정 시간 후 자동 만료되는 규칙을 추가할 수 있다.

```bash
# 5분간만 8080 포트 열기 (300초 후 자동 제거)
firewall-cmd --zone=public --add-port=8080/tcp --timeout=300
```

---

### --panic-on / --panic-off — 긴급 차단

보안 사고 발생 시 모든 네트워크 트래픽을 즉시 차단한다. SSH 세션도 끊기므로 콘솔 접근이 가능할 때만 사용한다.

```bash
# 모든 트래픽 차단
firewall-cmd --panic-on

# 차단 해제
firewall-cmd --panic-off

# panic 상태 확인
firewall-cmd --query-panic
```

---

## 패킷이 서버에 도달하는 전체 흐름

외부에서 패킷이 서버의 서비스에 도달하기까지 여러 단계를 거친다. 방화벽은 그 중 하나일 뿐이며, 접속이 안 되는 원인이 항상 방화벽인 것은 아니다.

```
외부 클라이언트 → [네트워크 경로(라우터, 스위치)] → 서버 NIC 도착
                                                     ↓
                                              [1. netfilter / firewalld]
                                              허용 목록에 있는가?
                                                ↓ YES         ↓ NO
                                              통과          차단 (REJECT/DROP)
                                                ↓
                                              [2. 포트에서 LISTEN 중인 프로세스가 있는가?]
                                              (ss -tuln으로 확인)
                                                ↓ YES         ↓ NO
                                              전달         RST 패킷 반환 (Connection refused)
                                                ↓
                                              [3. SELinux]
                                              프로세스 도메인이 해당 자원에 접근 가능한가?
                                                ↓ YES         ↓ NO
                                              처리          거부 (AVC 로그)
                                                ↓
                                              [4. 애플리케이션 로직]
                                              정상 응답
```

이 흐름에서 핵심은 **각 계층이 독립적**이라는 점이다. 방화벽을 열어도 프로세스가 LISTEN하지 않으면 접속이 안 되고, 프로세스가 LISTEN해도 SELinux가 막으면 동작하지 않는다.

---

## 설정 후 검증

방화벽 규칙을 변경했을 때 "제대로 적용됐는가?"를 확인하는 방법이다.

### 서버 쪽 확인

```bash
# 1. firewalld에 규칙이 들어갔는지
firewall-cmd --list-all
# services: 에 http가 있는지, ports: 에 8080/tcp가 있는지 확인

# 2. 프로세스가 실제로 해당 포트에서 LISTEN 중인지
ss -tlnp | grep :80
# tcp  LISTEN  0  511  0.0.0.0:80  ...  users:(("httpd",pid=...))
```

방화벽을 열었는데 `ss`에서 해당 포트가 안 보이면, 서비스 자체가 실행되지 않았거나 다른 포트에서 동작하는 것이다.

---

### 클라이언트 쪽 확인

```bash
# 같은 서버에서 로컬 테스트
curl http://localhost/

# 다른 머신에서 원격 테스트 (서버 IP가 192.168.1.10이라면)
curl http://192.168.1.10/

# 포트만 열려있는지 확인 (TCP 연결 시도)
ss -tn dst 192.168.1.10:80
```

로컬(`localhost`)에서는 되는데 원격에서 안 되면, 방화벽이 원인일 가능성이 높다. 로컬에서도 안 되면 서비스 자체 문제다.

---

## 방화벽 트러블슈팅

"서비스 추가했는데 접속이 안 됩니다"는 가장 흔한 질문이다. 원인을 순서대로 좁혀가는 체크리스트:

```
1. firewalld가 실행 중인가?
   systemctl status firewalld
   → inactive면 방화벽 자체가 꺼져 있음

2. 올바른 zone에 규칙을 추가했는가?
   firewall-cmd --get-active-zones
   → 인터페이스가 어느 zone에 속하는지 확인
   → 예: enp1s0이 public인데 internal에 규칙을 추가했으면 적용 안 됨

3. 런타임에 반영됐는가?
   firewall-cmd --list-all
   → --permanent만 하고 --reload를 안 했으면 런타임에 없음

4. 서비스가 실제로 LISTEN하고 있는가?
   ss -tlnp | grep :<포트번호>
   → LISTEN 중인 프로세스가 없으면 방화벽 문제가 아님

5. SELinux가 차단하고 있는가?
   ausearch -m AVC -ts recent
   → 비표준 포트면 semanage port 필요

6. 네트워크 경로에 문제가 있는가?
   ping <서버IP>   (클라이언트에서)
   → 게이트웨이, 라우팅 문제 확인
```

### firewalld 로깅

기본적으로 firewalld가 차단한 패킷은 별도 로그를 남기지 않는다. 디버깅이 필요할 때 Rich Rule의 `log` 액션을 사용하면 특정 트래픽에 대해 로깅할 수 있다.

```bash
# HTTP 접근을 로그에 기록 (prefix로 구분)
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" service name="http" log prefix="HTTP_ACCESS: " level="info" accept' --permanent
firewall-cmd --reload
```

로그는 `/var/log/messages` 또는 `journalctl`에 기록되며, `prefix`로 grep하면 해당 규칙에 의한 로그만 필터링할 수 있다.

```bash
# 로그 확인
journalctl -k | grep "HTTP_ACCESS"
```

거부된 패킷을 로깅하려면 `accept` 대신 `log` 다음에 `reject`나 `drop`을 넣는다.

```bash
# 특정 IP에서 오는 패킷을 로그 남기고 차단
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.100" log prefix="BLOCKED: " level="warning" drop' --permanent
firewall-cmd --reload
```

---

# Part 2 — SELinux

## SELinux 동작 방식

SELinux의 핵심 질문은 하나다: **"이 프로세스가 이 파일(또는 자원)에 접근해도 되는가?"**

이 질문에 답하기 위해 SELinux는 모든 프로세스와 파일에 **보안 컨텍스트(Security Context)**라는 레이블을 붙이고, 그 레이블 간의 허용 규칙을 **정책(Policy)**으로 관리한다.

### 보안 컨텍스트 구조

보안 컨텍스트는 콜론(`:`)으로 구분된 4개 필드로 구성된다.

```
user:role:type:level
예시: system_u:system_r:httpd_t:s0
```

| 필드       | 예시                               | 설명                                       |
| -------- | -------------------------------- | ---------------------------------------- |
| user     | `system_u`, `unconfined_u`       | SELinux 사용자 식별자                          |
| role     | `system_r`, `object_r`           | 접근 역할 (프로세스는 `system_r`, 파일은 `object_r`) |
| **type** | `httpd_t`, `httpd_sys_content_t` | **가장 중요한 필드. 허용 규칙의 기준**                 |
| level    | `s0`, `s0-s0:c0.c1023`           | MLS(Multi-Level Security) 수준             |

실무에서 가장 중요한 것은 **type** 필드다. SELinux 정책의 대부분은 "어떤 type의 프로세스가 어떤 type의 파일에 접근할 수 있는가"를 정의하는 규칙으로 이루어져 있다.

> **RHEL 10 참고:** RHEL 10에서 SELinux는 기본적으로 `Enforcing` 모드로 설정되며, `targeted` 정책이 적용된다. Red Hat은 SELinux를 비활성화(Disabled)하지 말 것을 강력히 권고하며, 트러블슈팅 시에도 `Permissive` 모드를 사용하도록 안내한다.

**targeted 정책 vs mls 정책:** RHEL에서 기본으로 사용하는 정책은 `targeted`다. 이 정책에서는 네트워크 서비스 등 주요 프로세스만 SELinux 도메인으로 제한(confined)하고, 일반 사용자 프로세스는 `unconfined_t` 도메인에서 실행된다. 즉 일상적인 사용자 작업은 거의 제한을 받지 않지만, httpd, sshd, mysqld 같은 데몬은 엄격하게 격리된다.

```
targeted 정책에서:
  httpd    → httpd_t      (제한됨 - confined)
  sshd     → sshd_t       (제한됨 - confined)
  일반 사용자 → unconfined_t  (제한 없음 - unconfined)
```

`mls` 정책은 군사/기밀 등급 환경을 위한 Multi-Level Security 정책으로 일반 서버에서는 거의 사용하지 않는다.

---

### 보안 컨텍스트 확인

```bash
# 파일의 SELinux 컨텍스트 확인
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html

# 프로세스의 SELinux 컨텍스트 확인
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0    1234 ?  00:00:01 httpd

# 현재 터미널 세션의 컨텍스트 확인
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

---

### Type Enforcement

SELinux 정책의 핵심 메커니즘이다. 프로세스의 **도메인(domain)**과 파일의 **타입(type)** 간 허용 규칙을 정의한다. 프로세스가 실행되면 특정 도메인으로 전환되고, 그 도메인이 접근할 수 있는 파일 타입이 정책으로 제한된다.

```
httpd 프로세스 (도메인: httpd_t)
    ├── /var/www/html/  (타입: httpd_sys_content_t)  → 허용 (정책에 allow 규칙 있음)
    ├── /tmp/           (타입: tmp_t)                 → 거부 (허용 규칙 없음)
    └── /data/mysql/    (타입: mysqld_db_t)           → 거부 (허용 규칙 없음)
```

"기본적으로 거부(default deny)" 원칙: 명시적으로 허용하는 규칙이 없으면 모든 접근은 거부된다.

---

### 도메인 전환 (Domain Transition)

프로세스가 다른 프로그램을 실행할 때, 실행된 프로그램이 다른 SELinux 도메인으로 자동 전환되는 메커니즘이다.

```
systemd (init_t) → httpd 바이너리 실행 → httpd 프로세스는 httpd_t 도메인으로 전환
```

이 전환은 정책에 미리 정의되어 있다. 덕분에 systemd가 httpd를 시작하면 httpd는 자동으로 격리된 도메인에서 실행된다. 사용자가 직접 `/usr/sbin/httpd`를 실행하면 `unconfined_t`에서 시작하므로 같은 바이너리라도 다른 제한이 적용될 수 있다.

---

## 동작 모드

SELinux는 3가지 모드로 동작한다.

| 모드             | 정책 적용   | 로그 기록 | 권장 사용        |
| -------------- | ------- | ----- | ------------ |
| **Enforcing**  | O       | O     | 운영 환경 기본값    |
| **Permissive** | X (기록만) | O     | 트러블슈팅, 정책 개발 |
| **Disabled**   | X       | X     | 사용 강력 비권장    |

`Permissive`는 정책을 실제로 적용하지 않지만 거부 메시지는 로그에 남긴다. "SELinux를 끄지 않고 문제를 파악할 수 있는" 안전한 디버깅 모드다.

`Disabled`는 파일 레이블링 자체가 중단되어, 나중에 다시 활성화할 때 전체 파일 시스템 재레이블링이 필요하다.

> **RHEL 10 경고:** RHEL 10 공식 문서는 Disabled 사용을 강력히 비권장한다. Disabled 상태에서 생성된 파일은 SELinux 레이블이 부여되지 않으므로, 다시 Enforcing으로 전환하면 광범위한 접근 거부가 발생한다. `Disabled` → `Enforcing` 직접 전환 대신 반드시 `Permissive`를 먼저 거쳐 `fixfiles -F onboot`와 재부팅으로 전체 재레이블링을 수행해야 한다.

### 현재 모드 확인

```bash
# 현재 동작 모드 확인
getenforce
# Enforcing

# 상세 상태 확인 (정책 이름, 버전 등 포함)
sestatus
```

`sestatus` 출력 예시:

```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```

`Current mode`와 `Mode from config file`이 다르다면, `setenforce`로 임시 변경한 상태라는 뜻이다.

---

### 임시 모드 변경 (재부팅 시 원복)

```bash
# Permissive로 전환
setenforce 0

# Enforcing으로 전환
setenforce 1

# 확인
getenforce
```

`setenforce`로 변경한 내용은 재부팅하면 `/etc/selinux/config`의 설정으로 돌아온다.

---

### 영구 모드 변경

```bash
# /etc/selinux/config 파일 수정
vi /etc/selinux/config
```

```ini
# /etc/selinux/config
SELINUX=enforcing       # enforcing / permissive / disabled
SELINUXTYPE=targeted    # 사용할 정책 타입
```

변경 후 재부팅해야 적용된다.

> **주의:** `Disabled` → `Enforcing`으로 직접 전환하면 파일 레이블이 없어서 부팅 실패가 발생할 수 있다. `Disabled`에서 복귀할 때는 반드시 `Permissive`를 먼저 거쳐 파일 시스템 재레이블링 후 `Enforcing`으로 올린다.

---

## SELinux Boolean

Boolean은 정책의 일부를 재부팅 없이 켜고 끌 수 있는 스위치다. 정책을 새로 컴파일하지 않고도 특정 동작의 허용 여부를 조정할 수 있다.

예를 들어 `httpd_can_network_connect_db`라는 Boolean은 Apache 웹 서버가 DB에 네트워크로 연결할 수 있는지를 제어한다. 기본값은 `off`이며, PHP 앱이 MariaDB에 접속해야 한다면 이 Boolean을 켜야 한다.

```bash
# 모든 Boolean 목록과 현재 값 확인
getsebool -a

# 특정 서비스 관련 Boolean 검색
getsebool -a | grep httpd

# 특정 Boolean 값 확인
getsebool httpd_can_network_connect_db

# Boolean 임시 변경 (재부팅 시 원복)
setsebool httpd_can_network_connect_db on

# Boolean 영구 변경 (-P 옵션)
setsebool -P httpd_can_network_connect_db on
```

`-P` 옵션을 붙이면 영구 적용된다. Boolean의 현재 의미(어떤 동작을 허용하는지)를 보려면 `selinux-policy-devel` 패키지 설치 후 `semanage boolean -l`을 사용한다.

> **RHEL 10 참고:** RHEL 10에서 Boolean 수는 수백 개에 달한다. 서비스별로 필요한 Boolean을 찾으려면 `getsebool -a | grep <서비스명>` 패턴이 유용하다. 자주 쓰이는 예시:
>
> - `httpd_can_network_connect` — httpd가 네트워크 연결 허용
> - `httpd_can_network_connect_db` — httpd → DB 연결 허용
> - `httpd_enable_homedirs` — httpd가 사용자 홈 디렉토리 접근 허용
> - `ftpd_anon_write` — vsftpd 익명 쓰기 허용

---

## SELinux 로그 분석

SELinux가 접근을 거부할 때 감사 로그(audit log)에 AVC(Access Vector Cache) 거부 메시지를 남긴다.

### audit.log 직접 확인

```bash
# 최근 AVC 거부 메시지 전체 조회
ausearch -m AVC,USER_AVC,SELINUX_ERR,USER_SELINUX_ERR -ts recent

# 오늘 발생한 거부만
ausearch -m AVC -ts today

# 특정 프로세스 관련
ausearch -m AVC -c httpd
```

AVC 메시지 예시:

```
type=AVC msg=audit(1553609555.619:127): avc: denied { write } for
pid=4097 comm="passwd" path="/root/test" dev="dm-0" ino=17142697
scontext=unconfined_u:unconfined_r:passwd_t:s0-s0:c0.c1023
tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
```

메시지 해석:

- `denied { write }` — write 권한이 거부됨
- `comm="passwd"` — 접근을 시도한 프로세스
- `path="/root/test"` — 접근 대상 파일
- `scontext` — 접근을 시도한 프로세스의 컨텍스트 (source)
- `tcontext` — 접근 대상 파일의 컨텍스트 (target)
- `tclass=file` — 접근 대상 오브젝트 클래스

---

### sealert — 사람이 읽기 쉬운 분석 결과

`setroubleshoot-server` 패키지가 설치되어 있다면 `sealert`로 거부 원인과 해결책을 자동 분석해준다.

```bash
# 설치
dnf install setroubleshoot-server

# 모든 AVC 이벤트 분석
sealert -l "*"

# /var/log/messages에서 SELinux 차단 메시지 검색
grep "SELinux is preventing" /var/log/messages
```

`sealert` 출력에는 문제 설명, 신뢰도 기반 해결책 제안, `restorecon`이나 `setsebool` 명령어가 함께 나온다.

> **RHEL 10 참고:** `setroubleshoot-server` 패키지가 설치되어 있으면 AVC 거부 발생 시 `/var/log/messages`에 사람이 읽기 쉬운 형태의 메시지(`SELinux is preventing ...`)가 자동 기록된다. 이 메시지에 포함된 `sealert -l <UUID>` 명령을 실행하면 상세 분석 결과와 해결 방법을 확인할 수 있다.

---

## 파일 컨텍스트 관리

### restorecon — 컨텍스트 복구

파일을 복사하거나 이동하면 원래 위치의 컨텍스트가 따라오지 않고 목적지 규칙과 달라질 수 있다. `restorecon`은 SELinux 정책에 정의된 기본 컨텍스트로 복원한다.

```bash
# 특정 파일 컨텍스트 복원
restorecon -v /var/www/html/index.html

# 디렉토리 전체 재귀 복원
restorecon -R -v /var/www/html/
```

`-v`는 변경 내용을 출력, `-R`은 재귀 적용이다.

---

### cp vs mv — 컨텍스트 동작의 차이

파일 이동 방식에 따라 SELinux 컨텍스트 결과가 완전히 다르다. 이 차이를 모르면 컨텍스트 불일치 문제를 만나게 된다.

| 명령                      | 컨텍스트 동작                             | 결과                  |
| ----------------------- | ----------------------------------- | ------------------- |
| `cp` (복사)               | **새 파일 생성** → 목적지 디렉토리의 기본 컨텍스트를 상속 | 대부분 정상              |
| `cp --preserve=context` | 원본 컨텍스트를 강제 유지                      | 목적지 규칙과 불일치 가능      |
| `mv` (이동)               | **원본 inode 유지** → 원본 컨텍스트가 그대로 따라감  | 다른 디렉토리로 옮기면 불일치 발생 |

```bash
# 예시: 홈 → 웹 루트로 이동
mv ~/index.html /var/www/html/
ls -Z /var/www/html/index.html
# user_home_t ← 원본 컨텍스트가 유지됨! httpd가 읽지 못함

# 복사로 하면:
cp ~/index.html /var/www/html/
ls -Z /var/www/html/index.html
# httpd_sys_content_t ← 목적지 컨텍스트 상속됨, 정상
```

`mv`로 옮긴 후에는 반드시 `restorecon`을 실행해야 한다.

---

### chcon — 임시 컨텍스트 변경

`chcon`은 파일의 컨텍스트를 즉시 변경하지만, `restorecon`이 실행되면 정책 기본값으로 되돌아간다. 테스트 목적으로만 사용한다.

```bash
# 컨텍스트 임시 변경
chcon -t httpd_sys_content_t /srv/myweb/index.html

# restorecon 실행하면 원래대로 돌아감
restorecon -v /srv/myweb/index.html
```

영구 변경이 필요하면 `chcon` 대신 `semanage fcontext` + `restorecon` 조합을 써야 한다.

---

### semanage fcontext — 컨텍스트 규칙 영구 등록

비표준 경로에 서비스 파일을 두는 경우, 해당 경로에 적용할 컨텍스트를 정책에 영구 등록해야 한다.

```bash
# /srv/myweb/ 하위 파일을 httpd가 읽을 수 있도록 컨텍스트 등록
semanage fcontext -a -t httpd_sys_content_t "/srv/myweb(/.*)?"

# 등록 후 실제 파일에 적용
restorecon -R -v /srv/myweb
```

`semanage fcontext`는 정책에 규칙을 등록하는 것이고, `restorecon`은 그 규칙을 실제 파일에 적용하는 두 단계가 항상 함께 쓰인다.

---

### semanage port — 포트 라벨 관리

서비스가 비표준 포트를 사용하면, SELinux가 해당 포트에 대한 바인드를 거부한다. `semanage port`로 포트에 적절한 타입 라벨을 등록해야 한다.

```bash
# 현재 http 관련 포트 라벨 확인
semanage port -l | grep http_port_t
# http_port_t   tcp   80, 443, 488, 8008, 8009, 8443

# httpd가 9876 포트에서 실행되도록 허용
semanage port -a -t http_port_t -p tcp 9876

# 등록 확인
semanage port -l | grep 9876
```

이 설정 없이 `httpd.conf`에서 `Listen 9876`을 지정하면 `systemctl start httpd`가 실패하고, audit.log에 `name_bind` 거부 메시지가 남는다.

---

### matchpathcon — 기대 컨텍스트 조회

파일의 현재 컨텍스트가 정책상 맞는지 검증한다. `restorecon`을 실행하기 전에 "어떤 컨텍스트가 되어야 하는지"를 미리 확인할 때 유용하다.

```bash
matchpathcon -V /var/www/html/*
# /var/www/html/index.html has context ..., should be system_u:object_r:httpd_sys_content_t:s0
```

`-V` 옵션은 현재 컨텍스트와 정책 기본값을 비교해서 불일치를 보여준다.

---

## 명령어 요약

### firewall-cmd — 상태 확인

| 명령어                                  | 용도                    |
| ------------------------------------ | --------------------- |
| `systemctl status firewalld`         | firewalld 서비스 상태      |
| `firewall-cmd --state`               | firewalld 동작 중인지 확인   |
| `firewall-cmd --get-default-zone`    | 현재 기본 zone 확인         |
| `firewall-cmd --get-zones`           | 사용 가능한 모든 zone 나열     |
| `firewall-cmd --get-active-zones`    | 활성 zone과 할당된 인터페이스 확인 |
| `firewall-cmd --list-all`            | 기본 zone 전체 설정 출력      |
| `firewall-cmd --zone=<z> --list-all` | 특정 zone 전체 설정 출력      |

### firewall-cmd — zone 관리

| 명령어                                                            | 용도                  |
| -------------------------------------------------------------- | ------------------- |
| `firewall-cmd --set-default-zone=<z>`                          | 기본 zone 변경 (자동 영구)  |
| `firewall-cmd --get-zone-of-interface=<dev>`                   | 인터페이스가 속한 zone 확인   |
| `firewall-cmd --zone=<z> --change-interface=<dev> --permanent` | 인터페이스를 다른 zone으로 이동 |
| `firewall-cmd --zone=<z> --add-source=<CIDR> --permanent`      | 출발지 IP를 zone에 할당    |

### firewall-cmd — 서비스·포트·적용

| 명령어                                                                | 용도                |
| ------------------------------------------------------------------ | ----------------- |
| `firewall-cmd --get-services`                                      | 정의된 서비스 목록        |
| `firewall-cmd --zone=<z> --add-service=<svc> --permanent`          | 서비스 허용 (영구)       |
| `firewall-cmd --zone=<z> --remove-service=<svc> --permanent`       | 서비스 제거 (영구)       |
| `firewall-cmd --zone=<z> --add-port=<port>/<proto> --permanent`    | 포트 허용 (영구)        |
| `firewall-cmd --zone=<z> --remove-port=<port>/<proto> --permanent` | 포트 제거 (영구)        |
| `firewall-cmd --reload`                                            | 영구 설정을 런타임에 반영    |
| `firewall-cmd --runtime-to-permanent`                              | 현재 런타임을 영구 저장     |
| `firewall-cmd --add-port=<port>/<proto> --timeout=<sec>`           | 일시적 허용            |
| `firewall-cmd --panic-on`                                          | 모든 트래픽 즉시 차단 (긴급) |
| `firewall-cmd --zone=<z> --add-rich-rule='...' --permanent`        | 정밀 규칙 추가          |

### SELinux — 모드·상태

| 명령어                      | 용도                           |
| ------------------------ | ---------------------------- |
| `getenforce`             | 현재 SELinux 모드 확인             |
| `sestatus`               | 상세 상태 (정책, 모드, config 모드 비교) |
| `setenforce 0`           | Permissive로 임시 전환 (재부팅 원복)   |
| `setenforce 1`           | Enforcing으로 임시 전환 (재부팅 원복)   |
| `vi /etc/selinux/config` | 영구 모드 변경 (재부팅 필요)            |

### SELinux — 컨텍스트

| 명령어                      | 용도                           |
| ------------------------ | ---------------------------- |
| `ls -Z <path>`           | 파일/디렉토리의 SELinux 컨텍스트 확인     |
| `ps -eZ | grep <proc>`   | 프로세스의 SELinux 도메인 확인         |
| `id -Z`                  | 현재 셸 세션의 컨텍스트 확인             |
| `chcon -t <type> <path>` | 컨텍스트 임시 변경 (restorecon 시 원복) |
| `restorecon -v <path>`   | 정책 기본 컨텍스트로 복원               |
| `restorecon -R -v <dir>` | 재귀적으로 디렉토리 전체 복원             |
| `matchpathcon -V <path>` | 현재 컨텍스트와 정책 기본값 비교           |

### SELinux — 정책·Boolean·로그

| 명령어                                              | 용도                                         |
| ------------------------------------------------ | ------------------------------------------ |
| `semanage fcontext -a -t <type> "<regex>"`       | 경로에 컨텍스트 규칙 영구 등록                          |
| `semanage port -a -t <type> -p <proto> <port>`   | 포트 라벨 등록                                   |
| `getsebool -a`                                   | 모든 Boolean 목록·현재 값                         |
| `setsebool -P <bool> on`                         | Boolean 영구 변경                              |
| `ausearch -m AVC -ts recent`                     | 최근 AVC 거부 메시지 조회                           |
| `sealert -l "*"`                                 | 모든 AVC 이벤트 분석 (`setroubleshoot-server` 필요) |
| `grep "SELinux is preventing" /var/log/messages` | 사람이 읽기 쉬운 차단 메시지 검색                        |

---

## 실습 시나리오 — firewalld 방화벽 정책 설정

> **안내:** 이 섹션은 **본인이 직접 실습하며 채워 넣는 공간**이다. 아래 시나리오는 흐름 가이드이며, 환경에 맞게 **명령, 출력, 이슈 메모**를 기록하면 된다.

### 목표

웹 서버를 설치하고, 방화벽이 차단하는 상태를 직접 확인한 뒤, 서비스·포트 허용 → 검증 → 런타임/영구 차이 → Rich Rule → 로깅 → 트러블슈팅까지 firewalld의 전체 워크플로우를 체험한다.

### Step 1 — httpd 설치 및 시작

```bash
dnf install httpd -y
systemctl start httpd
systemctl enable httpd
```

(출력 기록)

### Step 2 — 방화벽 현재 상태 확인 + zone target 확인

```bash
firewall-cmd --get-default-zone
firewall-cmd --list-all
```

(출력 기록 — 기본 zone이 `public`인지, `target: default`인지, 현재 허용된 services 목록 확인)

### Step 3 — 방화벽 열기 전 접근 테스트 (차단 체감)

```bash
# 서버 쪽: httpd가 80에서 LISTEN 중인지 확인
ss -tlnp | grep :80

# 로컬 접근 (loopback은 방화벽을 거치지 않음)
curl http://localhost/

# 외부에서 접근 시도 (다른 머신이 있다면, 또는 서버 IP로)
curl http://<서버IP>/
```

(출력 기록 — `ss`에서 httpd LISTEN 확인됨. localhost는 되지만 외부 IP로는 차단됨. "프로세스는 살아있는데 방화벽이 막고 있다"를 직접 체감)

### Step 4 — 서비스 허용 + 다계층 검증

```bash
# 서비스 추가
firewall-cmd --zone=public --add-service=http --add-service=https --permanent
firewall-cmd --reload

# 검증 1: 방화벽 규칙 반영 확인
firewall-cmd --list-all

# 검증 2: 프로세스 LISTEN 확인
ss -tlnp | grep :80

# 검증 3: 실제 접근 테스트
echo "Hello from firewalld lab" > /var/www/html/index.html
curl http://localhost/
```

(출력 기록 — services에 http/https 추가됨, curl 정상 응답. Step 3에서 안 되던 외부 접근도 이제 가능)

### Step 5 — 비표준 포트 허용

```bash
# 8080 포트 허용
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload

# 확인
firewall-cmd --list-ports
```

(출력 기록 — `8080/tcp` 표시 확인)

### Step 6 — 런타임 vs 영구 비교

```bash
# --permanent 없이 런타임만 추가
firewall-cmd --zone=public --add-port=9090/tcp
firewall-cmd --list-ports

# reload로 영구 설정만 남기기
firewall-cmd --reload
firewall-cmd --list-ports
```

(출력 기록 — reload 후 9090 사라짐 확인. `--permanent` 없이 추가한 것은 런타임에만 존재)

### Step 7 — Rich Rule (특정 IP 허용/차단)

```bash
# 특정 IP에서만 SSH 허용
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.50" service name="ssh" accept' --permanent
firewall-cmd --reload

# Rich Rule 확인
firewall-cmd --zone=public --list-rich-rules
```

(출력 기록 — Rich Rule이 추가됐는지 확인)

```bash
# 정리
firewall-cmd --zone=public --remove-rich-rule='rule family="ipv4" source address="10.0.0.50" service name="ssh" accept' --permanent
firewall-cmd --reload
```

### Step 8 — Rich Rule 로깅

```bash
# HTTP 접근을 로그에 기록
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" service name="http" log prefix="HTTP_ACCESS: " level="info" accept' --permanent
firewall-cmd --reload

# 접근 발생시키기
curl http://localhost/

# 로그 확인
journalctl -k | grep "HTTP_ACCESS"
```

(출력 기록 — `HTTP_ACCESS:` 접두사로 로그가 남는지 확인)

```bash
# 정리
firewall-cmd --zone=public --remove-rich-rule='rule family="ipv4" service name="http" log prefix="HTTP_ACCESS: " level="info" accept' --permanent
firewall-cmd --reload
```

### Step 9 — --timeout 일시적 허용

```bash
# 5분간만 3000 포트 열기
firewall-cmd --zone=public --add-port=3000/tcp --timeout=300
firewall-cmd --list-ports

# 5분 후 자동 제거되는지 확인 (또는 --reload로 즉시 확인)
```

(출력 기록 — timeout 후 자동으로 사라지는지 확인)

### Step 10 — 트러블슈팅 연습 (의도적 문제 → 원인 추적)

```bash
# 의도적으로 http 서비스 제거
firewall-cmd --zone=public --remove-service=http --permanent
firewall-cmd --reload

# 접속 시도
curl http://localhost/
curl http://<서버IP>/

# 트러블슈팅 체크리스트 순서대로 확인:
systemctl status firewalld          # 1. 실행 중?
firewall-cmd --get-active-zones     # 2. 올바른 zone?
firewall-cmd --list-all             # 3. 규칙 반영? ← 여기서 http가 빠져 있음을 발견
ss -tlnp | grep :80                 # 4. 프로세스 LISTEN?
```

(출력 기록 — `--list-all`에서 services에 http가 없음 발견 → 원인 특정)

```bash
# 복구
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

(출력 기록 — 복구 후 정상 접근 확인)

---

## 참고 (RHEL 공식 문서)

- [RHEL 10 - Configuring firewalls and packet filters](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/index)
- [RHEL 10 - Controlling network traffic using firewalld](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/controlling-network-traffic-using-firewalld)
- [RHEL 10 - Working with firewalld zones](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/working-with-firewalld-zones)
- [RHEL 10 - Getting started with SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/getting-started-with-selinux)
- [RHEL 10 - Changing SELinux states and modes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/changing-selinux-states-and-modes)
- [RHEL 10 - Troubleshooting problems related to SELinux](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/troubleshooting-problems-related-to-selinux)

