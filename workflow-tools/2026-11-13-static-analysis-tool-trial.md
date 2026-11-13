# CUBRID에 CodeQL·cppcheck·lizard·jscpd·PMD CPD 실사용 평가

실행 기준일은 **2026-11-13**, 대상은 CUBRID `develop` 커밋 `e64a04a79a`의 `src/` 디렉터리(C/C++ 약 1,180개 파일, CodeQL 기준 약 104만 줄)다. 실행 머신은 80코어·251GB 메모리다. 도구 목록과 역할은 [코드 이해·중복 탐지·리팩터링 도구](2026-11-13-code-intelligence-refactoring-tools.md)에 있다.

## 1. 결론

| 도구 | 버전 | 실행 시간 | 판정 | 한 줄 평가 |
| --- | --- | --- | --- | --- |
| lizard | 1.26.0 | 2초 | **바로 쓸 만함** | 복잡도·길이 순위가 즉시 나온다. 리팩터링 대상 선정과 PR 전후 비교에 적합 |
| PMD CPD | 7.29.0 | 10초 (식별자 무시 시 24초) | **바로 쓸 만함** | 중복 후보 품질이 가장 좋다. 문자셋 테이블 헤더를 빼야 한다 |
| jscpd | 5.4.3 | 6초 | 보조용 | 빠르고 JSON이 깔끔하지만 기본 설정에서 헤더(`.h`, `.hpp`)를 보지 않았다 |
| cppcheck | 2.22.0 | 14초 | **변경 파일 검사용으로 유용** | 실제 버그를 일부 잡지만 잡음이 많다. 규칙 몇 개만 골라 쓰면 좋다 |
| CodeQL | 2.27.4 | DB 1.5~10분, 분석 1~3시간 후 실패 | **쿼리를 골라서만 사용** | 버그의 질은 가장 좋지만, 전체 suite는 이 머신(80코어·251GB)에서 끝나지 않았다 |

이번 평가에서 도구가 찾은 실제 결함은 다음과 같다.

| 결함 | 찾은 도구 | 상태 |
| --- | --- | --- |
| `convert_db_value_to_string`의 `std::string (NULL)` → csql 비정상 종료 | cppcheck | 재현, [CBRD-28080](https://jira.cubrid.org/browse/CBRD-28080) |
| `sm_count_tokens` 문자열 끝 너머 읽기 | CodeQL | 코드로 확인, 미재현, 미등록 |
| `type_checking.c:20059` 괄호 누락으로 오류 코드 소실 | CodeQL | 코드로 확인, 미등록 |
| `double_write_buffer.cpp` `dwb_log` 포맷 인자 부족 2건 | CodeQL | 코드로 확인, 미등록 |
| `broker_util.c` assert만 있는 NULL 검사 | cppcheck | 코드로 확인, 경미 |

추천 조합: **lizard + PMD CPD로 리팩터링 후보를 고르고, cppcheck를 변경 파일에만 돌리고, CodeQL은 선별한 버그 쿼리만 주기적으로 돌린다.**

## 2. 설치

sudo 없이 사용자 영역에 설치했다. 모두 `~/.local/bin`에서 실행된다.

| 도구 | 설치 방법 | 참고 |
| --- | --- | --- |
| lizard | `uv tool install lizard` | Python |
| jscpd | `npm i -g jscpd` | mise의 Node |
| PMD CPD | GitHub release zip을 `~/.local/opt/pmd-bin-7.29.0`에 풀고 `bin/pmd` 링크 | 기존 Java 8로 실행됨 |
| cppcheck | 소스 tarball을 CMake+Ninja로 빌드, `~/.local/opt/cppcheck`에 설치 | 빌드 1~2분 |
| CodeQL | `codeql-bundle-linux64.tar.zst`(약 560MB)를 `~/.local/opt/codeql`에 풀기 | 쿼리 팩 포함 bundle |

## 3. lizard: 함수 복잡도와 길이

```bash
lizard src -l cpp -t 40 --csv > lizard.csv
```

- 2초, 메모리 44MB. 함수 25,111개를 측정했다.
- CCN(순환 복잡도) 50 초과 376개, 100 초과 96개, 500줄 초과 114개.

| CCN | 줄 수 | 함수 |
| --- | --- | --- |
| 1095 | 5201 | `pt_evaluate_db_value_expr` (`parser/type_checking.c`) |
| 624 | 2143 | `pt_to_regu_variable` (`parser/xasl_generation.c`) |
| 493 | 1722 | `pt_eval_expr_type` (`parser/type_checking.c`) |
| 446 | 869 | `tp_value_find_converter` (`object/object_domain_convert.cpp`) |
| 275 | 1284 | `qexec_execute_mainblock_internal` (`query/query_executor.c`) |

평가:

- 가장 가볍고 결과를 바로 읽을 수 있다. CSV에 파일·함수·시작/끝 줄이 있어서, 다른 도구의 줄 번호를 함수 이름으로 바꾸는 데도 썼다(아래 CPD 표).
- 거대한 `switch`가 많은 파서·타입 검사 코드가 상위를 차지한다. 숫자만으로 "나쁜 코드"라고 단정하면 안 되고, 수정 위험도와 테스트 범위를 가늠하는 지표로 쓴다.
- 활용: PR에서 바뀐 함수의 CCN 증감 표시, 리팩터링 전후 비교.

## 4. 중복 탐지: PMD CPD와 jscpd

```bash
# 정확한 복사본
pmd cpd --dir src --language cpp --minimum-tokens 150 --format csv --no-fail-on-violation --no-fail-on-error
# 이름·리터럴만 다른 복사본 (테이블 헤더 제외 목록 사용)
pmd cpd --file-list files.txt --language cpp --minimum-tokens 150 \
  --ignore-identifiers --ignore-literals --format csv --no-fail-on-violation --no-fail-on-error
# jscpd
jscpd src --format "c,cpp" --min-tokens 100 --reporters json --output out/
```

| 실행 | 시간 | 최대 메모리 | 결과 |
| --- | --- | --- | --- |
| CPD 기본 | 10초 | 1.6GB | 중복 그룹 277개 |
| CPD 식별자·리터럴 무시, 전체 `src` | 약 11분 | 12GB | 그룹 3,016개, 상위는 대부분 문자셋 테이블 |
| CPD 식별자·리터럴 무시, `jisx0212.h`·`ksc5601.h`·`CRC.h` 제외 | 24초 | 6.4GB | 그룹 2,443개, 상위가 실제 코드 |
| jscpd | 6초 | 3.4GB | 클론 5,039개, 중복 5.9% (줄 기준). `.c`·`.cpp` 557개 파일만 검사 |

두 도구가 공통으로 찾은 대표 후보(함수 이름은 lizard CSV로 매핑):

| 후보 | 위치 | 성격 |
| --- | --- | --- |
| LIKE 평가 함수 | `broker/cas_str_like.c` `str_eval_like` ↔ `method/method_query_util.cpp` `cubmethod::str_eval_like` | 모듈을 넘은 복사본. 공통화 1순위 |
| CAS 요청 처리 | `broker/cas.c` ↔ `broker/cas_cgw.c` `process_request` | 172줄 복사 |
| 정렬 merge | `storage/external_sort.c` `sort_exphase_merge_elim_dup` ↔ `sort_exphase_merge` | 같은 파일 안 변형 |
| B-tree 정렬 입력 | `storage/btree_load.c` `btree_sort_get_next_parallel` ↔ `btree_sort_get_next` | 병렬 버전 복사 |
| 뷰 변환 lambda | `parser/view_transform.c` `mq_class_lambda` ↔ `mq_inline_view_lambda` | 여러 구간 반복 |
| 날짜 변환 | `query/string_opfunc.c` `db_to_timestamp` ↔ `db_to_datetime` | 234줄 |
| schema info | `method/method_schema_info.cpp` ↔ `method/method_struct_schema_info.cpp` | 150줄 |

식별자 무시 모드에서만 나온 후보:

- `parser/parse_tree_cl.c` ↔ `query/query_dump.c` (약 400줄, 출력 함수 쌍)
- `storage/btree_sort.c` ↔ `storage/external_sort.c` (274줄)
- `object/object_domain_convert.cpp` 내부 반복 구간 (500줄)

평가:

- **CPD**가 더 유용했다. 식별자 무시 옵션으로 "변수명만 바꾼 복사본"을 찾는다. 다만 C 문자셋 테이블처럼 숫자 배열이 큰 파일이 있으면 매칭이 폭발한다. 그래서 테이블 파일을 빼면 11분이 24초로 줄었다.
- PMD 7.29의 `--exclude`에 절대 경로를 줬지만 효과가 없었다. `--file-list`로 대상 목록을 직접 주는 방식이 확실했다.
- CPD는 `#if` 블록이 닫히지 않는 일부 헤더에서 lexical error를 내고 건너뛴다(`px_interrupt.hpp` 등 4개). 결과에는 큰 영향이 없다.
- **jscpd**는 결과 JSON이 에이전트에게 넘기기 좋고 중복 비율 통계를 준다. 그러나 `--format "c,cpp"`만으로는 `.h`, `.hpp`를 검사하지 않았다. 헤더까지 보려면 확장자 매핑 설정이 필요하다. CPD와 후보가 거의 겹치므로 둘 중 하나만 쓰면 된다.

## 5. cppcheck

`build_preset_debug_gcc`의 `compile_commands.json`을 그대로 사용했다.

```bash
cppcheck --project=compile_commands.json --enable=warning,performance,portability \
  --inline-suppr -j 60 --quiet --cppcheck-build-dir=out/ --xml --output-file=all.xml
```

- 전체 14초, `src/storage`만 7초. 결과 831건, 136개 파일. `cubrid-cci` 서브모듈도 포함됐다.
- 건수 상위는 잡음에 가깝다: `uninitMemberVarNoCtor` 390, `dangerousTypeCast` 162, `uninitMemberVar` 69, `assertWithSideEffect` 30.

직접 확인한 결과:

| 위치 | 규칙 | 판정 |
| --- | --- | --- |
| `method/method_query_util.cpp:473` | `nullPointer` | **실제 결함, 재현됨 → [CBRD-28080](https://jira.cubrid.org/browse/CBRD-28080).** `std::string (NULL)` 생성(정의되지 않은 동작) |
| `broker/broker_util.c:632` | `nullPointerRedundantCheck` | **실제지만 경미.** NULL 검사가 `assert`뿐이라 release 빌드에서는 그대로 역참조 |
| `storage/bestspace.cpp:1936` | `uninitvar` | 이론상 맞음. shard 수가 0일 때만 발생하므로 실질 영향 없음 |
| `transaction/mvcc_table.cpp:642` | `identicalInnerCondition` | **오탐.** atomic 값을 다시 읽는 double-check 패턴 |
| `base/object_representation_sr.h:127` | `memsetClass` | **오탐.** `std::atomic` 멤버 앞까지만 의도적으로 `memset` |
| `object/schema_*.cpp` 등 | `missingFile` | compile DB가 현재 소스와 맞지 않음(오래된 빌드 디렉터리) |

CBRD-28080 경위: cppcheck는 "`db_value_coerce` 실패 시 NULL"이라는 경로를 지적했다. 별도 에이전트가 호출 경로를 추적한 결과, 실제로 터지는 경로는 다른 곳이었다. 컬렉션 원소에 NULL이 있으면 VARCHAR 변환은 성공하지만 `db_get_char`가 NULL을 돌려준다. 아래 두 문장으로 csql이 `basic_string::_M_construct null not valid`로 비정상 종료한다(서버는 유지).

```sql
CREATE TABLE cbrd_nullstr_t (c SET(INT) DEFAULT {1, NULL});
CREATE OR REPLACE PROCEDURE cbrd_nullstr_p() AS v cbrd_nullstr_t.c%TYPE; BEGIN NULL; END;
```

옛 broker 쪽 원본(`cas_execute.c`)에는 NULL 검사가 있었고, C++로 복사하면서 빠졌다. 중복 탐지에서 나온 `cas_str_like.c` ↔ `method_query_util.cpp` 복사 관계와 같은 파일이다. **도구가 가리킨 줄은 맞았고, 원인은 사람이(또는 에이전트가) 추적해야 했다.**

평가:

- 빠르고 compile DB를 바로 읽는다. 빌드 없이도 돌릴 수 있다.
- 전체 결과를 사람이 보기에는 잡음이 많다. 동시성 코드는 오탐이 나기 쉽다.
- 쓸 만한 규칙: `nullPointer`, `ctunullpointer`, `uninitvar`, `invalidPrintfArgType_*`, `nullPointerRedundantCheck`. 이 규칙만 걸러서 **PR 변경 파일**에 돌리는 방식이 적합하다.
- compile DB를 최신 빌드로 갱신한 뒤 써야 `missingFile`이 사라진다.

## 6. CodeQL

CUBRID 빌드를 tracing하지 않고 `--build-mode=none`(소스만 읽는 방식)으로 DB를 만들었다.

```bash
codeql database create db --language=cpp --build-mode=none --source-root=src --threads=40
codeql database analyze db codeql/cpp-queries:codeql-suites/cpp-code-scanning.qls \
  --format=sarif-latest --output=results.sarif --threads=40 --ram=96000
```

### 첫 시도: 실패에서 배운 것

처음에는 `--source-root=.`(worktree 전체)로 만들었다.

| 단계 | 시간 | 최대 메모리 | 결과 |
| --- | --- | --- | --- |
| DB 생성 (worktree 전체) | 9분 50초 | 29GB | 2.0GB DB, 약 104만 줄 |
| `security-and-quality` 183개 쿼리 | 1시간 9분 | 78GB | 127개 완료 후 **내부 오류로 중단** |
| `code-scanning` 61개 쿼리 (같은 DB) | 53분 | 76GB | **같은 오류로 중단** |

오류는 `java.lang.IllegalArgumentException: Maximum array length exceeded`였다. transitive closure 계산이 Java 배열 한계(약 21억 원소)를 넘었다. 원인은 범위다. worktree 전체를 잡으면서 `build_preset_debug`, `build_preset_debug_gcc` 두 빌드 디렉터리의 TBB 소스·테스트와 생성된 `csql_grammar.c`가 **두 벌씩** 들어갔다. 실제로 완료된 쿼리 결과의 상당수가 이 파일들을 가리켰다.

**교훈: CodeQL은 반드시 `src`처럼 범위를 좁혀서 만든다.** 빌드 디렉터리와 서드파티 코드를 넣으면 시간·메모리가 폭증하고 잡음이 늘어난다.

### 완료된 127개 쿼리에서 확인한 결과

건수가 많은 쿼리는 대부분 잡음이었다: `MistypedFunctionArguments` 97,919건(build-mode none에서 매크로·정의를 정확히 모르는 영향으로 보임), `LocalVariableHidesGlobalVariable` 12,305건, 미사용 변수·함수 수천 건, `StackAddressEscapes` 1,760건(상위가 TBB 헤더).

건수가 적은 버그 쿼리에서 실제 결함이 나왔다.

| 위치 | 쿼리 | 판정 |
| --- | --- | --- |
| `object/schema_manager.c` `sm_count_tokens` | `NestedLoopSameVar` | **실제 결함(미재현).** 문자열이 공백 없이 토큰으로 끝나면, 안쪽 루프가 `\0`에서 멈춘 뒤 바깥 루프의 `i++`가 종료 문자를 지나 버퍼 밖을 읽는다. `class_->loader_commands`를 나눌 때 호출된다 |
| `parser/type_checking.c:20059` | `AmbiguousAssignmentOfComparison` | **실제 결함.** `err = pt_set_default_data_type (...) < 0` 괄호 누락으로 `err`에 오류 코드 대신 `1`이 들어가 반환된다. 음수로 오류를 판별하는 호출자는 실패를 놓친다 |
| `object/trigger_manager.c:5579` | `NestedLoopSameVar` | 의도된 코드. 안쪽 루프가 바깥 루프의 `t`를 이어서 진행한다 |
| `base/hide_password.cpp:715`, `broker/broker_log_replay.c:1168` | `SnprintfOverflow` | 확인 필요. `snprintf` 반환값을 다음 크기 계산에 그대로 쓴다 |
| `heaplayers/lea_heap.c:227` | `SizeCheck2` | 서드파티 할당기 코드, 낮은 우선순위 |

평가(첫 시도 기준):

- 다른 도구가 못 본 **의미 수준의 버그**(연산자 우선순위, 루프 변수 공유)를 잡았다. cppcheck에 없는 결과다.
- 비용이 크다. 범위를 잘못 잡으면 70GB 넘는 메모리와 한 시간을 쓰고 실패한다.
- 결과는 전체 목록이 아니라 **건수가 적은 버그·보안 쿼리 위주로** 보는 것이 효율적이다.

### 두 번째 시도: `src`만, 기본 `code-scanning` 61개 쿼리

| 단계 | 시간 | 최대 메모리 | 결과 |
| --- | --- | --- | --- |
| DB 생성 (`--source-root=src`) | 1분 33초 | 16GB | 파일 1,172개 |
| `code-scanning` 61개 쿼리 | 3시간 17분 | 78GB | **56개 완료**, 데이터 흐름 쿼리 4개가 결과 2GiB 한도 초과(`CleartextFileWrite`, `UncontrolledFormatString`, `ExposedSystemData`, `CleartextTransmission`), 이후 메모리 부족으로 종료. DB가 캐시 포함 30GB로 커짐 |

DB 생성은 빨라졌지만, 오염 데이터 흐름(taint tracking) 쿼리는 `src`만으로도 감당하지 못했다. CodeQL이 매긴 DB 품질도 한계를 보여준다. 함수의 90%, 코드 줄의 94%는 오류 없이 추출됐지만, **컴파일 단위 기준으로는 33%만 오류가 없었다.** build-mode none은 실제 빌드의 매크로·include 설정을 모르기 때문이다.

완료된 56개 쿼리의 결과는 첫 시도보다 훨씬 적고 정확했다(잡음 쿼리가 기본 suite에 없음).

| 쿼리 | 건수 | 확인 결과 |
| --- | --- | --- |
| `WrongNumberOfFormatArguments` | 2 | **실제 결함.** `storage/double_write_buffer.cpp:1426`, `:3290`의 `dwb_log`가 `%d` 세 개에 인자 두 개. DWB 디버그 로그를 켤 때만 영향 |
| `RedundantNullCheckSimple` | 7 | `lock_free.c`, `view_transform.c`. 역참조 뒤 NULL 검사, 확인 필요 |
| `IntMultToLong` | 9 | `dynamic_array.c`, `binaryheap.c` 등 `int` 곱셈 후 `size_t` 변환. 큰 입력에서만 의미 |
| `TOCTOUFilesystemRace` | 8 | `ddl_log.c`, `broker.c` 등. 대부분 관리 도구 경로 |
| `ComparisonWithWiderType` | 57 | 루프 변수 폭 문제. 일부는 무한 루프 가능성 확인 필요 |
| `PointerOverflow` | 2 | `query/vacuum.c:5026`. 일반적인 끝 포인터 비교라 오탐으로 보임 |

### CodeQL 평가

- **찾는 버그의 질이 가장 좋다.** 괄호 누락(`type_checking.c`), 버퍼 끝을 넘는 루프(`sm_count_tokens`), 포맷 인자 개수 불일치(`dwb_log`)는 다른 네 도구가 잡지 못했다.
- **비용이 가장 크고 불안정하다.** 이 80코어·251GB 머신에서도 전체 suite를 한 번도 끝까지 돌리지 못했다. 데이터 흐름 보안 쿼리는 build-mode none + CUBRID 규모에서 결과가 폭발한다.
- 쓸 만한 방식:
  - 쿼리를 골라 돌린다. `Likely Bugs` 중 건수가 적은 쿼리(`AmbiguousAssignmentOfComparison`, `NestedLoopSameVar`, `WrongNumberOfFormatArguments`, `RedundantNullCheckSimple`)만 지정하면 빠르게 끝난다.
  - 정확도가 필요하면 build-mode none 대신 실제 빌드를 tracing한다. ccache를 끄고 clean build를 해야 컴파일러 호출이 기록된다.
  - "latch를 잡고 release 없이 return" 같은 CUBRID 전용 질의는 직접 쓴 쿼리를 하위 디렉터리 DB에 돌리는 방식이 현실적이다. 이번 평가에서는 시도하지 않았다.

## 7. 도입 제안

| 용도 | 도구와 방식 | 비용 |
| --- | --- | --- |
| 리팩터링 후보 선정 | lizard CSV + PMD CPD(식별자 무시, 문자셋 테이블 제외) 결과를 함수 단위로 합쳐 에이전트에게 전달 | 1분 이내 |
| PR 검증 | 변경된 `.c`·`.cpp`에만 cppcheck, 고신뢰 규칙만 표시 (`nullPointer`, `ctunullpointer`, `uninitvar`, `invalidPrintfArgType_*`, `nullPointerRedundantCheck`) | 수 초 |
| 주기적 버그 탐색 | CodeQL `src` DB에 선별 쿼리만 실행. DB는 주 1회 정도 새로 만든다 | DB 1.5분 + 쿼리 수 분(추정) |
| 중복 제거 실행 | CPD 후보 → Serena·clangd로 호출부 탐색 → Coccinelle·ast-grep 교체 → 빌드·테스트 | 작업 단위 |

운영 주의:

- 모든 도구에서 빌드 디렉터리(`build_preset_*`), `3rdparty`, 생성 코드, 문자셋 테이블(`jisx0212.h`, `ksc5601.h`, `CRC.h`)을 빼야 한다. 빼지 않으면 잡음이 늘고 CPD·CodeQL은 시간과 메모리가 폭증한다.
- cppcheck와 CodeQL의 결과는 **위치는 맞아도 원인 설명은 틀릴 수 있다**(CBRD-28080). 결과를 바로 고치지 말고 호출 경로와 재현으로 확인한다.
- 동시성 코드(atomic double-check, lock-free)는 두 정적 분석기 모두 오탐이 많다.

다음 단계 후보:

1. CodeQL 선별 쿼리 세트를 `.qls` 파일로 만들어 `src` DB에 돌리고, 실제 실행 시간을 잰다.
2. 위 표의 미등록 결함(`sm_count_tokens`, `type_checking.c:20059`, `dwb_log`)을 확인해 이슈로 만들지 정한다.
3. 변경 파일 대상 cppcheck를 hook이나 skill로 만든다.

## 8. 후속 작업 결과 (같은 날)

위 다음 단계 후보를 모두 진행했다. 각 작업은 별도 에이전트가 맡았고, 결함마다 재현을 먼저 시도한 뒤 중복 검사를 거쳐 이슈로 등록했다.

### 만든 도구

| 도구 | 위치 | 내용 |
| --- | --- | --- |
| CodeQL 선별 suite | `my-cubrid/static-analysis/codeql/` (`cubrid-bugs.qls`, `run.sh`) | 쿼리 59개. `src` DB 생성 약 2분, 분석 중앙값 3분 44초, 최대 메모리 약 35GiB. 결과 30건 중 19건이 실제 결함 |
| `cubrid-static-check <base>` | `my-cubrid/static-analysis/static-check/`, `just sync`로 설치 | 변경이 새로 추가한 cppcheck·clang-tidy `bugprone-*` 결과, 새 CPD 복제, lizard 복잡도 증가만 보고. 보통 PR 15~60초. `cubrid-code-review`, `cubrid-pr-create` skill에 선택 단계로 연결 |
| format attribute 측정 | `my-cubrid/static-analysis/format-attr/` | 로깅 함수 29개에 printf format attribute를 붙였을 때의 경고 분류. 218줄 중 90줄이 실제 불일치 |
| clone drift 감사 | `my-cubrid-docs/code-analysis/clone-drift/` | CPD와 같은 이름 함수 쌍 탐지를 함께 써서, 한쪽 복사본에만 들어간 수정을 찾는다 |
| ASan preset | `my-cubrid` `CMakeUserPresets.json` (`asan_debug_gcc`, `asan_noscope_debug_gcc`) | `use_system_malloc=yes`와 함께 쓰면 CUBRID 자체 할당기를 우회해 범위 밖 읽기를 잡는다 |

### 등록한 이슈

| 이슈 | 결함 | 찾은 방법 | 재현 |
| --- | --- | --- | --- |
| [CBRD-28080](https://jira.cubrid.org/browse/CBRD-28080) | PL/CSQL `%TYPE` + NULL 원소 컬렉션 DEFAULT → `std::string (NULL)`로 csql 비정상 종료 | cppcheck | SQL 두 문장으로 재현 |
| [CBRD-28083](https://jira.cubrid.org/browse/CBRD-28083) | `pt_coerce_value_internal` 괄호 누락으로 오류 코드 대신 1 반환 | CodeQL | gdb 오류 주입 |
| [CBRD-28084](https://jira.cubrid.org/browse/CBRD-28084) | `sm_count_tokens` 문자열 끝 너머 읽기 | CodeQL | 재현, ASan으로도 확인 |
| [CBRD-28090](https://jira.cubrid.org/browse/CBRD-28090) | `cubrid broker info`의 `REJECT_CLIENT_FLAG` 값이 낡은 값으로 표시 | CodeQL(후보 단계 쿼리) | 재현 |
| [CBRD-28091](https://jira.cubrid.org/browse/CBRD-28091) | `catcls_get_db_collation` realloc 뒤 잘못된 포인터 검사 | CodeQL | gdb 오류 주입 |
| [CBRD-28092](https://jira.cubrid.org/browse/CBRD-28092) | `db_macro.c` `%.17g`가 24바이트 버퍼를 1바이트 넘침 | CodeQL | gdb watchpoint로 확인 |
| [CBRD-28093](https://jira.cubrid.org/browse/CBRD-28093) | `REPLACE`에서 `db_is_deleted` 오류 코드 소실 | CodeQL | gdb 오류 주입 |
| [CBRD-28094](https://jira.cubrid.org/browse/CBRD-28094) | 디버그 로그 format 인자 불일치 16곳, `LOG_THREAD_TRAN_ARGS`가 trid 대신 포인터 출력 | CodeQL + format attribute | 빌드 경고로 확인. 수정 커밋은 로컬에만 있음 |
| [CBRD-28095](https://jira.cubrid.org/browse/CBRD-28095) | `coll_name` 1바이트 넘침 | CBRD-28091 조사 중 발견 | gdb 오류 주입 |
| [CBRD-28096](https://jira.cubrid.org/browse/CBRD-28096) | `xts_save_packable_object` 4바이트 heap 범위 밖 읽기 (PL/CSQL 호출마다) | ASan | 재현 |
| [CBRD-28097](https://jira.cubrid.org/browse/CBRD-28097) | Java SP·PL/CSQL 동적 SQL의 `?` 개수 세기가 `no_backslash_escapes` 무시 | clone drift 감사 | 재현 |
| [CBRD-28098](https://jira.cubrid.org/browse/CBRD-28098) | `pl_status_server`가 `new[]` 메모리를 `free`로 해제 | ASan | 재현 |

기존 이슈 보강: [CBRD-24699](https://jira.cubrid.org/browse/CBRD-24699)에 루프 변수 재사용 분석과 재현 시도 결과를, [CBRD-27875](https://jira.cubrid.org/browse/CBRD-27875)에 두 문장짜리 결정적 재현을 댓글로 남겼다.

### 배운 점

- **도구가 가리킨 줄은 맞아도 원인 설명은 틀릴 수 있다.** CBRD-28080은 cppcheck가 지적한 경로가 아니라 다른 경로로 터졌다. 결과는 호출 경로 추적과 재현으로 확인해야 한다.
- **도구를 섞어 쓰면 서로 보완된다.** CPD는 C++로 다시 쓴 복사본(CBRD-28080, CBRD-28097)을 놓쳤고, 같은 이름 함수 쌍 탐지가 찾았다. ASan은 valgrind가 못 본 범위 밖 읽기(CBRD-28084)를 `use_system_malloc=yes`와 함께 잡았다.
- **ASan 빌드 자체도 검증해야 한다.** 처음 ASan preset은 컴파일에 linuxbrew `as` 2.47, 링크에 시스템 `ld` 2.35.2를 써서, 링커가 `push`를 `call`로 잘못 바꿨다. 그 결과 서버가 가짜 assert로 죽었고, 하마터면 CUBRID 결함으로 등록할 뻔했다. preset을 같은 binutils로 링크하도록 고친 뒤 다시 돌려 기존 결과를 확인했다.
- **중복 검사가 실제로 걸렀다.** `query_graph.c` 결함은 이미 CBRD-24699로 등록돼 있었다.
