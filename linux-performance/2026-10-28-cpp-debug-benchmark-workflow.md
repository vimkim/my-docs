# Linux C++ 개발자의 디버깅·벤치마크·성능 실험 절차

조사 기준일: **2026-10-28**. 소스를 수정하고, 빌드 옵션을 바꾸고, 테스트와 CLI 분석기를 실행할 수 있는 Linux 개발 환경을 가정한다. 대상은 DB 엔진, 서버, 스토리지 계층, 라이브러리 같은 C++ 시스템 소프트웨어다. 특정 제품의 성능 개선 결과를 보고하는 문서는 아니다.

[Linux 성능 측정 도구 가이드](2026-10-28-linux-db-performance-survey.md)가 OS와 하드웨어에서 무엇을 관찰할지 설명한다면, 이 문서는 **재현 → 빌드 준비 → 측정 → 코드 분석 → 한 가지 변경 → 정확성·성능 재검증**을 연결한다. 아래 순서와 도구 조합은 공식 자료를 바탕으로 구성한 실무 제안이다.

## 1. 질문에 따라 시작점을 고른다

| 현재 질문 | 먼저 할 일 | 남길 증거 |
| --- | --- | --- |
| crash·잘못된 결과·메모리 손상이 있는가 | 최소 입력으로 재현하고 GDB·ASan·UBSan으로 원인 위치 확인 | 입력, 기대값, 실제 결과, stack, 정확한 binary |
| hang·교착·간헐적 race인가 | 전체 thread stack, lock 소유 관계, TSan 또는 rr 재현 | 멈춘 시점의 stack, 획득 순서, 실패한 실행 기록 |
| 변경 후 실제 workload가 느려졌는가 | 동일 작업량의 반복 A/B와 정확성 검사 | 개별 elapsed·처리량·latency, 완료량·오류, 환경 |
| 어느 함수·코드 줄이 비싼가 | 최적화+심볼 빌드로 `perf stat`·`record`·`annotate` | 요청당 CPU 비용, 호출 경로, source/assembly |
| 작은 함수의 구현 대안을 비교하는가 | 실제 입력 분포를 옮긴 microbenchmark | 입력 크기·분포, 최적화 방지, 반복 원본 |
| 스레드를 늘릴수록 비효율적인가 | concurrency sweep와 저·고동시성 profile 비교 | 완료량, CPU/요청, wait, 공유 데이터 위치 |

정확성 검사는 성능 비교의 전제다. UB·data race가 있는 코드는 빌드 옵션과 스케줄링에 따라 다른 결과를 낼 수 있다. sanitizer는 실행된 경로에서 오류를 찾는 도구이며, 통과만으로 모든 입력의 안전성이 증명되지는 않는다. [UBSan](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html), [TSan](https://clang.llvm.org/docs/ThreadSanitizer.html).

## 2. 측정할 binary와 진단할 binary를 준비한다

### 2.1 빌드 목적을 분명히 한다

| 목적 | 예시 설정 | 결과를 해석할 때 확인할 것 |
| --- | --- | --- |
| 최종 성능 비교 | 실제 배포와 같은 최적화·assert·allocator·LTO 설정 | 기준선과 후보에서 코드 변경 외 조건을 맞춘다 |
| 함수·라인 profile | 위 설정에 `-g`, 필요하면 별도 debuginfo | `-g`와 최적화는 함께 쓸 수 있다. inline·변수 제거는 남는다 |
| 변수·제어 흐름 디버깅 | `-Og -g`, 필요하면 문제 TU만 낮은 최적화 | 재현 여부를 다시 확인한다. 이 실행의 속도를 배포 성능으로 쓰지 않는다 |
| 메모리·UB 검사 | 별도 `-O1 -g -fsanitize=address,undefined` | 실행시간·메모리·주소 배치가 바뀐다 |
| data race 검사 | 별도 `-O1 -g -fsanitize=thread` | ASan과 합치지 않는다. scheduler 변경을 고려한다 |

GCC는 `-g`를 최적화와 함께 지원하고 디버깅에는 `-Og`를 권장한다. frame pointer 보존도 별도 코드 생성 조건이다. `-fno-omit-frame-pointer`를 profiling 편의상 추가했다면 그 빌드와 원래 배포 빌드의 차이도 기록한다. [GCC debugging options](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html), [GCC optimization options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html).

### 2.2 기존 CMake 프로젝트에 적용하는 예

**실행 위치:** 분석할 프로젝트 root. CMake·Ninja와 해당 sanitizer runtime이 준비되어 있고, `app` 실행 파일과 CTest 테스트가 이미 정의된 프로젝트를 가정한다. `app` 경로와 프로그램 인자는 실제 프로젝트 것으로 바꾼다. 아래 `-O2`는 예시이므로 배포 설정이 다르면 그것을 유지한다. 프로젝트가 옵션을 덮어쓸 수 있으므로 `compile_commands.json`과 실제 링크 명령까지 확인한다.

```bash
# 최적화+심볼. 일반 Ninja 같은 single-config generator를 가정한다.
cmake -S . -B build/prof -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_CXX_FLAGS_RELWITHDEBINFO='-O2 -g -DNDEBUG' \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmake --build build/prof --parallel 2 --verbose
ctest --test-dir build/prof --output-on-failure --no-tests=error

# 디버깅용: profile 빌드와 디렉터리를 분리한다.
cmake -S . -B build/debug -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS_DEBUG='-Og -g'
cmake --build build/debug --parallel 2

# ASan+UBSan: 컴파일과 최종 링크 모두에 sanitizer를 적용한다.
cmake -S . -B build/asan -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS_DEBUG='-O1 -g -fsanitize=address,undefined -fno-omit-frame-pointer' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=address,undefined'
cmake --build build/asan --parallel 2
ASAN_OPTIONS=detect_leaks=1 UBSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir build/asan --output-on-failure --no-tests=error

# TSan은 독립 실행으로 검사한다.
cmake -S . -B build/tsan -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS_DEBUG='-O1 -g -fsanitize=thread' \
  -DCMAKE_EXE_LINKER_FLAGS='-fsanitize=thread'
cmake --build build/tsan --parallel 2
ctest --test-dir build/tsan --output-on-failure --no-tests=error
```

`CMAKE_BUILD_TYPE`은 single-config generator의 선택이다. 프로젝트 preset·toolchain·target별 옵션이 있으면 그 인터페이스를 우선 사용한다. 여러 언어·DSO·플러그인이 있는 프로젝트는 관련 C/C++ 컴파일과 각 최종 링크까지 검사 범위를 설계한다. [CMake build type](https://cmake.org/cmake/help/latest/variable/CMAKE_BUILD_TYPE.html), [ASan 사용법](https://clang.llvm.org/docs/AddressSanitizer.html), [GCC sanitizer 옵션·조합 제한](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html).

### 2.3 binary와 증거를 함께 보존한다

아래 `artifacts/`는 이번 실험의 출력 위치다. 실험마다 새 작업 디렉터리를 사용하거나 출력 경로를 바꾸어 이전 결과를 보존한다. 이후 예제도 이 디렉터리를 준비한 상태를 가정한다.

```bash
mkdir -p artifacts
git rev-parse HEAD > artifacts/revision.txt
git diff HEAD --binary > artifacts/worktree.patch
git status --porcelain=v1 > artifacts/git-status.txt
c++ --version > artifacts/compiler.txt
cmake --version > artifacts/cmake.txt
cp build/prof/compile_commands.json artifacts/
readelf -n build/prof/app > artifacts/elf-notes.txt
sha256sum build/prof/app > artifacts/binary.sha256
```

소스 commit만 같아도 compiler·라이브러리·생성 파일·빌드 옵션에 따라 binary는 달라진다. 실행 파일, DSO, debuginfo, 입력·설정, 빌드 로그를 한 실험 단위로 보관한다. 위 patch에는 untracked 파일 내용이 없으므로 빌드에 쓰인 파일은 별도로 보존한다. build ID가 없으면 hash와 원본 binary로 식별한다. GDB와 perf를 분석할 때 나중에 재빌드한 다른 파일로 바꾸지 않는다. 별도 debug 파일은 build ID 또는 debug link로 대응시킬 수 있다. [GDB separate debug files](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Separate-Debug-Files.html).

## 3. GDB·sanitizer·rr로 실행 상태를 확인한다

### 3.1 crash 또는 hang의 상태를 확보한다

**실행 위치:** 프로젝트 root. `--case failing`은 실제 재현 인자로 교체한다.

```bash
# 재현 실행 중 signal에서 멈추면 전체 스레드 상태를 저장한다.
gdb -q -nx -batch \
  -ex 'set pagination off' -ex run \
  -ex 'thread apply all bt full' \
  --args ./build/debug/app --case failing > artifacts/gdb-run.txt 2>&1

# 기존 core와 정확히 일치하는 binary를 사용한다.
gdb -q -nx -batch \
  -ex 'set pagination off' -ex 'thread apply all bt full' \
  ./build/prof/app ./core > artifacts/gdb-core.txt 2>&1

# 시험 프로세스에 attach해 stack을 보고 즉시 detach한다.
APPPID=12345
gdb -q -nx -batch -ex 'set pagination off' \
  -ex 'thread apply all bt full' -ex detach \
  ./build/prof/app -p "$APPPID" > artifacts/gdb-live.txt 2>&1
```

attach와 일반 breakpoint는 실행을 멈춘다. 타임아웃·lock 순서·race의 발생 양상이 바뀔 수 있으므로 성능 판정 실행과 구분한다. core는 순간 상태이며 과거 실행 순서는 없다. 자동 core 생성 여부·저장 위치는 호스트 설정에 따르고, GDB 안에서는 `generate-core-file`로 snapshot을 만들 수 있다. [GDB attach](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Attach.html), [core generation](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Core-File-Generation.html).

대화형 세션에서 함수와 변수 이름을 실제 대상으로 바꾸어 다음 순서로 좁힌다.

```gdb
info threads
thread apply all bt
thread 3
frame 2
info args
info locals
break source.cpp:120 if request_id == 42
watch -location object->state
disassemble /s hot_function
```

최적화 빌드에서 `optimized out`이나 줄 이동이 보이면 assembly·register·inline frame을 함께 본다. `volatile`을 붙여 변수를 보이게 하는 것은 원래 실행을 바꾸는 실험이다. watchpoint도 하드웨어 슬롯·크기 제한이나 software fallback을 확인한다. `set scheduler-locking step`은 stepping 중 다른 스레드를 멈추므로 협력이 필요한 경로에서는 진행을 막을 수 있다. [GDB watchpoints](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Set-Watchpoints.html), [thread execution](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Thread-Stops.html).

### 3.2 간헐적 실패를 rr로 한 번 기록한다

rr이 설치되어 있고 CPU·PMU·가상화 조건을 만족하는 작은 재현에서 사용한다.

```bash
rr record ./build/debug/app --case failing
rr replay
```

replay의 GDB에서 깨진 객체에 watchpoint를 잡고 `reverse-continue`로 마지막 변경을 거슬러 올라간다. 기록 중 나타난 실패는 반복 분석할 수 있다. 다만 rr은 사용자 thread를 한 번에 하나씩 실행하는 방식이므로 실제 병렬 성능·weak-memory 현상을 그대로 재현하는 수단은 아니다. CPU 지원과 VM의 performance counter 노출도 확인한다. [rr 사용 예](https://rr-project.org/), [현재 요구 조건](https://github.com/rr-debugger/rr#system-requirements), [rr 설계 논문의 scheduling 제약](https://arxiv.org/html/1705.05937v1#S2.SS2).

## 4. 실제 workload와 microbenchmark를 연결한다

### 4.1 두 종류의 실험에 같은 질문을 하지 않는다

| 실험 | 답하는 질문 | 반드시 고정하거나 기록할 것 |
| --- | --- | --- |
| 실제 요청·도구의 end-to-end harness | 사용자가 기다리는 시간과 완료 처리량이 개선되는가 | 입력·결과, 서비스 상태, 동시성·도착률, warmup, 완료량·오류 |
| 엔진 내부 단계 harness | decode·lookup·flush 같은 단계의 비용이 변했는가 | 호출 경계, 입력 분포, cache·allocator 상태, 완료되는 비동기 작업 |
| 함수 microbenchmark | 제한된 입력·상태에서 구현 A/B 중 어느 쪽이 유리한가 | 입력 크기·분포, 최적화 방지, setup 포함 범위, 실제 생성 코드 |

microbenchmark의 반복 평균은 요청별 p99가 아니다. 사용자 latency가 목표라면 요청 시작·완료를 직접 기록하고 실패·timeout·queueing도 남긴다. DB 서버가 별도 프로세스라면 client 명령을 감싼 CPU 계수를 서버 비용으로 쓰지 않는다. OS·DB 범위와 closed/open-loop 부하는 [기존 가이드](2026-10-28-linux-db-performance-survey.md)의 4·5·7절을 따른다.

### 4.2 Google Benchmark로 작은 경로를 검증한다

아래는 **`bench_sum.cpp`에 저장할 독립 예제**다. Google Benchmark **v1.10.0**의 헤더·라이브러리가 compiler search path에 준비되어 있어야 한다. 실제 프로젝트에는 기존 `benchmark::benchmark` CMake target을 연결하고, 합산 함수 대신 측정할 함수를 사용한다.

**실행 위치:** 예제 파일을 저장한 작업 디렉터리. 이 예제의 입력은 합계가 `uint64_t` 범위에 들어가며, 실제 함수를 옮길 때는 별도의 기대값으로 정확성을 먼저 확인한다.

```cpp
#include <benchmark/benchmark.h>
#include <cstdint>
#include <numeric>
#include <vector>

static void BM_Sum(benchmark::State& state) {
  const auto n = static_cast<std::size_t>(state.range(0));
  std::vector<std::uint64_t> values(n);
  std::iota(values.begin(), values.end(), std::uint64_t{1});
  auto* data = values.data();
  benchmark::DoNotOptimize(data);
  for (auto _ : state) {
    benchmark::ClobberMemory();
    auto result = std::accumulate(data, data + n, std::uint64_t{0});
    benchmark::DoNotOptimize(result);
  }
  state.SetItemsProcessed(state.iterations() * state.range(0));
}
BENCHMARK(BM_Sum)->Arg(1024)->Arg(1 << 20);
BENCHMARK_MAIN();
```

```bash
c++ -std=c++17 -O2 -g bench_sum.cpp -lbenchmark -pthread -o bench_sum
./bench_sum --benchmark_min_time=0.5s \
  --benchmark_min_warmup_time=0.2 --benchmark_repetitions=10 \
  --benchmark_enable_random_interleaving=true \
  --benchmark_out=artifacts/sum.json --benchmark_out_format=json
```

이 예제는 준비한 배열의 반복 읽기를 측정한다. 데이터 생성·할당은 timed loop 밖이고, 반복하면 cache가 warm해질 수 있다. `DoNotOptimize`는 결과 사용을 강제하는 보조 수단이며 계산식 자체의 constant folding·재사용을 모두 막지는 않는다. 입력을 escape시키고 memory barrier를 두었어도 실제 hot loop가 남는지 assembly로 확인한다. [Google Benchmark v1.10.0 최적화 방지](https://github.com/google/benchmark/blob/v1.10.0/docs/user_guide.md#preventing-optimization).

반복 원본을 JSON에 보관한다. random interleaving은 **같은 실행 파일 안의 benchmark 반복 순서**를 섞으며, 서로 다른 commit의 프로세스 실행 순서까지 섞지는 않는다. 서로 다른 binary는 7절의 A/B 순서를 사용한다. [random interleaving](https://github.com/google/benchmark/blob/v1.10.0/docs/random_interleaving.md).

시간 기준과 제외 범위도 명시한다. setup을 반복마다 제외하려고 `PauseTiming`을 과도하게 호출하면 그 제어 비용이 커질 수 있다. [Google Benchmark timers](https://github.com/google/benchmark/blob/v1.10.0/docs/user_guide.md#cpu-timers).

| 대상 | 선택할 시간 기준·구성 |
| --- | --- |
| 호출 thread 안에서 끝나는 CPU 함수 | 기본 CPU time과 real time을 함께 확인 |
| 함수 안에서 worker thread를 만들거나 사용하는 작업 | `MeasureProcessCPUTime()`으로 전체 process CPU, `UseRealTime()`으로 elapsed 기준을 선택 |
| framework가 여러 benchmark thread를 실행하는 실험 | `Threads(N)`과 실제 공유/개별 입력을 명시하고 wall 기준 확장성을 확인 |
| I/O·비동기 작업 | 완료까지 기다리는 경계를 만들고 wall time 측정. 제출만 재는 결과와 구분 |

### 4.3 코드에 의미 있는 계측을 심는다

다음 C++17 예제는 동기 scope의 **inclusive elapsed 합계와 호출 수**를 thread별로 모은다. `Metrics`는 해당 thread가 소유하고, 종료 뒤 join한 다음 집계한다. 객체가 stack unwinding으로 파괴되는 예외 경로도 센다.

```cpp
#include <chrono>
#include <cstdint>

struct Metrics { std::uint64_t calls = 0, elapsed_ns = 0; };
class ScopeTimer {
  using Clock = std::chrono::steady_clock;
  Metrics& metrics_;
  Clock::time_point start_ = Clock::now();
public:
  explicit ScopeTimer(Metrics& metrics) : metrics_(metrics) {}
  ScopeTimer(const ScopeTimer&) = delete;
  ScopeTimer& operator=(const ScopeTimer&) = delete;
  ~ScopeTimer() noexcept {
    const auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
        Clock::now() - start_).count();
    ++metrics_.calls;
    metrics_.elapsed_ns += static_cast<std::uint64_t>(ns);
  }
};
```

`steady_clock`은 뒤로 조정되지 않는 시간 기준이다. 이 timer에는 I/O·lock·선점 대기가 들어간다. CPU 사용량은 별도로 `CLOCK_THREAD_CPUTIME_ID` 또는 `getrusage(RUSAGE_THREAD)`로 측정한다. [C++ steady_clock 명세](https://eel.is/c++draft/time.clock.steady), [Linux clock 종류](https://man7.org/linux/man-pages/man2/clock_gettime.2.html), [getrusage](https://man7.org/linux/man-pages/man2/getrusage.2.html).

사용할 때는 thread별 `Metrics metrics;`를 만들고 측정할 block의 시작에 `ScopeTimer timer(metrics);`를 둔다. block 종료 시 누적되며, 해당 thread를 join한 뒤 `calls`와 `elapsed_ns`를 읽는다.

중첩 scope의 합계를 더하면 시간이 중복된다. coroutine·callback으로 다른 thread에서 끝나는 작업은 요청 ID와 시작/완료 사건을 연결해야 한다. 같은 `Metrics`를 여러 thread가 갱신하면 data race가 생기므로 hot path에는 thread별 저장을 사용한다. 빈 scope와 계측 on/off를 같은 조건에서 비교하고, 아주 짧은 함수는 여러 작업을 묶어 잰다. histogram·샘플링과 성공/실패별 분리도 목적에 맞게 추가한다.

반복 조사에서는 `request_begin/end`, `latch_wait_begin/end` 같은 의미 단위의 USDT·LTTng UST probe를 검토한다. 비활성·활성 수집의 비용을 각각 측정하고, probe 인자에 요청 ID·결과·완료량을 담는다. 매 호출 `printf`보다 thread별 counter·histogram 집계를 먼저 고려한다. [bpftrace USDT](https://bpftrace.org/docs/release_024/language#probes), [LTTng UST 원본 사용법](https://github.com/lttng/lttng-ust#using).

## 5. profile을 소스·생성 코드와 연결한다

### 5.1 CPU 비용을 좁힌다

**실행 위치:** 프로젝트 root. 수초 이상 실행되고 정상 종료하는 고정 workload를 가정한다. 도구·PMU 지원·권한을 먼저 확인한다. 아래는 해당 명령과 자식 대상으로 수집하며 `-a`를 쓰지 않는다.

```bash
cat /proc/sys/kernel/perf_event_paranoid
perf version
perf list

perf stat -e task-clock,cycles,instructions,context-switches,page-faults \
  -o artifacts/perf-stat.txt -- ./build/prof/app --case fixed
perf record -F 99 -e cycles --call-graph dwarf,16384 \
  -o artifacts/perf.data -- ./build/prof/app --case fixed
perf report --stdio -i artifacts/perf.data > artifacts/perf-report.txt
perf annotate --stdio -i artifacts/perf.data \
  --symbol='hot_function' > artifacts/perf-annotate.txt
```

권한은 `perf_event_paranoid` 숫자 하나만으로 결정하지 않는다. 이벤트·user/kernel 범위·capability·컨테이너 정책을 함께 확인한다. PMU가 없으면 지원되는 software 이벤트로 범위를 좁힐 수 있지만, 그 결과로 cycles·IPC를 추정하지 않는다. [커널 perf 권한 모델](https://docs.kernel.org/admin-guide/perf-security.html).

`hot_function`은 report의 실제 symbol로 교체한다. C++ overload·inline·DSO가 섞이면 대상과 소스 대응을 먼저 확인한다. DWARF stack dump의 비용·잘림, lost samples, PMU multiplex·unsupported 이벤트를 기록한다. fp 수집은 호출 경로의 frame pointer 보존이 필요하다. [perf record](https://man7.org/linux/man-pages/man1/perf-record.1.html), [annotate](https://man7.org/linux/man-pages/man1/perf-annotate.1.html).

샘플 비중과 호출 수는 다르다. 한 함수의 비율이 커졌어도 전체 실행이 빨라졌다면 절대 비용은 줄었을 수 있다. 동일 완료량당 instructions·CPU time·elapsed를 비교하고, 낮은 CPU 사용률의 긴 지연은 요청 timer·off-CPU·run queue·I/O 경로로 넘긴다. IPC나 HITM의 단일 임계치로 원인을 확정하지 않는다. [perf stat](https://man7.org/linux/man-pages/man1/perf-stat.1.html), [기존 가이드의 해석 주의점](2026-10-28-linux-db-performance-survey.md#6-결과를-잘못-비교하기-쉬운-지점).

### 5.2 컴파일러가 만든 코드를 확인한다

**실행 위치:** 프로젝트 root. `kernel.cpp`는 분리한 작은 함수 또는 실제 TU다. 실제 프로젝트의 include·macro·ISA·최적화 옵션을 가져와 실행한다.

```bash
# GCC vectorization 성공/실패 사유
g++ -std=c++17 -O2 -g -fopt-info-vec-all=artifacts/vectorization.txt \
  -c kernel.cpp -o artifacts/kernel.o

# Clang을 사용하는 프로젝트라면 대응 진단을 확인한다.
clang++ -std=c++17 -O2 -g -Rpass=loop-vectorize \
  -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize \
  -c kernel.cpp -o artifacts/kernel-clang.o 2> artifacts/clang-vectorization.txt

# 최종 링크된 binary의 source/assembly도 확인한다.
objdump -d -C -S build/prof/app > artifacts/app-assembly.txt
```

TU의 진단은 vectorization·inline 여부의 단서이며 최종 실행 비용은 아니다. LTO·linking·runtime dispatch가 결과를 바꿀 수 있으므로 실제 실행한 binary의 코드와 profile을 연결한다. [GCC opt-info](https://gcc.gnu.org/onlinedocs/gcc/Developer-Options.html), [Clang optimization reports](https://clang.llvm.org/docs/UsersManual.html#options-to-emit-optimization-reports), [objdump](https://sourceware.org/binutils/docs/binutils/objdump.html).

### 5.3 할당·호출 수를 별도 실험으로 분해한다

```bash
# 도구가 설치된 작은 재현에서 각 명령을 별도 실행한다.
heaptrack ./build/prof/app --case fixed
# 위 실행이 출력한 실제 결과 파일 경로로 바꾼다.
heaptrack_print heaptrack.app.PID.gz > artifacts/heaptrack.txt

valgrind --tool=callgrind --callgrind-out-file=artifacts/callgrind.out \
  ./build/prof/app --case fixed
callgrind_annotate artifacts/callgrind.out > artifacts/callgrind.txt

valgrind --tool=cachegrind --cache-sim=yes --branch-sim=yes \
  --cachegrind-out-file=artifacts/cachegrind.out \
  ./build/prof/app --case fixed
cg_annotate artifacts/cachegrind.out > artifacts/cachegrind.txt
```

heaptrack으로 allocation 횟수·크기·stack을 확인하고, pool 안의 논리 할당은 별도로 센다. Callgrind는 호출 관계·실행 명령 수를, Cachegrind는 명령 계수와 선택한 cache/branch simulation을 분해한다. 이 실행들의 wall time을 원래 처리량과 비교하지 않는다. 시뮬레이션 cache miss는 실제 PMU·NUMA·coherence 사건과 다르다. [heaptrack CLI](https://github.com/KDE/heaptrack), [Callgrind](https://valgrind.org/docs/manual/cl-manual.html), [Cachegrind](https://valgrind.org/docs/manual/cg-manual.html).

## 6. 가설에 맞는 변경 하나를 고른다

수정 전에 **원인이 맞으면 어떤 지표가 어떻게 바뀌는지** 적는다. 다음은 확인 방향이며 자동 처방이 아니다.

| 관측과 가설 | 구분할 실험·변경 | 정확성·해석 조건 |
| --- | --- | --- |
| 반복 allocation이 CPU 비용을 늘린다 | 호출 수 계측 후 재사용·예약·batching 하나를 비교 | 생명주기·peak memory·오류 경로도 검사 |
| 공유 통계 counter가 확장성을 막는다 | field 접근·cacheline 위치 확인 후 thread별 누적을 비교 | 최종 합계·조회 시점·메모리 사용·공유 방식 유지 |
| 독립 필드가 같은 cacheline에 있다 | `perf c2c`와 struct layout으로 확인 후 분리·배치 변경 | false/true sharing 구분, 메모리 footprint·cache locality 재검증 |
| lock 대기 또는 spin이 지배한다 | 획득 대기·보유시간·소유자를 계측하고 critical section 축소 | `futex`가 전체 사용자 lock 비용은 아니다. fairness·기아·정합성 검사 |
| loop 의존성·메모리 접근이 병목이다 | compiler report·assembly 확인 후 loop 순서·batch·SIMD 중 하나 비교 | 수치 오차·alignment·경계·ISA dispatch·alias 계약 확인 |

false sharing은 cacheline과 실제 필드·접근 주체를 연결해 확인한다. 모든 공유를 padding으로 해결할 수 없고, 배치 변경이 다른 locality를 악화시킬 수도 있다. [커널 false-sharing 가이드](https://docs.kernel.org/kernel-hacking/false-sharing.html).

floating-point reduction의 재결합은 결과를 바꿀 수 있다. `-ffast-math`·`-Ofast`는 허용 의미를 바꾸므로 단순한 속도 옵션으로 추가하지 않는다. `__restrict__`는 G++ 확장이며, 호출자가 지켜야 하는 alias 계약을 먼저 증명한다. [GCC 수치 최적화 의미](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html#Optimize-Options), [C++ restricted pointers](https://gcc.gnu.org/onlinedocs/gcc/Restricted-Pointers.html).

PGO·LTO·allocator·라이브러리 교체도 각각 독립 후보로 다룬다. PGO는 대표적인 훈련 부하·profile 파일·compiler 버전을 보관하고, 학습에 쓰지 않은 입력으로 재검증한다. LTO는 관련 TU와 링크 단계에 적용되는지 확인한다. 특정 작은 입력에 과적합한 개선을 전체 workload의 개선으로 일반화하지 않는다. [GCC profile generation](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html), [GCC profile-use·LTO](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html).

## 7. 반복 A/B로 원래 목표를 재검증한다

### 7.1 입력과 시작 상태를 함께 맞춘다

1. 정확한 baseline/candidate와 주 판정 지표를 정한다. 성공한 고정 작업량의 elapsed 또는 정해진 부하의 처리량·latency처럼 경계를 명시한다.
2. 각 binary의 정확성·오류·완료량을 먼저 검증한다. 테스트·checksum·독립 기대값을 사용한다.
3. 실행별 seed·데이터 상태·cache 조건·warmup·설정을 맞춘다. 변경된 DB나 파일은 다음 실행 전에 동일 snapshot으로 복원한다.
4. 한 번에 하나씩 A/B를 실행하고 순서를 교대하거나 무작위화한다. 원본 개별값을 남기고 중앙값·분산·필요한 신뢰 구간을 함께 본다.
5. 설명용 profile은 별도 실행으로 수집한다. 최종 판정은 debugger·sanitizer·고밀도 tracing 없이 원래 빌드 조건에서 반복한다.
6. 여러 입력 크기·분포와 concurrency에서 재검증한다. microbenchmark의 개선과 실제 workload 결과를 나란히 남긴다.

주파수·열·SMT·NUMA·다른 작업·주소 배치가 분산을 바꿀 수 있다. affinity나 governor를 바꾸면 적용 범위와 전후 값을 기록한다. 작은 차이를 보려고 설정을 바꾼 경우 실제 운영 조건에서도 다시 확인한다. [Google Benchmark의 분산 요인](https://github.com/google/benchmark/blob/v1.10.0/docs/reducing_variance.md).

### 7.2 독립 실행 파일의 순서를 교대하는 작은 harness

**실행 위치:** 동일하게 빌드한 `baseline/`·`candidate/` checkout의 부모 디렉터리. Python 3를 사용하며, 두 `app --case fixed`가 동일 고정 작업량을 처리하고 정상 종료한다고 가정한다. 여기에는 재시작·snapshot 복원·warmup이 없다. 상태를 바꾸는 workload라면 해당 준비를 명시적으로 추가한다.

```python
import json
import pathlib
import subprocess
import time

out = pathlib.Path(f"artifacts-ab-{time.time_ns()}")
out.mkdir()
with (out / "runs.jsonl").open("w") as report:
    for index, variant in enumerate(["baseline", "candidate", "candidate", "baseline"] * 5):
        command = [f"./{variant}/build/prof/app", "--case", "fixed"]
        with (out / f"{index:02d}-{variant}.stdout").open("wb") as stdout, \
             (out / f"{index:02d}-{variant}.stderr").open("wb") as stderr:
            started = time.perf_counter()
            result = subprocess.run(command, stdout=stdout, stderr=stderr)
            elapsed = time.perf_counter() - started
        report.write(json.dumps({"run": index, "variant": variant,
                                 "command": command, "elapsed_s": elapsed,
                                 "exit_code": result.returncode}) + "\n")
        report.flush()
        if result.returncode != 0:
            raise SystemExit("workload failed; inspect retained stdout/stderr")
```

이 harness의 시간은 process 시작·종료와 출력까지 포함한다. 정상 종료만 검사하므로 실제 결과·완료량 검증을 프로젝트 harness에 추가해야 한다. 예상과 다른 결과나 host 간섭은 버릴 값으로 숨기지 말고 원본과 제외 사유를 보관한다. ABBA 반복은 순서 편향을 줄이려는 예시이며, 모든 분산 요인을 제거하거나 통계적 유의성을 보장하지 않는다.

## 8. 실험 기록과 반복 작업용 스킬

| 항목 | 기록 내용 |
| --- | --- |
| 질문·예측 | 무엇이 느린가, 원인 가설, 맞을 때 변할 지표와 반증 조건 |
| 비교 대상 | commit·dirty patch, binary hash/build ID, compiler·flags·DSO·allocator |
| 입력·정확성 | seed·분포·크기·상태 복원, 기대값·checksum·완료량·오류 |
| 실행 조건 | CPU·kernel·NUMA·주파수·cache·동시성·도착률·warmup |
| 명령·구간 | 작업 디렉터리, 실행 명령, PID/TID/cgroup, timer·counter의 시작/끝 |
| 원본 | 개별 A/B 값, stdout/stderr·JSON, profile·symbols·입력·빌드 로그 |
| 수집 품질 | lost samples·multiplex·지원 이벤트, instrumentation on/off 영향 |
| 결론 | 정확성 결과, end-to-end와 micro 결과, 변동 폭, 적용 범위·남은 불확실성 |

`linux-perf`는 수집·hotspot 분석, `performance-patterns`는 관측에서 수정 후보를 찾는 절차로 활용한다. `research`는 공식 근거와 재현 절차를 문서로 남기는 데 사용한다. 설치된 스킬의 휴리스틱은 조사 출발점이며, 이 문서에서는 PMU 지원·권한·명령 옵션을 공식 문서로 확인하고 단일 지표로 원인을 단정하지 않는다. [Intel 스킬 원본](https://github.com/intel/intel-performance-skills).

실제 작업에서는 다음처럼 요청 범위를 지정할 수 있다.

> 이 C++ workload의 느린 구간을 재현하고 baseline을 남겨라. 배포와 같은 최적화·심볼 빌드를 사용하고, 정확성 검사를 먼저 통과시켜라. perf와 필요한 최소 계측으로 원인 가설을 구분한 뒤 한 가지 변경을 비교하라. 원래 조건의 반복 A/B와 실제 workload로 검증하고, 명령·binary 식별자·원본 결과·결론을 이 저장소의 실험 문서에 저장하라.

## 9. 이 문서의 검증 범위

도구 기능과 옵션은 위의 공식 자료에서 확인했다. 제품별 `app`·`kernel.cpp`·`hot_function`·PID·core·heaptrack 결과 파일명은 적용 대상에 맞게 바꾸는 예시다. 실제 서버의 성능 수치나 최적화 효과를 이 문서에서 주장하지 않는다.

2026-10-28에 임시 C++17 프로그램과 문서에서 추출한 코드를 사용해 다음을 확인했다. 검증 환경은 GCC 11.5.0, CMake 3.26.5, Ninja 1.10.2, GDB 14.2, perf 5.14.0, rr 5.9.0, Valgrind 3.24.0이다. Google Benchmark v1.10.0은 임시 디렉터리에서 빌드·링크했다.

| 확인한 항목 | 결과와 범위 |
| --- | --- |
| 최적화+심볼·디버깅 빌드 | 별도 CMake fixture의 빌드·실행 성공, 같은 입력의 출력 일치. 제품의 CTest 테스트를 실행한 결과는 아니다 |
| GDB·심볼·assembly | breakpoint에서 두 thread의 stack·사용자 소스 위치 확인. build ID 조회와 `objdump` 성공 |
| perf | 시험 명령과 자식만 대상으로 stat·record·report·annotate 성공. 사용자 함수와 assembly 해석 확인. 커널 심볼 접근 제한 경고가 있어 커널 심볼 해석은 검증 범위에서 제외 |
| rr·Callgrind | 임시 프로그램의 record·replay 출력 일치. Callgrind 수집·CLI annotate 성공 |
| Google Benchmark | 4.2절 코드를 그대로 컴파일. 두 입력 크기, warmup·반복·random interleaving·JSON 저장 확인. 빠른 기능 확인을 위해 최소 시간은 `0.02s`, warmup은 `0.01`, 반복은 2회로 줄였으며 성능 판정에는 사용하지 않음 |
| ScopeTimer | 4.3절 코드를 컴파일하고 정상·예외 경로와 thread별 독립 집계를 확인 |
| A/B Python harness | 7.2절 코드를 그대로 실행해 정상 20회·원본 파일 40개 보관 확인. 의도적인 exit 7을 기록하고 2번째 실행 뒤 중단하는 동작도 확인 |
| sanitizer | GCC의 UBSan·TSan runtime 파일 누락으로 compiler 링크 확인 단계 실패. 해당 검사는 실행 완료로 계산하지 않음 |

GCC vectorization 진단의 생성도 확인했다. 실제 제품의 재현 입력, core 분석, race 탐지, heaptrack·Cachegrind와 PGO·LTO 효과는 각 프로젝트에서 별도로 검증해야 한다.
