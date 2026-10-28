# Linux에서 DB·시스템 소프트웨어 성능을 측정하는 방법

조사 기준일: **2026-10-28**. Linux 호스트의 sudo 권한이 있고, 필요한 커널 설정·모듈·빌드까지 바꿀 수 있는 개발 환경을 가정한다. 특정 배포판·CPU·DBMS에 한정하지 않는다. 이 문서는 공식 문서와 도구의 원본 소스를 바탕으로 구성한 **진단 절차와 도구 선택 가이드**다. 현재 서버에 도구를 설치하거나 커널을 변경하거나 벤치마크를 실행한 결과는 아니다.

**먼저 증상과 완료 작업량을 정의하고, 호스트 오류와 자원 상태를 확인한다.** DB 내부 통계와 `pidstat`·`iostat`·소수 이벤트의 `perf stat`을 이용해 CPU 작업 증가, CPU 효율 저하, 대기 증가를 구분한다. CPU 호출 경로가 필요하면 `perf record`로 이어간다. 이 선택 순서와 오버헤드 평가는 실무적 판단이며, 도구 간 실측 순위가 아니다.

[가설과 실험으로 원인을 좁히는 예제](2026-10-28-performance-investigation-walkthrough.md)는 같은 완료량의 비용 비교, 반증 조건, 요청과 OS 측정의 연결, 결론 작성까지 보여 준다. 예제 수치는 설명용으로 구성했다.

소스를 수정하고 다시 빌드할 수 있다면 [Linux C++ 디버깅·성능 실험 가이드](2026-10-28-cpp-debug-benchmark-workflow.md)로 이어간다. 빌드 구성, GDB·sanitizer, 함수 벤치마크, 직접 계측과 코드 변경 전후의 검증 절차를 다룬다.

## 1. 먼저 구분할 측정 목적

| 알고 싶은 것 | 먼저 볼 지표·도구 | 다음 단계 | DB 개발에서의 예 |
| --- | --- | --- | --- |
| 변경 후 실제로 느려졌는가 | 고정 작업량의 경과시간, 처리량, p50·p95·p99, 오류 수 | 반복 A/B, 동일 데이터·빌드·실행계획 확인 | 같은 SELECT 100만 건의 서버 비용과 사용자 응답시간 비교 |
| CPU가 더 많은 일을 하는가 | `perf stat`의 instructions, task-clock, cycles | `perf record`, `perf annotate`, DB 함수별 계수 | 행 디코딩·복사·압축·해시 연산 증가 |
| 명령 수는 비슷한데 CPU 시간이 늘었는가 | IPC, branch miss, CPU별 PMU 지표 | Top-down, `perf mem`, PCM·uProf | 포인터 추적, 캐시·TLB miss, NUMA 원격 접근 |
| CPU 사용률이 낮은데 응답이 늦는가 | DB wait event, off-CPU stack, run queue 지연 | `perf sched`, BCC `offcputime`·`runqlat`, PSI | latch·I/O·네트워크 대기와 CPU 배정 대기 구분 |
| 스레드를 늘려도 처리량이 늘지 않는가 | 스레드별 CPU, 대기, cacheline contention | `perf c2c`, 사용자 lock 계측, 메모리 대역폭 | 공용 buffer counter, allocator, log mutex 병목 |
| 저장장치 또는 파일시스템이 병목인가 | `iostat`, DB physical I/O·WAL 통계 | `biolatency`, `biosnoop`, block·FS trace | checkpoint writeback, fsync tail, random read |
| 메모리가 커지거나 메모리 압박이 있는가 | RSS·PSS, cgroup memory, PSI, reclaim | heaptrack·allocator 통계·DAMON·NUMA 통계 | 누수, fragmentation, buffer pool 상주량 구분 |
| 가끔만 긴 지연이 발생하는가 | 요청별 latency와 같은 구간의 OS·DB 지표 | 이벤트 trace, scheduler·IRQ trace, rtla | commit p99와 writeback·scheduler 지연 상관 확인 |

도구 기능의 근거는 3절의 각 행에 연결했다. 위 표의 도구 조합은 관측 대상을 좁히기 위한 제안이다.

### 1.1 도구를 실행하기 전에 정할 조사 순서

1. **문제를 수치로 쓴다.** 어느 입력·동시성에서 어떤 지표가 얼마나 변했는지 적는다. 기준선과 후보의 정확성·성공 완료량을 먼저 맞춘다.
2. **호스트 상태를 확인한다.** 5.1절의 kernel 오류, 메모리, CPU별 상태와 I/O를 본다. 요청 오류·retry·timeout도 같은 구간에서 확인한다.
3. **자원과 thread 시간을 나누어 본다.** 아래 USE 표를 채우고, 요청을 처리하는 thread의 실행·CPU 배정 대기·blocking·유휴 시간을 구분한다.
4. **경쟁하는 가설을 쓴다.** 예: 코드 작업량 증가, 메모리 stall 증가, CPU 배정 지연. 각 가설이 맞을 때 변할 지표와 반증할 관측을 적는다.
5. **가설을 구분할 최소 측정을 한다.** 함수 profile, 요청 timer, run queue, FS·block trace 중 필요한 경계만 고른다. 측정 범위·누락·계측 부하를 함께 남긴다.
6. **한 가지 변경으로 예측을 검증한다.** 7절 형식으로 반복 A/B와 원본을 남긴다. 목표 지표·정확성·오류·tail을 원래 실행 조건에서 다시 확인한다.

벤치마크 결과에는 **현재 성능을 제한하는 것으로 의심되는 요인과 근거**를 적는다. client CPU나 quota가 제한한 결과를 엔진 경로의 처리 능력으로 해석하지 않는다. 원인이 확정되지 않으면 후보와 다음 구분 실험을 남긴다. [Active benchmarking](https://www.brendangregg.com/activebenchmarking.html).

### 1.2 자원별 USE 확인표

USE는 자원별 **사용 정도(utilization), 처리하지 못한 수요(saturation), 오류(errors)**를 확인하는 방법이다. 아래는 DB 개발용 시작 표다. 각 칸에 값·단위·측정 구간·원본 위치를 기록한다. 확인할 수 없는 칸은 `unknown`과 사유를 남긴다. [USE method](https://www.brendangregg.com/usemethod.html), [Linux 확인표](https://www.brendangregg.com/USEmethod/use-linux.html).

| 자원 | 사용 정도 | 처리하지 못한 수요 | 오류·실패 |
| --- | --- | --- | --- |
| CPU·CPU quota | CPU별·thread별 실행 시간, `cpu.stat` | run queue·배정 지연, throttling 변화 | kernel hardware 오류, 관측 가능한 제한 관련 실패 |
| 메모리 용량 | `MemAvailable`, RSS/PSS, cgroup memory | reclaim·swap·memory PSI | OOM·할당 실패, `memory.events` |
| 저장장치·FS | IOPS·bytes·busy, 공간·inode 사용 | queue·await·I/O PSI, 요청별 latency | I/O 오류·reset, 공간 부족, DB I/O 실패 |
| 네트워크 | 인터페이스별 bytes/초와 용량 | 송수신 queue·drop·처리 지연 | NIC 오류·TCP 재전송·연결 실패 |
| 메모리·I/O interconnect | 지원되는 대역폭·uncore 계수 | 지원 도구의 stall·혼잡 지표 | 플랫폼이 제공하는 오류 계수·로그 |
| DB 내부·소프트웨어 제한 | worker·connection·FD·lock 사용 | 대기열·대기시간, 제한까지 남은 여유 | 거절·timeout·retry·FD 할당 실패 |

이 표는 원인 후보를 찾는 출발점이다. device busy와 최대 처리량을 같게 보거나, 연결 수를 CPU 사용률처럼 해석하지 않는다. 오류 누적값은 같은 구간의 변화량을 보고, 로그에 오류가 없다는 사실만으로 모든 오류를 배제하지 않는다.

### 1.3 요청과 관련된 thread 시간을 확인한다

`perf record`의 CPU stack, `perf sched`의 배정 지연, off-CPU stack, DB wait를 함께 해석한다. **일을 기다리는 유휴 worker**와 **진행 중인 요청을 지연시키는 wait**를 구분한다. 요청 ID·worker 상태·단계 timer가 이 구분에 필요할 수 있다. [Thread State Analysis](https://www.brendangregg.com/tsamethod.html), [off-CPU 분석](https://www.brendangregg.com/offcpuanalysis.html).

여러 thread의 CPU·wait 합계를 요청 elapsed에서 단순히 빼지 않는다. 병렬 실행과 겹치는 구간이 있기 때문이다. 요청, 함수, syscall, FS, block I/O는 각각 시작·완료 경계가 다르다. [경계별 해석 예](2026-10-28-performance-investigation-walkthrough.md#5-요청과-os-사건의-측정-경계를-맞춘다)를 따라 필요한 연결 정보를 선택한다.

## 2. 전권으로 가능한 것과 여전히 필요한 조건

sudo는 다른 프로세스와 커널을 관찰하고 system-wide 이벤트를 수집할 수 있게 해 준다. `perf`는 `CAP_PERFMON`을 통한 제한된 권한 위임도 지원한다. 다만 권한이 생겨도 CPU에 없는 PMU 기능이나 하이퍼바이저가 노출하지 않는 가상 PMU가 생기지는 않는다. [커널 perf 권한 모델](https://docs.kernel.org/admin-guide/perf-security.html), [perf_event_open의 지원·오류 조건](https://man7.org/linux/man-pages/man2/perf_event_open.2.html).

| 조건 | 확인할 내용 | 해석에 미치는 영향 |
| --- | --- | --- |
| CPU·가상화 | CPU 모델, stepping, microcode, 소켓·NUMA·SMT, bare metal/VM, 노출된 PMU | PEBS·IBS·SPE·Intel PT·uncore는 지원 범위가 다르다. VM 안에서는 일부 또는 전체가 없을 수 있다 |
| 커널·도구 | 커널 버전·config, 배포판 perf 패키지, `perf list`, tracepoint 목록 | 최신 문서의 옵션이나 이벤트가 현재 호스트에 있다고 가정하지 않는다 |
| 커널 구성 변경 | tracing·BPF·lockstat·delayacct·DAMON 등 필요한 기능 | debug 계측을 켠 커널과 원래 커널의 결과를 같은 성능 기준선으로 섞지 않는다 |
| 심볼·빌드 | 실제 실행 파일과 일치하는 debuginfo·build ID, 소스, 라이브러리 | strip·inline·최적화 때문에 함수·소스라인 해석이 제한될 수 있다 |
| 스택 복원 | 사용자 frame pointer 또는 DWARF unwind, 커널 unwinder | `fp`는 호출 경로의 라이브러리에도 frame pointer가 필요하다. `dwarf`는 사용자 stack dump 비용을 추가한다 |
| 필터·범위 | 서버 PID·TID, 자식 프로세스, cgroup, CPU, 측정 구간 | 클라이언트만 측정하면 별도 DB 서버의 비용이 빠진다. `-a`는 같은 호스트의 다른 작업도 포함한다 |
| 수집 신뢰성 | lost samples·buffer overrun, 이벤트 지원 여부, PMU multiplex 비율 | 누락·스케일링이 큰 결과는 정밀한 회귀 판정 근거로 쓰기 어렵다 |

스택 수집 방식과 PMU 옵션은 [perf record](https://man7.org/linux/man-pages/man1/perf-record.1.html), multiplex·이벤트 그룹은 [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html)을 기준으로 확인한다. 커널 lockdown·배포판 보안 정책·컨테이너 capability 제한도 실제 실행 조건에 포함한다. [kernel_lockdown](https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html).

## 3. 측정 옵션 비교표

**오버헤드 표기:** `낮음`은 보통 주기적 카운터 조회나 소수 이벤트 집계를 시작점으로 삼기 좋다는 뜻이다. `가변`은 샘플링 주파수, 이벤트 발생률, 스택 수집, 대상 범위에 따라 크게 달라진다. `높음`은 전수 계측·실행 에뮬레이션·디버그 기능 등으로 원래 실행을 크게 바꿀 수 있다는 뜻이다. 모두 정성적 판단이다. 실제 오버헤드는 같은 부하에서 수집을 켠 실행과 끈 실행을 비교해야 한다.

### 3.1 전체 상태·프로세스·자원 압박

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 1. GNU `/usr/bin/time -v` | 경과시간, user/system CPU, 최대 RSS, fault·context switch. 단독 엔진·유틸리티의 전후 비교 | 일반 실행 가능. root가 필수는 아니다 | 낮음. 감싼 명령과 자식의 자원 사용이며, 이미 별도로 실행 중인 DB 서버는 포함하지 않는다. 최대 RSS는 메모리 사용량의 시계열이 아니다 | [GNU Time](https://www.gnu.org/software/time/manual/time.html) |
| 2. `vmstat`·`mpstat`·`sar` | CPU별 사용률, run queue, swap, fault·I/O 등 시스템 추세 | 전체 호스트를 읽고 `sar` 이력을 수집·보관 | 낮음. 초 단위 평균은 짧은 tail spike를 숨긴다. CPU `%iowait` 하나로 DB의 I/O 대기시간을 계산할 수 없다 | [sysstat](https://sysstat.github.io/), [proc 통계](https://docs.kernel.org/filesystems/proc.html) |
| 3. `pidstat` | PID·TID별 CPU, fault, RSS, I/O, voluntary/involuntary switch. worker 편중 확인 | 다른 계정의 DB 스레드까지 관찰 | 낮음. 간격 사이에 종료한 작업과 세부 호출 경로는 별도 추적이 필요하다 | [pidstat](https://man7.org/linux/man-pages/man1/pidstat.1.html) |
| 4. `/proc/PID/{stat,status,io,smaps_rollup,smaps}` | RSS·PSS·shared/private, 매핑별 사용량, 프로세스 I/O. 공유 buffer pool과 private heap 구분 | 타 프로세스·매핑을 읽을 권한 확보 | 일반 카운터는 낮음, 큰 주소 공간의 `smaps`는 가변. RSS 합산은 공유 메모리를 중복 계산하며 read/write byte와 SQL 논리 I/O는 다르다 | [proc filesystem](https://docs.kernel.org/filesystems/proc.html) |
| 5. PSI `/proc/pressure/*` | CPU·memory·I/O 자원 압박으로 작업이 멈춘 시간 비율. 처리량 저하와 메모리 압박 확인 | `CONFIG_PSI`가 필요하며 cgroup별 pressure도 사용 | 조회는 낮음. 원인 함수는 알려 주지 않는다. `some`·`full`의 의미를 구분하고 system-level CPU `full`을 해석하지 않는다 | [PSI](https://www.kernel.org/doc/html/latest/accounting/psi.html) |
| 6. cgroup v2 `cpu.stat`·`memory.stat`·`io.stat` | DB 프로세스 집단의 CPU, throttling, memory·I/O 계수. 다중 프로세스 DB 분리 | 전용 cgroup 구성과 controller 활성화 가능 | 조회는 낮음. 상위 제한과 FS의 writeback 귀속 지원을 확인한다. shared inode의 I/O 귀속은 부정확할 수 있다. 그룹 변경·제한은 실험 조건 변경이다 | [cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), [writeback](https://docs.kernel.org/admin-guide/cgroup-v2.html#writeback) |
| 7. PCP·Prometheus `node_exporter` | 장기간 OS 추세와 과거 기준선. 야간 checkpoint·백업·회귀 시점 비교 | 호스트 collector와 보관 구성을 설정 가능 | 낮음~가변. collector·주기·시계열 수에 의존한다. node_exporter만으로 함수 profile·DB wait를 알 수 없다 | [PCP](https://pcp.io/), [node_exporter](https://github.com/prometheus/node_exporter) |
| 8. schedstat·taskstats delay accounting | 실행·run queue 대기, 동기 block I/O·reclaim 등의 지연 계수 | `CONFIG_SCHEDSTATS`, `CONFIG_TASK_DELAY_ACCT`·`CONFIG_TASKSTATS` 등. delayacct 런타임 활성화 가능 | 낮음~가변. 누적값의 차이를 사용한다. delayacct는 활성화 후 시작한 task부터 유효하며 모든 비동기 I/O를 요청별로 설명하지는 않는다 | [schedstat](https://docs.kernel.org/scheduler/sched-stats.html), [delayacct](https://docs.kernel.org/accounting/delay-accounting.html) |

### 3.2 CPU·스케줄러·동기화

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 9. `perf stat` | instructions, cycles, task-clock, faults, switch, CPU별 metric. 동일 작업의 CPU 비용 회귀 판정 | user+kernel, 타 PID·CPU·system-wide·cgroup. PMU 이벤트 지원 필요 | 소수 counting 이벤트는 낮음. 이벤트를 많이 열면 multiplex된다. IPC·cache-miss 의미는 CPU·이벤트에 의존한다 | [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html) |
| 10. `perf record` → `report`·`annotate` | 샘플의 함수·호출 경로·소스/어셈블리 분포. hot loop, 디코딩·복사 경로 분석 | 커널까지 포함해 attach 가능. debuginfo와 unwind 준비 | 가변. sample rate·stack depth·DWARF 크기에 의존한다. 샘플 비중은 호출 횟수나 요청 지연 분포가 아니다 | [record](https://man7.org/linux/man-pages/man1/perf-record.1.html), [report](https://man7.org/linux/man-pages/man1/perf-report.1.html), [annotate](https://man7.org/linux/man-pages/man1/perf-annotate.1.html) |
| 11. FlameGraph | perf/BPF 등의 stack 집계를 SVG로 시각화. 넓어진 CPU 경로 비교 | 수집 권한은 원래 수집기에 따름. 결과 생성은 offline | 시각화 자체는 대상 프로그램에 추가 부하가 없다. 일반 flame graph의 x축은 시간 순서가 아니다. 폭의 단위는 원본 가중치다 | [FlameGraph 원본](https://github.com/brendangregg/FlameGraph) |
| 12. `perf stat -M`·pmu-tools `toplev` | Retiring·Bad Speculation·Frontend/Backend 등 CPU 병목 분류 | CPU별 metric과 Top-down 지원. 모델·SMT·측정 범위 확인 | 낮음~가변. 파생 metric이며 세대 간 정의가 다르다. 비반복 부하에서 여러 pass의 결과를 섞으면 오판할 수 있다 | [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html), [toplev manual](https://github.com/andikleen/pmu-tools/wiki/toplev-manual) |
| 13. `perf sched` | wakeup→실행 지연, task별 runtime과 scheduler timeline. worker가 CPU를 못 받는 구간 확인 | scheduler tracepoint를 system-wide 수집 가능 | 가변. `timehist`의 wait에는 runnable이 아닌 시간도 포함되고 scheduling delay는 runnable→실행 구간이다. 전체 wait를 lock 대기로 부르지 않는다 | [perf sched](https://man7.org/linux/man-pages/man1/perf-sched.1.html) |
| 14. BCC `offcputime`·`runqlat` | off-CPU stack의 누적 시간과 run queue latency histogram | BPF·scheduler hook·stack 지원, 타 PID 필터 | 가변. off-CPU에는 sleep·I/O·lock·선점 이후 대기가 섞인다. runqlat과 조합해 CPU 배정 지연을 구분한다 | [offcputime](https://github.com/iovisor/bcc/blob/master/tools/offcputime.py), [runqlat](https://github.com/iovisor/bcc/blob/master/tools/runqlat.py) |
| 15. `perf lock` | 커널 lock contention과 lock별 통계. FS·메모리 관리 경로의 lock 병목 | 커널 이벤트·BPF 등 사용 모드별 지원 필요 | 가변. **DB 사용자 공간의 mutex·latch를 일반적으로 직접 측정하는 도구가 아니다** | [perf lock](https://man7.org/linux/man-pages/man1/perf-lock.1.html) |
| 16. `CONFIG_LOCK_STAT`·`/proc/lock_stat` | 커널 lock class별 contention·hold/wait 시간과 지점 | 계측용 커널을 빌드하고 lock 통계를 켤 수 있음 | 높아질 수 있음. lockdep 계열 계측이 실행을 바꾸므로 정상 빌드의 최종 성능과 분리한다. 사용자 DB lock 통계는 아니다 | [Lock Statistics](https://docs.kernel.org/locking/lockstat.html) |
| 17. `perf c2c` | cacheline별 공유·HITM 등 contention. 공용 통계·latch 구조의 확장성 분석 | Intel precise load/store, AMD IBS, Arm SPE 등 플랫폼별 지원 | 가변. CPU별 제한이 있으며 upstream 문서는 Zen 3를 지원하지 않는다고 명시한다. 공유 cacheline만으로 false sharing을 확정하지 못하며 필드·접근 주체와 true sharing을 확인한다 | [perf c2c](https://man7.org/linux/man-pages/man1/perf-c2c.1.html), [커널 false-sharing 가이드](https://docs.kernel.org/kernel-hacking/false-sharing.html) |
| 18. `perf mem` | 메모리 접근 샘플의 주소·latency·data source. 느린 load의 코드·데이터 위치 | PEBS·IBS Op·SPE 등 지원 PMU와 커널 | 가변. 모든 load를 전수 기록하지 않는다. Intel의 use-latency에는 pipeline queueing도 들어가므로 순수 DRAM 지연과 같지 않다 | [perf mem](https://man7.org/linux/man-pages/man1/perf-mem.1.html) |

사용자 공간 latch는 **spin 구간의 CPU profile**, pthread 함수 uprobe·USDT, DB 내부 wait 계수, futex syscall을 함께 본다. futex만 보면 syscall 없이 성공한 lock과 사용자 공간 spin이 빠진다. uprobe의 함수 진입→복귀 시간도 lock 획득 대기 외 실행을 포함할 수 있다. [futex](https://man7.org/linux/man-pages/man2/futex.2.html), [uprobe](https://docs.kernel.org/trace/uprobetracer.html).

### 3.3 이벤트 추적·커널 내부 계측

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 19. `strace` | syscall 횟수·인자·반환·시간. 예상 밖 fsync·read·futex 패턴 확인 | ptrace attach로 타 계정·스레드 추적 가능 | 높아질 수 있음. syscall 빈도가 높으면 실행을 크게 교란한다. `-c`·`-T`·`-w`의 시간 의미를 구분한다 | [strace](https://man7.org/linux/man-pages/man1/strace.1.html) |
| 20. ftrace·tracefs | 정적 tracepoint, function/function_graph, IRQ·preemption latency | 커널 tracer 설정·함수 필터·instance 구성 가능 | 가변~높음. 필터 없는 함수 전수 추적은 부담이 크다. `tracing_on=0`만으로 모든 계측 비용이 제거되지는 않는다 | [ftrace](https://docs.kernel.org/trace/ftrace.html) |
| 21. `trace-cmd`·KernelShark | ftrace 설정·저장·텍스트 분석. scheduler·IRQ·block 사건을 한 timeline으로 연결 | root 수집, `trace-cmd report`로 headless 분석 | 가변. per-CPU buffer·파일 크기·누락 확인. KernelShark는 선택적 GUI이므로 원격 서버에서는 실행하지 않고 필요하면 결과를 로컬로 옮긴다 | [trace-cmd record](https://www.trace-cmd.org/Documentation/trace-cmd/trace-cmd-record.1.html), [KernelShark](https://kernelshark.org/) |
| 22. BCC 도구 모음 | `profile`, `funclatency`, `syscount` 등 목적별 ready-made 계측 | 커널 BPF 기능과 도구별 hook, 보통 LLVM·header 등 런타임 준비 | 가변. 커널 안에서 집계하면 이벤트 전송량을 줄일 수 있으나 probe 자체의 발생 비용은 남는다. 커널·도구 버전 조합을 검증한다 | [BCC](https://github.com/iovisor/bcc) |
| 23. `bpftrace` | 짧은 스크립트로 histogram·count·stack 집계. 의심 syscall·함수의 임시 측정 | tracepoint·kprobe·uprobe·USDT 등. BPF와 probe 유형별 지원 | 가변. 이벤트마다 출력하기보다 집계를 먼저 사용한다. 고빈도 함수·stack 수집은 부하가 클 수 있다 | [bpftrace language](https://bpftrace.org/docs/release_024/language) |
| 24. libbpf·CO-RE | 반복 실행 가능한 전용 계측기. DB 요청 ID와 커널 이벤트를 연결하는 수집기 개발 | BTF·CO-RE 적용 조건, verifier·hook·helper 지원 | 구현에 따라 가변. CO-RE가 모든 커널 API·probe 의미까지 호환시키는 것은 아니다. map 용량·drop·수명 관리가 필요하다 | [libbpf overview](https://docs.kernel.org/bpf/libbpf/libbpf_overview.html) |
| 25. kprobe·uprobe·USDT | 커널/사용자 함수, 개발자가 심은 정적 probe의 인자·구간. page fetch·log flush 같은 의미 단위 계측 | 커널 event 기능, 정확한 binary/offset, USDT는 미리 삽입한 probe | 가변. inline·최적화·ABI·심볼 변경에 민감하다. 함수 단위 uprobe보다 의미가 명확한 USDT가 반복 조사에 유용할 수 있다 | [kprobe](https://docs.kernel.org/trace/kprobetrace.html), [uprobe](https://docs.kernel.org/trace/uprobetracer.html), [bpftrace probes](https://bpftrace.org/docs/release_024/language#probes) |
| 26. LTTng kernel·UST | 커널과 앱 tracepoint를 장기간·동일 trace 체계로 수집. 복합 요청 흐름 분석 | kernel tracer 모듈·session daemon, 앱 UST instrumentation 준비 | 가변. 활성 event·buffer·consumer 대역폭·discard를 관리한다. tracing 없이도 생기는 instrumentation 비용과 활성 수집 비용을 구분한다 | [LTTng 문서](https://lttng.org/docs/v2.15/), [lttng-tools](https://github.com/lttng/lttng-tools) |
| 27. SystemTap | 커널·사용자 공간의 스크립트 계측·symbol 기반 분석. 기존 tapset 활용 | backend별 커널 모듈·권한·debuginfo 등. 커널과 맞는 준비 필요 | 가변~높음. 선택 backend와 probe에 따라 조건이 다르다. 전수 함수 tracing은 제한된 재현에서 사용한다 | [SystemTap](https://sourceware.org/systemtap/) |
| 28. `rtla osnoise`·`timerlat` | OS 간섭·timer IRQ·실행 스레드 지연. DB tail spike의 호스트 jitter 조사 | osnoise/timerlat tracer를 갖춘 커널, CPU별 측정 설정 | 가변·능동 측정. osnoise의 반복 측정 스레드와 timerlat의 주기적 wakeup이 부하를 추가한다. DB query latency의 직접 측정값은 아니다 | [osnoise](https://docs.kernel.org/tools/rtla/rtla-osnoise.html), [timerlat](https://docs.kernel.org/tools/rtla/rtla-timerlat.html) |

### 3.4 저장장치·파일시스템·page cache

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 29. `iostat -x`·block stat | IOPS, bytes, await, queue, device busy. DB 파일 장치의 부하 확인 | 장치·partition·device-mapper 계층 조회 | 낮음. await는 queue+service를 포함한다. 병렬 NVMe·RAID의 `%util=100`이 최대 성능 한계와 같지는 않다. 여러 계층을 합하면 I/O를 중복 셀 수 있다 | [iostat](https://man7.org/linux/man-pages/man1/iostat.1.html), [block stat](https://docs.kernel.org/block/stat.html) |
| 30. BCC `biolatency`·`biosnoop` | block I/O latency histogram·요청별 시간. 평균에 숨은 flush/read tail 확인 | BPF·block hook, device별 수집 | 가변. biolatency의 queue 포함 여부(`-Q`)를 기록한다. worker·writeback이 제출한 I/O의 현재 PID를 원 SQL 실행자로 단정하지 않는다 | [biolatency](https://github.com/iovisor/bcc/blob/master/tools/biolatency.py), [biosnoop](https://github.com/iovisor/bcc/blob/master/tools/biosnoop.py) |
| 31. `blktrace`·`blkparse`·`btt` | queue·merge·issue·complete 등 block 요청 생애주기. I/O 크기·병합·스케줄링 분석 | 지원 block 장치·debugfs·커널 tracing | 가변~높음. 출력량이 많고 kernel·장치 계층별 차이가 있다. request→SQL 매핑은 별도 correlation이 필요하다 | [blktrace](https://www.man7.org/linux/man-pages/man8/blktrace.8.html), [blktrace 소스](https://git.kernel.org/pub/scm/linux/kernel/git/axboe/blktrace.git/) |
| 32. FS·writeback·`io_uring` tracepoint | 파일시스템 journal, dirty/writeback, 비동기 제출·완료. fsync가 느린 층 구분 | ftrace/BPF/perf로 현재 커널에 있는 event를 골라 수집 | 가변. syscall 반환·I/O 완료·내구성 보장은 서로 다른 경계다. async request와 completion ID를 연결해야 한다 | [event tracing](https://docs.kernel.org/trace/events.html), [io_uring tracepoint 정의](https://github.com/torvalds/linux/blob/master/include/trace/events/io_uring.h) |
| 33. BCC `cachestat`·파일 `cachestat(2)` | BCC의 page-cache 활동 추정과 syscall의 파일 범위별 cached·dirty·writeback 상태. warm/cold 조건 검증 | BCC는 kernel hook, `cachestat(2)`는 지원 커널 필요 | 가변. **같은 이름의 서로 다른 수단**이다. hook 기반 hit 추정은 DB buffer hit와 다르며 syscall은 시점별 상태이지 hit rate가 아니다 | [BCC cachestat](https://github.com/iovisor/bcc/blob/master/tools/cachestat.py), [cachestat(2)](https://man7.org/linux/man-pages/man2/cachestat.2.html) |

DB buffer pool hit, Linux page-cache hit, CPU cache hit는 서로 다른 계층이다. buffered read에서 DB buffer miss가 발생해도 OS page cache에서 충족되면 block read가 없을 수 있다. 반대로 write는 요청을 실행한 스레드보다 나중에 writeback worker가 장치로 보낼 수 있다. 따라서 DB 논리 page fetch, DB 물리 read/write, 프로세스 I/O, 장치 I/O를 각각 기록한다. [proc I/O 정의](https://docs.kernel.org/filesystems/proc.html), [buffered I/O와 writeback](https://docs.kernel.org/filesystems/vfs.html).

### 3.5 메모리·NUMA·할당·실행 시뮬레이션

프로세스 heap이나 RSS로 설명되지 않는 host 메모리 증가는 kernel slab도 확인한다. `/proc/meminfo`의 `Slab`, `SReclaimable`, `SUnreclaim`과 `slabtop -o`를 비교한다. `slabtop`의 cache 크기를 곧바로 회수 가능한 메모리나 누수 크기로 해석하지 않는다. [proc 메모리 통계](https://docs.kernel.org/filesystems/proc.html), [slabtop](https://man7.org/linux/man-pages/man1/slabtop.1.html).

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 34. `numastat`·`/proc/PID/numa_maps`·`numactl` | node별 메모리 배치·할당 통계. buffer pool의 NUMA 배치 확인 | 타 PID 조회, CPU·메모리 정책을 통제한 실험 가능 | 조회는 낮음. 배치 통계는 실제 원격 load 횟수가 아니다. `numactl`의 바인딩은 관찰이 아니라 실험 조건 변경이다 | [numactl](https://github.com/numactl/numactl), [numa_maps](https://man7.org/linux/man-pages/man5/proc_pid_numa_maps.5.html) |
| 35. DAMON·`damo` | 주소 영역별 접근 빈도·working set 추정. RSS 중 활발히 쓰는 영역 파악 | `CONFIG_DAMON` 및 선택 backend·sysfs. privileged monitor 구성 | 샘플링·영역 수에 따라 가변. monitoring과 DAMOS reclaim 등의 동작을 구분한다. DAMOS를 켜면 측정 대상 상태를 바꾼다 | [DAMON 사용법](https://docs.kernel.org/admin-guide/mm/damon/usage.html) |
| 36. `heaptrack` | allocation stack, 할당 횟수·일시 객체·남은 할당. query 중 반복 malloc 분석 | launch preload 또는 GDB를 통한 attach. unwind·symbol 준비 | 가변~높음. 할당마다 stack을 수집한다. custom pool 내부의 개별 할당은 일반 malloc trace에서 안 보일 수 있다. CLI `heaptrack_print` 사용 가능 | [heaptrack](https://github.com/KDE/heaptrack) |
| 37. jemalloc 통계·heap profiling | allocated/active/resident, arena·fragmentation, 표본 할당 stack. allocator 비용 분석 | 해당 allocator를 사용하고 profiling 지원으로 빌드·설정 | 통계 조회~표본 설정에 따라 가변. allocator를 바꾸면 기준선도 바뀐다. custom pool과 DB 논리 메모리 통계는 별도로 필요하다 | [jemalloc manual](https://jemalloc.net/jemalloc.3.html) |
| 38. Valgrind Massif | 시간에 따른 heap·선택적 stack의 점유와 allocation tree. 큰 메모리 peak 원인 | 대상 프로그램을 Valgrind 아래에서 재실행 | 높음. 원래 처리량·스케줄링 측정과 분리한다. 기본 heap 계측과 `--pages-as-heap=yes`의 의미가 다르다 | [Massif](https://valgrind.org/docs/manual/ms-manual.html) |
| 39. Valgrind Callgrind·Cachegrind | 함수별 실행 명령·호출 관계와 선택적 cache/branch simulation. 작은 hot path 비교 | 재실행과 지원 ISA, 정확한 symbol. CLI 분석 가능 | 높음. **시뮬레이션의 cache·명령 계수와 실제 PMU 결과는 다르다.** 다중 스레드 wall time·실제 coherence 판정용으로 쓰지 않는다 | [Callgrind](https://valgrind.org/docs/manual/cl-manual.html), [Cachegrind](https://valgrind.org/docs/manual/cg-manual.html) |

### 3.6 CPU 제조사·하드웨어 전용 계측

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 40. Intel PCM | 채널별 DRAM 대역폭, NUMA·UPI, PCIe·uncore 등. scan·join의 대역폭 한계 | 지원 Intel CPU, perf/MSR·PCI 접근 등 기능별 조건 | counting은 대체로 낮음. uncore 값은 대개 소켓·장치 범위라 한 DB PID에 직접 귀속하지 못한다. 다른 PMU 도구와 자원 충돌 확인 | [Intel PCM](https://github.com/intel/pcm) |
| 41. Intel VTune CLI | hotspots·uarch·memory access·threading 분석과 source attribution | 분석별 CPU·드라이버 또는 perf 수집 조건. 고급 HW 분석은 지원 Intel 대상 확인 | 가변. `vtune -collect`·`-report`로 headless 사용. 분석별 event·stack·메모리 수집 비용이 다르다 | [VTune collect](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2025-0/collect.html) |
| 42. Intel Processor Trace·`perf record -e intel_pt//` | 제어 흐름을 hardware trace로 기록·복원. 짧은 경로·branch 흐름 분석 | Intel PT, kernel perf AUX 지원, 실행 binary 보존 | 가변~높음. trace 출력·decode 비용과 저장량이 크다. PT의 제어 흐름을 모든 데이터 접근·정확한 명령별 시간으로 해석하지 않는다 | [perf Intel PT](https://man7.org/linux/man-pages/man1/perf-intel-pt.1.html) |
| 43. AMD uProf·IBS | CPU hotspot·PMC·정밀 instruction/op sample·memory 분석 | 지원 AMD 모델, kernel perf 또는 AMD 구성·드라이버 조건 | 가변. CLI 수집·보고 가능. IBS·uncore 지원과 metric 정의는 CPU 세대별로 확인한다 | [AMD uProf](https://www.amd.com/en/developer/uprof.html), [CLI 수집](https://docs.amd.com/r/en-US/57368-uProf-user-guide/Data-Collection-With-CLI) |
| 44. Arm SPE·`perf` | load/store·branch 등 operation 표본과 latency·data source. Arm DB 메모리 병목 | SPE가 있는 Arm64 CPU, 커널·firmware·perf 지원 | 가변. 통계적 표본이며 collision·샘플 선택 특성을 고려한다. Intel PEBS·AMD IBS와 수치를 무조건 같은 의미로 비교하지 않는다 | [perf Arm SPE](https://man7.org/linux/man-pages/man1/perf-arm-spe.1.html) |
| 45. `turbostat`·RAPL | 실제 busy 주파수·온도·전력·idle state. 동일 instructions인데 시간 차이가 나는 환경 확인 | 지원 x86 CPU·MSR·전력 domain 접근 | 대체로 낮음. package 전력은 DB만의 전력이 아니다. 에너지/transaction에는 유휴·동시 작업과 domain 범위를 기록한다 | [turbostat 원본 manual](https://github.com/torvalds/linux/blob/master/tools/power/x86/turbostat/turbostat.8), [powercap](https://docs.kernel.org/power/powercap/powercap.html) |

### 3.7 네트워크와 DB 내부 계측

| 번호·옵션 | 얻는 정보와 DB 활용 | 전권으로 활용하는 범위·전제 | 오버헤드·해석 한계 | 원문 |
| --- | --- | --- | --- | --- |
| 46. `ss -ti`·`nstat` | TCP RTT·retransmission·송수신 queue, protocol counter. 클라이언트 대기와 DB 실행 구분 | 다른 프로세스 소켓·namespace 조회 | 대체로 낮음. protocol 누적값은 해당 DB만의 값이 아닐 수 있다. 같은 시간 구간의 차이를 본다 | [ss](https://man7.org/linux/man-pages/man8/ss.8.html), [nstat](https://man7.org/linux/man-pages/man8/nstat.8.html) |
| 47. `ethtool -S`·IRQ/softirq 통계 | NIC drop·queue·driver 통계, CPU별 IRQ 처리. network-heavy DB의 RX 병목 | NIC·driver 통계와 IRQ affinity를 조사 가능 | 조회는 낮음. 카운터 이름·의미는 driver마다 다르다. offload·coalescing·affinity 변경은 별도 실험이다 | [ethtool](https://man7.org/linux/man-pages/man8/ethtool.8.html), [네트워크 통계](https://docs.kernel.org/networking/statistics.html) |
| 48. `tcpdump`·BCC TCP 도구 | packet timeline·retransmission, TCP lifecycle. 네트워크 지연·반복 연결 분석 | packet capture 또는 kernel TCP probe | 가변. filter·snaplen·수집량 조절. TLS payload는 해독되지 않으며 offload·packet drop을 고려한다. SQL별 귀속은 별도 연결 정보가 필요하다 | [tcpdump](https://www.man7.org/linux/man-pages/man8/tcpdump.8.html), [BCC](https://github.com/iovisor/bcc) |
| 49. DB 내부 plan·wait·I/O·단계별 timer | SQL/transaction/operator별 실행·buffer·WAL·lock 통계. OS 결과를 DB 의미에 연결 | DB 권한·설정·버전별 지원. 예: PostgreSQL EXPLAIN·누적 통계 | 가변. 세밀한 노드 timer도 비용이 있다. DB의 buffer hit를 OS cache hit와 같게 보지 않는다. EXPLAIN ANALYZE는 실제로 실행한다 | [EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html), [PG 통계](https://www.postgresql.org/docs/current/monitoring-stats.html) |
| 50. 직접 계측: clock·`getrusage`·`perf_event_open` | 의미 있는 코드 구간의 elapsed/CPU time·자원·PMU 계수. 함수 미세 최적화와 내부 regression 지표 | 코드 수정 가능, PMU는 perf 권한·지원 필요. `RUSAGE_THREAD`로 스레드 범위 구분 | 측정 빈도·읽기 방식에 따라 가변. timer·counter 읽기 비용을 먼저 측정한다. thread 이동·중첩·비동기 작업을 고려한다 | [clock_gettime](https://man7.org/linux/man-pages/man2/clock_gettime.2.html), [getrusage](https://man7.org/linux/man-pages/man2/getrusage.2.html), [perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) |

## 4. 부하 생성과 성능 판정에 함께 쓰는 도구

아래 도구는 주로 **무엇을 실행할지** 정한다. `perf`·tracing은 **그 실행에 비용이 어디서 발생했는지** 설명한다. 한 도구가 두 역할을 일부 제공해도 계층을 구분한다.

| 도구·방법 | 맞는 용도 | 함께 기록할 것 | 주요 한계·원문 |
| --- | --- | --- | --- |
| `fio` | block/file I/O의 크기·queue depth·동기/비동기·직접/버퍼 I/O 조건별 latency·IOPS·대역폭 | job file, read/write 비율, block size, iodepth, engine, fsync·direct 조건 | SQL·WAL·DB buffer pool을 재현하지 않는다. direct I/O도 장치 내부 cache를 없애지는 않는다. write 대상은 DB 파일과 분리된 시험 파일·장치로 지정한다. [fio](https://fio.readthedocs.io/en/latest/fio_doc.html) |
| `pgbench`·`sysbench` | SQL transaction 부하, 동시성·rate 변화, script별 처리량 | workload script, client 수, 성공/실패·retry, latency 원본, client 사용률 | 지원 DB·driver·workload에 의존한다. 기본 workload를 공인 TPC 결과로 부르지 않는다. [pgbench](https://www.postgresql.org/docs/current/pgbench.html), [sysbench](https://github.com/akopytov/sysbench) |
| YCSB | KV·cloud serving DB의 read/update/scan·분포별 비교 | binding·workload·key 분포·record 수·성공/오류·latency 설정 | binding·client runtime 비용도 포함된다. 엔진 내부의 한 경로만 분리한 결과가 아니다. [YCSB](https://github.com/brianfrankcooper/YCSB) |
| 고정 SQL harness·microbenchmark | 특정 코드 변경의 동일 작업량·정확성·회귀 검증 | 정확한 revision, 실행계획, 입력·결과, 완료 작업량, 서버 CPU·I/O | end-to-end harness와 함수 microbenchmark를 함께 사용한다. 짧은 함수만 빨라져도 실제 서비스가 빨라지는 것은 아니다. 반복·warmup·CPU/real time 구분 참고: [Google Benchmark](https://github.com/google/benchmark/blob/main/docs/user_guide.md) |

부하 방식도 결과를 바꾼다. 완료 후 다음 요청을 보내는 closed-loop 방식은 서버가 느려질수록 유입률이 줄어든다. 목표 도착률을 둔 실험에서는 실행시간 외에 schedule lag·queueing·누락 요청도 보아야 한다. pgbench의 `--rate`와 latency·schedule lag 구분을 참고한다. [pgbench rate·latency 정의](https://www.postgresql.org/docs/current/pgbench.html).

## 5. 바로 적용할 수 있는 수집 예

다음은 **Linux DB 서버의 터미널**에서 실행할 예시다. 도구가 설치되어 있고, 해당 이벤트를 지원하며, 짧은 수집 구간에 부하가 이미 실행 중이라고 가정한다. 실제 실행하지 않은 예시이며 버전별 `--help`·`perf list`를 먼저 확인한다. 아래 파일 경로는 현재 작업 디렉터리를 기준으로 한다. PID·파일명·경로는 실제 값으로 바꾼다.

### 5.1 환경과 대상 확인

```bash
uname -a
lscpu
perf version
perf list
cat /proc/sys/kernel/perf_event_paranoid
cat /proc/self/cgroup

# 예시 PID: 부하 생성기가 아니라 측정할 DB 서버 PID로 교체한다.
DBPID=12345
ps -L -p "$DBPID" -o pid,tid,psr,comm
```

여러 독립 worker 프로세스를 쓰는 DB는 필요한 PID 목록이나 전용 cgroup을 대상으로 삼는다. 이미 존재하는 서버를 client 명령의 자식으로 취급하지 않는다. CPU·system-wide·PID 범위는 [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html)을 따른다.

아래는 **같은 서버에서 실행하는 초기 상태 확인 예**다. journal은 systemd가 있는 현재 boot의 최근 10분을 가정한다. 실제 문제 시간과 boot가 다르면 그 범위로 바꾼다. 도구·권한이 없으면 빈 결과를 정상으로 해석하지 말고 `unknown`으로 남긴다.

```bash
date --iso-8601=seconds
uptime
journalctl -k -b --since '10 minutes ago' --no-pager
free -m
vmstat 1 5
mpstat -P ALL 1 5
sar -n DEV,EDEV,TCP,ETCP 1 5

# 메모리 증가를 heap으로 설명할 수 없을 때 추가한다.
rg '^(MemAvailable|Slab|SReclaimable|SUnreclaim):' /proc/meminfo
slabtop -o
```

`free`는 `free`뿐 아니라 `available` 추정을 본다. `vmstat` 첫 보고서의 일부 값은 boot 이후 평균이다. 위 명령은 서로 다른 시점의 상태를 순서대로 읽는다. 인과관계를 비교할 실행에서는 시작·끝 시간을 맞춘 5.3절 수집과 DB 통계를 사용한다. kernel 오류는 [journalctl](https://man7.org/linux/man-pages/man1/journalctl.1.html), 메모리 열은 [free](https://man7.org/linux/man-pages/man1/free.1.html)의 정의를 따른다.

### 5.2 전체 수치와 CPU profile

각 예시는 별도 수집이다. 회귀 수치를 판정하는 실행과 원인 분석용 tracing 실행을 나누고, 동시 수집은 그 영향이 허용되는 경우에 한다.

```bash
# 30초 동안 서버의 카운터 집계. sleep은 측정 구간을 정한다.
sudo perf stat -p "$DBPID" \
  -e task-clock,cycles,instructions,context-switches,cpu-migrations,page-faults \
  -o perf-stat.txt -- sleep 30

# 낮은 빈도에서 시작하는 stack sampling 예시.
sudo perf record -F 99 -e cycles --call-graph dwarf,16384 \
  -p "$DBPID" -o perf-cpu.data -- sleep 30
sudo perf report --stdio -i perf-cpu.data

# 소스·assembly 분석: 실제 hot symbol로 교체한다.
sudo perf annotate --stdio -i perf-cpu.data --symbol=hot_function
```

`--call-graph fp`는 frame pointer가 보존된 빌드에 적합하다. `dwarf,16384`의 16 KiB 사용자 stack dump도 비용·복원 한계가 있으므로 결과의 `[unknown]`·끊긴 stack·lost samples를 확인한다. 커널 stack 복원은 커널 빌드의 unwinder에 따른다. [perf record](https://man7.org/linux/man-pages/man1/perf-record.1.html).

위 30초 예시는 고정 시간 관찰이다. 두 버전의 처리량이 다르면 instructions 총량도 다른 작업량을 반영한다. 회귀 비교에서는 같은 구간의 성공 transaction 수로 나누거나, 동일 작업량의 시작·완료 구간을 맞춘다. warmup·유휴 시간이 섞이면 요청당 비용도 왜곡될 수 있다. 구간을 엄밀히 맞춰야 한다면 `perf stat --control` 또는 코드 내부 `perf_event_open`의 enable/disable을 사용할 수 있다. [perf stat 구간 제어](https://man7.org/linux/man-pages/man1/perf-stat.1.html), [perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html).

### 5.3 프로세스·장치·scheduler 관찰

```bash
pidstat -u -r -d -w -t -p "$DBPID" 1 10

# -y로 부팅 이후 평균인 첫 보고서를 생략한다.
iostat -x -y 1 10
cat /proc/pressure/cpu /proc/pressure/memory /proc/pressure/io

# 다른 작업도 기록되는 system-wide scheduler trace다.
sudo perf sched record -a -o perf-sched.data -- sleep 10
sudo perf sched timehist -i perf-sched.data -p "$DBPID"
```

`pidstat`, `iostat`, `perf sched timehist`의 각 시간·범위 정의는 위 비교표의 원문을 따른다. 장치 전체 부하와 DB PID의 부하를 자동으로 같은 값으로 대응시키지 않는다.

### 5.4 BPF로 fsync 경과시간 분포 확인

`bpftrace`의 syscall tracepoint를 사용하는 예다. fsync만 포함하며 fdatasync·비동기 I/O·실제 DB commit 전체를 측정하지 않는다. 진입과 복귀 사이의 **경과시간**이므로 off-CPU 시간도 포함한다. [bpftrace](https://bpftrace.org/docs/release_024/language).

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_fsync /pid == $1/ {
  @start[tid] = nsecs;
}
tracepoint:syscalls:sys_exit_fsync /@start[tid]/ {
  @fsync_us = hist((nsecs - @start[tid]) / 1000);
  delete(@start[tid]);
}
interval:s:10 { exit(); }
' "$DBPID"
```

구간 끝에 아직 복귀하지 않은 호출은 histogram에 없고 `@start`에 남는다. 실패한 fsync도 포함되므로 오류 분석이 필요하면 exit 반환값별로 집계한다. 사용 전에 해당 syscall tracepoint가 `bpftrace -l`에 있는지 확인한다.

### 5.5 kernel event를 파일로 남기기

```bash
sudo trace-cmd record -o sched-trace.dat \
  -e sched:sched_switch -e sched:sched_wakeup -- sleep 5
sudo trace-cmd report -i sched-trace.dat > sched-trace.txt
```

이 예는 scheduler 이벤트만 짧게 수집한다. FS·block·IRQ로 확장할 때는 현재 커널에 존재하는 이벤트와 필드를 확인하고 필터를 추가한다. 기존 수집 세션이 있다면 trace instance·설정 충돌도 먼저 확인한다. 원격 서버에서 GUI를 실행할 필요는 없다. [trace-cmd record](https://www.trace-cmd.org/Documentation/trace-cmd/trace-cmd-record.1.html).

### 5.6 CPU FlameGraph까지 생성한다

**실행 위치:** 5.2절의 단일 `cycles` 이벤트 기록인 `perf-cpu.data`가 있는 작업 디렉터리. Perl과 [FlameGraph 저장소](https://github.com/brendangregg/FlameGraph)의 checkout이 준비되어 있다고 가정한다. `FLAMEGRAPH_DIR`는 실제 checkout 경로로 바꾼다. binary·DSO·debuginfo는 수집 당시 파일을 사용한다.

```bash
FLAMEGRAPH_DIR=/path/to/FlameGraph
git -C "$FLAMEGRAPH_DIR" rev-parse HEAD > flamegraph-revision.txt
sudo perf script -i perf-cpu.data \
  -F comm,pid,tid,cpu,time,event,ip,sym,dso > perf-cpu.stacks
"$FLAMEGRAPH_DIR/stackcollapse-perf.pl" perf-cpu.stacks > perf-cpu.folded
"$FLAMEGRAPH_DIR/flamegraph.pl" --title='DB server CPU samples' \
  --countname=samples perf-cpu.folded > perf-cpu.svg
```

이 변환은 새 profile을 수집하지 않는다. stack을 복원할 수 없는 샘플도 원본에서 확인하고, unknown이 많은 그래프로 세부 호출 경로를 확정하지 않는다. 그래프는 생성 파일을 로컬로 옮기거나 문서 viewer로 확인한다. 원격 서버에서 브라우저를 실행하지 않는다. [perf script](https://man7.org/linux/man-pages/man1/perf-script.1.html), [FlameGraph 생성·읽기](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).

위 설정은 `period` 필드를 제외해 폭을 stack별 **샘플 수**로 맞춘다. `period`를 포함한 입력은 변환기가 그 값으로 가중할 수 있으므로 단위를 함께 바꿔야 한다. 시간 순서·호출 횟수·요청 p99를 나타내지 않는다. 넓은 상위 함수에는 자식 경로의 샘플도 들어간다. 실제 생성 코드와 함수 호출 수를 별도로 확인하고, 동일 완료량당 CPU 비용과 함께 비교한다. [변환기 원본](https://github.com/brendangregg/FlameGraph/blob/master/stackcollapse-perf.pl), [호출 경로에서 가설로 이어가는 예](2026-10-28-performance-investigation-walkthrough.md#4-호출-경로로-가설을-좁히고-한-가지-변경을-시험한다).

## 6. 결과를 잘못 비교하기 쉬운 지점

| 흔한 해석 | 확인·수정할 내용 |
| --- | --- |
| elapsed가 10초이므로 CPU도 10초를 썼다 | 다중 스레드 CPU time은 각 스레드의 사용량을 합하므로 elapsed보다 클 수 있다. elapsed·task-clock·thread 수를 함께 기록한다. [clock_gettime](https://man7.org/linux/man-pages/man2/clock_gettime.2.html) |
| instructions가 줄면 성능이 개선됐다 | 동일 작업량인지 확인한다. memory stall·주파수·동기화·I/O 대기가 악화될 수 있다. instructions와 elapsed/latency를 함께 판정한다. [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html) |
| 많은 PMU 이벤트를 동시에 열면 모두 전수 측정된다 | 하드웨어 슬롯이 부족하면 multiplex되어 일부 시간만 측정한 뒤 총량을 추정한다. counter의 running/enabled 시간과 `<not supported>`·`<not counted>`를 확인한다. 비율에 쓰는 이벤트는 가능한 한 같은 그룹·구간에서 측정한다. [perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) |
| CPU flame graph의 넓은 함수가 p99의 원인이다 | CPU sample은 on-CPU 분포다. 드문 off-CPU tail은 요청 timer·wait·event trace로 확인한다. [perf record](https://man7.org/linux/man-pages/man1/perf-record.1.html), [offcputime](https://github.com/iovisor/bcc/blob/master/tools/offcputime.py) |
| futex가 많으면 DB lock의 전체 비용을 측정했다 | 사용자 공간에서 성공한 lock·spin·custom latch가 빠질 수 있다. CPU profile·DB lock 통계를 결합한다. [futex](https://man7.org/linux/man-pages/man2/futex.2.html) |
| cache miss·page fault·DB buffer miss는 같은 사건이다 | CPU cache, 가상메모리 fault, DB buffer pool은 다른 층이다. minor fault는 장치 I/O를 요구하지 않을 수 있고 buffered read는 fault 계수와 일대일 대응하지 않는다. [getrusage](https://man7.org/linux/man-pages/man2/getrusage.2.html), [proc](https://docs.kernel.org/filesystems/proc.html) |
| read/write 호출 시간이 디스크 latency다 | OS cache·queue·FS·worker·비동기 완료를 구분한다. block 요청 지연과 syscall·SQL 경과시간은 다른 경계다. [block 통계](https://docs.kernel.org/block/stat.html) |
| global `drop_caches`로 모든 cold 조건이 만들어졌다 | DB buffer, OS page cache, 장치 cache가 별개다. drop_caches는 dirty 객체를 버리지 않으며 시스템 전체에 영향을 준다. cold 조건은 계층별로 명시하고 격리 환경에서 설계한다. [drop_caches](https://docs.kernel.org/admin-guide/sysctl/vm.html#drop-caches) |
| cache-misses 이벤트를 CPU가 다른 서버끼리 그대로 비교한다 | 이벤트의 실제 의미·지원·PMU 구현과 실행 ISA가 다를 수 있다. 같은 CPU의 전후 비교를 기본으로 하고 다른 환경 비교는 별도 설계한다. [perf list](https://man7.org/linux/man-pages/man1/perf-list.1.html) |
| 평균 latency가 같으므로 tail도 같다 | p95·p99·최대값·histogram과 충분한 완료 요청 수를 함께 남긴다. 요청 발생 방식·오류·timeout·schedule lag도 기록한다. [pgbench](https://www.postgresql.org/docs/current/pgbench.html) |
| profile의 비중이 늘었으므로 그 함수의 절대 비용이 늘었다 | 전체 실행시간·완료 작업량도 비교한다. 다른 함수가 빨라지면 비중만 커질 수 있다. `Self`와 `Children` 누적 비용을 구분한다. [perf report](https://man7.org/linux/man-pages/man1/perf-report.1.html) |

## 7. DB 성능 실험의 최소 기록 형식

아래는 재현성과 해석을 위한 권장 기록 항목이다. 절대적인 합격 임계치를 제시하는 표는 아니다.

| 묶음 | 남길 내용 |
| --- | --- |
| 문제·가설 | 어느 workload의 어떤 지표가 변했는가, 경쟁 가설, 예측할 지표, 반증 조건 |
| 비교 대상 | baseline/candidate의 정확한 commit·binary build ID, compiler·최적화 옵션, debug·sanitizer·allocator·kernel config |
| 하드웨어·실행 환경 | CPU·microcode·메모리·NUMA·SMT·가상화, 커널·도구 버전, 저장장치·FS·mount, governor·주파수·thermal 상태 |
| DB 조건 | 설정·buffer pool·WAL/durability, schema·index·통계·실행계획, 데이터 생성 seed·크기·실제 저장 형태, 시작 상태 |
| 부하 조건 | 요청 script·입력·분포·동시성·rate, client와 server 위치, warmup, 측정 길이·완료 작업량, client 자원 사용률 |
| cache·간섭 | DB buffer와 OS cache의 warm/cold 조건, 데이터가 각 cache에 들어가는지, background maintenance·backup·다른 작업 |
| 판정 지표 | 정확성·오류·retry·timeout, elapsed·TPS/QPS·latency 분포, 서버 CPU time·instructions/성공 작업·cycles, 논리/물리 I/O·WAL |
| 수집 품질 | PID·TID·CPU·cgroup 범위, user/kernel 포함 여부, events·sample rate·unwind, multiplex·lost events·buffer overrun |
| 반복과 원본 | A/B 순서 교대·여러 번 반복한 개별값과 분산, 사용 명령·원본 결과·stderr, 성능 수집 on/off 차이 |
| 자원·오류 확인 | USE 각 칸의 값·구간·원본, 확인하지 못한 `unknown`과 사유, kernel·DB 오류의 변화량 |
| 제한 요인과 구분 실험 | 의심되는 제한 요인, 지지·반대 증거, 한 가지 변경, 예상 결과와 실제 결과 |
| 결론·다음 행동 | 해결·일부 설명·미확정 중 현재 상태, 목표 지표·정확성·tail 결과, 적용 범위, 남은 질문과 다음 구분 실험 |

실제 회귀 판정은 **수집 부하가 작은 반복 실험**으로 하고, 원인을 찾는 고밀도 tracing·Valgrind·debug kernel 실험은 별도로 연결한다. 최적화 후에는 원래 조건의 처리량·latency·정확성을 다시 확인한다. 특정 지표 하나의 개선만으로 DB 전체 개선을 확정하지 않는다.

## 8. 용도별 도입 우선순위

| 단계 | 먼저 준비할 수단 | 다음 도구로 넘어갈 조건 |
| --- | --- | --- |
| 일상적인 코드 회귀 비교 | 고정 workload, DB 통계, sysstat, `perf stat`·`record`, 일치하는 symbol | CPU hot path 또는 대기 원인이 확인됐는데 세부 구분이 필요할 때 |
| 대기·I/O·동시성 분석 | bpftrace/BCC, `perf sched`, `iostat`, 요청별 latency | 함수·lock·I/O 요청 사이의 구체적인 관계를 추적해야 할 때 |
| cache·NUMA·대역폭 분석 | `perf mem`·`c2c`, CPU별 Top-down, numastat, PCM·uProf·SPE | 지원 하드웨어가 있고 일반 profile만으로 지연을 설명하기 어려울 때 |
| 반복하는 전문 조사 | libbpf·USDT·LTTng, 내부 timer·counter | 매번 임시 probe를 작성하거나 SQL 의미와 kernel 사건을 반복 연결할 때 |
| 커널·미세 경로의 집중 실험 | lockstat, delayacct, rtla, DAMON, PT, Callgrind·Massif | 재현 범위를 좁혔고 계측이 실행을 바꾸는 정도를 별도로 검증할 수 있을 때 |

각 단계는 도구 수를 늘리는 목표가 아니다. 현재 질문에 필요한 관측 범위·분해능·수집 비용을 선택하는 기준이다.
