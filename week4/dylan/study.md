# Rocky Linux 스터디 — 4주차
## 서비스 관리와 프로세스 제어

> **기준 버전:** Rocky Linux 10 (RHEL 10 기반, 커널 6.12.0)

---

## 이번 주 전체 맥락

```
부팅
  ↓
커널 로드
  ↓
PID 1: systemd 실행      ← 모든 것의 시작점
  ↓
유닛 파일 읽기 + 의존성 해결
  ↓
서비스들 병렬 시작       ← 예: sshd, NetworkManager, firewalld...
  ↓
로그인 가능 상태

이후:
  프로세스들이 실행됨
  각 서비스 = cgroup 하나 (자원 격리)
  systemctl 로 서비스 제어
  ps/top/kill 로 프로세스 조회·제어
  cron/at 으로 작업 스케줄링
```

---

## 1. systemd 구조

> 출처: [RHEL 10 Working with systemd unit files](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files)

### 1-1. systemd란

systemd는 PID 1로 실행되는 Linux 시스템·서비스 관리자입니다. 부팅부터 종료까지 모든 서비스의 시작/종료/감시를 담당합니다.

```
전통적인 SysV init:
  부팅 순서 = 스크립트를 순서대로 하나씩 실행 (직렬)
  /etc/rc.d/rc3.d/S10network → S15sshd → S20httpd ...
  → 느림, 의존성 관리 수동

systemd:
  의존성 그래프를 분석해서 병렬 실행 가능한 것은 동시에 실행
  → 빠름, 의존성 자동 해결
  → 서비스 상태 추적, 실패 시 자동 재시작
  → cgroup으로 자원 제어
```

### 1-2. 유닛 (Unit)

systemd가 관리하는 기본 단위를 **유닛**이라 합니다. 유닛 종류:

```
서비스 유닛 (.service)  ← 우리가 가장 많이 다루는 것
  예: sshd.service, httpd.service, firewalld.service

타겟 유닛  (.target)   ← 여러 유닛의 묶음 (그룹 상태 정의)
  예: multi-user.target (텍스트 모드), graphical.target (GUI)
  이전 SysV runlevel과 유사 개념

마운트 유닛 (.mount)   ← 파일시스템 마운트
  /etc/fstab → systemd가 .mount 유닛으로 자동 변환

타이머 유닛 (.timer)   ← cron 대체, 주기적 작업 실행

소켓 유닛  (.socket)   ← 소켓 활성화 (실제 연결 올 때 서비스 시작)

디바이스   (.device)   ← 하드웨어 장치 관리
```

### 1-3. 유닛 파일 위치와 우선순위

```
/usr/lib/systemd/system/     ← RPM 패키지가 설치한 기본 파일
/run/systemd/system/          ← 런타임에 동적 생성 (재부팅 시 사라짐)
/etc/systemd/system/          ← 관리자가 직접 수정/추가한 파일 ← 최우선

우선순위: /etc/ > /run/ > /usr/lib/
→ /etc/에 같은 이름 파일 만들면 기본 파일보다 우선 적용
```

패키지 기본 파일을 직접 수정하면 패키지 업데이트 시 덮어씌워집니다. **드롭인 파일**을 쓰면 원본을 보존하면서 일부만 오버라이드할 수 있습니다.

```
드롭인 파일 위치:
/etc/systemd/system/httpd.service.d/override.conf

→ 원본 /usr/lib/systemd/system/httpd.service는 유지
→ override.conf의 설정만 덮어씀
→ 패키지 업데이트해도 override.conf는 안 사라짐

쉬운 방법:
# systemctl edit httpd   ← 드롭인 파일을 에디터로 자동 열기
```

### 1-4. 서비스 유닛 파일 구조

```ini
# /usr/lib/systemd/system/sshd.service 예시

[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8)
After=network.target sshd-keygen.target   ← 이 유닛들 다음에 시작
Wants=sshd-keygen.target                  ← 약한 의존성 (없어도 시작 가능)

[Service]
Type=notify                               ← 시작 완료 시 systemd에 알림
EnvironmentFile=-/etc/sysconfig/sshd      ← 환경변수 파일 (없어도 무시)
ExecStartPre=/usr/sbin/sshd-keygen       ← 시작 전 실행
ExecStart=/usr/sbin/sshd -D $OPTIONS      ← 메인 프로세스
ExecReload=/bin/kill -HUP $MAINPID        ← reload 시 실행
KillMode=process
Restart=on-failure                        ← 실패 시 자동 재시작
RestartSec=42s

[Install]
WantedBy=multi-user.target               ← enable 시 이 타겟에 연결
```

#### [Unit] 섹션 주요 옵션

```
Description=   유닛 설명 (systemctl status에서 보임)
After=         이 유닛들이 시작된 '후에' 시작
Before=        이 유닛들이 시작되기 '전에' 시작
Requires=      강한 의존성 — 나열된 유닛 실패 시 이 유닛도 실패
Wants=         약한 의존성 — 나열된 유닛 실패해도 이 유닛은 계속
Conflicts=     충돌 관계 — 나열된 유닛과 동시에 실행 불가
```

After/Requires 차이:

```
After=B.service
  → "B가 시작된 후에 A를 시작해라" (순서)
  → B가 실패해도 A는 시작 시도

Requires=B.service
  → "A는 B가 필요하다" (의존성)
  → B가 실패하면 A도 실패

After + Requires 둘 다 쓰는 경우가 많음:
  After=database.service
  Requires=database.service
  → database가 먼저 시작되어야 하고, database 실패 시 이 서비스도 실패
```

#### [Service] 섹션 주요 옵션

```
Type= 서비스 유형:
  simple   기본값. ExecStart 프로세스 자체가 메인 프로세스
  forking  전통 데몬 방식. 부모 프로세스가 fork 후 종료. PIDFile= 필요
  notify   systemd에 준비 완료 알림을 직접 보냄 (sd_notify 사용)
  oneshot  한 번 실행 후 종료 (스크립트 실행에 적합)
  idle     다른 작업 다 끝난 후 실행

Restart= 재시작 조건:
  no          재시작 안 함 (기본값)
  on-failure  비정상 종료 시만 재시작 (exit code != 0 또는 시그널)
  always      항상 재시작
  on-abnormal 비정상 종료 + 타임아웃 시
  on-abort    abort 시그널 받았을 때만

RestartSec=  재시작 전 대기 시간 (기본값 100ms)

ExecStart=   시작 명령
ExecStop=    종료 명령 (없으면 systemd가 SIGTERM 보냄)
ExecReload=  reload 명령

User=        실행할 사용자 (보안: root 대신 전용 계정 사용)
Group=       실행할 그룹
WorkingDirectory=  작업 디렉토리
Environment= 환경변수 설정
```

#### [Install] 섹션

```
WantedBy=multi-user.target
  → systemctl enable 실행 시
  → /etc/systemd/system/multi-user.target.wants/sshd.service 심볼릭 링크 생성
  → 시스템이 multi-user.target에 도달하면 sshd 자동 시작

systemctl enable = WantedBy 타겟의 .wants/ 에 심볼릭 링크 생성
systemctl disable = 그 심볼릭 링크 제거
```

### 1-5. 타겟 (Target) — 시스템 상태 정의

```
타겟 = 여러 유닛의 묶음. 시스템의 "상태"를 정의

주요 타겟:
  poweroff.target      종료
  rescue.target        단일 사용자 모드 (root 전용, 네트워크 없음)
  multi-user.target    다중 사용자 텍스트 모드 (서버 기본값)
  graphical.target     GUI 모드 (multi-user.target 포함)
  emergency.target     긴급 모드 (루트 파일시스템만 읽기 마운트)

SysV runlevel ↔ systemd target 비교:
  runlevel 0 = poweroff.target
  runlevel 1 = rescue.target
  runlevel 3 = multi-user.target
  runlevel 5 = graphical.target
  runlevel 6 = reboot.target
```

```bash
# 현재 타겟 확인
systemctl get-default

# 기본 타겟 변경 (다음 부팅부터 적용)
systemctl set-default multi-user.target

# 지금 즉시 타겟 전환 (서버 GUI 끄기 등)
systemctl isolate multi-user.target
```

---

## 2. systemctl — 서비스 관리

> 출처: [RHEL 10 Managing system services with systemctl](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl)

### 2-1. 서비스 상태 조회

```bash
# 상태 확인 — 가장 많이 씀
systemctl status httpd

# 출력 예시:
# ● httpd.service - The Apache HTTP Server
#    Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; ...)
#    Active: active (running) since Mon 2025-01-13 09:00:00 KST; 2h ago
#  Main PID: 1234 (httpd)
#    CGroup: /system.slice/httpd.service
#            ├─1234 /usr/sbin/httpd -DFOREGROUND
#            └─1235 /usr/sbin/httpd -DFOREGROUND
# Jan 13 09:00:00 server httpd[1234]: Server configured, ...

# Loaded 줄: 파일 로드됨 + 활성화 상태 (enabled/disabled)
# Active 줄: 현재 실행 상태

# 단순히 실행 중인지만 확인 (스크립트용)
systemctl is-active httpd    # 실행 중이면 "active" 출력, exit 0
systemctl is-enabled httpd   # 부팅 시 시작 설정이면 "enabled", exit 0

# 전체 유닛 목록
systemctl list-units --type service          # 현재 로드된 서비스만
systemctl list-units --type service --all    # 비활성 포함 전체
systemctl list-unit-files --type service     # 설치된 서비스 파일 목록

# 특정 서비스의 의존성 확인
systemctl list-dependencies httpd            # httpd가 의존하는 것
systemctl list-dependencies --after httpd    # httpd 이전에 시작해야 하는 것
systemctl list-dependencies --before httpd   # httpd 이후에 시작되는 것
```

### 2-2. 서비스 제어

```bash
# 시작 / 중지 / 재시작
systemctl start httpd
systemctl stop httpd
systemctl restart httpd      # stop → start (연결 끊김 발생)

# 설정 파일만 다시 읽기 (가능한 서비스에 한해)
systemctl reload httpd       # 서비스 중단 없이 설정 적용
systemctl reload-or-restart httpd  # reload 불가능하면 restart

# 부팅 시 자동 시작 설정
systemctl enable httpd       # 다음 부팅부터 자동 시작
systemctl disable httpd      # 자동 시작 해제 (지금 실행 중인 것은 안 끔)
systemctl enable --now httpd # enable + 지금 바로 시작
systemctl disable --now httpd # disable + 지금 바로 중지

# 마스킹 (완전 차단 — 수동으로도 시작 불가)
systemctl mask httpd
systemctl unmask httpd
```

restart vs reload:

```
restart:
  httpd stop → httpd start
  → 새 프로세스 생성, 기존 연결 끊김
  → 설정 파일 오류여도 중지까지는 됨

reload:
  httpd 프로세스에 SIGHUP (또는 ExecReload 명령)
  → 설정 파일 다시 읽음
  → 기존 연결 유지됨
  → 모든 서비스가 지원하지는 않음 (지원 여부는 ExecReload= 존재 여부)
```

enable vs start:

```
start  = 지금 당장 실행 (재부팅하면 다시 꺼짐)
enable = 부팅 시 자동 시작 설정 (지금은 안 켜짐)

보통 둘 다:
systemctl enable --now httpd
```

mask vs disable:

```
disable: enable 취소. 하지만 수동으로 start는 가능
mask:    /etc/systemd/system/xxx.service → /dev/null 심볼릭 링크
         수동 start도 차단. 절대 실행하면 안 되는 서비스에 사용
```

### 2-3. 유닛 파일 수정 후 반드시 해야 하는 것

```bash
# 유닛 파일 직접 수정 후
systemctl daemon-reload

# 이걸 안 하면:
# → systemd가 디스크의 변경을 모름
# → start/enable 명령이 오작동 가능
```

### 2-4. 커스텀 서비스 유닛 파일 만들기

예시: 간단한 Python 웹서버를 서비스로 등록

```bash
# 1. 실행 파일 준비 (예시)
cat > /usr/local/bin/myapp.sh << 'EOF'
#!/bin/bash
cd /opt/myapp
python3 -m http.server 8080
EOF
chmod +x /usr/local/bin/myapp.sh

# 2. 전용 사용자 생성 (보안)
useradd --system --no-create-home myapp

# 3. 유닛 파일 작성
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Custom App
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/local/bin/myapp.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 4. 등록 및 시작
systemctl daemon-reload
systemctl enable --now myapp

# 5. 확인
systemctl status myapp
```

---

## 3. 프로세스 조회 및 제어

### 3-1. 프로세스란

```
프로그램 = 디스크에 저장된 실행 파일 (수동적, 정적)
프로세스 = 실행 중인 프로그램 인스턴스 (능동적, 메모리에 로드됨)

각 프로세스는:
  PID (Process ID) — 고유 식별 번호
  PPID (Parent PID) — 부모 프로세스 ID
  UID/GID — 실행 사용자/그룹 (권한 결정)
  상태 — Running, Sleeping, Stopped, Zombie 등
  우선순위 — nice 값 (-20~+19)
```

프로세스 트리:

```
PID 1: systemd
  ├─ sshd (PID 1001)
  │    └─ bash (PID 2001) ← ssh 접속한 사용자의 쉘
  │         └─ vim (PID 3001) ← 사용자가 실행한 에디터
  ├─ httpd (PID 1002)
  │    ├─ httpd (PID 1003)  ← worker 프로세스
  │    └─ httpd (PID 1004)
  └─ cron (PID 1005)
```

### 3-2. ps — 프로세스 스냅샷 조회

```bash
# 내 세션의 프로세스만 (기본)
ps

# 자주 쓰는 옵션 조합
ps aux      # 모든 프로세스, 상세 정보
ps -ef      # 모든 프로세스, PPID 포함 (SysV 스타일)

# aux 출력 열 의미:
# USER  PID  %CPU  %MEM  VSZ    RSS    TTY  STAT  START  TIME  COMMAND
# root  1234  0.0   0.1  12345  6789   ?    Ss    09:00  0:01  sshd

# STAT (상태) 열:
# R — Running/Runnable (실행 중 또는 실행 대기)
# S — Sleeping (인터럽트 가능한 대기)
# D — Uninterruptible Sleep (I/O 대기 — kill도 안 됨)
# T — Stopped (Ctrl+Z로 정지됨)
# Z — Zombie (종료됐지만 부모가 wait() 안 함)
# s — 세션 리더
# + — 포그라운드 프로세스 그룹

# 특정 프로세스 찾기
ps aux | grep httpd
pgrep -l httpd       # 이름으로 PID 찾기 (더 간단)

# 프로세스 트리 보기
ps auxf              # 트리 형태 출력
pstree               # 더 보기 좋은 트리
pstree -p            # PID 포함

# VSZ vs RSS
# VSZ (Virtual Size): 프로세스가 요청한 가상 메모리 전체
# RSS (Resident Set Size): 실제 RAM에 올라간 크기 ← 실제 메모리 사용량
```

### 3-3. top — 실시간 프로세스 모니터링

```bash
top   # 실행 후 대화형 명령어:
      # q — 종료
      # k — kill (PID 입력 후 시그널 입력)
      # r — renice (nice 값 변경)
      # M — 메모리 사용량 순 정렬
      # P — CPU 사용량 순 정렬 (기본값)
      # 1 — CPU별 상세 보기
      # u — 특정 사용자 필터
      # f — 컬럼 설정
      # s — 갱신 간격 변경
```

top 헤더 읽는 법:

```
top - 10:30:00 up 2 days, 3:10, 2 users, load average: 0.15, 0.10, 0.08
Tasks: 180 total, 1 running, 178 sleeping, 1 stopped, 0 zombie
%Cpu(s): 2.3 us, 0.5 sy, 0.0 ni, 96.8 id, 0.4 wa, 0.0 hi, 0.0 si
MiB Mem:   7843.2 total,  4231.1 free,  2104.5 used,  1507.6 buff/cache
MiB Swap:  2048.0 total,  2048.0 free,     0.0 used.  5321.2 avail Mem

① load average: 0.15, 0.10, 0.08
   순서대로 1분, 5분, 15분 평균 실행 대기 프로세스 수
   CPU 코어 수와 비교: 4코어 시스템에서 load 4.0이면 100% 부하
   0.15면 여유 있는 상태

② %Cpu(s):
   us = user (사용자 프로그램)
   sy = system (커널)
   ni = nice (우선순위 조정된 프로세스)
   id = idle (놀고 있는 비율)
   wa = wait (I/O 대기) ← 높으면 디스크/네트워크 병목
   hi = hardware interrupt
   si = software interrupt

③ buff/cache:
   OS가 디스크 캐시로 쓰는 메모리
   실제로 필요하면 즉시 해제됨 → "낭비"가 아님
   avail Mem = 실제로 쓸 수 있는 가용 메모리
```

더 좋은 대안:

```bash
htop     # 색상, 마우스 지원 (dnf install htop)
```

### 3-4. 시그널과 kill

시그널 = 프로세스에 보내는 비동기 통지 메시지

```
주요 시그널:
번호  이름     동작
1     SIGHUP   재시작/설정 파일 재로드 (많은 데몬이 이걸 이용)
2     SIGINT   인터럽트 (Ctrl+C 와 동일)
9     SIGKILL  강제 종료 — 프로세스가 무시 불가, 즉시 종료
15    SIGTERM  정상 종료 요청 — 프로세스가 받아서 graceful 종료 가능
17    SIGCHLD  자식 프로세스 종료 알림
18    SIGCONT  정지된 프로세스 재개
19    SIGSTOP  프로세스 정지 — 무시 불가
20    SIGTSTP  정지 (Ctrl+Z 와 동일, 무시 가능)
```

SIGTERM vs SIGKILL:

```
SIGTERM (15) — "정중하게 종료 요청"
  프로세스가 신호를 받아서 처리
  → 열린 파일 닫기, DB 연결 종료, 임시 파일 정리
  → 데이터 유실 없이 종료

SIGKILL (9) — "강제 즉시 종료"
  커널이 직접 프로세스 제거
  프로세스 코드가 실행될 틈이 없음
  → 정리 작업 없이 강제 종료 → 데이터 손상 위험
  → D 상태(Uninterruptible Sleep)에도 효과 없음 (하드웨어 I/O 대기 중)

올바른 순서:
  1. kill PID        (기본 = SIGTERM, graceful)
  2. 안 죽으면 기다리기
  3. kill -9 PID     (강제, 최후의 수단)
```

```bash
# 기본: SIGTERM 전송
kill 1234
kill -15 1234     # 동일

# 강제 종료
kill -9 1234
kill -KILL 1234   # 동일

# 이름으로 종료
pkill httpd        # 이름으로 SIGTERM
pkill -9 httpd     # 이름으로 SIGKILL
killall httpd      # 동일 이름 모두

# 시그널 목록
kill -l

# 시그널 확인 (어떤 시그널에 죽었는지)
echo $?    # 종료 코드 = 128 + 시그널 번호
           # SIGKILL로 죽었으면 137 (128+9)
```

### 3-5. nice와 우선순위

```
CPU 스케줄러가 어떤 프로세스를 먼저 실행할지 결정
→ nice 값으로 우선순위 조절 가능

nice 값 범위: -20 ~ +19
  -20 = 최고 우선순위 (남들보다 CPU 많이 받음)
  0   = 기본값
  +19 = 최저 우선순위 (남들한테 양보)

"nice" = 다른 프로세스에게 nice(양보)하다
nice 값이 높을수록 더 양보 = 우선순위 낮음
```

```bash
# 우선순위 낮춰서 실행 (백업처럼 중요하지 않은 작업)
nice -n 10 tar -czf backup.tar.gz /data
nice -n 19 find / -name "*.log"    # 최저 우선순위로 파일 검색

# 실행 중인 프로세스 우선순위 변경
renice -n 5 -p 1234      # PID 1234의 nice값을 5로
renice -n -5 -p 1234     # -5로 (root만 가능)
renice -n 10 -u alice    # alice의 모든 프로세스에 적용

# 일반 사용자: nice 값 증가만 가능 (더 낮은 우선순위로만)
# root: 감소 가능 (더 높은 우선순위로)

# top에서 확인:
# PR = 실제 커널 우선순위 (PR = 20 + NI)
# NI = nice 값
```

### 3-6. 포그라운드 / 백그라운드

```bash
# 포그라운드 실행 (기본) — 터미널 점령
sleep 1000

# 백그라운드로 전환
Ctrl+Z     # 실행 중인 프로세스 일시 정지 (SIGTSTP)
bg         # 일시 정지된 프로세스를 백그라운드로 재개
fg         # 백그라운드 프로세스를 포그라운드로
fg %2      # 2번 작업을 포그라운드로

# 처음부터 백그라운드 실행
sleep 1000 &

# 현재 백그라운드 작업 목록
jobs
jobs -l    # PID 포함

# 터미널 닫아도 계속 실행 (HUP 시그널 무시)
nohup sleep 1000 &
nohup ./script.sh > output.log 2>&1 &
```

---

## 4. 작업 스케줄링

### 4-1. cron — 반복 작업

cron은 주기적으로 반복 실행되는 작업 스케줄러입니다.

#### crontab 형식

```
분  시  일  월  요일  명령어
│   │   │   │   │
│   │   │   │   └── 0-7 (0과 7 모두 일요일)
│   │   │   └────── 1-12
│   │   └────────── 1-31
│   └────────────── 0-23
└────────────────── 0-59
```

예시:

```
# 매일 새벽 2시에 백업
0 2 * * * /opt/backup.sh

# 5분마다 실행
*/5 * * * * /opt/monitor.sh

# 월,수,금 오전 9시에
0 9 * * 1,3,5 /opt/report.sh

# 매월 1일 자정에
0 0 1 * * /opt/monthly.sh

# 월요일~금요일 9-18시 매 시간 정각
0 9-18 * * 1-5 /opt/hourly.sh

# 특수 키워드
@reboot   /opt/startup.sh      # 부팅 시 한 번
@daily    /opt/daily.sh        # 매일 자정 (= 0 0 * * *)
@weekly   /opt/weekly.sh       # 매주 일요일 자정
@monthly  /opt/monthly.sh      # 매월 1일 자정
@hourly   /opt/hourly.sh       # 매 시간 정각
```

```bash
# crontab 편집 (현재 사용자)
crontab -e

# crontab 목록 보기
crontab -l

# crontab 삭제
crontab -r

# 다른 사용자 crontab 관리 (root만)
crontab -e -u alice
crontab -l -u alice
```

#### 시스템 crontab 파일

```bash
# 시스템 전체 cron 작업 (root 권한)
/etc/crontab           # 사용자 필드 추가됨
/etc/cron.d/           # 개별 패키지가 설치하는 cron 파일
/etc/cron.hourly/      # 매 시간 실행할 스크립트
/etc/cron.daily/       # 매일 실행
/etc/cron.weekly/      # 매주 실행
/etc/cron.monthly/     # 매월 실행

# /etc/crontab 형식 (사용자 필드 있음):
# 분 시 일 월 요일 사용자 명령어
  0  2  *  *  *   root   /opt/backup.sh
```

#### cron 접근 제어

```
/etc/cron.allow  존재하면 여기 있는 사용자만 crontab 사용 가능
/etc/cron.deny   여기 있는 사용자는 crontab 사용 불가

cron.allow 있으면 cron.deny 무시
둘 다 없으면 root만 사용 가능 (시스템마다 다를 수 있음)
```

#### cron 주의사항

```bash
# cron 환경변수는 최소한:
# PATH=/usr/bin:/bin 정도만
# .bashrc 안 읽음

# 반드시 절대 경로 사용:
# 나쁨: backup.sh
# 좋음: /opt/backup.sh

# 출력이 있으면 메일로 보내거나 리다이렉트:
0 2 * * * /opt/backup.sh > /var/log/backup.log 2>&1

# 환경변수 직접 설정:
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com
0 2 * * * /opt/backup.sh
```

### 4-2. at — 일회성 예약 작업

```bash
# 설치 확인
dnf install at
systemctl enable --now atd

# 명령어 예약 (대화형)
at 14:30              # 오늘 14:30에
at 14:30 tomorrow     # 내일 14:30에
at 14:30 Jan 15       # 1월 15일 14:30에
at now + 30 minutes   # 지금부터 30분 후
at now + 2 hours
at now + 3 days

# 입력 후 명령어 입력하고 Ctrl+D 로 종료
at 14:30
at> /opt/deploy.sh
at> mail -s "배포완료" admin@example.com < /dev/null
at> (Ctrl+D)
# job 3 at Mon Jan 13 14:30:00 2025

# 파일로 예약
at 14:30 < /opt/scheduled_tasks.sh

# 예약 목록 확인
atq
at -l   # 동일

# 예약 취소
atrm 3    # job 번호로 취소

# 예약 내용 확인 (root만)
at -c 3
```

### 4-3. systemd timer — cron 대체

systemd timer는 cron보다 정밀하고 로그 관리가 편합니다.

```bash
# timer 유닛 구성 (cron 대신 사용 가능)
# 1. service 유닛
cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/opt/backup.sh
User=root
EOF

# 2. timer 유닛
cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=*-*-* 02:00:00    # 매일 02:00
Persistent=true               # 놓친 실행 있으면 즉시 실행

[Install]
WantedBy=timers.target
EOF

# 3. 등록
systemctl daemon-reload
systemctl enable --now backup.timer

# 4. 타이머 목록 확인
systemctl list-timers

# OnCalendar 형식 예시:
# OnCalendar=daily                  매일 자정
# OnCalendar=weekly                 매주 월요일 자정
# OnCalendar=Mon 02:30              매주 월요일 02:30
# OnCalendar=*-*-* 00/6:00:00       6시간마다
# OnCalendar=2025-01-15 14:00:00    특정 날짜·시간

# cron과 달리 실패 로그도 journalctl로:
journalctl -u backup.service
```

---

## 5. cgroup 기반 자원 제어

> 출처:
> [RHEL 10 Setting system resource limits by using control groups](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/setting-system-resource-limits-for-applications-by-using-control-groups)
> [RHEL 10 Using systemd to manage resources](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/using-systemd-to-manage-resources-used-by-applications)

### 5-1. cgroup이란

**Control Group (cgroup)** = 프로세스 그룹에 자원 제한을 거는 커널 기능

```
cgroup 없을 때의 문제:
  서버 A: DB 서버 (중요)
  서버 B: 로그 분석 (덜 중요)
  
  → 로그 분석이 CPU 100% 점유하면 DB 응답 느려짐
  → nice로 조절 가능하지만 메모리/I/O는 못 막음

cgroup 있으면:
  DB 서비스 cgroup: CPU 70% 이상 보장
  로그 서비스 cgroup: CPU 30% 이내로 제한
  → 중요 서비스 자원 보장
```

### 5-2. cgroups v1 vs v2

```
RHEL 10은 기본으로 cgroups v2 사용

cgroups v1:
  컨트롤러마다 별도 계층(트리)
  → /sys/fs/cgroup/cpu/ 따로
  → /sys/fs/cgroup/memory/ 따로
  → 복잡, 일관성 문제

cgroups v2:
  단일 계층에 모든 컨트롤러
  → /sys/fs/cgroup/ 하나
  → 일관된 인터페이스
  → RHEL 10 기본값
```

### 5-3. cgroup 계층 구조

```
/sys/fs/cgroup/                     ← 루트 cgroup
├── system.slice/                   ← 시스템 서비스 그룹
│   ├── httpd.service/              ← httpd의 cgroup
│   ├── sshd.service/               ← sshd의 cgroup
│   └── mysql.service/
├── user.slice/                     ← 사용자 세션 그룹
│   └── user-1000.slice/
│       └── session-1.scope/        ← 사용자의 프로세스
└── machine.slice/                  ← 가상머신/컨테이너 그룹
```

systemd가 서비스를 시작하면 자동으로 서비스마다 cgroup 하나씩 생성됩니다.

```bash
# cgroup 계층 확인
systemd-cgls                         # 전체 cgroup 트리
systemd-cgls httpd.service           # 특정 서비스만

# 리소스 사용량 실시간 모니터링
systemd-cgtop                        # CPU/메모리/I/O 순 정렬

# 특정 프로세스의 cgroup 확인
cat /proc/1234/cgroup
# 0::/system.slice/httpd.service
```

### 5-4. systemctl로 자원 제한 설정

systemctl을 통해 cgroup 설정을 바꾸는 것이 RHEL 10 권장 방법입니다.

#### CPU 자원 제어

```bash
# CPUWeight= : 상대적 CPU 가중치 (1~10000, 기본값 100)
# 비율 기반으로 분배 (절대값 아님)
systemctl set-property httpd.service CPUWeight=200
systemctl set-property backup.service CPUWeight=50
# → httpd : backup = 200 : 50 = 4 : 1 비율로 CPU 분배

# CPUQuota= : CPU 사용 상한선 (절대값)
systemctl set-property backup.service CPUQuota=30%
# → backup 서비스는 CPU 30% 이상 못 씀
# → 멀티코어에서 30%는 전체 코어 합산 기준

# 영구 적용 확인
systemctl cat httpd.service            # 현재 설정 확인
cat /etc/systemd/system/httpd.service.d/50-CPUWeight.conf
```

#### 메모리 자원 제어

```bash
# MemoryMax= : 절대 상한 (이 이상 사용 시 OOM killer 호출)
systemctl set-property httpd.service MemoryMax=1G

# MemoryHigh= : 소프트 상한 (넘으면 스로틀링, 강제 종료는 아님)
systemctl set-property httpd.service MemoryHigh=800M

# MemoryMin= : 보장 최소 메모리 (시스템이 부족해도 이것만큼은 보장)
systemctl set-property db.service MemoryMin=512M

# 단위: K, M, G, T 사용 가능
```

#### I/O 자원 제어

```bash
# IOWeight= : I/O 가중치 (1~10000, 기본값 100)
systemctl set-property backup.service IOWeight=10    # I/O 낮은 우선순위

# IOReadBandwidthMax=, IOWriteBandwidthMax= : 대역폭 제한
systemctl set-property backup.service IOWriteBandwidthMax="/dev/sda 50M"
# → /dev/sda 쓰기 속도를 초당 50MB로 제한
```

#### 적용 확인

```bash
# 현재 cgroup 설정 확인
systemctl show httpd.service | grep -E "CPU|Memory|IO"

# 런타임 상태 확인
systemctl status httpd
# CGroup 줄 아래에 프로세스 목록과 cgroup 경로 표시

# /sys/fs/cgroup 직접 확인
cat /sys/fs/cgroup/system.slice/httpd.service/cpu.weight
cat /sys/fs/cgroup/system.slice/httpd.service/memory.max
```

### 5-5. cgroup 자원 제어 원리

```
CPU Weight 작동 방식:
  httpd:  CPUWeight=200
  backup: CPUWeight=50
  mysql:  CPUWeight=100
  
  총 가중치 = 200 + 50 + 100 = 350
  
  CPU 100% 부하 상태에서:
  httpd  = 200/350 ≈ 57%
  backup = 50/350  ≈ 14%
  mysql  = 100/350 ≈ 29%

  단, CPU 여유 있을 때는 제한 없이 모두 사용 가능
  (= 보장이지 상한선이 아님)

CPUQuota 작동 방식:
  CPUQuota=30%  ← 상한선
  → 시스템이 놀아도 30% 이상 못 쓰게 차단
```

### 5-6. slice 단위 자원 제어

개별 서비스가 아닌 그룹(slice) 전체에 제한을 걸 수 있습니다.

```bash
# system.slice = 모든 시스템 서비스의 부모
# user.slice   = 모든 사용자 세션의 부모

# 예: 사용자 전체 세션이 CPU 50% 이상 못 쓰게
systemctl set-property user.slice CPUQuota=50%

# 예: 시스템 서비스 전체 메모리 제한
systemctl set-property system.slice MemoryMax=4G
```

---

## 6. journalctl — 서비스 로그 조회

> systemd는 journald로 모든 로그를 중앙 관리합니다.

```bash
# 전체 로그 (최신 것부터)
journalctl -r

# 특정 서비스 로그
journalctl -u httpd
journalctl -u httpd -u sshd   # 여러 서비스

# 실시간 추적 (tail -f 느낌)
journalctl -u httpd -f

# 최근 N줄
journalctl -u httpd -n 50

# 시간 범위 필터
journalctl -u httpd --since "2025-01-13 09:00"
journalctl -u httpd --since "1 hour ago"
journalctl --since today

# 에러/경고 레벨만
journalctl -p err           # error 이상
journalctl -p warning       # warning 이상
journalctl -u httpd -p err  # 특정 서비스의 에러만

# 이번 부팅 로그만
journalctl -b
journalctl -b -1    # 이전 부팅
journalctl -b -2    # 2번 전 부팅

# 커널 로그 (dmesg 대체)
journalctl -k

# 로그 디스크 사용량
journalctl --disk-usage

# 오래된 로그 정리
journalctl --vacuum-time=30d    # 30일 이전 삭제
journalctl --vacuum-size=500M   # 500MB 이하로 정리
```

---

## 7. 전체 빠른 참조 치트시트

```bash
# ── systemctl ─────────────────────────────────────────────
systemctl status httpd          # 상태 확인
systemctl start/stop/restart httpd
systemctl reload httpd          # 설정 재로드 (연결 유지)
systemctl enable/disable httpd  # 부팅 시 자동 시작 설정
systemctl enable --now httpd    # enable + 지금 시작
systemctl mask/unmask httpd     # 완전 차단 / 해제
systemctl daemon-reload         # 유닛 파일 수정 후 필수
systemctl list-units --type service
systemctl list-unit-files --type service
systemctl list-timers           # 타이머 목록
systemctl get-default           # 현재 기본 타겟
systemctl set-default multi-user.target

# ── 프로세스 조회 ──────────────────────────────────────────
ps aux                          # 전체 프로세스 스냅샷
ps -ef                          # PPID 포함
ps auxf                         # 트리 형태
pgrep -l httpd                  # 이름으로 PID 찾기
pstree -p                       # 트리 + PID
top / htop                      # 실시간 모니터링

# ── 프로세스 제어 ──────────────────────────────────────────
kill PID                        # SIGTERM (정상 종료 요청)
kill -9 PID                     # SIGKILL (강제 종료)
kill -1 PID                     # SIGHUP (재로드)
pkill httpd                     # 이름으로 SIGTERM
killall httpd
nice -n 10 command              # 낮은 우선순위로 실행
renice -n 5 -p PID              # 실행 중인 프로세스 우선순위 변경
jobs                            # 백그라운드 작업 목록
bg / fg                         # 백그라운드/포그라운드 전환
nohup command &                 # 터미널 닫아도 계속 실행

# ── cron ──────────────────────────────────────────────────
crontab -e                      # crontab 편집
crontab -l                      # 목록 보기
crontab -r                      # 삭제

# 형식: 분 시 일 월 요일 명령어
# */5 * * * * command           5분마다
# 0 2 * * * command             매일 02:00
# 0 9 * * 1-5 command           월~금 09:00

# ── at ────────────────────────────────────────────────────
at 14:30                        # 오늘 14:30 (Ctrl+D 로 종료)
at now + 30 minutes
atq                             # 예약 목록
atrm 3                          # 예약 취소

# ── cgroup 자원 제어 ───────────────────────────────────────
systemctl set-property httpd.service CPUWeight=200
systemctl set-property httpd.service CPUQuota=50%
systemctl set-property httpd.service MemoryMax=1G
systemctl set-property httpd.service IOWeight=10
systemd-cgls                    # cgroup 트리 확인
systemd-cgtop                   # 실시간 자원 사용량

# ── 로그 ──────────────────────────────────────────────────
journalctl -u httpd             # 서비스 로그
journalctl -u httpd -f          # 실시간 로그
journalctl -u httpd -p err      # 에러만
journalctl -b                   # 이번 부팅 로그
journalctl --since "1 hour ago"
```

---

## 📚 주요 출처

| 문서 | URL |
|------|-----|
| RHEL 10 Using systemd unit files | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/ |
| RHEL 10 Managing system services with systemctl | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl |
| RHEL 10 Setting system resource limits (cgroups) | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/setting-system-resource-limits-for-applications-by-using-control-groups |
| RHEL 10 Using systemd to manage resources | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/using-systemd-to-manage-resources-used-by-applications |
| RHEL 10 Using cgroupfs to manually manage cgroups | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/using-cgroupfs-to-manually-manage-cgroups |
