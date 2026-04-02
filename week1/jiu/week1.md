## RHEL 개요
### RHEL이란?
**RHEL(Red Hat Enterprise Linux)** 은 Red Hat이 개발·배포하는 **상용 엔터프라이즈 리눅스 배포판**이다.  
개인용 데스크톱보다 **기업 환경에서의 안정성, 호환성, 장기 지원**에 초점을 맞추고 있으며, 서버 운영, 클라우드 인프라, 보안이 중요한 환경에서 널리 사용된다.

### 리눅스 배포판
리눅스 배포판은 패키지 관리 방식과 지향점에 따라 여러 계열로 나뉜다.  
대표적으로 **Debian 계열**과 **Red Hat 계열**이 널리 알려져 있다.

**Debian 계열**
- 패키지 관리 시스템: `apt`
- 커뮤니티 중심 개발
- 패키지 다양성과 접근성 중시
- 예시: Debian, Ubuntu, Linux Mint

**RedHat 계열**
- 패키지 관리 시스템: `rpm`, `yum`, `dnf`
- 기업 환경과 운영 안정성 중시
- 장기 지원 및 엔터프라이즈 생태계 강조
- 예시: RHEL, CentOS, Fedora

### 엔터프라이즈 리눅스 생태계

일반적인 커뮤니티 리눅스 배포판은 비교적 빠르게 변화하는 반면, 엔터프라이즈 리눅스는 **안정성, 호환성, 장기 지원**을 더 중요하게 본다.  
RHEL 생태계는 보통 아래 흐름으로 이해할 수 있다.

```
Fedora → CentOS Stream → RHEL → Rocky Linux
```

**Fedora**
- upstream
- 커뮤니티 중심의 빠른 배포판
- 새로운 기술과 기능이 먼저 통합된다

**CentOS**
- midstream
- 다음 RHEL 마이너 버전에 반영될 내용을 미리 보여주는 성격

**RHEL**
- 기업용으로 안정화된 엔터프라이즈 리눅스
- 검증, 지원, 보안 업데이트, 장기 운영을 중시

**Rocky Linux**
- downstream
- 커뮤니티 기반 엔터프라이즈 리눅스
- RHEL과 높은 호환성을 목표로 하는 downstream 배포판

<br>

## 설치 및 초기 설정
### 설치
[RedHat Developers](https://developers.redhat.com/) 가입 후 [공식 다운로드 페이지](https://developers.redhat.com/products/rhel/download#publicandprivatecloudreadyrhelimages) 에서 `rhel-10.0-aarch64-dvd.iso` 설치
UTM으로 가상머신 세팅, 한국어 설정, root 사용자 활성화

<br>

## 파일 시스템
RHEL도 Ubuntu와 마찬가지로 **표준적인 리눅스 디렉터리 구조(Filesystem Hierarchy Standard, FHS)** 를 따른다.  
따라서 큰 디렉터리 구조는 비슷하지만, 패키지 관리나 보안 도구 등에서 배포판 차이가 난다.

### 디렉토리 구조
- `/bin` : 기본 명령어
    - 모든 사용자가 사용할 수 있는 핵심 실행 파일
    - 예: `ls`, `cp`, `mv`, `rm`
- `/etc` : 설정 파일
    - 시스템 전역 설정 파일
    - 예: 네트워크 설정, 서비스 설정, 사용자 정책
- `/home` : 사용자 홈 디렉터리
    - 일반 사용자별 작업 공간
    - 예: `/home/username`
- `/var` : 로그, 캐시, 가변 데이터
    - 실행 중 계속 바뀌는 데이터
    - 예: 로그, spool, cache, PID 파일
- `/usr` : 사용자 공간 프로그램과 라이브러리
    - 대부분의 명령어, 라이브러리, 문서
- `/tmp` : 임시 파일
    - 프로그램 실행 중 잠시 쓰는 파일
    - 재부팅 시 정리될 수 있음
- `/boot` : 부팅 관련 파일
    - 커널, initramfs, bootloader 관련 파일
- `/dev` : 디바이스 파일
    - 디스크, 터미널, 랜덤 장치 등을 파일처럼 표현
- `/proc`, `/sys` : 커널/시스템 정보
    - 실제 파일이 아니라 커널이 제공하는 가상 파일 시스템
    - 시스템 상태 확인에 자주 사용

### 절대경로 vs 상대경로

**절대 경로**
루트(`/`)부터 시작하는 전체 경로
```bash
/etc/ssh/sshd_config
```

**상대 경로**
현재 위치를 기준으로 한 경로
```bash
../logs  
./script.sh
```

### 기본 탐색 명령어

```bash
pwd                       # 현재 위치 확인
ls                        # 목록 보기
ls -al                    # 숨김 파일 포함 상세 보기
cd /etc                   # 특정 디렉터리로 이동
cd ..                     # 상위 디렉터리로 이동
cd ~                      # 홈 디렉터리로 이동
tree .                    # 디렉터리 구조를 트리 형태로 표시 (설치 필요)
find /etc -name "*.conf"  # 파일 검색
which ls                  # 명령어 위치 확인
```

<br>

## 기본 명령어
기본적인 파일 탐색·복사·이동·삭제 명령어는 Ubuntu와 거의 동일하게 사용할 수 있다.

#### cheat sheet
```bash
ls -alh              # 상세 목록 보기
cd /path/to/dir      # 디렉터리 이동
pwd                  # 현재 위치 확인
cp src dst           # 파일 복사
cp -r src dst        # 디렉터리 복사
mv old new           # 이름 변경/이동
rm file              # 파일 삭제
rm -r dir            # 디렉터리 삭제
```

#### `ls` - 목록 보기
```bash
ls
```
**자주 쓰는 옵션**
- `-l` : 상세 정보 출력
- `-a` : 숨김 파일 포함
- `-h` : 사람이 읽기 쉬운 크기 단위

보통 `ll`을 `ls -alF` 또는 `ls -al` 형태의 alias로 등록해서 사용하는 경우가 많다.

#### `cd` - 디렉토리 이동
```bash
cd /    # 루트 디렉터리
cd ~    # 홈 디렉터리
cd      # 홈 디렉터리
cd ..   # 상위 디렉터리
cd -    # 이전 작업 디렉터리
```

#### `pwd` - 현재 경로 확인
현재 작업 중인 디렉터리의 절대 경로를 출력한다.

#### cp - 파일/디렉터리 복사
```bash
cp <src> <dst>
```
**자주 쓰는 옵션**:
- `-r` : 디렉터리 재귀 복사
- `-i` : 덮어쓰기 전 확인
- `-v` : 복사 과정 출력

#### `mv` - 이동 / 이름 변경
```bash
mv old new
```

#### `rm` - 삭제
```bash
rm filename
rm -rf dirname
```
**자주 쓰는 옵션**:
- `-r` : 디렉터리 재귀 삭제
- `-f` : 확인 없이 강제 삭제


<br>

## 도움말 활용하기
#### `man`
명령어의 공식 설명서를 확인할 수 있다.
```bash
man ls
```
**자주 쓰는 조작법**
- `/키워드` : 검색
- `n` : 다음 검색 결과
- `q` : 종료

#### `--help`
짧고 빠르게 옵션을 확인할 때 유용하다.
```bash
ls --help
```

#### `whatis`
명령어 한줄 설명을 볼 수 있다.
```bash
whatis ls
```

#### `apropos`
키워드와 관련된 명령어를 찾을 수 있다.
명령어 이름이 정확히 기억나지 않을 때 유용하다.
```bash
appropos copy
```
