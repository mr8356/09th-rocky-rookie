# 5주차 스터디 개요

> RHEL에서 네트워크 인터페이스와 IP 설정을 확인하고, 네트워크 연결 상태를 점검하는 방법 및 방화벽과 SELinux의 기본 보안 개념을 학습하고자 한다.

1~4주차에 걸쳐 RHEL의 기본 환경과 명령어, 패키지 및 사용자/권한 관리, 파일 시스템과 스토리지 관리, 서비스와 프로세스 제어 등을 학습했다. 이번에는 나아가 **네트워크 연결 상태를 확인하고 기본적인 네트워크 및 보안 설정 이해를 목표**로 한다.

리눅스 서버는 대부분 네트워크를 통해 다른 시스템과 통신하며, 실제 운영 환경에서는 `IP 주소, DNS, 게이트웨이, 네트워크 인터페이스 설정`이 매우 중요하다. 또한 외부 접근을 제어하는 `방화벽`과 시스템 보안 정책을 담당하는 `SELinux`를 함께 이해해야 한다.

---

## 1. 네트워크 기본 개념

리눅스에서 네트워크를 이해할 때 가장 기본이 되는 개념은 다음과 같다.

- `IP 주소`: 네트워크 상에서 장치를 식별하는 주소
- `게이트웨이(Gateway)`: 외부 네트워크로 나갈 때 거치는 기본 경로
- `DNS`: 도메인 이름을 IP 주소로 변환하는 시스템
- `네트워크 인터페이스`: 실제 네트워크 통신을 담당하는 장치 또는 가상 장치

즉, 서버가 정상적으로 네트워크에 연결되려면 적절한 IP 주소, DNS 설정, 게이트웨이 정보가 필요하다.

---

## 2. 네트워크 인터페이스란

네트워크 인터페이스는 **시스템이 네트워크와 연결되는 접점**이다.

RHEL에서는 보통 `ens160`, `eth0`과 같은 이름으로 확인할 수 있다.

네트워크 인터페이스를 통해 시스템은 IP를 할당받고, 패킷을 송수신하며, 외부와 통신할 수 있다.

---

## 3. `nmcli`와 `nmtui`

RHEL에서는 네트워크 관리를 위해 NetworkManager를 사용하며, 이를 제어하는 도구로 `nmcli`와 `nmtui`가 있다.

- `nmcli`: 명령줄 기반 네트워크 관리 도구
- `nmtui`: 텍스트 UI 기반 네트워크 관리 도구

---

## 4. 방화벽과 SELinux

### 방화벽 (firewalld)

방화벽을 시스템으로 들어오거나 나가는 네트워크 트래픽을 제어하는 기능이다. RHEL에서는 보통 `firewalld` 서비스를 통해 관리한다.

### SELinux

SELinux(Security-Enhanced Linux)는 리눅스 보안 모듈로, 파일/프로세스/서비스가 어떤 자원에 접근할 수 있는지 정책 기반으로 제어한다.

즉:

- 방화벽은 네트워크 접근 제어
- SELinux는 시스템 내부 자원 접근 제어

를 담당한다고 이해하면 된다.

## 실습

## 1. 현재 네트워크 상태 확인

먼저 현재 시스템이 어떤 네트워크 인터페이스를 가지고 있는지, IP 설정은 어떻게 되어 있는지 확인했다.

<img width="653" height="482" alt="image" src="https://github.com/user-attachments/assets/7d4af3e8-7b22-4395-a421-4807d18d3715" />


- `ip a` : 네트워크 인터페이스와 IP 주소 확인
- `ip route` : 라우팅 테이블 및 기본 게이트웨이 확인
- `nmcli device status` : 장치 상태 확인
- `nmcli connection show` : 저장된 네트워크 연결 정보 확인

이 과정으로 현재 시스템의 인터페이스, IP, 기본 경로 등을 확인했다.

## 2. DNS 설정 확인

DNS 정보는 외부 도메인 접근에 매우 중요하다.

현재 시스템에서 DNS 서버가 어떻게 설정되어 있는지 확인했다.

<img width="548" height="174" alt="image" src="https://github.com/user-attachments/assets/bc694296-246e-4ae6-a693-077e54559e09" />


- `cat /etc/resolv.conf` : 현재 사용 중인 DNS 설정 확인
- `nmcli dev show | grep DNS` : NetworkManager 관점의 DNS 정보 확인

도메인 이름 해석이 어떤 DNS 서버를 통해 이루어지는지 파악했다.

## 3. 네트워크 연결 점검 실습

### 3-1. ping

네트워크가 실제로 동작하는지 점검하기 위해 몇 가지 기본적인 테스트 명령어를 사용했다.

<img width="654" height="449" alt="image" src="https://github.com/user-attachments/assets/d5d1af5d-f376-4f1c-b346-c4010d782185" />


- `ping -c 4 8.8.8.8` : 외부 IP로의 통신 가능 여부 확인
- `ping -c 4 google.com` : DNS 해석 포함 외부 통신 확인

IP ping과 도메인 ping 모두 성공했는데, 만약에 하나가 성공하고 하나가 실패하면 DNS 문제 가능성이 있고, 둘 다 실패하면 네트워크 연결 자체가 문제일 가능성이 크다.

### 3-2. ss

<img width="640" height="237" alt="image" src="https://github.com/user-attachments/assets/ccbd501f-ea34-4895-a85b-811443bde499" />


- `ss -tuln` : 현재 열려 있는 TCP/UDP 포트 확인

옵션 의미:

- `t` : TCP
- `u` : UDP
- `l` : listening 상태
- `n` : 숫자 형태로 출력

이를 통해 현재 시스템에서 어떤 포트가 대기 중인지 확인할 수 있다.

### 3-3. hostname과 호스트 정보 확인

<img width="290" height="88" alt="image" src="https://github.com/user-attachments/assets/4fa0fff6-8024-4500-87cb-107b6e793fd8" />


- `hostname` : 현재 시스템 호스트 이름 확인
- `hostname -I` : 현재 시스템 IP 주소 확인

## 4. nmcli 기반 네트워크 정보 확인

`nmcli` 를 사용하면 현재 활성화된 연결과 각 인터페이스의 세부 설정을 확인할 수 있다.

<p align="center">
<img width="45%" height="400" alt="image" src="https://github.com/user-attachments/assets/fd146cea-d078-4ba9-9704-0697040b9d2f" />
<img width="45%" height="340" alt="image" src="https://github.com/user-attachments/assets/f8a32fcf-6792-43f9-9b58-012ce073bc4f" />
</p>

- `nmcli connection show` : 전체 연결 목록 확인
- `nmcli connection show "ens160"` : 특정 연결 상태 확인
- `nmcli device show` : 각 장치별 IP, 게이트웨이, DNS 정보 확인

실제 연결 이름은 환경에 따라 `ens160` , `System etho0` , `Wired connection 1` 등 다를 수 있다.

## 5. nmtui 실습

텍스트 UI 기반으로 네트워크 설정을 확인하기 위해 `nmtui` 를 실행했다.

<p align="center">
<img width="45%" height="450" alt="image" src="https://github.com/user-attachments/assets/3f33a1bd-c849-403f-9716-3aad6f8935d7" />
<img width="45%" height="489" alt="image" src="https://github.com/user-attachments/assets/8c6a9a60-81fd-4f12-b755-b30ab1953a13" />
</p>


특별히 설정을 바꾸진 않았고, 현재 인터페이스와 IP/DNS 항목이 어떤 식으로 구성되어 있는지 정도로만 확인했다.

`nmtui` 에서는 다음의 작업이 가능하다.

- 연결 수정
- 인터페이스 활성화/비활성화
- 호스트 이름 설정

## 6. 네트워크 재시작 및 인터페이스 제어

특정 인터페이스를 비활성화하거나 다시 활성화할 때는 `nmcli` 를 사용할 수 있다.

예를 들어 인터페이스 이름이 `ens160` 이라면:

<img width="709" height="113" alt="image" src="https://github.com/user-attachments/assets/3bbc9f9e-a7b1-49fb-b257-9e2dd38717e5" />


- `disconnect` : 인터페이스 활성화
- `connect` : 비활성화

또는 연결 이름 기준으로도 재활성화가 가능하다.

<img width="656" height="130" alt="image" src="https://github.com/user-attachments/assets/1023a570-caa3-4404-913d-1846d4ac539a" />


- `connection down "이름 지정"`
- `connection up "이름 지정"`

## 7. 방화벽 실습

RHEL에서는 `firewalld`  서비스를 사용해 방화벽을 관리한다.

### 7-1. 방화벽 상태 확인

<img width="650" height="352" alt="image" src="https://github.com/user-attachments/assets/458d9a64-f121-4bab-94b2-848911e47ebf" />

- `systemctl status firewalld` : 방화벽 서비스 상태 확인
- `firewall-cmd --state` : 실행 중 여부 확인

### 7-2. 현재 허용 서비스 확인

<img width="421" height="342" alt="image" src="https://github.com/user-attachments/assets/e16db914-910f-4134-8e79-903757847313" />


- `sudo firewall-cmd --list-all`

기본 zone과 허용된 서비스 / 포트를 확인할 수 있다.

### 7-3. 서비스 허용 실습

예를 들어 HTTP 서비스를 허용할 수 있다.

<img width="492" height="405" alt="image" src="https://github.com/user-attachments/assets/bd1e3037-bb46-4e60-8d12-e92e6f09ba04" />


- 일시적 적용
    - `sudo firewall-cmd --add-service=http` : http 서비스에 일시적 적용
    - `sudo firewall-cmd --list-all` : 적용됐는지 확인

<img width="580" height="430" alt="image" src="https://github.com/user-attachments/assets/3c526608-dc0c-4bdf-83de-d1de694f3ee5" />


- 영구 반영 시에는
    - `sudo firewall-cmd --permanaet --add-service=http` : 영구 설정
    - `sudo firewall-cmd --reload` : 영구 설정 반영
    - `sudo firewall-cmd --list-all`


## 8. SELinux 실습

SELinux는 현재 시스템 보안 정책 상태를 확인하는 정도로 실습을 진행했다.

<img width="378" height="273" alt="image" src="https://github.com/user-attachments/assets/f0f345be-459a-41e5-9131-7732bfe4a825" />


- `getenforce` : 현재 SELinux 모드 확인 (`Enforcing` , `Permissive` , `Disabled` )
- `sestatus` : SELinux 전체 상태 상세 확인

**SELinux 모드**

- `Enforcing`: 정책을 실제로 강제 적용
- `Permissive`: 정책 위반을 기록만 하고 차단하지 않음
- `Disabled`: SELinux 비활성화

즉, SELinux는 단순 on/off가 아니라 정책 강제 수준을 조절하는 보안 기능이라고 이해할 수 있다.

## 9. 정리

이번 5주차 실습에서는 아래의 내용들을 익혔다.

- 네트워크 인터페이스, IP, 게이트웨이, DNS의 기본 개념
- `ip` , `nmcli` , `nmtui` 를 통한 네트워크 설정 확인 방법
- ping, ss를 통한 네트워크 상태 점검 방법
- `firewalld` 를 통한 기본 방화벽 확인 및 서비스 허용 방법
- `SELinux` 의 기본 개념과 현재 보안 정책 상태 확인 방법

특히 이번 실습으로 서버의 네트워크는 단순히 IP 설정으로 끝나는 것이 아니라, **인터페이스 상태, 라우팅, DNS, 포트, 방화벽, 보안 정책**까지 함께 확인해야 정상적인 통신 환경을 이해할 수 있다는 점을 확인할 수 있었다.
