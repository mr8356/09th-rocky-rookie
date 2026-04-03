## 4주차 스터디 개요

> RHEL에서 서비스와 프로세스를 관리하는 방법을 이해하고, `systemd` 기반의 서비스 제어와 프로세스 조회/종료, 작업 스케줄링의 기초를 학습하고자 한다.

1주차에선 RHEL 기본 환경과 명령어, 2주차에선 패키지 및 사용자/권한 관리, 3주차에선 파일 시스템과 스토리지 관리를 학습했다. 이번 주차에서는 한단계 더 나아가 `실제로 시스템에서 동작 중인 서비스와 프로세스를 확인하고 제어하는 방법`을 익히고자 한다.

리눅스에서는 다양한 백그라운드 서비스가 실행되고, 사용자가 실행한 프로그램 또한 프로세스 단위로 관리된다.

따라서, 서비스의 상태를 확인하고 시작/중지/재시작하는 방법, 현재 동작 중인 프로세스를 조회하고 제어하는 방법, 그리고 정해진 시간에 작업을 자동으로 실행하는 방법을 아는 것이 시스템 운영의 중요한 기초로 연결될 수 있다.

## 1. systemd란?

`systemd` 는 현대 리눅스 배포판에서 사용하는 시스템 및 서비스 관리자이다.

시스템 부팅 과정에서 필요한 서비스들을 시작하고, 실행 중인 서비스를 관리하며, 서비스 상태를 확인하거나 자동 시작 여부를 제어하는 역할을 한다.

RHEL에서는 대부분의 서비스 관리가 `systemd`  기반으로 이루어지며, 이를 제어할 때 주로 `systemctl`  명령어를 사용한다.

즉, `systemd` 는 시스템 전체를 관리하는 기반 구조이고, `systemctl` 은 이를 제어하는 명령어라고 이해할 수 있다.

---

## 2. 서비스란?

서비스는 사용자의 직접적인 입력 없이도 백그라운드에서 계속 동작하며 시스템 기능을 제공하는 프로그램이다.

예를 들어:

- `sshd` : 원격 접속 서비스
- `crond` : 작업 스케줄링 서비스
- `firewalld` : 방화벽 서비스

이러한 서비스는 시스템 부팅 시 자동으로 시작될 수도 있고, 필요에 따라 수동으로 `시작/중지/재시작`할 수도 있다.

---

## 3. 프로세스란?

프로세스(process)는 실행 중인 프로그램의 인스턴스를 의미힌다.

즉, 명령어를 실행하거나, 서비스가 동작하면 시스템 안에서는 프로세스 단위로 실행된다.

- 터미널에서 `sleep 1000` 을 실행하면 하나의 프로세스가 생성되고
- `sshd` , `crond` 같은 서비스도 내부적으로는 프로세스로 동작한다.

프로세스는 `PID(Process ID)`라는 고유 번호를 가지며, 이를 통해 조회하거나 종료할 수 있다.

---

## 4. 작업 스케줄링이란?

작업 스케줄링은 특정 명령어를 정해진 시간에 자동으로 실행하도록 예약하는 기능이다.

- `cron` : 반복 작업 예약
- `at` : 1회성 작업 예약

등을 사용한다.

필자의 경우, ETL 파이프라인을 구축할 당시, AWS EC2 내부에서 `cron job` 을 통해 데이터 분석 처리 스케줄링을 고려한 적이 있었다.

즉, 사용자가 직접 매번 명령어를 입력하지 않아도 시스템이 자동으로 작업을 실행하도록 설정할 수 있다.

---

## 1. 서비스 관리 실습

### 1-1. systemctl 기본 확인

먼저 `systemctl` 을 통해 현재 시스템에서 서비스 관리가 어떻게 이루어지는지 확인했다.

<img width="655" height="472" alt="image" src="https://github.com/user-attachments/assets/ab79c814-12e4-4817-aeae-d01f28c5e70d" />


- `systemctl list-units --type=service | head -20` : 현재 로드된 서비스 유닛 목록을 앞부분 20줄만 확인

다양한 서비스 리스트와 각 서비스마다의 `loaded` , `active` , `running`  등의 상태가 표시되는 걸 확인했다.

---

### 1-2. 특정 서비스 상태 확인

실습에서는 비교적 익숙한 `sshd`  또는 `crond`  같은 서비스를 대상으로 상태를 확인했다.

<p align="center">
<img width="45%" height="235" alt="image" src="https://github.com/user-attachments/assets/369fd646-bee4-40ce-ad35-9673ac384168" />
<img width="45%" height="342" alt="image" src="https://github.com/user-attachments/assets/81569bf6-67d2-4f55-92d4-1e968c462969" />
</p>


- `systemctl status 서비스명` : 해당 서비스의 현재 상태 확인

이 명령어를 통해 서비스가 현재 실행 중인지, 최근 로그는 어떤지, 자동 시작 설정이 되어 있는지 등을 확인했다.

### 1-3. 서비스 시작, 중지, 재시작

서비스는 필요에 따라 수동으로 제어할 수도 있다.

- `sudo systemctl start crond` : 서비스 시작
- `sudo systemctl stop crond` : 서비스 중지
- `sudo systemctl restart crond` : 서비스 재시작

<img width="649" height="73" alt="image" src="https://github.com/user-attachments/assets/0c1f89db-ee6a-429b-9938-fe4e2190524b" />



중지 후 다시 상태를 확인하여, 서비스 상태가 바뀌는 것을 확인

### 1-4. 서비스 자동 시작 설정

시스템 부팅 시 특정 서비스를 자동으로 시작하도록 설정

- `sudo systemctl enable crond` : 부팅 시 자동 시작 설정
- `sudo systemctl disable crond` : 부팅 시 자동 시작 해제

이 과정으로 서비스의 현재 실행 상태와, 부팅 시 자동 실행 여부는 별개의 개념이라는 것을 알 수 있고, 아래 명령을 실행해서 자동 시작 여부를 확인할 수 있다.

<img width="423" height="45" alt="image" src="https://github.com/user-attachments/assets/ab6156ac-c140-4a6a-a472-ab924426f783" />


- `systemctl is-enabled crond`

## 2. 프로세스 관리 실습

### 2-1. 현재 실행 중인 프로세스 확인

리눅스에서는 `ps` , `top` 같은 명령어를 통해 현재 실행 중인 프로세스를 확인한다.

- `ps -ef` : 전체 프로세스를 포맷 형태로 출력
- `ps aux` : CPU, 메모리 사용량 등을 포함해 상세 출력

<p align="center">
<img width="45%" height="291" alt="image" src="https://github.com/user-attachments/assets/b632c66f-b850-431f-8e52-e9782f5af5d3" />
<img width="45%" height="233" alt="image" src="https://github.com/user-attachments/assets/66fcbe10-c3d9-40ba-a0bb-cf55c3be4ce8" />
</p>


이 명령을 통해 각 프로세스의 사용자, PID, CPU 사용량, 실행 명령 등을 확인할 수 있다.


### 2-2. 실시간 프로세스 조회

현재 실행 중인 프로세스를 실시간으로 보고 싶을 땐 `top` 을 사용할 수 있다.

<img width="642" height="187" alt="image" src="https://github.com/user-attachments/assets/7dc0ee0f-9210-4007-94e9-f438c129f552" />



- `top` : CPU, 메모리, 실행 시간 등을 기준으로 ‘실시간’ 프로세스 상태 표시
- 종료는 q 눌러서 나가면 된다.

`top` 은 현재 시스템 자원 사용 상황을 빠르게 확인할 수 있어서 서버 운영 시에 자주 사용된다.

### 2-3. 테스트 프로세스 생성

프로세스 종료 실습을 위해 오래 실행되는 테스트 프로세스를 하나 생성했다.

터미널이 점유될 것을 감안하여, 백그라운드로 실행했다.

<img width="529" height="89" alt="image" src="https://github.com/user-attachments/assets/9b399fb0-d4fe-49aa-9a19-68dee37f55dc" />



- `sleep 1000` : 1000초 동안 대기하는 프로세스를 백그라운드로 실행
- `ps -ef | grep sleep` : pid 확인

### 2-4. 프로세스 종료

실행 중인  프로세스는 `kill` 을 통해 종료한다.

<img width="324" height="52" alt="image" src="https://github.com/user-attachments/assets/ee19dcb5-bf48-4992-a410-7ba291ff94a7" />


본인은 방금 실행한 sleep 프로세스를 종료시켰다.


- `kill pid`

정상 종료가 되지 않는 경우 강제 종료도 가능함.

- `kill -9 PID` : 강제 종료

### 2-5. 프로세스 우선순위 개념

리눅스에서는 프로세스마다 CPU 스케줄링 우선순위를 조절할 수 있으며, `nice` 와 `renice` 명령어로 관리한다.

### nice 값과 함께 실행

<img width="396" height="55" alt="image" src="https://github.com/user-attachments/assets/2bca2342-8464-4aaf-b380-5a39cf6d2694" />


- `nice -n 10 sleep 1000 &` : 우선순위를 낮춰서 실행

### 실행 중인 프로세스 우선순위 조정

<img width="423" height="68" alt="image" src="https://github.com/user-attachments/assets/967cd64d-f561-4be1-b34a-3e8ed736b282" />


- `renice 5 -p PID` : 실행 중인 프로세스의 nice 값 변경

`nice`  값이 클수록 우선순위가 낮아진다.

## 3. 작업 스케줄링 실습

### 3-1. cron 서비스 확인

반복 작업 예약을 위해서는 `crond`  서비스가 실행 중이어야 한다.

running 여부 확인 (`systemctl status crond` )

<img width="554" height="85" alt="image" src="https://github.com/user-attachments/assets/4a2af3f9-6d1f-4ff4-86d7-9e4ca5d9239b" />



### 3-2. crontab 등록

사용자 단위 반복 작업은 `crontab` 으로 관리할 수 있다.

<img width="413" height="195" alt="image" src="https://github.com/user-attachments/assets/30c849fa-5845-48aa-b53e-3ef1c8923b5e" />


테스트 작업 등록을 해보았고,

`*`의 순서 별로 범위를 지정할 수 있다.

- 첫 번째 : 매분
- 두 번째: 매시간
- 세 번째: 매일
- 네 번째: 매월
- 다섯 번째: 매요일
- `date >> ...` : 현재 시간 문자열을 파일 끝에 계속 추가

<img width="480" height="144" alt="image" src="https://github.com/user-attachments/assets/1d48425d-7da0-433f-85de-c026dc7bed7a" />


지정한대로 매분마다 잘 실행되고 있음을 확인했다.

### 3-3. at 명령 실습

`at` 은 한 번만 실행할 작업을 예약할 때 사용한다.

먼저 `atd` 서비스 상태를 확인 (running 여부)

<img width="393" height="111" alt="image" src="https://github.com/user-attachments/assets/291c29db-7539-4286-bc20-f797c0ba09e4" />


만약에 여기서 꺼져 있으면, `sudo systemctl start atd` 로 키면 된다.

<img width="419" height="93" alt="image" src="https://github.com/user-attachments/assets/0edea588-f898-4280-b936-5fcb4f6a52a6" />


`Ctrl + D` 로 프롬프트를 닫고 `atq` 를 통해 예약 목록을 확인한다. 여기서 `ls -l /home/minsubyun/at_test.txt` 로 파일이 생겼는지 확인을 하는데, `atq` 에서 보이는(내가 지정한) 시간대가 지나야 생성이 된다.

즉, 지정한 대로 잘 된 것이다.

<img width="657" height="206" alt="image" src="https://github.com/user-attachments/assets/97afecef-8497-4a97-b43e-f3e60b6dc1c8" />


## 4. 컨트롤 그룹(cgroup)

### 4-1. cgroup이란?

cgroup(control group)은 리눅스 커널이 제공하는 기능으로, 실행 중인 프로세스들을 **그룹 단위로 묶어 CPU, 메모리, I/O, 네트워크 대역폭 같은 시스템 자원을 할당하고 제한할 수 있도록 해주는 기능**이다.

즉, 개별 프로세스를 하나씩 관리하는 것이 아니라, 여러 프로세스를 하나의 작업 그룹으로 묶어서 자원을 관리할 수 있다는 점이 핵심이다.

예를 들어 cgroup을 사용하면 다음과 같은 제어가 가능하다.

- 특정 그룹의 CPU 사용량 제한
- 특정 그룹이 사용할 수 있는 메모리 크기 제한
- 특정 그룹의 디스크 I/O 우선순위 조정
- 특정 장치에 대한 접근 허용 또는 거부
- 특정 그룹의 프로세스를 일시 정지하거나 다시 시작

이처럼 cgroup은 시스템 관리자가 **자원 할당, 제한, 우선순위 지정, 모니터링**을 보다 세밀하게 수행할 수 있도록 도와준다.

---

### 4-2. cgroup이 필요한 이유

리눅스 시스템에서는 여러 서비스와 프로세스가 동시에 실행된다.

이때 어떤 프로세스가 CPU나 메모리를 과도하게 점유하면 다른 작업의 성능에 영향을 줄 수 있다.

cgroup은 이러한 문제를 해결하기 위해 프로세스를 그룹으로 나누고, 각 그룹마다 사용할 수 있는 자원을 제어할 수 있게 한다.

즉, 시스템 전체 자원을 무조건 동일하게 쓰게 하는 것이 아니라, **업무 목적이나 중요도에 따라 그룹별로 자원을 분배**할 수 있게 해준다.

예를 들면,

- 웹 서비스 그룹에는 CPU를 더 많이 할당
- 백그라운드 배치 작업은 CPU 우선순위를 낮게 설정
- 특정 그룹의 메모리 사용량을 제한하여 시스템 전체 장애를 방지

와 같은 운영이 가능하다.

따라서 cgroup은 단순한 성능 관리뿐 아니라, **서비스 안정성 확보와 자원 격리** 측면에서도 중요한 기능이다.

---

### 4-3. cgroup의 구조

cgroup은 계층적 구조를 가진다.

즉, 상위 cgroup 아래에 하위 cgroup을 만들 수 있고, 하위 cgroup은 부모 cgroup의 일부 속성을 상속받을 수 있다.

이 점은 일반적인 리눅스 프로세스 구조와 비슷해 보인다.

리눅스 프로세스 역시 부모-자식 관계를 가지는 단일 트리 구조를 이루기 때문이다.

하지만 둘 사이에는 중요한 차이가 있다.

### 리눅스 프로세스 모델

리눅스의 프로세스는 하나의 부모 프로세스를 가지며, 전체 시스템은 하나의 프로세스 트리로 구성된다.

즉, 프로세스 구조는 **단일 계층 구조**이다.

### cgroup 모델

반면 cgroup은 하나의 트리만 존재하는 것이 아니라, **자원 종류별로 서로 다른 계층 구조가 동시에 존재할 수 있다.**

즉:

- 프로세스 모델은 하나의 트리
- cgroup 모델은 여러 개의 독립적인 트리 가능

이라는 차이가 있다.

이렇게 여러 계층이 존재하는 이유는, cgroup이 CPU, 메모리, 디스크 I/O처럼 **서로 다른 자원(subsystem)** 을 각각 관리하기 때문이다.

---

## 4-4. cgroup과 subsystem

cgroup은 자원을 제어하기 위해 **subsystem(서브시스템)** 과 연결된다.

여기서 subsystem은 CPU, 메모리, 디스크 I/O처럼 **제어 대상이 되는 개별 자원 종류**를 의미한다.

즉, cgroup 자체가 자원을 직접 의미하는 것이 아니라,

각 자원별 subsystem과 연결되어 해당 자원 사용을 제어하는 구조라고 이해하면 된다.

Red Hat 문서 기준으로 소개되는 대표적인 subsystem은 다음과 같다.

- **blkio** : 디스크, SSD, USB 같은 블록 장치의 I/O 접근 제어
- **cpu** : CPU 사용 시간 제어
- **cpuacct** : CPU 사용량 측정 및 보고
- **cpuset** : 특정 CPU 코어와 메모리 노드 할당
- **devices** : 장치 접근 허용/거부
- **freezer** : 작업 일시 중지 및 재개
- **memory** : 메모리 사용량 제한 및 보고
- **net_cls** : 네트워크 패킷에 식별자 부여
- **ns** : namespace 관련 subsystem

즉, cgroup은 단순히 “프로세스 그룹”이라기보다,

**각 자원 subsystem과 연결되어 해당 그룹의 자원 사용을 제어하는 관리 구조**라고 볼 수 있다.

---

### 4-5. cgroup으로 가능한 작업

cgroup을 사용하면 시스템 관리자는 프로세스 그룹 단위로 다음과 같은 작업을 수행할 수 있다.

- 자원 사용량 제한
- 자원 우선순위 조정
- 특정 자원 접근 거부
- 자원 사용량 모니터링
- 실행 중인 시스템에서 동적 재구성

즉, 단순히 “얼마까지 쓸 수 있는지 제한”하는 것뿐만 아니라,

**현재 동작 중인 시스템에서도 설정을 다시 조정하고 모니터링할 수 있다**는 점이 중요하다.

또한 설정한 cgroup은 서비스 설정을 통해 재부팅 후에도 유지되도록 구성할 수 있다.

---

### 4-6. cgroup과 서비스/프로세스 관리의 연결점

4주차 주제인 서비스와 프로세스 관리 관점에서 보면, cgroup은 단순히 이론적인 기능이 아니라 **실제 시스템 자원 제어의 기반 개념**이다.

예를 들어 서비스는 내부적으로 하나 이상의 프로세스로 실행되는데,

이 프로세스들을 cgroup 단위로 묶으면 서비스 전체에 대해 CPU나 메모리 자원 정책을 적용할 수 있다.

즉,

- `systemctl`은 서비스의 시작/중지/재시작 같은 **동작 관리**
- `cgroup`은 그 서비스가 사용하는 **자원 관리**

라는 관점으로 연결해서 이해할 수 있다.

따라서 cgroup은 이번 주차에서 깊게 실습하지 않더라도,

**서비스와 프로세스를 단순히 실행 여부만이 아니라 자원 사용 관점에서도 관리할 수 있게 해주는 커널 기반 기능**이라는 점에서 중요하다고 느꼈다.

[참고 자료 - Red Hat Docs 1장 컨트롤 그룹](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/6/html/resource_management_guide/ch01)

## 5. 정리

이번 4주차 실습을 통해 아래 내용을 익혔다.

- `systemd`와 `systemctl`을 통한 서비스 관리 방법
- 서비스의 상태 확인, 시작, 중지, 재시작, 자동 시작 설정 방법
- `ps`, `top`, `kill`, `nice`, `renice`를 통한 프로세스 조회 및 제어 방법
- `cron`, `at`을 활용한 작업 스케줄링 기초
- `cgroup`이 시스템 자원 제어의 기반 개념이라는 점

특히 이번 실습을 통해 리눅스 시스템에서는 서비스와 프로세스가 서로 밀접하게 연결되어 있으며,

운영자는 단순히 프로그램을 실행하는 것을 넘어 **시스템에서 어떤 서비스가 동작 중인지, 어떤 프로세스가 자원을 사용하고 있는지, 어떤 작업이 자동으로 예약되어 있는지까지 관리해야 한다**는 점을 확인할 수 있었다.
