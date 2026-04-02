## 패키지 관리 개요

### RHEL의 패키지 관리 방식

RHEL 계열은 **RPM 기반 패키지 관리 방식**을 사용한다.
소프트웨어는 보통 `.rpm` 형식의 패키지로 관리되며
설치, 업데이트, 삭제는 `dnf` 같은 패키지 관리 도구를 통해 수행한다.

패키지 관리는 다음의 작업을 포함한다.
- 필요한 의존성 설치
- 버전 관리
- 업데이트 적용
- 삭제 및 검증
- 저장소(Repository)에서 패키지 검색

### `rpm`, `yum`, `dnf`
#### `rpm`

가장 기본적인 저수준 패키지 관리 도구다.  
개별 `.rpm` 파일을 직접 설치하거나, 이미 설치된 패키지 정보를 조회할 때 사용한다.
```bash
rpm -qa  
rpm -qi bash  
rpm -ql bash
```

#### `yum`

과거 RHEL/CentOS에서 널리 쓰이던 고수준 패키지 관리 도구다.  
의존성 해결과 저장소 기반 설치를 지원한다.

#### `dnf`

현재 RHEL에서 주로 사용하는 고수준 패키지 관리 도구다.  
`yum`의 후속 성격으로 이해하면 되고, 실무에서는 보통 `dnf`를 사용하면 된다.

#### `rpm` vs `dnf`
- `rpm` : 개별 패키지 정보를 직접 다루는 저수준 도구
- `dnf` : 의존성과 저장소까지 포함해 패키지를 관리하는 고수준 도구


### 패키지와 저장소(Repository)

Repository는 패키지를 보관하고 배포하는 저장소다.  
`dnf`는 이 저장소를 참고해 패키지를 검색하고 설치한다.

보통 패키지 관리 흐름은 다음과 같다.
1. 저장소에서 패키지 검색
2. 필요한 의존성 확인
3. 패키지 다운로드 및 설치
4. 설치 이력과 버전 관리


<br>

## 패키지 설치, 조회, 삭제
### 패키지 설치
```bash
sudo dnf install tree
sudo dnf install git vim curl
```

패키지를 설치할 때 필요한 의존성도 함께 설치된다.

### 패키지 조회
```bash
# 패키지 검색
dnf search nginx

# 설치 여부 확인
dnf list installed
dnf list installed git

# 패키지 상세 정보 확인
dnf info git
rpm -qi git

# 패키지가 설치한 파일 목록 확인
rpm -qi git
```

### 패키지 업데이트
```bash
# 특정 패키지 업데이트
sudo dnf update git

# 전체 패키지 업데이트
sudo dnf update
```

### 패키지 삭제
```bash
sudo dnf remove tree
```

삭제 시 불필요한 의존성도 함께 정리될 수 있다.


<br>

## 시스템 업데이트와 Repository 관리
#### 시스템 업데이트
RHEL에서는 보안 패치와 버그 수정 적용을 위해 정기적인 업데이트가 필요하다.
```bash
sudo dnf check-update  # 최신 상태 반영
sudo dnf update
```

#### Repository 확인
```bash
# 현재 활성화된 저장소 확
dnf repolist

# 모든 저장소 확인
dnf repolist all
```

Repository 설정 파일은 `/etc/yum.repos.d/` 경로에 있다.

RHEL에서 패키지 문제를 볼 때는 다음 세 가지를 함께 본다.
- 저장소가 활성화되어 있는지
- 패키지 이름이 정확한지
- 시스템이 최신 메타데이터를 가지고 있는지

<br>

## 사용자 및 그룹 관리
### 사용자와 그룹
리눅스는 **다중 사용자 시스템(Multi-user system)** 이기 때문에, 사용자와 그룹 개념이 중요하다.

**사용자 계정**
각 사용자는 고유한 계정을 가진다.  
로그인, 파일 소유, 권한 관리의 기준이 된다.

**그룹**
여러 사용자에게 공통 권한을 부여하기 위한 묶음

**Root 사용자**
시스템 전체를 제어할 수 있는 최고 권한 사용자다.  
모든 작업을 root로 직접 수행하기보다는 `sudo`를 통해 필요한 작업만 권한 상승하는 것이 일반적이다.

**UID, GID**
- UID: 사용자 식별 번호
- GID: 그룹 식별 번호
계정과 그룹은 이름뿐 아니라 위와 같은 숫자 ID로도 관리된다.

### 사용자 관리
```bash
# 사용자 생성
sudo useradd alice

# 비밀번호 설정
sudo passwd alice

# 사용자 정보 확인
id alice

# 사용자 수정
sudo usermod -aG wheel alice

# 사용자 삭제
sudo userdel alice

# 홈 디렉터리까지 함께 삭제
sudo userdel -r alice

```

### 그룹 관리
```bash
# 그룹 생성
sudo groupadd developers

# 그룹 정보 확인
getent group developers

# 사용자를 그룹에 추가
sudo usermod -aG developers alice

# 그룹 삭제
sudo groupdel developers
```

### 계정 정보 확인 파일
사용자와 그룹 정보는 아래 파일에서 확인할 수 있다.
- `/etc/passwd` : 사용자 계정 기본 정보
- `/etc/shadow` : 암호 정보
- `/etc/group` : 그룹 정보


<br>

## 파일 권한과 소유권 관리
### 권한과 소유권 
리눅스의 파일은 **소유자(owner)**, **그룹(group)**, **기타 사용자(others)** 기준으로 권한이 나뉜다.
파일을 권한은 `ls -l` 명령어로 확인할 수 있다.

### `chmod`
파일 권한 변경
```bash
chmod 644 note.txt  
chmod 755 script.sh
```

### `chown`
소유자 변경
```bash
sudo chown alice note.txt  
sudo chown alice:developers note.txt
```

### `chgrp`

```bash
sudo chgrp developers note.txt
```

### `sudo`와 관리자 권한

`sudo`는 일반 사용자가 일시적으로 관리자 권한으로 명령을 실행할 수 있게 해준다.

```bash
sudo dnf update  
sudo useradd bob
```

- `sudo` : 일시적 권한 상승
- `/etc/sudoers` : sudo 정책 설정 파일
- `wheel` 그룹 : 관리자 권한을 위임할 때 자주 사용하는 그룹

보통은 root로 직접 로그인해 계속 작업하기보다, 필요한 작업만 `sudo`를 통해 수행하는 것이 더 안전하다.
