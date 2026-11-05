# TLA+ 주변 도구와 실무 대안 비교

2026-11-05 KST 기준. 공식 문서, 도구 개발자가 쓴 사례, 실제 사용 프로젝트의 저장소를 확인했다. 아래 추천은 검증 대상과 보장 범위에 따른 판단이다. GitHub 별 수는 관심의 누적 지표이고, 최근 커밋은 유지보수의 한 단서다. 어느 쪽도 검증 성능이나 산업 보급률을 직접 보여주지 않는다.

분산 시스템 설계를 검토한다면 TLA+와 TLC를 비교 기준으로 두고, 작성 경험을 바꾸고 싶을 때 Quint를 먼저 살펴볼 만하다. 메시지를 주고받는 상태 머신이 자연스러운 팀에는 P가 유력하다. Python 계열 문법과 자체 검사기를 원하면 FizzBee가 후보가 된다. 이 판단은 각 도구의 모델과 작업 방식에 따른 것이며, 모든 문제에서 우월한 도구가 있다는 뜻은 아니다. [Quint 검사기 설명](https://quint.sh/docs/model-checkers), [P 소개](https://p-org.github.io/P/whatisP/), [FizzBee 문서](https://fizzbee.io/)

## GitHub 현황

GitHub 통계는 2026-11-05 KST에 API로 수집했다. 최근 커밋은 저장소의 기본 브랜치에서 확인한 committer 날짜를 한국 시간으로 바꾼 값이다. 릴리스 날짜나 이슈 활동을 포함하는 `updated_at`과 다르며, 문서·CI·병합 커밋도 포함한다.

| 도구 | 부류 | GitHub stars | 기본 브랜치 최근 커밋 KST |
| --- | --- | ---: | --- |
| [TLA+ / TLC](https://github.com/tlaplus/tlaplus) | 설계 모델 | 3,192 | [2026-11-04](https://github.com/tlaplus/tlaplus/commit/309e489a512e8017f1cdc8388fa6a5e8a01cf597) |
| [Apalache](https://github.com/apalache-mc/apalache) | 설계 모델 | 623 | [2026-10-12](https://github.com/apalache-mc/apalache/commit/9a13950e0ebddcb3d7a79cb9e854b0866e1c5712) |
| [Quint](https://github.com/quint-co/quint) | 명세 언어와 도구 | 1,917 | [2026-11-04](https://github.com/quint-co/quint/commit/07d7a1a69c2093b5bc656caf3b5dfefa0d766cd0) |
| [P](https://github.com/p-org/P) | 상태 머신과 검사 도구 | 3,728 | [2026-08-04](https://github.com/p-org/P/commit/260da928f67569ca7a8c0039a02d0bad7cde6096) |
| [SPIN](https://github.com/nimble-code/Spin) | 설계 모델 | 479 | [2026-11-04](https://github.com/nimble-code/Spin/commit/e21106ff7e58afe79275b3ca10179556afa97aa2) |
| [Alloy 6](https://github.com/AlloyTools/org.alloytools.alloy) | 관계·구조·동작 모델 | 884 | [2026-06-11](https://github.com/AlloyTools/org.alloytools.alloy/commit/ed89fdb16c58aea17a55e312c44f99e31aa63ee1) |
| [FizzBee](https://github.com/fizzbee-io/fizzbee) | 설계 모델과 테스트 | 370 | [2026-11-05](https://github.com/fizzbee-io/fizzbee/commit/9ed474e3343711479d024598d75e227ec20ab4a8) |
| [Veil](https://github.com/verse-lab/veil) | 모델과 안전성 증명 | 354 | [2026-11-05](https://github.com/verse-lab/veil/commit/0a20c4c7154bdb020bda4404e5051e4298a06d92) |
| [CBMC](https://github.com/diffblue/cbmc) | C와 C++ 구현 | 1,161 | [2026-11-05](https://github.com/diffblue/cbmc/commit/8e018d0c0ef6767172dbbeaa7dcf4abe51c9eda3) |
| [Verus](https://github.com/verus-lang/verus) | Rust 구현과 증명 | 3,559 | [2026-11-05](https://github.com/verus-lang/verus/commit/1c5a2778d75b758794f5e1bb162d48a236c0b8a0) |
| [Kani](https://github.com/model-checking/kani) | Rust 구현 | 3,641 | [2026-11-04](https://github.com/model-checking/kani/commit/47689dc3ac222e6aefed440e69b55d0c192a6337) |
| [Dafny](https://github.com/dafny-lang/dafny) | 검증 지향 언어 | 3,613 | [2026-11-03](https://github.com/dafny-lang/dafny/commit/8c137ba351c60a521cfb122011770fdbea9a0453) |
| [Lean 4](https://github.com/leanprover/lean4) | 범용 증명 보조기 | 9,884 | [2026-11-05](https://github.com/leanprover/lean4/commit/14ad5bae33b5f136efd92ef5322dae0904111841) |
| [TLA++](https://github.com/zoratu/tlaplusplus) | Rust TLA+ 검사기 | 4 | [2026-10-12](https://github.com/zoratu/tlaplusplus/commit/ff7ce8f247a6a660fe746108c84b45b673fad3fd) |
| [tla-rs](https://github.com/fabracht/tla-rs) | Rust TLA+ 검사기 | 80 | [2026-10-29](https://github.com/fabracht/tla-rs/commit/9438572f8278df75427460597ca48b0e0026684c) |

원본 수치, 기본 브랜치, 커밋 SHA와 제목은 [GitHub API 스냅숏](2026-11-05-github-tool-snapshot.json)에 보존했다. Rust로 작성한 TLA+ 검사기 TLA++와 tla-rs도 비교를 위해 포함했다.

## 검증 대상과 보장 범위

| 도구 | 검증 대상과 방식 | 결과를 해석할 때의 조건 |
| --- | --- | --- |
| TLA+와 TLC | 상태, 전이, 불변식, 시간 속성으로 표현한 설계를 명시적 상태 탐색으로 검사한다. | 탐색을 완료하면 설정한 유한 모델의 도달 가능한 상태와 관련 동작을 검사한 것이다. 노드 수나 자료 범위를 바꾼 모든 모델에 대한 정리는 아니다. [TLC 모델의 의미](https://lamport.azurewebsites.net/tla/model-popup.html), [TLC 설명](https://lamport.azurewebsites.net/tla/book-02-03-04.pdf) |
| Apalache | TLA+와 Quint의 검증 문제를 SMT 제약으로 바꾸는 기호적 검사기다. | 보통은 정한 길이까지의 실행을 검사한다. 별도로 귀납적 불변식도 검사하므로, 모든 사용법을 단순한 깊이 제한 탐색으로 설명하면 부정확하다. 일반적으로 매개변수는 고정되거나 제한된다. [공식 README](https://github.com/apalache-mc/apalache) |
| Quint | 타입이 있는 명세 언어, REPL, 테스트, 무작위 시뮬레이터를 제공한다. `verify`는 Apalache 또는 TLC를 호출한다. | 독립적인 새 검증 엔진과 구분해야 한다. `run`은 표본 탐색이다. 기본 Apalache 검사는 `--max-steps` 제한이 있고, TLC는 `--backend=tlc`로 선택한다. [모델 검사기](https://quint.sh/docs/model-checkers), [CLI](https://quint.sh/docs/quint) |
| P | 비동기 이벤트로 통신하는 상태 머신과 속성 모니터를 작성한다. 기본 검사기는 실행 스케줄을 탐색한다. | 정해진 스케줄 수를 실행한 `p check` 성공을 전체 상태 공간의 증명으로 해석하면 안 된다. 별도 PEx는 유한 인스턴스의 완전 탐색을 목표로 하지만, 실행 깊이와 종료 사유도 확인해야 한다. [기본 검사기](https://p-org.github.io/P/getstarted/usingP/), [PEx](https://p-org.github.io/P/advanced/pex/) |
| SPIN과 Promela | 동시 실행 프로세스, 채널, 단언, LTL 속성을 검사한다. | 완전 탐색과 메모리를 줄이기 위한 bitstate 탐색을 구분해야 한다. bitstate는 상태를 놓칠 수 있으므로 오류가 없다는 결과의 의미가 다르다. [SPIN 탐색 방식](https://spinroot.com/spin/Man/Quick.html) |
| Alloy 6 | 집합과 관계의 제약을 표현한다. 6부터 가변 상태와 시간 논리를 직접 지원한다. | 객체 범위가 제한된다. 기본 시간 범위는 10단계지만, `for 1.. steps`와 NuSMV 또는 nuXmv를 쓰면 그 유한 상태 공간에서 시간 길이를 미리 제한하지 않는 완전 검사가 가능하다. [Alloy 6](https://alloytools.org/alloy6.html) |
| FizzBee | Python과 비슷한 명세로 상태, 동시 실행, 실패를 모델링하고 자체 검사기로 탐색한다. | 전체 검사는 작은 설정으로 완료해야 한다. 시뮬레이션이나 `max_actions` 제한에서 성공한 결과와 구분해야 한다. 공식 가이드도 상한이 무한 경로를 잘라낼 수 있다고 경고한다. [검증 가이드](https://fizzbee.io/reference/verification/) |
| CBMC | C와 C++ 구현의 메모리 안전, 정의되지 않은 동작, 사용자 단언을 검사한다. 루프를 전개해 SAT 또는 SMT로 푼다. | 별도 설계 모델보다 실제 함수 코드에 가깝다. 입력 가정과 루프 상한이 검증 범위를 결정한다. unwinding assertion으로 전개가 충분했는지도 확인해야 한다. [CBMC](https://www.cprover.org/cbmc/), [루프 전개 설명](https://model-checking.github.io/cbmc-training/faq/loop-unwinding.html) |

## 산업 적용 사례

"업계 표준"은 분야를 붙여서 판단하는 편이 정확하다. AWS 엔지니어들은 2011년부터 TLA+를 사용한 경험을 논문으로 공개했고, 이후에도 실무 경험을 발표했다. 따라서 분산 시스템의 설계 검증에서 TLA+와 TLC는 충분한 산업 사례가 있는 비교 기준이다. 이 사실만으로 모든 회사에서 기본 도구라고 단정할 수는 없다. [AWS의 2015년 논문](https://www.amazon.science/publications/how-amazon-web-services-uses-formal-methods), [AWS의 2024년 회고 발표](https://conf.tlapl.us/2024/)

P는 현재의 산업 사례를 확인하기 좋은 대안이다. 공식 사례 문서는 S3 강한 일관성 설계의 핵심 분산 프로토콜을 P로 모델링했다고 설명하며, EBS, DynamoDB, Aurora, EC2의 팀도 명시한다. 또한 PObserve는 서비스 로그를 P 모니터에 넣어 실행 중 관찰한 동작을 검사한다. 로그 검사는 관찰하지 않은 모든 실행까지 증명하는 기능과 구분해야 한다. [P 산업 사례](https://p-org.github.io/P/casestudies/), [PObserve 소개](https://p-org.github.io/P/whatisP/)

SPIN에는 NASA 우주 임무의 알고리즘, Lucent PathStar 스위치, 네덜란드 홍수 방벽의 제어 알고리즘 사례가 있다. 이는 오래 검증된 도구라는 근거다. 상당수가 과거 사례이므로 현재의 인기 상승을 보여주는 자료로 쓰기는 어렵다. Alloy는 관계 제약과 구조적 설계를 분석하는 용도로 비교할 가치가 있고, 공식 사이트에 보안 정책과 웹 프로토콜을 포함한 사례 목록이 있다. 그 목록을 곧바로 기업 전체의 생산 환경 도입 실적으로 세면 안 된다. [SPIN 사례](https://spinroot.com/spin/success.html), [Alloy 사례 목록](https://alloytools.org/citations/case-studies.html)

새로운 후보에도 실제 사용 근거가 있다. Quint 개발팀은 Emerald에서 명세로 생성한 실행을 Rust 구현에 재생하는 Quint Connect 테스트를 CI에 넣고 메모리 누수를 찾은 사례를 공개했다. FizzBee는 SlateDB의 자체 저장소에서 사용을 확인할 수 있다. `specs/README.md`는 읽기·쓰기·flush 모델과 저장소 GC 경계 프로토콜 모델을 명시한다. 홈페이지에 실린 개인 추천보다 이런 저장소 자료가 적용 범위를 판단하기에 낫다. [Emerald 사례](https://quint.sh/posts/quint_connect_emerald), [SlateDB 명세](https://github.com/slatedb/slatedb/blob/main/specs/README.md)

## AI 에이전트 연동

AI 에이전트와의 연결은 신생 도구만의 특징이 아니다. TLA+ 조직은 공식 AgentSkills를 제공하고, VS Code 확장에도 MCP 서버가 있다. Quint는 명세 작성용 `quint-lang`과 모델링용 `quint-modeling` 스킬을 제공하며 Codex CLI, Claude Code, Cursor, Gemini CLI 지원을 명시한다. P의 PeasyAI는 설계 문서를 P 코드로 바꾸고 컴파일과 검사기를 호출하는 MCP 서버다. FizzBee는 명세 작성, 검사, 디버깅, 모델 기반 테스트용 스킬 네 개를 배포한다. AI가 만든 명세가 사용자의 의도를 표현하는지는 여전히 사람이 검토해야 한다. [TLA+ AgentSkills](https://github.com/tlaplus/AgentSkills), [Quint 스킬](https://github.com/quint-co/quint/blob/main/skills/README.md), [PeasyAI](https://p-org.github.io/P/getstarted/peasyai/), [FizzBee 스킬](https://github.com/fizzbee-io/fizzbee#ai-coding-assistant-skills)

## 용도별 선택

용도에 따른 선택은 다음처럼 좁힐 수 있다.

- 기존 TLA+ 명세가 있다면 TLC를 유지하면서 Apalache를 같은 모델에 비교한다. 공식 문서도 어느 쪽이 효율적인지는 명세에 달렸다고 설명한다. 공통 벤치마크 없이 "차세대라 더 빠르다"고 고르면 근거가 부족하다. [Apalache 설명](https://apalache-mc.org/docs/apalache/index.html)
- 새 분산 시스템 명세를 개발자 도구와 함께 작성하려면 Quint부터 시험할 만하다. 메시지와 상태 머신 중심의 모델, AWS 사례, 로그 모니터링이 더 중요하다면 P를 비교한다. Python 계열 문법과 자체 검사기를 선호하면 FizzBee의 작은 모델부터 검토한다. 이는 위 문서에 근거한 추천이다.
- 데이터 모델, 권한 관계, 구성 제약이 핵심이면 Alloy 6을 살펴본다. 프로세스와 채널의 LTL 속성이 핵심이면 SPIN을 비교한다. 기존 C와 C++ 함수의 포인터와 경계 조건을 검사하려면 CBMC가 더 직접적인 후보다.

## 코드 증명과 연구 도구

Verus, Dafny, Lean은 별도 부류다. Verus는 Rust 코드와 명세·증명을 함께 작성해 기능적 정확성을 정리 증명으로 확인한다. Dafny는 계약과 루프 불변식을 가진 전용 언어와 정적 검증기를 제공한다. Lean은 프로그래밍 언어이자 증명 보조기다. 이들은 깊이 제한을 늘려 반례를 찾는 작업과 다른 노력이 필요하다. 별 수가 높더라도 "TLC를 교체할 모델 검사기" 순위에 그대로 넣기는 어렵다. [Verus](https://verus-lang.github.io/verus/guide/), [Dafny](https://dafny.org/), [Lean](https://lean-lang.org/)

새로운 연구 도구 중에는 Veil도 비교할 가치가 있다. Lean 안에서 하나의 상태 전이 모델을 작성하고, 구체적·기호적 모델 검사로 반례를 찾은 뒤 귀납적 불변식과 대화형 증명으로 안전성 증명을 진행한다. CAV 2025에서 발표한 연구 프레임워크이며, 공식 사이트는 IDE 안의 반례 시각화도 제공한다. 모델 검사에서 일반적인 안전성 증명까지 이어가고 싶을 때 살펴볼 후보지만, 현재 자료로 산업 전반의 도입을 주장할 수는 없다. [Veil](https://veil.dev/), [개발자가 쓴 Lean 사례 소개](https://lean-lang.org/use-cases/veil/)

현재 도구에서 눈에 띄는 특징은 명세 작성부터 검사, 테스트, 에이전트 호출까지 연결하는 작업 방식이다. Quint, P, FizzBee가 구체적인 기능을 제공하고 TLA+도 에이전트 연동을 지원한다. 다만 "요즘 급상승하는 도구"를 별 수 한 번과 마지막 커밋만으로 순위 매기려면 근거가 모자란다. 성장 추세에는 같은 기간의 별 증가나 기여자·사용 프로젝트의 시계열이 더 필요하다.
