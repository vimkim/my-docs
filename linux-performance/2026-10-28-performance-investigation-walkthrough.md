# C++·DB 성능 문제를 가설과 실험으로 좁히는 절차

작성일: **2026-10-28**. Linux에서 C++ 서버와 DB 엔진의 성능 회귀를 조사하는 개발자를 위한 예제다. **이 문서의 수치·함수 이름·관측 결과는 설명용으로 구성한 가상 사례다. 실제 제품이나 현재 서버에서 측정한 결과가 아니다.**

목표는 느린 구간을 찾은 뒤, 어떤 증거로 원인 가설을 좁히고 어떤 변경으로 검증할지 익히는 것이다. 도구·권한·수집 명령은 [측정 가이드](2026-10-28-linux-db-performance-survey.md), 빌드·소스 계측·반복 A/B는 [C++ 개발 가이드](2026-10-28-cpp-debug-benchmark-workflow.md)를 따른다. [Gregg 자료와 비교한 검토](2026-10-28-gregg-performance-coverage-review.md)에서 확인한 방법론의 부족을 보완한다.

## 1. 문제와 성공 조건을 먼저 쓴다

> 같은 데이터에서 100만 행의 8개 컬럼을 읽는 scan이 버전 A보다 B에서 느려졌다. 두 버전의 결과는 같다. 우선 이 scan의 경과시간 회귀를 설명하고 줄인다.

이 예제는 단일 client·단일 scan worker, 동일 실행계획·schema·데이터·설정, warm DB buffer를 가정한다. client가 잰 query elapsed와 같은 실행 구간의 서버 계수를 기록한다. 실제 조사에서는 cache 상태·배경 작업·단계 경계를 원본으로 확인해야 한다.

문제에는 다음 네 가지를 붙인다.

| 항목 | 이 예제에서 정한 내용 |
| --- | --- |
| 비교 대상 | A와 B의 source commit, binary hash/build ID, compiler·flags·DSO를 각각 보존 |
| 완료 작업량 | 반환 행 1,000,000개, 선택 컬럼 8개, 기대 결과와 비교한 전체 결과 검증 |
| 주 판정 지표 | 정상 완료 scan의 query elapsed. 오류·timeout이 생긴 실행은 성공 표본으로 숨기지 않음 |
| 성공 조건 | 정확성을 유지하며 B의 회귀를 줄이고, 예측한 원인 지표도 함께 변하는지 확인 |

서비스 p99 개선은 이 scan 실험과 별개의 목표다. 한 query가 반환한 100만 행을 100만 개의 요청 latency 표본으로 계산하지 않는다.

## 2. 초기 자원 상태와 thread 시간을 확인한다

[측정 가이드의 USE 표](2026-10-28-linux-db-performance-survey.md#12-자원별-use-확인표)를 채운다. 아래 관측도 가상 사례의 일부다. 실제 기록에는 각 행의 시간 구간·원본 파일·권한 제한을 붙인다.

| 확인한 것 | 예제 관측 | 현재 해석 |
| --- | --- | --- |
| 오류·실패 | 해당 구간 kernel·DB 로그에서 관련 오류를 발견하지 못함. query 오류·retry·timeout 변화 없음 | 복구·retry 경로의 증거가 약하다. 모든 오류를 배제한 것은 아님 |
| CPU·quota | scan worker의 실행이 주 비용. CPU 배정 지연 변화가 작고, 대상·상위 cgroup의 throttling 증가 없음 | CPU 실행 비용을 먼저 분해한다 |
| 메모리·장치 I/O | 양쪽 scan의 DB physical read가 0. reclaim·swap·I/O pressure 악화가 관측되지 않음 | 장치 read가 늘었다는 가설의 근거가 약하다. CPU cache·메모리 stall은 별도로 남음 |
| client·네트워크 | client CPU와 송수신 queue·오류에서 새 제한 요인을 발견하지 못함 | 서버 작업량 변화를 조사할 근거가 생김 |
| worker·연결·FD | 가상 사례의 한 client·worker가 동작하며 제한 근처의 대기·거절 없음 | pool capacity 부족을 우선 원인으로 보지 않음 |
| interconnect | `unknown`: 이 사례에서는 계측하지 않음 | 대역폭 병목이 없다고 단정하지 않음 |

오래 잠든 worker의 off-CPU 합계가 가장 크더라도 scan의 원인으로 바로 선택하지 않는다. 진행 중인 요청을 맡은 worker인지 확인하고, 그 요청 구간의 실행·runnable 지연·blocking을 본다. [Thread State Analysis](https://www.brendangregg.com/tsamethod.html).

이 단계는 확인할 자원과 오류를 빠뜨리지 않기 위한 것이다. 사용률이 낮거나 평균 latency가 비슷하다는 이유만으로 자원을 완전히 제외하지 않는다. [USE method](https://www.brendangregg.com/usemethod.html).

## 3. 동일 완료량의 비용으로 경쟁 가설을 구분한다

표의 시간은 수집 부하가 작은 반복 실행, 계수는 동일 workload의 별도 진단 실행을 나타낸다. 실제 실험에서도 계측을 켠 실행과 끈 실행을 식별해 보존한다. 서로 다른 실행의 시간과 계수를 같은 실행의 값처럼 섞지 않는다.

| 지표 | 기준선 A | 회귀 버전 B |
| --- | --- | --- |
| 개별 query elapsed, 초 | 10.0, 10.2, 9.9, 10.1, 10.0 | 12.0, 11.8, 12.2, 12.1, 11.9 |
| elapsed 중앙값, 초 | 10.0 | 12.0 |
| 정상 완료 행 수 | 1,000,000 | 1,000,000 |
| 진단 실행의 서버 CPU time, 초 | 8.0 | 10.0 |
| 같은 진단 구간의 instructions | 24,000,000,000 | 30,000,000,000 |
| 같은 진단 구간의 cycles | 24,000,000,000 | 30,000,000,000 |
| instructions/행 | 24,000 | 30,000 |
| IPC = instructions/cycles | 1.0 | 1.0 |

elapsed 중앙값은 A 대비 **20% 증가**, instructions/행은 **25% 증가**다. 같은 완료량에서 CPU 작업량 증가를 먼저 조사할 이유가 있다. IPC가 같다는 사실만으로 메모리 병목이나 혼합 원인을 배제하지 않는다. [perf stat의 계수·범위](https://man7.org/linux/man-pages/man1/perf-stat.1.html).

| 경쟁 가설 | 구분할 다음 증거 | 약해지거나 반증되는 조건 |
| --- | --- | --- |
| 같은 행을 읽는 코드가 더 많은 작업을 한다 | CPU profile에서 커진 호출 경로, 해당 함수의 호출 수·작업량 | 그 경로의 비용 증가가 재현되지 않거나 제거해도 요청당 CPU 비용이 유지됨 |
| 같은 작업이 CPU cache·메모리 stall 때문에 느려졌다 | 지원 PMU의 메모리 관련 지표, `perf mem`·Top-down, 같은 코드 경로의 비용 | 해당 입력·실험에서 stall 변화 없이 다른 원인만 바꿔 회귀가 재현·해소됨 |
| CPU를 배정받지 못하거나 I/O·lock을 기다린다 | 요청 worker의 runnable 지연·off-CPU와 요청 timer | 해당 대기 증가가 없고, 추가 CPU 작업을 제거했을 때 elapsed도 회복됨 |

후속 실험은 현재 가장 유력한 가설부터 한다. 어느 가설도 지지되지 않으면 측정 경계·완료량·대상 PID를 다시 확인한다. 지원되지 않는 PMU 이벤트는 0으로 해석하지 않는다.

## 4. 호출 경로로 가설을 좁히고 한 가지 변경을 시험한다

[측정 가이드의 FlameGraph 절차](2026-10-28-linux-db-performance-survey.md#56-cpu-flamegraph까지-생성한다)로 CPU stack을 변환한다. 예제의 B에서 주목한 호출 경로는 다음과 같다.

```text
execute_scan
  decode_row
    decode_column
      validate_row_layout
```

그래프의 폭은 이 경로를 지나는 샘플 수다. `decode_row`의 inclusive 비중에는 자식 비용도 들어간다. 높은 비중만으로 호출 횟수나 한 호출의 비용을 알 수 없다. [CPU FlameGraph 해석](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).

실제 binary의 source·inline·assembly 대응을 확인한 뒤, 별도 계측으로 `validate_row_layout`의 호출 수를 센다고 가정한다. 전체 호출마다 출력하는 대신 thread별 counter를 사용하고 계측 on/off 차이도 확인한다. 예제에서는 A가 행당 1회, B가 선택 컬럼당 1회로 각각 100만·800만 회 호출한다.

**원인 가설:** B가 행마다 불변인 layout 검증을 컬럼마다 반복해 CPU 비용을 늘렸다.

**변경 전 예측:** 불변 검증의 중복만 제거한 C에서는 호출 수가 행당 1회로 줄고 instructions/행·CPU time이 A에 가까워진다. 원래 조건의 elapsed도 개선되어야 한다.

**반증 조건:** 호출 수만 줄고 CPU 비용이나 elapsed가 회복되지 않으면 이 변경만으로 회귀를 설명하지 못한다. 검증 결과가 컬럼·동시 변경에 의존하면 이동 자체가 올바르지 않을 수 있다.

설명용 의사 코드는 다음과 같다. API·형식은 실제 엔진 코드가 아니다.

```cpp
// B: 컬럼을 읽을 때마다 같은 행의 layout을 검사한다.
for (auto column : selected_columns) {
  const auto checked = validate_row_layout(row, schema_snapshot);
  consume(decode_column(row, checked, column));
}

// C: 선택 컬럼이 있는 경우에만 같은 검증을 한 번 수행한다.
if (!selected_columns.empty()) {
  const auto checked = validate_row_layout(row, schema_snapshot);
  for (auto column : selected_columns) {
    consume(decode_column(row, checked, column));
  }
}
```

이 이동은 같은 행을 읽는 동안 row와 schema snapshot이 변하지 않고, 검증에 반복해야 할 부작용이 없다는 조건이 필요하다. `decode_column` 안의 컬럼별 경계 검사는 유지한다. 빈 projection, NULL·가변 길이·손상 입력·오류 반환·예외 경로도 비교한다. microbenchmark만 통과했다는 이유로 엔진 전체에서 안전하다고 판단하지 않는다.

## 5. 요청과 OS 사건의 측정 경계를 맞춘다

위 사례는 CPU 작업량을 따라갔다. 같은 절차로 wait를 조사할 때도 측정 시작·완료를 먼저 적는다.

| 알고 싶은 것 | 필요한 경계와 연결 정보 | 그 값만으로 확정할 수 없는 것 |
| --- | --- | --- |
| 사용자 응답시간이 늘었는가 | client 요청 시작→응답 완료, 오류·timeout, 도착 방식 | 서버 CPU 시간과 개별 I/O 비용 |
| 어느 DB 단계가 늘었는가 | 요청 ID와 decode·lock wait·flush 시작/완료 | 다른 요청과 공유하는 작업의 비용 배분 |
| thread가 CPU를 기다렸는가 | 같은 worker의 wakeup·runnable→실행, 실행 중 요청 | 모든 off-CPU 시간이 lock 대기라는 결론 |
| FS 작업이 느린가 | 해당 파일·FS 함수 시작→완료와 요청/worker 정보 | 실제 장치 요청만의 latency |
| 장치 요청이 느린가 | block queue·issue·complete, 장치와 요청 식별자 | 그 요청이 어느 SQL의 응답을 얼마나 지연시켰는가 |

PMU·software 이벤트는 계수나 표본, tracepoint·kprobe·uprobe·USDT는 사건의 관측 지점을 제공한다. `perf`·ftrace·BPF는 목적에 맞게 이 지점에서 수집한다. 결과는 counter, stack, histogram, timeline 등으로 표현한다. **관측 지점 → 수집 방식 → 결과 표현**을 구분하면 도구를 바꿔도 같은 질문을 유지할 수 있다. [perf 이벤트 분류](https://www.brendangregg.com/perf.html#Events).

fsync가 길다면 먼저 file·FS 구간과 block 구간을 구분한다. 사용 중인 FS에 맞는 BCC `ext4slower`·`xfsslower` 또는 해당 tracepoint를 선택한다. `ext4slower` 같은 도구가 선택한 file operation latency는 모든 fsync를 대표하지 않는다. 실제 함수·이벤트 범위를 확인한다. async·writeback worker가 개입하면 시간 근접성만으로 SQL에 귀속하지 말고 추가 식별 정보를 확보한다. [ext4slower 원본](https://github.com/iovisor/bcc/blob/master/tools/ext4slower.py), [block 통계 경계](https://docs.kernel.org/block/stat.html).

간헐적인 회귀에서는 전체 구간의 profile과 histogram에 시간축도 추가한다. latency heat map은 시간별 분포 변화를, FlameScope는 선택한 시간 구간의 CPU 경로를 살펴보는 데 쓴다. 이 역시 on-CPU 관측만으로 요청 p99의 원인을 확정하는 도구는 아니다. [Heat maps](https://www.brendangregg.com/heatmaps.html), [FlameScope](https://www.brendangregg.com/flamescope.html).

## 6. 원래 조건에서 검증하고 결론의 범위를 쓴다

가상 사례에서 C를 A·B와 같은 조건으로 반복하고 결과를 다음처럼 기록한다. 별도 계측으로 호출 수를 확인한 뒤, 시간 판정은 그 계측을 끈 실행에서 한다.

| 지표 | 변경 C |
| --- | --- |
| 개별 query elapsed, 초 | 10.1, 10.2, 10.0, 10.1, 10.1 |
| elapsed 중앙값, 초 | 10.1 |
| 정상 완료 행 수 | 1,000,000 |
| 진단 실행의 서버 CPU time, 초 | 8.1 |
| instructions/행 | 24,300 |
| 별도 계측한 layout 검증 호출 수 | 1,000,000 |
| 결과·오류 검사 | 기대 결과 일치, 예제의 오류 경로 검사 통과 |

비교 기준을 붙여 쓴다. C의 elapsed 중앙값은 **B 대비 약 15.8% 감소**, **A 대비 1.0% 증가**다. 이 다섯 값만으로 A와 통계적으로 동등하다고 주장하지 않는다. instructions/행도 A에 가까워져 최초 예측과 같은 방향으로 변했다.

예제에서 쓸 수 있는 결론은 다음과 같다.

> 이 warm-buffer scan에서는 B의 중복 layout 검증이 CPU 비용 증가의 주요 원인이라는 설명을 지지한다. 불변 검증의 중복을 제거한 C에서 호출 수·행당 instructions·query elapsed가 함께 줄었다. 장치 I/O 개선이나 전체 서비스의 p99 개선은 이 실험으로 확인하지 않았다.

실제 작업에서는 원본 실행별 결과, 변경 diff, 정확성 검사와 반대 증거를 붙인다. 다른 입력 크기·projection·cache·동시성에도 적용되는지는 별도로 확인한다. 주 지표가 개선되지 않거나 정확성을 해치면 변경을 유지하지 않고 다음 가설로 돌아간다. [Active benchmarking](https://www.brendangregg.com/activebenchmarking.html).

## 7. 스레드 수나 부하를 바꾸기 전에 모델을 정한다

고정 작업량을 N개 worker로 나눌 때는 직렬 부분과 병렬 부분을 구분한다. 직렬 비율을 $f$라 하고 나머지가 추가 비용 없이 완전히 병렬화된다는 이상적 모델의 speedup은 다음과 같다.

$$
S(N) = \frac{1}{f + (1-f)/N}
$$

예를 들어 $f=0.1$, $N=8$이면 약 4.7배다. 실제 코드의 lock·통신·메모리 경쟁은 이 모델의 가정을 바꾼다. 한 번의 높은 동시성 profile에서 lock 비중을 읽어 곧바로 직렬 비율로 대입하지 않는다. 입력 크기를 늘리는 scaling과 고정 작업량의 scaling도 구분한다. [Amdahl의 원 논문](https://doi.org/10.1145/1465482.1465560).

안정된 부하와 같은 시스템 경계에서는 Little’s law로 평균 진행 중 요청 수 $L$, 처리율 $\lambda$, 평균 응답시간 $W$의 관계를 확인할 수 있다.

$$
L = \lambda W
$$

예를 들어 처리율이 초당 2,000건이고 평균 응답시간이 0.005초라면 평균 진행 중 요청은 10건이다. 여기에는 선택한 경계 안의 대기와 실행이 포함된다. 이 계산은 thread 수 권장값이나 p99 예측이 아니다. 같은 기간·요청 집합을 사용하고, 계속 적체가 커지는 구간과 정상 상태를 혼동하지 않는다. [Little와 Graves의 설명](https://link.springer.com/chapter/10.1007/978-0-387-73699-0_5).

실험에서는 worker 수나 도착률 중 한 변수를 바꾸며 처리량·latency·CPU/완료량·wait를 함께 그린다. plateau가 보이면 그 구간의 제한 요인을 조사한다. 그래프 모양만으로 lock이나 false sharing을 확정하지 않는다.

## 8. 다음 조사에 복사할 기록 양식

아래 양식에 원본 경로와 실제 값을 채운다. 실제 실험 기록은 [측정 가이드 7절](2026-10-28-linux-db-performance-survey.md#7-db-성능-실험의-최소-기록-형식)의 환경·수집 품질 항목도 포함한다.

```markdown
## 문제와 기준선

- 증상·주 지표·완료 작업량:
- baseline/candidate commit·binary 식별자:
- 입력·기대 결과·시작 상태·반복 순서:

## 자원과 측정 경계

- USE 관측·시간 구간·오류 변화·원본:
- unknown과 사유:
- 요청·thread·PID/cgroup·counter의 시작/완료:
- 명령·도구 버전·수집 누락·계측 on/off 영향:

## 가설과 구분 실험

- 현재 제한 요인 후보와 경쟁 가설:
- 지지 증거·반대 증거:
- 한 가지 변경과 정확성 조건:
- 맞을 때 변할 지표와 반증 조건:

## 결과와 다음 행동

- 개별 A/B 값·완료량·오류·원본:
- 정확성·목표 성능·tail 결과:
- 예측과 관측의 일치 또는 불일치:
- 해결·일부 설명·미확정 중 현재 상태:
- 적용 범위·남은 질문·다음 구분 실험:
```

## 9. 확인한 범위

문서의 상대 링크·절 제목, shell 예제의 문법, 가상 사례의 중앙값·증감률·행당 계수를 확인했다. 합성한 `perf script` 형식의 3개 샘플을 folded stack으로 변환하고, 합계 3을 유지한 SVG가 생성되는지도 확인했다. 변환에 사용한 FlameGraph revision은 `41fee1f99f9276008b7cd112fca19dc3ea84ac32`다.

이 확인에는 실제 `perf.data`의 해석, 호스트 수집 권한·PMU 지원, DB 성능 개선 검증이 포함되지 않는다. C++ 예제는 검증 위치 이동을 설명하는 의사 코드다.
