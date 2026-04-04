> **원문:** Red Hat Enterprise Linux 10 — Monitoring and managing system status and performance
> 

> **범위:** Process의 개념과 Lifecycle, Process 조회/제어(ps, top, kill, nice, renice), cgroup v2의 구조와 Resource Isolation, systemd의 cgroup 통합, 성능 모니터링 도구(vmstat, iostat, sar), /proc Filesystem, TuneD Profile
> 

> **핵심 연결:** cgroup → K8s Pod Resource Limits, Steal Time → EC2 Burstable Instance, nice → Workload Priority 설계
> 

---

# 1. Process란 무엇인가 — Kernel이 프로그램을 실행하는 방식

## 1-1. Program vs Process vs Thread

이 세 개념의 구분이 Linux 시스템 관리의 출발점입니다.

**Program**은 Disk에 저장된 실행 가능한 파일입니다. `/usr/sbin/sshd`, `/usr/bin/python3` 같은 Binary 파일이 Program입니다. 그 자체로는 아무 일도 하지 않는 **정적인 코드 덩어리**입니다.

**Process**는 Program이 Memory에 로드되어 **실행 중인 Instance**입니다. 같은 Program에서 여러 개의 Process를 만들 수 있습니다. 예를 들어 `python3 app1.py`와 `python3 app2.py`는 같은 `/usr/bin/python3` Program이지만 서로 다른 두 개의 Process입니다. 각 Process는 고유한 **PID (Process ID)**, 독립된 **Memory 공간(Virtual Address Space)**, 고유한 **File Descriptor Table**을 가집니다.

**Thread**는 하나의 Process 내에서 실행되는 **경량 실행 단위**입니다. 같은 Process의 Thread들은 Memory 공간을 공유합니다. Linux Kernel은 내부적으로 Process와 Thread를 모두 **Task**라는 동일한 구조체(`task_struct`)로 관리합니다. `ps -eLf` 명령으로 Thread 단위까지 볼 수 있습니다.

## 1-2. fork()와 exec() — Process 생성의 원리

Linux에서 새로운 Process가 만들어지는 과정을 이해해야 `ps`의 PPID(Parent PID) 컬럼, Zombie Process, Orphan Process가 왜 발생하는지 이해할 수 있습니다.

**fork() System Call:**

Parent Process가 `fork()`를 호출하면 Kernel은 **Parent의 거의 완벽한 복사본**인 Child Process를 생성합니다. Child는 Parent의 Memory, File Descriptor, 환경 변수 등을 복사받습니다. 이 시점에서 Parent와 Child는 **동일한 코드**를 실행합니다. fork()의 리턴 값으로 Parent인지 Child인지 구분합니다 (Parent에게는 Child의 PID, Child에게는 0이 리턴됩니다).

현대 Linux에서는 실제로 전체 Memory를 복사하지 않고 **Copy-on-Write (COW)** 방식을 사용합니다. fork() 직후에는 Parent와 Child가 같은 Memory Page를 공유하다가, 한쪽이 수정할 때만 해당 Page를 복사합니다. 이것이 fork()가 빠른 이유입니다.

**exec() System Call:**

fork()로 만들어진 Child Process가 `exec()`를 호출하면, 현재 Process의 Memory를 **새로운 Program의 코드와 데이터로 완전히 교체**합니다. PID는 유지됩니다.

예를 들어, Shell에서 `ls` 명령을 실행하면:

1. Bash(Parent)가 `fork()`로 Child를 생성합니다.
2. Child가 `exec("/bin/ls")`를 호출하여 자신의 코드를 ls로 교체합니다.
3. ls가 실행되고 종료됩니다.
4. Parent(Bash)가 `wait()`으로 Child의 종료를 회수합니다.

이 fork-exec 모델은 Unix 계열 OS의 가장 핵심적인 설계입니다.

## 1-3. Process Lifecycle과 State

모든 Process는 아래의 State를 오가며, 최종적으로 종료됩니다.

| State | ps 코드 | 설명 | 발생 상황 |
| --- | --- | --- | --- |
| **Running** | `R` | CPU에서 실행 중이거나, Run Queue에서 CPU 할당을 대기 | 정상 실행 중 |
| **Sleeping (Interruptible)** | `S` | Event나 I/O를 대기. Signal로 깨울 수 있음 | 대부분의 Daemon이 이 상태 — 요청을 기다리는 중 |
| **Sleeping (Uninterruptible)** | `D` | I/O를 대기. Signal로 깨울 수 **없음** | Disk I/O 완료를 기다리는 중 |
| **Stopped** | `T` | SIGSTOP 또는 SIGTSTP로 일시 정지 | `Ctrl+Z`로 정지, 또는 Debugger가 붙은 상태 |
| **Zombie** | `Z` | 실행은 종료되었지만 Parent가 wait()으로 회수하지 않은 상태 | Parent가 바쁘거나 버그로 wait()을 안 부르는 경우 |

**D State가 SRE에게 중요한 이유:**

D State(Uninterruptible Sleep) Process는 `kill -9`로도 죽일 수 없습니다. Kernel이 I/O 완료를 보장하기 위해 Signal을 차단하기 때문입니다. D State Process가 쌓인다면 **Disk I/O에 심각한 병목이 있다**는 신호입니다.

> **AWS EBS 연결:** EC2 Instance에서 D State Process가 급증하면 **EBS Volume의 Latency가 비정상적으로 높아진 상황**을 의심해야 합니다. gp2 Volume의 Burst Credit이 소진되었거나, EBS-optimized가 아닌 Instance Type에서 Throughput 한계에 도달한 경우 발생합니다. CloudWatch의 `VolumeQueueLength` Metric이 급등하면 D State Process도 함께 증가합니다.
> 

**Zombie Process의 구조적 원인:**

Zombie Process는 Process Table에 Entry만 남아 있는 상태입니다. Memory나 CPU는 사용하지 않지만, PID를 점유합니다. 소수의 Zombie는 문제가 안 되지만, 대량으로 쌓이면 PID 고갈(`/proc/sys/kernel/pid_max`)이 발생하여 새 Process를 생성할 수 없게 됩니다.

해결 방법:

1. Parent Process에 SIGCHLD를 보내 wait()을 유도: `kill -SIGCHLD <PPID>`
2. Parent Process를 종료하면 Zombie의 Parent가 PID 1(systemd)로 바뀌고, systemd가 자동으로 회수합니다.

**Orphan Process:**

Parent가 먼저 종료되면 Child는 Orphan이 됩니다. Linux Kernel은 Orphan Process의 Parent를 **PID 1(systemd)**로 자동 재할당합니다. systemd는 이 Orphan들을 주기적으로 wait()하여 회수합니다. 이것이 SysVinit 대비 systemd의 장점 중 하나입니다 — SysVinit은 Orphan 회수에 취약했습니다.

---

# 2. Process 조회 — ps, top, 그리고 /proc

## 2-1. ps — Process의 정적 Snapshot

ps는 실행 순간의 Process 상태를 **스냅샷**으로 보여줍니다. 실시간 갱신이 아닙니다.

```bash
# BSD 스타일 — 가장 흔히 사용
ps aux

# UNIX(POSIX) 스타일 — PPID가 포함되어 부모-자식 관계 확인에 유리
ps -ef

# Process Tree 형태 — 부모-자식 계층 시각화
ps axjf

mr8356@mr8356:~$ ps axjf
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
      0       2       0       0 ?             -1 S        0   0:00 [kthreadd]
      2       3       0       0 ?             -1 S        0   0:00  \_ [pool_workqueue_release]
      2       4       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-rcu_gp]
      2       5       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-sync_wq]
      2       6       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-slub_flushwq]
      2       7       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-netns]
      2      10       0       0 ?             -1 I<       0   0:00  \_ [kworker/0:0H-events_highpri]
      2      12       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-mm_percpu_wq]
      2      14       0       0 ?             -1 I        0   0:00  \_ [rcu_tasks_kthread]
      2      15       0       0 ?             -1 I        0   0:00  \_ [rcu_tasks_rude_kthread]
      2      16       0       0 ?             -1 I        0   0:00  \_ [rcu_tasks_trace_kthread]

# 커스텀 출력 — 보고 싶은 필드만 선택
ps -eo pid,ppid,user,%cpu,%mem,stat,nice,cmd --sort=-%cpu | head -20

# 특정 사용자의 Process만
ps -u mr8356

# 특정 이름의 Process 찾기
ps aux | grep sshd
# 또는 pgrep 사용 (grep 자체가 결과에 포함되지 않음)
pgrep -a sshd
```

**ps aux 출력의 각 필드가 의미하는 것:**

| 필드 | 의미 | SRE 관점에서의 활용 |
| --- | --- | --- |
| `USER` | Process 소유자 | root로 실행되면 안 되는 앱이 root인지 확인 |
| `PID` | Process ID | kill, renice 등의 대상 지정 |
| `%CPU` | CPU 사용률 | 순간적인 값. 장기 추이는 top이나 sar로 |
| `%MEM` | 물리 Memory 사용률 | OOM Kill 위험 평가 |
| `VSZ` | Virtual Memory Size (KB) | Process가 **예약한** 전체 가상 Memory. 실제 사용량과 다름 |
| `RSS` | Resident Set Size (KB) | **실제로** 물리 Memory에 올라가 있는 크기. 진짜 메모리 사용량 |
| `STAT` | Process State | R/S/D/Z/T + 보조 플래그 (s=Session Leader, l=Multi-threaded 등) |
| `START` | 시작 시각 | Process가 얼마나 오래 실행되었는지 |
| `COMMAND` | 실행 명령어 | 어떤 인자로 실행되었는지 확인 |

**VSZ vs RSS 차이가 중요한 이유:**

VSZ는 Process가 **요청한** 가상 메모리 전체 크기입니다. 실제로 물리 RAM에 올라가 있지 않을 수 있습니다 (Memory-mapped File, 아직 접근하지 않은 영역 등). RSS는 **현재 물리 RAM에 실제로 올라가 있는** 크기입니다. OOM Killer는 RSS 기반으로 판단합니다.

Java Application에서 흔히 보는 현상: VSZ가 8GB이지만 RSS는 2GB. JVM이 `-Xmx8g`로 8GB를 예약했지만, 실제로 사용 중인 Heap은 2GB뿐인 상태입니다.

## 2-2. top — 실시간 Process 모니터링

```bash
# 기본 실행 — 3초마다 갱신
top

# 특정 사용자의 Process만
top -u mr8356

# Batch Mode — 스크립트에서 사용 (1회 출력)
top -bn1 | head -30

# 갱신 간격 변경 (0.5초)
top -d 0.5
```

**top 실행 중 단축키:**

| 키 | 동작 |
| --- | --- |
| `P` | CPU 사용률 순 정렬 |
| `M` | Memory 사용률 순 정렬 |
| `k` | Process Kill (PID 입력 프롬프트) |
| `r` | Process Renice (nice 값 변경) |
| `1` | CPU Core별 사용률 표시 (전체/Core별 토글) |
| `c` | 전체 Command Line 표시 |
| `H` | Thread 표시 모드 토글 |
| `f` | 표시 필드 선택 화면 |
| `q` | 종료 |

**top 헤더의 CPU 줄 해석 — SRE 필수 지식:**

```
%Cpu(s): 14.4 us,  0.2 sy,  0.0 ni, 85.0 id,  0.0 wa,  0.0 hi,  0.5 si,  0.0 st
```

| 필드 | 의미 | 정상 범위 | 이상 징후 |
| --- | --- | --- | --- |
| `us` (User) | 사용자 공간 CPU. 애플리케이션 연산. | 상황에 따라 다름 | 높으면 앱의 CPU-bound 작업이 많은 것 |
| `sy` (System) | Kernel 공간 CPU. System Call, I/O 처리. | 보통 5-10% 미만 | 20% 이상이면 과도한 System Call 또는 Context Switch |
| `ni` (Nice) | nice 값이 조정된 Process의 CPU | 낮음 | Batch Job이 CPU를 점유 중 |
| `id` (Idle) | 여유 CPU | 높을수록 좋음 | 0에 가까우면 CPU 부족 |
| `wa` (I/O Wait) | CPU가 I/O 완료를 기다리는 시간 | 5% 미만 | **10% 이상이면 Disk I/O 병목!** EBS Latency 의심 |
| `hi` (HW Interrupt) | Hardware Interrupt 처리 | 매우 낮음 | NIC Interrupt 등 |
| `si` (SW Interrupt) | Software Interrupt 처리 | 낮음 | Network 패킷 처리가 많을 때 상승 |
| `st` (Steal Time) | **가상화 환경에서 다른 VM에 빼앗긴 CPU** | 0% | **5% 이상이면 Instance 업그레이드 필요!** |

<img width="1120" height="739" alt="image" src="https://github.com/user-attachments/assets/09cee70a-8368-4791-aa64-f50834b54cc5" />
<img width="2048" height="1112" alt="Image" src="https://github.com/user-attachments/assets/b3fa296d-1c8c-4705-91d2-aee0eab92512" />
<img width="1894" height="1218" alt="Image" src="https://github.com/user-attachments/assets/8440cd6f-ee0a-48fe-b500-cbd7ab8e1145" />
<img width="1263" height="613" alt="Image" src="https://github.com/user-attachments/assets/fd9c1e4d-0a96-4305-964b-19cc2ac1d973" />

### 1. hi (Hardware Interrupt): "Top Half"

하드웨어 장치가 CPU에 물리적 신호를 보내 즉각적인 처리를 요구할 때 발생하는 점유율입니다.

- **트리거(Trigger):** NIC(랜카드), 디스크 컨트롤러, 키보드 등의 하드웨어가 CPU의 **IRQ(Interrupt Request) 라인**에 전기 신호를 보냅니다.
- **커널 동작:** 1. CPU는 실행 중이던 명령을 즉시 중단하고 **IDT(Interrupt Descriptor Table)**를 참조하여 해당 하드웨어의 **ISR(Interrupt Service Routine)**을 실행합니다.
2. 이 단계에서는 다른 인터럽트가 발생하지 못하도록 차단(Masking)되는 경우가 많습니다.
3. 따라서 하드웨어로부터 데이터를 메모리로 옮기는 등 **최소한의 필수 작업**만 수행하고 즉시 종료해야 합니다.
- **상승 요인:** 하드웨어 자체의 입출력이 극심할 때(예: 초당 수만 개의 네트워크 패킷 유입) 상승합니다. 하지만 워낙 순식간에 끝나기 때문에 보통은 매우 낮게 유지됩니다.

---

### 2. si (Software Interrupt): "Bottom Half" (SoftIRQ)

하드웨어 인터럽트 처리 중 "지금 당장 안 해도 되는 무거운 작업"을 뒤로 미뤄서 처리할 때 발생하는 점유율입니다.

- **트리거(Trigger):** 하드웨어 인터럽트(hi) 처리 직후, 커널이 "나머지 복잡한 계산은 나중에 할게"라며 예약한 작업을 실행할 때 발생합니다.
- **커널 동작:**
    1. `hi` 단계에서 데이터를 메모리에 적재했다면, `si` 단계에서는 그 데이터를 분석합니다
    2. 네트워크를 예로 들면, 패킷을 읽어와서 **TCP/IP 프로토콜 스택을 통과시키고 체크섬을 확인하는 등 CPU 연산이 많이 필요한 작업**이 여기서 일어납니다.
    3. 이 작업은 **`ksoftirqd`**라는 커널 스레드(아까 보신 `kthreadd`의 자식)에 의해 처리되거나, 인터럽트 종료 직후 커널 컨텍스트에서 처리됩니다.
- **상승 요인:** 네트워크 트래픽이 많을 때 가장 흔하게 상승합니다. 패킷 하나하나를 분석하고 라우팅하는 과정이 전부 `si` 점유율로 잡히기 때문입니다.

> **AWS EC2 Steal Time 딥다이브:** `st`(Steal Time)는 가상화 Hypervisor가 이 VM의 CPU를 다른 VM에 할당한 시간의 비율입니다. 물리 호스트가 과부하 상태이거나, **Burstable Instance(t2, t3, t4g)**에서 CPU Credit이 소진되면 발생합니다.
> 

> `t3.micro`는 기본 CPU 성능이 10%입니다. Credit이 있을 때는 100%까지 Burst 가능하지만, Credit이 0이 되면 10%로 Throttle됩니다. 이때 `st` 값이 급등합니다. CloudWatch의 `CPUCreditBalance` Metric이 0에 수렴하면 성능 급감을 예상해야 합니다.
> 

> 해결: Credit이 자주 소진되면 `t3.unlimited` 모드를 켜거나(추가 비용 발생), `m5`/`c5` 같은 Fixed Performance Instance로 전환해야 합니다.
> 

> **K8s 연결:** K8s Node에서 `top`의 `wa`가 높으면 해당 Node의 모든 Pod가 I/O 병목의 영향을 받습니다. `kubectl top nodes`로는 `wa`를 볼 수 없습니다 — 반드시 Node에 SSH하여 `top`으로 확인해야 합니다. `st`가 높으면 Node Pool의 Instance Type을 재검토해야 합니다.
> 

## 2-3. /proc Filesystem — Kernel이 노출하는 Process 정보

`/proc`는 디스크에 실제 파일이 존재하지 않는 **가상 Filesystem**입니다. Kernel이 실시간으로 생성하는 Process와 시스템 정보를 파일 인터페이스로 노출합니다. `ps`와 `top`도 내부적으로 `/proc`를 읽어서 정보를 표시합니다.

```bash
# 특정 Process의 상세 상태
cat /proc/<PID>/status

# Process의 Command Line 인자
cat /proc/<PID>/cmdline | tr '\0' ' '

# Process의 환경 변수 (보안상 중요)
cat /proc/<PID>/environ | tr '\0' '\n'

# Process의 Memory Map
cat /proc/<PID>/maps | head -20

# Process의 Open File Descriptor 목록
ls -la /proc/<PID>/fd/
mr8356@mr8356:~$ ls -la /proc/1925/fd/
total 0
dr-x------. 2 mr8356 mr8356  4 Mar 28 22:03 .
dr-xr-xr-x. 9 mr8356 mr8356  0 Mar 28 22:03 ..
lrwx------. 1 mr8356 mr8356 64 Mar 28 22:03 0 -> /dev/pts/0
lrwx------. 1 mr8356 mr8356 64 Mar 28 23:16 1 -> /dev/pts/0
lrwx------. 1 mr8356 mr8356 64 Mar 28 23:16 2 -> /dev/pts/0
lrwx------. 1 mr8356 mr8356 64 Mar 28 23:16 255 -> /dev/pts/0

# Open File Descriptor 수 확인
ls /proc/<PID>/fd | wc -l

# 시스템 전체 Memory 정보
cat /proc/meminfo
mr8356@mr8356:~$ cat /proc/meminfo
MemTotal:        3661624 kB
MemFree:         2900728 kB
MemAvailable:    3212464 kB
Buffers:            4928 kB
Cached:           421604 kB
SwapCached:            0 kB
Active:           287752 kB
Inactive:         247480 kB
Active(anon):     119080 kB
Inactive(anon):        0 kB
Active(file):     168672 kB

# CPU 정보
mr8356@mr8356:~$ cat /proc/cpuinfo
processor	: 0
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint
CPU implementer	: 0x61
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0x000
CPU revision	: 0

processor	: 1
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm ssbs sb paca pacg dcpodp flagm2 frint
CPU implementer	: 0x61
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0x000
CPU revision	: 0

# 시스템 Load Average
cat /proc/loadavg

mr8356@mr8356:~$ cat /proc/loadavg
0.00 0.00 0.00 1/242 2772
```

**File Descriptor 모니터링이 SRE에게 중요한 이유:**

모든 Network Connection, Open File, Pipe 등은 File Descriptor를 소비합니다. Process당 열 수 있는 File Descriptor 수에는 Limit이 있습니다(`ulimit -n`, 기본 보통 1024). 고트래픽 웹 서버나 DB에서 동시 연결이 많아지면 "Too many open files" 오류가 발생합니다.

```bash
# 현재 Shell의 File Descriptor Limit 확인
ulimit -n

# 시스템 전체 Limit 확인
cat /proc/sys/fs/file-max

# 특정 Service의 Limit을 systemd Unit File에서 설정
# [Service] 섹션에 추가:
# LimitNOFILE=65535
```

> **실무:** Nginx, PostgreSQL, Redis 등의 systemd Unit File에는 거의 반드시 `LimitNOFILE=65535` 이상이 설정되어 있습니다. 이 값을 확인하지 않고 배포하면 Production에서 갑자기 연결 거부가 발생할 수 있습니다.
> 

---

# 3. Process 제어 — Signal, kill, Graceful Shutdown

## 3-1. Signal의 개념

Signal은 Kernel이나 다른 Process가 **특정 Process에게 보내는 비동기 알림**입니다. Process는 Signal을 받으면 미리 등록된 Signal Handler를 실행하거나, 기본 동작(Default Action)을 수행합니다.

| Signal | 번호 | Default Action | 설명 | catch 가능 여부 |
| --- | --- | --- | --- | --- |
| `SIGHUP` | 1 | 종료 | 원래는 Terminal 연결 끊김. 현재는 **설정 Reload** 용도로 관례적 사용 | O |
| `SIGINT` | 2 | 종료 | `Ctrl+C`가 보내는 Signal | O |
| `SIGQUIT` | 3 | Core Dump + 종료 | `Ctrl+\`가 보내는 Signal | O |
| `SIGKILL` | 9 | 강제 종료 | **catch/무시 불가.** Kernel이 직접 Process를 제거 | **X** |
| `SIGTERM` | 15 | 종료 | Graceful Shutdown 요청. **기본 kill Signal** | O |
| `SIGSTOP` | 19 | 일시 정지 | **catch/무시 불가.** Process 실행을 정지 | **X** |
| `SIGCONT` | 18 | 재개 | SIGSTOP으로 정지된 Process를 재개 | O |
| `SIGUSR1` | 10 | 종료 | 사용자 정의 Signal. 앱마다 용도가 다름 | O |
| `SIGUSR2` | 12 | 종료 | 사용자 정의 Signal | O |
| `SIGCHLD` | 17 | 무시 | Child Process가 종료되면 Parent에게 전달 | O |

**SIGTERM vs SIGKILL — 반드시 구분해야 하는 이유:**

`SIGTERM`(15): Process에게 "종료 준비를 해라"라는 **요청**입니다. Process는 Signal Handler에서 DB Connection 정리, 임시 파일 삭제, 로그 Flush 등의 **정리 작업(Cleanup)**을 수행한 후 스스로 종료합니다. 잘 만들어진 앱은 SIGTERM을 받으면 30초 이내에 Graceful Shutdown을 완료합니다.

`SIGKILL`(9): Kernel이 **즉시** Process를 제거합니다. Process는 어떤 정리 작업도 할 수 없습니다. 열린 File Descriptor가 정리되지 않고, 임시 파일이 남고, DB Transaction이 중단됩니다.

**올바른 순서: 항상 SIGTERM 먼저, SIGKILL은 최후의 수단**

```bash
# 1단계: Graceful Shutdown 요청 (SIGTERM)
kill 12345        # SIGTERM이 기본
kill -15 12345    # 명시적으로 SIGTERM

# 2단계: 잠시 대기 (10-30초)
sleep 10

# 3단계: 아직 살아있으면 강제 종료
kill -9 12345     # SIGKILL
```

> **K8s 직접 연결:** `kubectl delete pod`은 정확히 이 순서를 따릅니다.
> 

> 1. Container에 SIGTERM을 전송합니다.
> 

> 2. `terminationGracePeriodSeconds` (기본 30초) 동안 기다립니다.
> 

> 3. 기한 내에 종료되지 않으면 SIGKILL을 전송합니다.
> 

> 3-2. kill, killall, pkill
> 

> 앱이 SIGTERM을 처리하지 않으면 매번 SIGKILL로 강제 종료되어 데이터 손실 위험이 있습니다. **Production 앱은 반드시 SIGTERM Handler를 구현**해야 합니다. `terminationGracePeriodSeconds`를 앱의 Cleanup 시간에 맞게 조정하는 것이 Graceful Shutdown 설계의 핵심입니다.
> 

## 3-2. kill, killall, pkill

```bash
# PID로 Signal 전송
kill <PID>              # SIGTERM (기본)
kill -SIGHUP <PID>      # 설정 Reload (nginx, Apache 등)
kill -9 <PID>           # SIGKILL (강제 종료)

# 이름으로 모든 Process에 Signal 전송
killall httpd           # httpd라는 이름의 모든 Process에 SIGTERM

# 패턴 매칭으로 Process에 Signal 전송
pkill -f "python.*myapp"   # Command Line에서 패턴 매칭
pkill -u baduser           # 특정 사용자의 모든 Process 종료

# Signal 목록 확인
kill -l
```

---

# 4. Process Priority — nice, renice, Scheduling 정책

## 4-1. Linux CFS Scheduler와 nice 값

Linux Kernel은 **CFS (Completely Fair Scheduler)**를 사용하여 CPU 시간을 Process들에게 분배합니다. CFS는 각 Process에게 "공정하게" CPU를 나눠주되, **nice 값**으로 가중치를 조절할 수 있습니다.

| 항목 | 설명 |
| --- | --- |
| nice 값 범위 | **-20** (최고 Priority) ~ **+19** (최저 Priority) |
| 기본값 | 0 |
| nice를 낮추기 (Priority 올리기) | root 권한 필요 (`-20 ~ -1`) |
| nice를 높이기 (Priority 내리기) | 일반 사용자도 가능 (`0 ~ +19`) |

nice 값의 차이 1은 CPU 가중치에서 약 **10%의 차이**를 만듭니다. nice 0인 Process와 nice 19인 Process가 CPU를 경쟁하면, nice 0이 압도적으로 많은 CPU 시간을 받습니다.

**Linux Kernel은 nice 값을 내부적으로 Priority(PR) 값으로 변환합니다:**

```
PR = 20 + nice
```

- nice -20 → PR 0 (가장 높은 일반 Priority)
- nice 0 → PR 20 (기본)
- nice 19 → PR 39 (가장 낮은 일반 Priority)

## 4-2. nice와 renice 실습

```bash
# 현재 Process들의 nice 값 확인
ps -eo pid,ni,pri,cmd --sort=-ni | head -20

# 낮은 Priority(높은 nice)로 프로그램 실행
nice -n 10 /opt/scripts/batch-job.sh

# 높은 Priority(낮은 nice)로 프로그램 실행 (root 필요)
sudo nice -n -5 /opt/critical-app/server

# 실행 중인 Process의 nice 값 변경
sudo renice -n 15 -p <PID>

# 특정 사용자의 모든 Process nice 값 변경
sudo renice -n 10 -u mr8356

# top에서도 'r' 키로 renice 가능
```

> **이전 Prometheus 스터디 연결:** Avalanche(부하 생성기)가 CPU 22.6%를 잡아먹었던 상황에서, `nice -n 19`로 실행하면 Prometheus와 CPU를 경쟁할 때 Avalanche의 Priority가 최저가 됩니다. CPU 여유가 있을 때는 둘 다 정상 실행되지만, CPU가 부족해지면 Prometheus가 우선적으로 CPU를 할당받아 모니터링 안정성을 보장합니다.
> 

## 4-3. Real-time Scheduling (RT Priority)

nice 값 외에 Linux에는 **Real-time Scheduling Policy**가 존재합니다. RT Process는 일반 Process보다 **항상** 먼저 CPU를 받습니다.

| Policy | 설명 |
| --- | --- |
| `SCHED_FIFO` | First-In-First-Out. 같은 Priority 내에서 먼저 온 Process가 먼저 실행. |
| `SCHED_RR` | Round-Robin. 같은 Priority 내에서 Time Slice만큼 돌아가며 실행. |
| `SCHED_OTHER` | 기본 CFS. nice 값 사용. |

```bash
# Process의 Scheduling Policy 확인
chrt -p <PID>

# Real-time Priority로 실행 (주의: 잘못 사용하면 시스템 먹통)
sudo chrt -f 50 /opt/critical-app/realtime-worker
```

---

# 5. cgroup (Control Group) — Resource Isolation의 핵심

## 5-1. cgroup이란 무엇이고, 왜 존재하는가

nice 값은 CPU **Priority**를 조절할 뿐, **절대적인 사용량 제한**은 할 수 없습니다. nice 19인 Process도 다른 Process가 없으면 CPU 100%를 사용합니다.

**cgroup**은 Process 그룹의 **CPU, Memory, I/O, Network** 사용량을 **절대값으로 제한**하는 Linux Kernel 기능입니다. "이 Process 그룹은 최대 CPU 50%까지, Memory 512MB까지만 사용 가능하다"를 강제할 수 있습니다.

RHEL 10은 **cgroup v2**를 사용합니다. cgroup v1과의 핵심 차이는 **단일 계층 구조(Unified Hierarchy)**입니다. v1에서는 CPU, Memory, I/O 각각의 Controller가 별도의 계층을 가졌지만, v2에서는 하나의 통합된 계층에서 모든 Controller를 관리합니다.

**cgroup이 중요한 이유 — 이것 없이는 Container도, K8s도 없습니다:**

| 기술 | cgroup의 역할 |
| --- | --- |
| **systemd** | 모든 Service를 cgroup으로 추적. fork()한 자식까지 포함. PID 파일 없이도 Service의 모든 Process를 정확히 파악 |
| **Docker/Podman** | Container = **cgroup(Resource 제한) + Namespace(격리)**. `docker run --memory=512m`은 cgroup Memory Limit 설정 |
| **Kubernetes** | Pod의 `resources.requests/limits`는 kubelet이 cgroup으로 변환하여 적용 |

## 5-2. systemd의 cgroup 계층 구조

systemd는 모든 Process를 **Slice → Scope/Service** 계층으로 조직합니다.

```
/ (root cgroup)
├── init.scope                    ← systemd 자체 (PID 1)
├── system.slice                  ← systemd가 관리하는 Service들
│   ├── sshd.service
│   ├── httpd.service
│   ├── kubelet.service
│   └── myapp.service
├── user.slice                    ← 사용자 Session들
│   ├── user-1000.slice           ← UID 1000 사용자
│   │   └── session-1.scope       ← 로그인 세션
│   └── user-0.slice              ← root 사용자
└── machine.slice                 ← VM/Container들
    └── container-xyz.scope
```

- **Slice**: cgroup 계층의 **단위**. 자원 할당의 그룹입니다.
- **Scope**: systemd가 만들지 않았지만 **외부에서 생성된 Process 그룹**. 사용자 Session, Container 등.
- **Service**: systemd Unit File로 정의된 Service의 Process 그룹.

```bash
# 현재 cgroup 계층 트리 확인
sudo systemd-cgls

mr8356@mr8356:~$ sudo systemd-cgls
[sudo] password for mr8356:
CGroup /:
-.slice
├─user.slice
│ └─user-1000.slice
│   ├─user@1000.service …
│   │ └─init.scope
│   │   ├─1908 /usr/lib/systemd/systemd --user
│   │   └─1910 (sd-pam)
│   └─session-1.scope
│     ├─1902 sshd-session: mr8356 [priv]
│     ├─1924 sshd-session: mr8356@pts/0
│     ├─1925 -bash
│     ├─2786 sudo systemd-cgls
│     ├─2790 sudo systemd-cgls
│     ├─2791 systemd-cgls
│     └─2792 less

# Slice별 실시간 Resource 사용량 (top과 유사하지만 cgroup 단위)
sudo systemd-cgtop

# 특정 Service의 cgroup 정보
sudo systemctl show myapp.service | grep -i cgroup

# 특정 Service가 사용하는 cgroup 경로
sudo systemctl show myapp.service -p ControlGroup
```

## 5-3. Service에 Resource 제한 설정하기

systemd Unit File의 `[Service]` Section에서 cgroup Resource Directive를 설정합니다.

```bash
# Drop-in Override로 Resource 제한 추가
sudo systemctl edit myapp.service

# 에디터에 입력:
# [Service]
# CPUQuota=50%
# MemoryMax=512M
# MemoryHigh=400M
# IOWeight=50
# TasksMax=256

sudo systemctl daemon-reload
sudo systemctl restart myapp.service

# 적용 확인
sudo systemctl show myapp.service | grep -E 'CPUQuota|Memory|IO|Tasks'
```

**Resource Directive 상세:**

| Directive | 설명 | 초과 시 동작 |
| --- | --- | --- |
| `CPUQuota=` | CPU 사용량 제한. 100% = 1 Core. 200% = 2 Core. | Throttle (느려짐) |
| `CPUWeight=` | CPU 상대적 가중치 (1-10000, 기본 100) | 경쟁 시 비율대로 분배 |
| `MemoryMax=` | Memory **Hard Limit**. 절대 초과 불가. | **OOM Kill** |
| `MemoryHigh=` | Memory **Soft Limit**. 초과 시 Reclaim 압력. | Throttle (Memory 회수 시도) |
| `MemoryLow=` | Memory **보호 최소값**. 이 이하로는 회수하지 않음. | 보호됨 |
| `IOWeight=` | I/O 상대적 가중치 (1-10000, 기본 100) | 경쟁 시 비율대로 분배 |
| `IODeviceLatencyTargetSec=` | 특정 Device의 I/O Latency 목표 | Throttle |
| `TasksMax=` | 최대 Process/Thread 수 | fork() 실패 |

**MemoryMax vs MemoryHigh — 결정적 차이:**

`MemoryMax`는 Hard Limit입니다. Process가 이 값을 초과하면 Kernel의 **OOM Killer**가 해당 cgroup의 Process를 즉시 Kill합니다. 데이터 손실 위험이 있습니다.

`MemoryHigh`는 Soft Limit입니다. 초과하면 Kernel이 해당 cgroup에 **Memory 회수 압력(Reclaim Pressure)**을 가합니다. Process가 즉시 죽지는 않지만 느려집니다. Page Cache를 반환하도록 유도합니다.

Best Practice: `MemoryHigh`를 `MemoryMax`의 80% 정도로 설정하여, Hard Limit에 도달하기 전에 Soft Limit에서 먼저 속도 조절이 이루어지도록 합니다.

> **K8s Resource Limits ↔ cgroup 직접 매핑:**
> 

> K8s Pod Spec에서 설정하는 `resources`는 kubelet이 **정확히** 아래의 cgroup Directive로 변환합니다:
> 

> K8s에서 Pod이 `OOMKilled` 상태로 재시작되는 것은 cgroup `MemoryMax`를 초과한 것입니다. `kubectl describe pod`에서 `Last State: OOMKilled`가 보이면, `limits.memory`를 올리거나 앱의 Memory 사용을 줄여야 합니다.
> 

> 
> 

> CPU `limits`는 초과해도 죽지 않고 **Throttle**됩니다. CPU Throttle이 발생하면 요청 Latency가 급증합니다. K8s에서 CPU Throttle은 `/sys/fs/cgroup/cpu.stat`의 `nr_throttled`, `throttled_usec` 값으로 확인합니다. Production에서 CPU limits를 너무 타이트하게 잡으면 Throttle로 인한 Latency 문제가 발생합니다.
> 

---

# 6. 성능 모니터링 도구

## 6-1. vmstat — CPU, Memory, I/O, Swap 종합 모니터링

```bash
# 1초 간격으로 5회 출력
vmstat 1 5
```

**출력 필드 해석:**

| 카테고리 | 필드 | 의미 | 이상 징후 |
| --- | --- | --- | --- |
| **Procs** | `r` | Run Queue 대기 Process 수 | CPU Core 수보다 높으면 CPU 부족 |
|  | `b` | Uninterruptible Sleep(D State) Process 수 | 1 이상이면 I/O 병목 의심 |
| **Memory** | `free` | 여유 Memory (KB) | 지속적 감소 시 Memory Leak 의심 |
|  | `buff` | Buffer Cache (KB) | Block Device I/O 캐시 |
|  | `cache` | Page Cache (KB) | File System 캐시 |
| **Swap** | `si` | Swap In (KB/s) | **0이 아니면 Memory 부족!** |
|  | `so` | Swap Out (KB/s) | **0이 아니면 Memory 부족!** |
| **IO** | `bi` | Block In (KB/s) | Disk Read |
|  | `bo` | Block Out (KB/s) | Disk Write |
| **CPU** | `us` | User CPU % | 앱 CPU 사용 |
|  | `sy` | System CPU % | Kernel CPU 사용 |
|  | `wa` | I/O Wait % | **높으면 Disk 병목** |
|  | `st` | Steal % | **높으면 VM CPU 빼앗김** |

> **핵심 판단 기준:** `si`/`so`(Swap)가 지속적으로 0이 아니면 **물리 Memory가 부족**한 것입니다. Swap은 Disk이므로 Memory 대비 수백 배 느립니다. Swap을 사용하는 순간 성능이 급감합니다. EC2에서는 Instance Type의 Memory를 업그레이드하거나, 앱의 Memory 사용을 줄여야 합니다.
> 

## 6-2. iostat — Disk I/O 상세 모니터링

```bash
# 확장 Disk 통계 1초 간격 (유휴 Device 제외)
iostat -dxz 1

# 특정 Device만
iostat -dxz /dev/nvme0n1 1
```

이전 Prometheus 스터디에서 WAL 튜닝 시 사용했던 명령어입니다. `await`(평균 I/O 대기 시간), `%util`(Device 사용률), `w/s`(초당 Write 수)를 중점적으로 봅니다.

## 6-3. sar — 장기 성능 데이터 수집

sar는 `sysstat` 패키지에 포함된 **장기 성능 데이터 수집/조회 도구**입니다. 10분마다 시스템 성능 데이터를 `/var/log/sa/` 디렉터리에 자동으로 기록합니다.

```bash
# sysstat 설치 (첫 스터디에서 설치함)
sudo dnf install sysstat -y
sudo systemctl enable --now sysstat

# CPU 통계 (1초 간격, 5회)
sar -u 1 5

# Memory 통계
sar -r 1 5

# Disk I/O 통계
sar -d 1 5

# Network 통계
sar -n DEV 1 5

# 과거 데이터 조회 — 어제의 CPU 통계
sar -u -f /var/log/sa/sa$(date -d yesterday +%d)

# 특정 시간대만 조회
sar -u -s 10:00:00 -e 12:00:00
```

> **SRE 실무:** "어제 오후 3시쯤 서비스가 느려졌다는 Report가 있는데, 원인을 찾아라" — 이 요청에 대응하려면 과거 데이터가 필요합니다. sar는 자동으로 수집하므로 `sar -u -s 15:00:00 -e 16:00:00 -f /var/log/sa/sa<어제날짜>`로 해당 시간대의 CPU/Memory/IO를 확인할 수 있습니다.
> 

## 6-4. TuneD — 시스템 성능 Profile 자동 적용

RHEL에는 **TuneD**라는 성능 튜닝 Daemon이 포함되어 있습니다. 미리 정의된 Profile을 적용하여 Workload 유형에 맞게 Kernel Parameter, Disk Scheduler, Power Management 등을 일괄 조정합니다.

```bash
# TuneD 설치 및 시작
sudo dnf install tuned -y
sudo systemctl enable --now tuned

# 사용 가능한 Profile 목록
tuned-adm list

# 현재 활성 Profile 확인
tuned-adm active

# Profile 적용
sudo tuned-adm profile throughput-performance

# 추천 Profile 확인 (시스템 환경 자동 분석)
tuned-adm recommend
```

**주요 Profile:**

| Profile | 용도 | 특성 |
| --- | --- | --- |
| `balanced` | 기본. 성능과 절전의 균형 | 대부분의 상황에 적합 |
| `throughput-performance` | **서버 워크로드** | CPU Governor를 performance로, I/O Scheduler 최적화 |
| `latency-performance` | **Low Latency 필요** | 절전 기능 비활성화, C-State 제한 |
| `virtual-guest` | **VM/EC2 환경** | 가상화 환경에 최적화 |
| `network-latency` | Network Latency 최소화 | TCP Buffer 조정, IRQ Balancing |

> **AWS EC2 연결:** EC2 Instance에서는 `virtual-guest` Profile이 권장됩니다. `tuned-adm recommend`를 실행하면 가상화 환경을 자동 감지하여 이 Profile을 추천합니다.

