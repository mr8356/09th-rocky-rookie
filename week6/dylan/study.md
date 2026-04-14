# Rocky Linux 스터디 — 6주차
## 셸 스크립트 및 자동화 기초

> **기준 버전:** Rocky Linux 10 (Bash 5.2)

---

## 이번 주 전체 맥락

```
지금까지 배운 것들:
  systemctl, nmcli, firewall-cmd, dnf ...

이것들을 매번 손으로 치는 게 아니라
→ 스크립트로 묶어서 자동화

셸 스크립트 = 커맨드를 순서대로 실행하는 텍스트 파일
            + 변수, 조건문, 반복문으로 로직 추가
```

---

## 1. Bash 셸 기본 구조

### 1-1. 셸이란

```
터미널에서 명령어를 입력하면:

사용자 입력: ls -la /etc
      ↓
셸 (Bash)    ← 명령어 해석, 변수 치환, 파이프 처리
      ↓
커널         ← 실제 실행
      ↓
결과 출력

셸 종류:
  /bin/bash   Bourne Again Shell — Linux 기본
  /bin/sh     POSIX 호환 최소 셸 (bash가 symlink인 경우 많음)
  /bin/zsh    macOS 기본
  /bin/fish   대화형 친화적

현재 셸 확인:
  echo $SHELL
  cat /etc/shells   # 시스템에 설치된 셸 목록
```

### 1-2. 스크립트 기본 구조

```bash
#!/bin/bash
# 이 줄 = shebang (셔뱅)
# 이 파일을 어떤 인터프리터로 실행할지 커널에게 알려줌
# /bin/bash 로 실행하라는 의미

# 주석은 # 으로 시작

echo "Hello, World!"
```

shebang이 없으면:

```
셔뱅 없는 경우 → 현재 셸이 실행
셔뱅 있는 경우 → 커널이 직접 /bin/bash 실행
→ 다른 사용자나 cron에서 실행해도 동일한 동작 보장
→ 항상 써주는 게 맞음
```

### 1-3. 스크립트 실행 방법

```bash
# 1. 실행 권한 부여 후 직접 실행 (가장 일반적)
chmod +x script.sh
./script.sh

# 2. bash로 직접 실행 (권한 없어도 됨)
bash script.sh

# 3. 현재 셸에서 실행 (변수/함수가 현재 셸에 남음)
source script.sh
. script.sh          # 동일

# 차이:
# ./script.sh  → 자식 프로세스 생성, 실행 완료 후 사라짐
# source       → 현재 셸 프로세스에서 직접 실행
#                ~/.bashrc 적용할 때 source 쓰는 이유
```

### 1-4. 종료 코드

셸에서 모든 명령은 실행 후 종료 코드를 반환합니다.

```bash
0        성공
1~255    실패 (의미는 프로그램마다 다름)
1        일반 오류
2        잘못된 사용법
126      실행 권한 없음
127      명령어 없음
130      Ctrl+C (128 + SIGINT 2)

# 직전 명령의 종료 코드 확인
echo $?

ls /etc
echo $?    # 0 (성공)

ls /없는경로
echo $?    # 2 (실패)

# 스크립트 내에서 명시적 종료
exit 0     # 성공으로 종료
exit 1     # 실패로 종료
```

종료 코드가 중요한 이유:

```bash
# && : 앞이 성공(0)이어야 뒤 실행
systemctl start httpd && echo "시작됨"

# || : 앞이 실패해야 뒤 실행
systemctl start httpd || echo "시작 실패"

# 스크립트 자동화에서 성공/실패 분기에 핵심
```

---

## 2. 변수와 환경 변수

### 2-1. 변수 기본

```bash
# 대입 (= 양쪽에 공백 없어야 함)
name="alice"
count=10
path="/var/log"

# 나쁜 예 (오류 발생)
name = "alice"   # 오류: = 양쪽 공백 안 됨

# 사용 — $ 붙이기
echo $name
echo "Hello, $name"
echo "Hello, ${name}"    # 중괄호로 명확하게 (권장)

# 중괄호가 필요한 경우
file="report"
echo "${file}_2025.log"   # report_2025.log
echo "$file_2025.log"     # 변수명이 file_2025 로 해석됨 → 비어있음
```

### 2-2. 변수 타입과 선언

```bash
# 기본: 모든 변수는 문자열
x=10
y=20
echo $x + $y     # "10 + 20" (문자열 연결)

# 정수 연산
echo $((x + y))  # 30
echo $((x * y))  # 200
echo $((y / x))  # 2

# 읽기 전용
readonly PI=3.14159
PI=3              # 오류

# 정수 선언
declare -i count=0
count+=1           # 정수 덧셈

# 배열
fruits=("apple" "banana" "cherry")
echo ${fruits[0]}         # apple
echo ${fruits[1]}         # banana
echo ${fruits[@]}         # 전체
echo ${#fruits[@]}        # 원소 개수 (3)

# 배열 추가
fruits+=("date")
```

### 2-3. 환경 변수

```bash
# 환경 변수 = 자식 프로세스에도 전달되는 변수
# 일반 변수 = 현재 셸에서만 유효

name="local"      # 일반 변수 (자식 프로세스에 안 보임)
export NAME="env" # 환경 변수 (자식 프로세스에 보임)

# 또는
export name       # 기존 변수를 환경 변수로 승격

# 환경 변수 목록 확인
env
printenv
printenv PATH     # 특정 변수만

# 주요 환경 변수
$HOME      # 홈 디렉토리  (/home/alice)
$USER      # 현재 사용자 (alice)
$PATH      # 명령어 탐색 경로
$SHELL     # 현재 셸 (/bin/bash)
$PWD       # 현재 디렉토리
$OLDPWD    # 이전 디렉토리 (cd - 가 이걸 씀)
$?         # 직전 명령 종료 코드
$$         # 현재 프로세스 PID
$0         # 스크립트 이름
$1 $2 ...  # 스크립트 인자
$#         # 인자 개수
$@         # 모든 인자 (각각 따옴표)
```

PATH 동작 방식:

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# 명령어 실행 시 PATH 순서대로 탐색
# which 로 어디서 찾는지 확인
which python3   # /usr/bin/python3

# PATH에 경로 추가
export PATH="$PATH:/opt/myapp/bin"

# 영구 적용 → ~/.bashrc 또는 ~/.bash_profile 에 추가
echo 'export PATH="$PATH:/opt/myapp/bin"' >> ~/.bashrc
source ~/.bashrc
```

### 2-4. 특수 변수와 문자열 처리

```bash
# 변수 기본값
echo ${name:-"기본값"}    # name 비어있으면 "기본값" 출력
echo ${name:="기본값"}    # name 비어있으면 "기본값" 대입도 함

# 문자열 길이
str="hello world"
echo ${#str}              # 11

# 부분 문자열 (슬라이싱)
echo ${str:0:5}           # hello (0번째부터 5글자)
echo ${str:6}             # world (6번째부터 끝까지)

# 문자열 치환
echo ${str/hello/bye}     # bye world (첫 번째만)
echo ${str//l/L}          # heLLo worLd (전체)

# 접두사/접미사 제거
file="report_2025.log"
echo ${file%.log}         # report_2025 (끝에서 .log 제거)
echo ${file#report_}      # 2025.log (앞에서 report_ 제거)
```

---

## 3. 표준 입출력과 리다이렉션

### 3-1. 표준 스트림

```
모든 프로세스는 3개의 표준 스트림을 가짐:

fd 0  stdin   표준 입력  (키보드)
fd 1  stdout  표준 출력  (터미널)
fd 2  stderr  표준 에러  (터미널)

→ 5주차에서 배운 fd 그대로
```

### 3-2. 리다이렉션

```bash
# stdout → 파일 (덮어쓰기)
ls -la > output.txt

# stdout → 파일 (이어쓰기)
echo "추가 내용" >> output.txt

# stderr → 파일
ls /없는경로 2> error.txt

# stdout + stderr → 파일
ls /없는경로 > all.txt 2>&1
ls /없는경로 &> all.txt    # 위와 동일 (bash 단축)

# 출력 버리기 (/dev/null = 블랙홀)
ls /없는경로 2> /dev/null   # 에러 메시지 숨기기
command > /dev/null 2>&1    # 출력 전부 버리기

# stdin ← 파일
mysql -u root -p mydb < dump.sql

# Here Document — 여러 줄 입력을 stdin으로
cat << EOF
첫째 줄
둘째 줄
셋째 줄
EOF

# 스크립트에서 자주 쓰는 패턴:
cat << EOF > /etc/myapp/config.conf
host=192.168.1.100
port=8080
debug=false
EOF
```

### 3-3. 파이프

파이프 = 앞 명령의 stdout을 뒤 명령의 stdin으로 연결

```bash
# 기본
ls -la | grep ".log"
ps aux | grep httpd
cat /etc/passwd | wc -l

# 여러 개 연결
cat /var/log/messages | grep ERROR | sort | uniq -c | sort -rn

# 파이프 + 파일 동시에 (tee)
apt install httpd 2>&1 | tee install.log
# → 터미널에도 출력하면서 파일에도 저장

# xargs — 파이프 결과를 인자로 변환
find /tmp -name "*.log" | xargs rm -f
# find 결과를 rm의 인자로 넘김
# rm -f file1.log file2.log file3.log 와 동일

# 파이프에서 종료 코드
# 기본: 마지막 명령의 종료 코드만
echo $PIPESTATUS    # 각 명령의 종료 코드 배열
```

### 3-4. 명령 치환

```bash
# 명령 실행 결과를 변수에 저장
today=$(date +%Y%m%d)
hostname=$(hostname)
free_mem=$(free -m | awk '/^Mem:/ {print $4}')

echo "오늘 날짜: $today"
echo "호스트명: $hostname"
echo "가용 메모리: ${free_mem}MB"

# 파일명에 날짜 붙이기 (자동화에서 자주 씀)
backup_file="backup_$(date +%Y%m%d_%H%M%S).tar.gz"
tar -czf $backup_file /var/www/html/
```

---

## 4. 조건문

### 4-1. if 문

```bash
if [ 조건 ]; then
    명령
elif [ 조건 ]; then
    명령
else
    명령
fi
```

```bash
# 실제 예시
if [ $USER = "root" ]; then
    echo "root입니다"
else
    echo "일반 사용자입니다"
fi
```

### 4-2. 조건 표현식

#### 문자열 비교

```bash
[ "$a" = "$b" ]     # 같음
[ "$a" != "$b" ]    # 다름
[ -z "$a" ]         # 비어있음 (zero length)
[ -n "$a" ]         # 비어있지 않음 (non-zero)
```

#### 숫자 비교

```bash
[ $a -eq $b ]    # equal (==)
[ $a -ne $b ]    # not equal (!=)
[ $a -gt $b ]    # greater than (>)
[ $a -lt $b ]    # less than (<)
[ $a -ge $b ]    # greater or equal (>=)
[ $a -le $b ]    # less or equal (<=)
```

#### 파일/디렉토리 검사

```bash
[ -f "/etc/hosts" ]    # 파일 존재
[ -d "/var/log" ]      # 디렉토리 존재
[ -e "/tmp/lock" ]     # 존재 (파일/디렉토리 모두)
[ -r "/etc/hosts" ]    # 읽기 가능
[ -w "/tmp/test" ]     # 쓰기 가능
[ -x "/usr/bin/python3" ]  # 실행 가능
[ -s "/var/log/app.log" ]  # 크기 > 0
```

#### AND / OR

```bash
# -a (AND), -o (OR) — 구식, 비권장
[ $a -gt 0 -a $a -lt 10 ]

# && || — 권장
[ $a -gt 0 ] && [ $a -lt 10 ]

# [[ ]] — bash 확장 문법 (권장)
[[ $a -gt 0 && $a -lt 10 ]]
[[ $str =~ ^[0-9]+$ ]]    # 정규식 매칭 ([[ ]] 만 지원)
```

`[` vs `[[` 차이:

```bash
# [ ] = POSIX, /bin/[ 명령어
# [[ ]] = bash 내장, 더 안전하고 기능 많음

# [ ] 는 변수에 공백 있으면 문제
file="my file.txt"
[ -f $file ]     # [ -f my file.txt ] 로 해석 → 오류
[ -f "$file" ]   # 따옴표 필수

# [[ ]] 는 자동으로 처리
[[ -f $file ]]   # 따옴표 없어도 안전
```

### 4-3. case 문

```bash
case $변수 in
    패턴1)
        명령
        ;;
    패턴2|패턴3)    # OR 조건
        명령
        ;;
    *)              # default
        명령
        ;;
esac
```

```bash
# 실제 예시
read -p "계속할까요? (yes/no): " answer
case $answer in
    yes|y|Y)
        echo "계속합니다"
        ;;
    no|n|N)
        echo "중단합니다"
        exit 0
        ;;
    *)
        echo "yes 또는 no를 입력해주세요"
        exit 1
        ;;
esac

# 서비스 제어 스크립트에서
case $1 in
    start)   systemctl start myapp ;;
    stop)    systemctl stop myapp ;;
    restart) systemctl restart myapp ;;
    status)  systemctl status myapp ;;
    *)       echo "Usage: $0 {start|stop|restart|status}" ;;
esac
```

---

## 5. 반복문

### 5-1. for 문

```bash
# 리스트 순회
for item in apple banana cherry; do
    echo $item
done

# 배열 순회
servers=("web01" "web02" "db01")
for server in "${servers[@]}"; do
    echo "서버 확인: $server"
    ping -c 1 $server > /dev/null && echo "  → 응답" || echo "  → 무응답"
done

# C 스타일 (숫자 반복)
for ((i=1; i<=5; i++)); do
    echo "반복 $i"
done

# seq 사용
for i in $(seq 1 10); do
    echo $i
done

# 범위 표현
for i in {1..10}; do
    echo $i
done

for i in {0..100..10}; do    # 0, 10, 20 ... 100
    echo $i
done

# 파일/디렉토리 순회
for file in /var/log/*.log; do
    size=$(du -sh "$file" | cut -f1)
    echo "$file : $size"
done

# 명령 결과 순회
for pid in $(pgrep httpd); do
    echo "httpd PID: $pid"
done
```

### 5-2. while 문

```bash
# 조건이 참인 동안 반복
count=1
while [ $count -le 5 ]; do
    echo "카운트: $count"
    ((count++))
done

# 무한 루프 (서비스 모니터링 등)
while true; do
    if ! systemctl is-active --quiet httpd; then
        echo "$(date): httpd 다운 감지, 재시작 중..."
        systemctl restart httpd
    fi
    sleep 30
done

# 파일 한 줄씩 읽기
while IFS= read -r line; do
    echo "처리: $line"
done < /etc/hosts

# 파이프로도 가능
cat /etc/passwd | while IFS=: read -r user pass uid gid comment home shell; do
    echo "사용자: $user, 홈: $home"
done
```

### 5-3. until 문

```bash
# 조건이 거짓인 동안 반복 (while의 반대)
# 서비스가 뜰 때까지 대기하는 패턴에 유용
until systemctl is-active --quiet httpd; do
    echo "httpd 시작 대기 중..."
    sleep 2
done
echo "httpd 시작됨"
```

### 5-4. break / continue

```bash
# break: 루프 탈출
for i in {1..10}; do
    [ $i -eq 5 ] && break
    echo $i
done
# 1 2 3 4

# continue: 현재 반복 건너뜀
for i in {1..10}; do
    [ $((i % 2)) -eq 0 ] && continue
    echo $i
done
# 1 3 5 7 9
```

---

## 6. 함수

```bash
# 함수 정의
function greet() {
    echo "안녕하세요, $1 님"    # $1 = 첫 번째 인자
}

# 또는 (function 키워드 없이)
greet() {
    local name=$1              # local = 함수 안에서만 유효한 변수
    echo "안녕하세요, $name 님"
}

# 호출
greet "alice"
greet "bob"

# 반환값 — 종료 코드로 (0=성공, 1=실패)
is_root() {
    [ $EUID -eq 0 ]    # root면 0 반환
}

if is_root; then
    echo "root 권한으로 실행 중"
fi

# 값을 반환하려면 echo + 명령 치환
get_free_mem() {
    free -m | awk '/^Mem:/ {print $4}'
}
free=$(get_free_mem)
echo "가용 메모리: ${free}MB"
```

`local` 변수 중요성:

```bash
count=100   # 전역 변수

increment() {
    count=$((count + 1))    # 전역 변수 수정됨!
}

safe_increment() {
    local count=0           # 함수 내 지역 변수
    count=$((count + 1))    # 전역에 영향 없음
    echo $count
}
```

---

## 7. 스크립트 실용 패턴

### 7-1. 인자 처리

```bash
#!/bin/bash

# 사용법 출력 함수
usage() {
    echo "Usage: $0 [-h] [-v] [-f 파일명]"
    echo "  -h  도움말"
    echo "  -v  상세 출력"
    echo "  -f  처리할 파일"
    exit 1
}

# 인자 없으면 사용법 출력
[ $# -eq 0 ] && usage

# getopts — 옵션 파싱
verbose=false
filename=""

while getopts "hvf:" opt; do
    case $opt in
        h) usage ;;
        v) verbose=true ;;
        f) filename=$OPTARG ;;
        *) usage ;;
    esac
done

$verbose && echo "상세 모드 활성화"
[ -n "$filename" ] && echo "파일: $filename"
```

### 7-2. 오류 처리

```bash
#!/bin/bash
set -e          # 오류 발생 시 즉시 종료
set -u          # 정의되지 않은 변수 사용 시 오류
set -o pipefail # 파이프 중간 실패도 감지

# 또는 한 줄로
set -euo pipefail   # 스크립트 상단에 항상 써주는 게 좋음

# 오류 처리 함수
error() {
    echo "[ERROR] $1" >&2    # stderr로 출력
    exit 1
}

# 사용
[ -f "$config" ] || error "설정 파일 없음: $config"

# trap — 종료 시 정리 작업
cleanup() {
    rm -f /tmp/myapp.lock
    echo "정리 완료"
}
trap cleanup EXIT          # 스크립트 종료 시 항상 실행
trap cleanup INT TERM      # Ctrl+C, kill 시에도 실행

# 락 파일 패턴 (중복 실행 방지)
LOCKFILE=/tmp/myapp.lock
if [ -f "$LOCKFILE" ]; then
    error "이미 실행 중입니다 (lock: $LOCKFILE)"
fi
touch $LOCKFILE
trap "rm -f $LOCKFILE" EXIT
```

### 7-3. 로깅

```bash
#!/bin/bash

LOGFILE="/var/log/myapp/deploy.log"
mkdir -p "$(dirname $LOGFILE)"

log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a $LOGFILE
}

log INFO  "배포 시작"
log WARN  "설정 파일 없음, 기본값 사용"
log ERROR "데이터베이스 연결 실패"

# 실행 결과 로깅
run() {
    log INFO "실행: $*"
    if "$@" >> $LOGFILE 2>&1; then
        log INFO "완료: $*"
    else
        log ERROR "실패: $* (종료코드: $?)"
        return 1
    fi
}

run systemctl restart httpd
run dnf update -y
```

---

## 8. 실전 자동화 스크립트

### 8-1. 디스크 사용량 모니터링

```bash
#!/bin/bash
set -euo pipefail

THRESHOLD=80
ALERT_MAIL="admin@example.com"
LOGFILE="/var/log/disk_monitor.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> $LOGFILE
}

# df 출력 파싱
while IFS= read -r line; do
    # "Filesystem 1K-blocks Used Available Use% Mounted"
    usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
    mount=$(echo "$line" | awk '{print $6}')

    if [ "$usage" -ge "$THRESHOLD" ]; then
        msg="경고: $mount 사용률 ${usage}% (임계값: ${THRESHOLD}%)"
        log "$msg"
        echo "$msg" | mail -s "[DISK ALERT] $(hostname)" $ALERT_MAIL
    fi
done < <(df -h | tail -n +2)

log "디스크 점검 완료"
```

### 8-2. 서비스 헬스체크 + 자동 복구

```bash
#!/bin/bash
set -euo pipefail

SERVICES=("httpd" "mariadb" "firewalld")
LOGFILE="/var/log/healthcheck.log"
MAX_RESTART=3

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a $LOGFILE; }

check_and_restart() {
    local service=$1
    local restart_count_file="/tmp/.restart_${service}"

    if systemctl is-active --quiet "$service"; then
        return 0
    fi

    # 재시작 횟수 확인
    local count=0
    [ -f "$restart_count_file" ] && count=$(cat "$restart_count_file")

    if [ "$count" -ge "$MAX_RESTART" ]; then
        log "ERROR: $service 재시작 한도 초과 ($MAX_RESTART회), 수동 확인 필요"
        return 1
    fi

    log "WARN: $service 다운 감지, 재시작 시도 ($((count+1))/$MAX_RESTART)"
    systemctl restart "$service"
    echo $((count + 1)) > "$restart_count_file"
    log "INFO: $service 재시작 완료"
}

for svc in "${SERVICES[@]}"; do
    check_and_restart "$svc" || true
done
```

### 8-3. 백업 스크립트

```bash
#!/bin/bash
set -euo pipefail

BACKUP_SRC="/var/www/html /etc/httpd"
BACKUP_DEST="/backup"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DEST}/backup_${DATE}.tar.gz"
LOGFILE="/var/log/backup.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a $LOGFILE; }

# 백업 디렉토리 확인
mkdir -p "$BACKUP_DEST"

# 남은 디스크 공간 확인 (1GB 이상)
free_gb=$(df -BG "$BACKUP_DEST" | awk 'NR==2 {gsub("G",""); print $4}')
if [ "$free_gb" -lt 1 ]; then
    log "ERROR: 디스크 공간 부족 (${free_gb}GB)"
    exit 1
fi

# 백업 실행
log "INFO: 백업 시작 → $BACKUP_FILE"
tar -czf "$BACKUP_FILE" $BACKUP_SRC
log "INFO: 백업 완료 ($(du -sh $BACKUP_FILE | cut -f1))"

# 오래된 백업 삭제
log "INFO: ${RETENTION_DAYS}일 이전 백업 삭제"
find "$BACKUP_DEST" -name "backup_*.tar.gz" \
    -mtime +$RETENTION_DAYS -delete

log "INFO: 완료. 현재 백업 목록:"
ls -lh "$BACKUP_DEST"/backup_*.tar.gz >> $LOGFILE
```

---

## 9. 유용한 텍스트 처리 도구

셸 스크립트에서 자주 쓰는 도구들입니다.

### grep — 패턴 검색

```bash
grep "ERROR" /var/log/messages           # 포함된 줄
grep -v "DEBUG" /var/log/app.log         # 제외
grep -i "error" /var/log/app.log         # 대소문자 무시
grep -n "error" /var/log/app.log         # 줄 번호
grep -r "password" /etc/               # 재귀 탐색
grep -E "ERROR|WARN" /var/log/app.log    # 확장 정규식 (OR)
grep -c "ERROR" /var/log/app.log         # 매칭 줄 수
```

### awk — 열 기반 처리

```bash
# 특정 열 출력
df -h | awk '{print $1, $5}'          # 1번, 5번 열
awk -F: '{print $1}' /etc/passwd      # : 구분자, 1번 열 (사용자명)

# 조건 + 열
awk '$3 > 1000 {print $1}' /etc/passwd  # UID > 1000인 사용자
df -h | awk 'NR > 1 {print $5, $6}'    # 1번째 줄 제외

# 합계
free -m | awk '/^Mem:/ {print $3 "MB used of " $2 "MB"}'

# 구분자 변경 후 출력
echo "a:b:c" | awk -F: '{print $2}'    # b
```

### sed — 스트림 편집

```bash
# 치환
sed 's/old/new/' file.txt           # 각 줄 첫 번째만
sed 's/old/new/g' file.txt          # 전체
sed -i 's/old/new/g' file.txt       # 파일 직접 수정

# 줄 삭제
sed '/^#/d' config.txt              # # 으로 시작하는 줄 삭제
sed '/^$/d' config.txt              # 빈 줄 삭제

# 특정 줄만
sed -n '5,10p' file.txt             # 5~10번 줄만 출력

# 설정 파일 값 변경 (자동화에서 자주 씀)
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
sed -i "s/^Listen .*/Listen 8080/" /etc/httpd/conf/httpd.conf
```

### cut — 열 잘라내기

```bash
cut -d: -f1 /etc/passwd             # : 구분자, 1번 필드
cut -d: -f1,3 /etc/passwd           # 1번, 3번 필드
cut -c1-10 file.txt                 # 1~10번 문자
```

---

*문서 작성 기준: Rocky Linux 10 / Bash 5.2*
*`set -euo pipefail` 은 스크립트 상단 필수 습관*