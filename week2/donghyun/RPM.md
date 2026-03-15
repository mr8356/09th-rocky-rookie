> **Reference:** [Packaging and distributing software (RHEL 10)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/packaging_and_distributing_software/index)

---

## 1. GPG(GNU Privacy Guard)란 무엇인가?

**GPG**는 데이터 암호화 및 디지털 서명을 위한 오픈소스 도구로, **OpenPGP** 표준을 구현한 것입니다. 리눅스 환경에서 "이 소프트웨어가 믿을 만한가?"를 판단할 때 사용하는 표준 기술입니다.

핵심은 **비대칭 키 암호화(Asymmetric Cryptography)** 방식에 있습니다.

- **Private Key (비밀키)**: 나만 가지고 있는 키. 패키지에 "서명(Sign)"할 때 사용합니다.
- **Public Key (공개키)**: 누구나 가질 수 있는 키. 패키지의 "서명"이 진짜인지 "검증(Verify)"할 때 사용합니다.

---

## 2. RPM에서 GPG가 작동하는 원리 (Signature & Verification)

RPM 패키지 구조에서 **GPG Signature**는 일종의 **디지털 봉인**입니다. SRE 관점에서 아래 과정을 이해하는 게 중요합니다.

### Step 1: Signing (배포자 측)

1. 패키지 제작자(예: Red Hat 또는 로컬 레포지토리 제작자)가 RPM 파일을 만듭니다.
2. RPM 파일의 내용을 **Hashing**하여 고유한 지문(Digest)을 추출합니다.
3. 제작자의 **Private Key**를 사용해 이 지문에 서명을 합니다.
4. 이 서명 데이터를 RPM 파일의 **Header** 섹션에 포함시킵니다.

### Step 2: Verification (사용자/SRE 측)

1. 서버에서 `dnf install`로 패키지를 내려받습니다.
2. 시스템은 배포자의 **Public Key**를 이미 가지고 있거나 가져옵니다.
3. RPM 파일의 서명을 **Public Key**로 복호화하여 제작자가 보낸 지문을 확인합니다.
4. 동시에 현재 받은 파일의 지문을 새로 계산하여 제작자의 지문과 일치하는지 비교합니다.
    - **일치하면**: "누군가 중간에 파일을 조작하지 않았음(**Integrity**)" + "진짜 제작자가 보낸 게 맞음(**Authenticity**)"으로 간주하고 설치를 진행합니다.
    - **불일치하면**: 설치를 거부합니다.

```bash
Total                                                                          887 kB/s | 6.0 MB     00:06
Rocky Linux 10 - BaseOS                                                        1.6 MB/s | 1.6 kB     00:00
Importing GPG key 0x6FEDFC85:
 Userid     : "Release Engineering (Rocky Linux 10) <releng@rockylinux.org>"
 Fingerprint: FC22 6859 C086 0BF0 DDB9 5B08 5B10 6C73 6FED FC85
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
Is this ok [y/N]: y
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
```

- **상황**: 록키 리눅스 10을 설치하고 처음으로 공식 **Repository**에서 패키지를 받을 때 나타납니다. 시스템이 "이 패키지들을 만든 록키 빌드팀의 **Public Key**를 장부에 등록합니다.
- **Userid**: 이 키의 주인(록키 리눅스 릴리스 엔지니어링 팀)입니다.
- **Fingerprint**: 키의 고유 식별 번호. 나중에 보안 사고가 의심될 때 공식 사이트에 공지된 번호와 대조해보는 용도입니다.
- **From**: 시스템에 이미 저장되어 있던 록키 리눅스의 **Public Key** 파일 경로

---

## 3. 왜 GPG를 알아야 하나요? (SRE 실무 포인트)

- **Supply Chain Security**: 최근 오픈소스 패키지 오염 사고가 많습니다. GPG 서명이 없는 패키지를 서버에 설치하는 건 출처 모를 음식을 그냥 먹는 것과 같습니다.
- **Custom Repository 운영**: 나중에 회사에서 사내 전용 **DNF Repository**를 운영하게 되면, 직접 만든 패키지에 **Private Key**로 서명을 하고 서버들에 **Public Key**를 배포해야 합니다. 이 메커니즘을 모르면 배포 자동화가 불가능합니다.

---

## 4. GPG 키 확인하기

터미널에서 아래 명령어를 실행해 보시기 바랍니다. 록키 리눅스가 신뢰하는 **Public Keys** 목록이 표시됩니다.

`Rocky Linux 10 (official key)`와 같은 항목이 보일 수 있습니다. 해당 키가 등록되어 있어야 `dnf install` 시 에러 없이 설치가 가능합니다.

```bash
mr8356@mr8356:~$ sudo rpm -q gpg-pubkey --qf '%{name}-%{version}-%{release} --> %{summary}\n'
gpg-pubkey-6fedfc85-682ae1a9 --> Release Engineering (Rocky Linux 10) <releng@rockylinux.org> public key
```

---

## Chapter 1. Introduction to RPM

- RPM Package Manager (RPM)는 RHEL 계열의 핵심 패키지 관리 시스템입니다. 일반 압축 파일과 달리 시스템 유지보수를 획기적으로 도와줍니다.
- **Advantages**: 패키지 간 독립적 설치/업데이트/삭제 가능, 특정 OS 및 **Architecture**에 최적화된 **Binary** 배포.
- **Main Tasks**: 설치, 업그레이드, 검증, **GPG signature**를 이용한 보안 서명, **DNF repository** 발행.

### 1.1. RPM packages 구조

1. **GPG signature**: 패키지 무결성 검증.
2. **Header (Metadata)**: 의존성 정보, 설치 경로 등 포함.
3. **Payload**: 실제 설치될 파일들이 담긴 **cpio archive**.
- **SRPM (Source RPM)**: 소스 코드와 빌드 지시서인 **spec file** 포함.
- **Binary RPM**: 빌드가 완료된 실행 파일 포함.

---

## Chapter 2. Setting up RPM packaging workspace

빌드를 시작하기 전에 표준 작업 공간(Workspace)을 만들어야 합니다.

- **Prerequisites**: `# dnf install rpmdevtools`
- **Procedure**: `$ rpmdev-setuptree`

### 폴더 구조

명령어를 실행하면 `~/rpmbuild/` 하위에 아래 구조가 생깁니다.

```bash
mr8356@mr8356:~$ tree ~/rpmbuild/
/home/mr8356/rpmbuild/
├── BUILD
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS

6 directories, 0 files
```

- **SOURCES**: 소스 코드 압축 파일(`.tar.gz`)과 **Patch** 파일 보관.
- **SPECS**: 빌드 설계도인 **spec file** 보관. (**SRE의 핵심 작업 공간**)
- **RPMS**: 빌드 완료된 **Binary RPM**이 저장되는 곳.
- **BUILD**: 실제 빌드가 진행되는 임시 공간.

---

## Chapter 5. Preparing software for RPM packaging

RPM 패키징을 위해 소프트웨어를 **Patch**하고, **LICENSE** 파일을 생성하며, **Tarball**로 아카이브하는 단계입니다.

### 5.1. Patching software

원본 소스(Upstream)는 그대로 두고, 수정 사항만 **Patch file**로 관리합니다.

### 🛠️ [실습] C 프로그램 소스 및 패치 생성

```bash
mr8356@mr8356:~/rpmtest$ cat cello.c
#include <stdio.h>
int main(void) {
    printf("Hello World\n");
    return 0;
}

mr8356@mr8356:~/rpmtest$ cat Makefile
cello:
	gcc -g -o cello cello.c
install:
	mkdir -p $(DESTDIR)/usr/bin
	cp cello $(DESTDIR)/usr/bin/
clean:
	rm -f cello
	
mr8356@mr8356:~/rpmtest$ cp -p cello.c cello.c.orig

mr8356@mr8356:~/rpmtest$ sed -i 's/Hello World/Hello World from my first patch!/g' cello.c

mr8356@mr8356:~/rpmtest$ cat cello.c
#include <stdio.h>
int main(void) {
    printf("Hello World from my first patch!\n");
    return 0;
}

mr8356@mr8356:~/rpmtest$ diff -Naur cello.c.orig cello.c > cello.patch

mr8356@mr8356:~/rpmtest$ mv cello.c.orig cello.c

mr8356@mr8356:~/rpmtest$ cat cello.patch
--- cello.c.orig	2026-03-15 12:37:22.570443747 +0900
+++ cello.c	2026-03-15 12:38:18.070230967 +0900
@@ -1,5 +1,5 @@
 #include <stdio.h>
 int main(void) {
-    printf("Hello World\n");
+    printf("Hello World from my first patch!\n");
     return 0;
 }
```

### 5.3. Creating a source code archive

재료를 묶어 `~/rpmbuild/SOURCES/`로 이동시킵니다. (Chapter 5.3.3)

```bash
mr8356@mr8356:~/rpmtest$ mkdir cello-1.0

mr8356@mr8356:~/rpmtest$ cp cello.c Makefile cello.patch cello-1.0/

mr8356@mr8356:~/rpmtest$ echo "GPLv3+ License" > cello-1.0/LICENSE

mr8356@mr8356:~/rpmtest$ tar -cvzf cello-1.0.tar.gz cello-1.0
cello-1.0/
cello-1.0/cello.c
cello-1.0/Makefile
cello-1.0/cello.patch
cello-1.0/LICENSE

mr8356@mr8356:~/rpmtest$ tree cello-1.0
cello-1.0
├── cello.c
├── cello.patch
├── LICENSE
└── Makefile

1 directory, 4 files

mr8356@mr8356:~/rpmtest$ ll
total 16
drwxr-xr-x. 2 mr8356 mr8356  71 Mar 15 12:40 cello-1.0
-rw-r--r--. 1 mr8356 mr8356 508 Mar 15 12:40 cello-1.0.tar.gz
-rw-r--r--. 1 mr8356 mr8356  81 Mar 15 12:37 cello.c
-rw-r--r--. 1 mr8356 mr8356 254 Mar 15 12:38 cello.patch
-rw-r--r--. 1 mr8356 mr8356 120 Mar 15 12:37 Makefile

mr8356@mr8356:~/rpmtest$ mv cello-1.0.tar.gz ~/rpmbuild/SOURCES/

mr8356@mr8356:~/rpmtest$ mv cello.patch ~/rpmbuild/SOURCES/
```

---

## Chapter 6. Packaging software

### 6.1. About spec files

**spec file**은 빌드 지침서입니다. **Preamble**(메타데이터)과 **Body**(빌드 스크립트)로 나뉩니다.

### 6.1.1. Preamble & 6.1.2. Body 핵심 항목

- **Name / Version / Release**: 패키지 식별 정보.
- **BuildRequires**: 빌드 시 필요한 도구 (`gcc`, `make`).
- **%prep**: 소스 압축 해제 및 패치 적용 (`%setup -q`, `%patch0 -p1`).
- **%install**: 가상 루트(`$BUILDROOT`)에 파일 배치.

### cello.spec 작성, Building RPMs

```bash
mr8356@mr8356:~/rpmbuild/SPECS$ cat cello.spec
Name:           cello
Version:        1.0
Release:        1%{?dist}
Summary:        Hello World example implemented in C
License:        GPLv3+
Source0:        %{name}-%{version}.tar.gz
Patch0:         cello.patch
BuildRequires:  gcc, make

%description
The long-tail description for our Hello World Example implemented in C.

%prep
%setup -q
%patch0 -p1

%build
make %{?_smp_mflags}

%install
%make_install

%files
%license LICENSE
%{_bindir}/%{name}

%changelog
* Sun Mar 15 2026 Jo Dong-hyeon <dh@konkuk.ac.kr> - 1.0-1
- First cello package with GPG verification

mr8356@mr8356:~/rpmbuild/SPECS$ rpmbuild -ba cello.spec
warning: %patchN is deprecated (1 usages found), use %patch N (or %patch -P N)
setting SOURCE_DATE_EPOCH=1773532800
error: Failed build dependencies:
	gcc is needed by cello-1.0-1.el10.aarch64
	make is needed by cello-1.0-1.el10.aarch64

RPM build warnings:
    %patchN is deprecated (1 usages found), use %patch N (or %patch -P N)
```

```bash
mr8356@mr8356:~/rpmbuild$ rpmbuild -ba SPECS/cello.spec
setting SOURCE_DATE_EPOCH=1773532800
Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.OIDJFV
+ umask 022
+ cd /home/mr8356/rpmbuild/BUILD
+ cd /home/mr8356/rpmbuild/BUILD
+ rm -rf cello-1.0
+ /usr/lib/rpm/rpmuncompress -x /home/mr8356/rpmbuild/SOURCES/cello-1.0.tar.gz
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd cello-1.0
+ rm -rf /home/mr8356/rpmbuild/BUILD/cello-1.0-SPECPARTS
+ /usr/bin/mkdir -p /home/mr8356/rpmbuild/BUILD/cello-1.0-SPECPARTS
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ echo 'Patch #0 (cello.patch):'
Patch #0 (cello.patch):
+ /usr/bin/patch --no-backup-if-mismatch -f -p0 --fuzz=0
patching file cello.c
+ RPM_EC=0
++ jobs -p
+ exit 0
Executing(%build): /bin/sh -e /var/tmp/rpm-tmp.aKrBLE
```

### 6.8. Signing RPM packages (GPG & Sequoia)

**RHEL 10**의 핵심은 **Sequoia PGP**를 통한 **PQC(양자 내성 암호)** 지원입니다. (Chapter 6.8.3)

```bash
mr8356@mr8356:~/rpmbuild$ rpm -Kv ~/rpmbuild/RPMS/$(arch)/cello-1.0-1.el10.$(arch).rpm
/home/mr8356/rpmbuild/RPMS/aarch64/cello-1.0-1.el10.aarch64.rpm:
    Header SHA256 digest: OK
    Header SHA1 digest: OK
    Payload SHA256 digest: OK
    MD5 digest: OK
```

---

## Step 1: 패키지 서명 (Signing - Chapter 6.8)

## Step 2: 저장소 생성 (Creating Repo - Chapter 6.5)

저장소의 핵심은 **Metadata입니다**. 

`dnf`가 "이 저장소에 어떤 패키지들이 있나?"를 알 수 있게 지도를 그려주는 작업이 필요합니다.

## Step 3: 시스템에 저장소 등록 (DNF Configuration)

리눅스 시스템에 사용자 저장소 경로를 등록해야 합니다.

### /etc/yum.repos.d/my-local.repo 파일 작성

```ini
[my-local-repo]
name=Jo Dong-hyeon Local Repository
baseurl=file:///home/mr8356/my-repo/
enabled=1
gpgcheck=1
# 아까 생성한 공개키 경로 (검증용)
gpgkey=file:///home/mr8356/jo-donghyeon.pub
EOF
```

---

## Step 4: 드디어 활용! (Installation)

사용자가 만든 `cello` 패키지를 인터넷이 되지 않는 환경에서도 설치할 수 있습니다.

### dnf로 설치하기

```bash
mr8356@mr8356:~/rpmbuild$ # dnf 캐시를 비우고 목록을 새로고침합니다.
sudo dnf clean all
sudo dnf repolist
25 files removed
repo id                                                                repo name
appstream                                                              Rocky Linux 10 - AppStream
baseos                                                                 Rocky Linux 10 - BaseOS
docker-ce-stable                                                       Docker CE Stable - aarch64
extras                                                                 Rocky Linux 10 - Extras
my-local-repo  

mr8356@mr8356:~/rpmbuild$ dnf search cello
error: /home/mr8356/.rpmmacros: line 8: Macro %_gpg_name has empty body
error: /home/mr8356/.rpmmacros: line 8: Macro %_gpg_name has empty body
Docker CE Stable - aarch64                                                                                                  72 kB/s |  18 kB     00:00
Jo Dong-hyeon Local Repository                                                                                             640 kB/s | 705  B     00:00
=============================================================== Name Exactly Matched: cello ===============================================================
cello.aarch64 : Hello World example implemented in C                                                        Jo Dong-hyeon Local Repository
```

```bash
mr8356@mr8356:~/rpmbuild$ sudo dnf install -y cello --nogpgcheck
Last metadata expiration check: 0:02:00 ago on Sun 15 Mar 2026 12:54:54 PM KST.
Dependencies resolved.
===========================================================================================================================================================
 Package                          Architecture                       Version                                Repository                                Size
===========================================================================================================================================================
Installing:
 cello                            aarch64                            1.0-1.el10                             my-local-repo                             10 k

Transaction Summary
===========================================================================================================================================================
Install  1 Package

Total size: 10 k
Installed size: 67 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                   1/1
  Installing       : cello-1.0-1.el10.aarch64                                                                                                          1/1
  Running scriptlet: cello-1.0-1.el10.aarch64                                                                                                          1/1

Installed:
  cello-1.0-1.el10.aarch64

Complete!

mr8356@mr8356:~/rpmbuild$ cello
Hello World from my first patch!
```

---

## RPM vs DNF: 역할의 본질적 차이

리눅스 패키지 관리 시스템은 계층 구조로 이루어져 있으며, **RPM**은 개별 부품을 다루는 하위 계층, **DNF**는 시스템 전체를 관리하는 상위 계층의 역할을 수행합니다.

### 1. RPM

**RPM**은 소프트웨어의 **포장 단위**이자 이를 처리하는 최하위 계층의 도구입니다.

- **패키지 단위(Artifact):** 실제 설치될 파일 본체, 설치 경로, 파일 권한 및 소유권 정보가 담긴 `.rpm` 파일 그 자체를 의미합니다.
- **제한적 기능:** `rpm -ivh` 명령어로 설치를 시도할 때, 필요한 의존 패키지가 없다면 "의존성 오류" 메시지만 출력하고 작업을 중단합니다. 스스로 외부에서 파일을 찾아오는 기능이 없습니다.
- **SRE/엔지니어 관점:** 패키지를 직접 **빌드(Build)**하거나, 보안 무결성을 위해 **GPG 서명(Sign)**을 할 때는 반드시 RPM 수준의 지식과 도구를 사용해야 합니다.

### 2. DNF

**DNF**는 RPM 상단에서 실행되며, **의존성(Dependency)**과 **저장소(Repository)**를 지능적으로 관리하는 도구입니다.

- **의존성 자동 해결:** 설치하려는 패키지에 필요한 추가 부품(의존성)이 있을 경우, 등록된 저장소에서 해당 파일들을 자동으로 찾아 함께 설치합니다.
- **저장소 관리(Metadata):** `/etc/yum.repos.d/`에 등록된 설정 파일을 참조하여 로컬 디렉토리나 원격 서버(HTTP/HTTPS)에서 패키지를 수소문하고 관리합니다.
- **SRE/엔지니어 관점:** 수백 대 이상의 서버에 패키지를 일괄적으로 **배포(Deployment)**하거나 최신 상태로 유지(Update)할 때는 반드시 DNF를 사용하여 효율성을 높여야 합니다.

---

```bash
mr8356@mr8356:~/rpmbuild$ cat /etc/yum.repos.d/my-local.repo
[my-local-repo]
name=Jo Dong-hyeon Local Repository
baseurl=file:///home/mr8356/my-repo/
enabled=1
gpgcheck=1
# 공개키 경로 (Chapter 6.8 GPG Verification)
gpgkey=file:///home/mr8356/jo-donghyeon.pub
```