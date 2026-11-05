# Herdr의 에이전트 자동화와 workspace 활용

확인 기준일은 **2026-11-05**, 설치된 client와 server는 **Herdr TUI 0.9.3**이다. 단축키는 이날 사용자 설정을 기준으로 한다. 아래의 `w1:p2`, `w2:p1`, `reviewer`는 설명용 예시이며, 실제 조작에는 현재 CLI가 반환한 값을 사용한다.

Herdr로 화면을 나눠 에이전트를 실행할 수 있다. Herdr skill을 사용하는 에이전트는 다른 pane을 열고, 작업을 전달하고, 상태를 기다리고, 결과를 읽는 과정까지 수행한다. 사람이 다른 터미널의 답변을 복사해서 붙이는 수고가 줄어든다. [공식 자동화 설명](https://herdr.dev/tui/docs/agent-automation/)

## 1. Herdr와 skill의 역할

Herdr는 터미널과 에이전트를 조작하는 CLI를 제공한다. skill은 Codex나 Claude에 그 CLI의 사용법과 조작 원칙을 알려주는 문서다. 작업을 나누고 결과를 판단하는 역할은 현재 대화 중인 에이전트가 맡는다.

| 구성 요소 | 역할 |
| --- | --- |
| Workspace | 프로젝트나 작업을 묶는 공간. 탭과 pane을 가진다 |
| Tab | 한 workspace 안의 터미널 배치 |
| Pane | 셸, 테스트, 서버, 에이전트가 실행되는 실제 터미널 |
| Agent | Herdr가 pane 안에서 인식한 코딩 에이전트 |
| Skill | 에이전트가 Herdr를 조작할 때 따르는 사용법 |
| Integration | 에이전트의 대화 세션을 Herdr에 연결한다. 일부 integration은 상태도 직접 보고한다 |

skill 설치만으로 작업 분배나 코드 병합이 자동으로 시작되지는 않는다. Herdr 내부에서 실행 중인 에이전트에 작업을 요청하면, 에이전트가 CLI를 사용해 진행한다. 로컬 skill은 먼저 `HERDR_ENV=1`인지 확인하도록 요구한다.

## 2. ㄴ 모양은 Git worktree workspace의 그룹 표시

사이드바의 `├─`, `└─`는 같은 Git 저장소의 worktree workspace를 묶어 보여준다.

```text
my-project                 원래 checkout의 workspace
├─ fix-login               로그인 수정용 worktree workspace
└─ improve-search          검색 개선용 worktree workspace
```

각 workspace는 자기 탭과 pane을 가진다. worktree마다 작업 파일과 브랜치를 따로 두고 Git 이력은 공유하므로, 서로 다른 수정을 병렬로 진행할 수 있다. 이 들여쓰기는 에이전트의 부모·자식 관계, 실행 순서, 자동 병합을 뜻하지 않는다. [공식 worktree 설명](https://herdr.dev/tui/docs/cli-reference/#worktrees), [0.9.3 사이드바 표시 구현](https://github.com/herdrdev/herdr/blob/v0.9.3/src/client/shell/sidebar.rs#L674)

`herdr worktree create`는 Git worktree를 만들면서 workspace로 연다. `herdr worktree open`은 이미 존재하는 worktree를 workspace로 연결한다. 일반 workspace에서 다른 worktree 폴더로 `cd`하는 것만으로 그룹에 등록되지는 않는다. [worktree 등록 구현](https://github.com/herdrdev/herdr/blob/v0.9.3/src/app/api/worktrees.rs#L92)

pane만 나눠 같은 작업 디렉터리에서 에이전트 둘을 실행하면 둘은 같은 파일을 수정한다. 같은 코드를 읽기만 하는 리뷰와 달리, 서로 다른 수정 작업에는 별도 worktree가 유용하다.

workspace를 닫는 것과 Git worktree를 삭제하는 것은 별도 동작이다. `workspace close`는 Herdr의 실행 공간을 닫는다. checkout 삭제는 `worktree remove`가 담당한다.

## 3. Pane ID와 agent 이름

| 식별자 | 예시 | 무엇을 가리키는가 |
| --- | --- | --- |
| Workspace ID | `w1` | workspace |
| Tab ID | `w1:t1` | workspace 안의 탭 |
| Pane ID | `w1:p2` | 특정 터미널 |
| Agent 이름 | `reviewer` | 현재 pane에서 실행 중인 에이전트의 별칭 |
| Agent 종류 | `codex`, `claude` | 실행할 프로그램의 종류 |

`agent` 명령의 대상은 **살아 있는 에이전트의 고유 이름 또는 그 에이전트가 있는 pane ID**다. `codex` 같은 종류 이름이나 `terminal_id`는 대상 식별자가 아니다. ID는 사이드바 순서가 아니며, 숫자만으로 구성된다고 가정하지 않는다. [Agent 대상 식별자](https://herdr.dev/tui/docs/cli-reference/#agents)

Herdr 안의 셸에서 다음 명령으로 현재 목록을 확인할 수 있다. 이 명령들은 읽기 전용이다.

```bash
herdr workspace list
herdr pane list
herdr agent list
```

`agent list`의 각 항목에서 `workspace_id`, `pane_id`, `agent`, `agent_status`를 볼 수 있다. 이름이 등록돼 있으면 `name`도 나온다. 수동으로 실행한 에이전트는 이름이 없을 수 있으며, 이때 pane ID를 사용한다. `cwd`와 터미널 제목을 함께 보면 어떤 작업인지 구분하기 좋다.

특정 workspace의 pane만 볼 수도 있다.

```bash
herdr pane list --workspace w1
```

기존 에이전트에 이름을 붙이는 명령은 다음과 같다. `w1:p2`에 실행 중인 에이전트가 있고 `reviewer`라는 이름이 비어 있다고 가정한다.

```bash
herdr agent rename w1:p2 reviewer
```

그 뒤에는 `herdr agent read reviewer`처럼 이름으로 찾을 수 있다. 이름은 같은 server 안에서 실행 중인 에이전트끼리 고유하다. 에이전트가 종료되거나 다른 에이전트로 교체되면 별칭도 해제된다.

Herdr는 `HERDR_WORKSPACE_ID`, `HERDR_TAB_ID`, `HERDR_PANE_ID` 환경변수를 프로세스에 전달한다. 실행 후 pane 이동 등으로 환경변수와 현재 배치가 달라질 수 있다. 일반적으로 호출 pane에는 `--current`를 사용하되, 현재 환경에서 조회가 실패하면 `agent list`나 `pane list`를 다시 확인한다. [Pane 이동과 식별자](https://herdr.dev/tui/docs/agent-automation/#three-primitives)

## 4. 다른 pane이나 workspace로 결과를 전달하는 방식

한 에이전트가 다른 에이전트의 출력을 읽고, 읽은 결과를 제3의 에이전트에게 프롬프트로 전달할 수 있다. 같은 server와 session에서는 workspace가 달라도 대상 pane ID로 지정할 수 있다.

```text
조정 에이전트가 있는 pane
    │
    ├─ agent read로 리뷰 결과 읽기 ── workspace A의 리뷰어
    │
    └─ agent prompt로 결과 전달 ─── workspace B의 구현자
                                       │
                              후속 답변을 다시 읽기
```

workspace 자체는 메시지를 받지 않는다. 그 안에서 실행 중인 특정 에이전트가 프롬프트를 받는다. 전달되는 것은 출력 텍스트나 요약이며, 상대 에이전트의 전체 대화 기억이 자동으로 합쳐지는 것은 아니다. [에이전트 간 작업 전달](https://herdr.dev/tui/docs/agent-automation/)

### 자연어로 지시하는 예시

대상 ID를 모르면 먼저 찾도록 요청한다.

> Herdr의 에이전트 목록을 보고 pane ID, 에이전트 이름, 작업 디렉터리, 상태를 표로 보여줘. 다른 pane에는 입력하지 마.

대상을 정한 뒤 결과를 전달한다.

> Herdr로 `w1:p2`의 완료된 리뷰 결과를 읽어 `w2:p1`의 에이전트에게 전달해줘. 지적 사항과 근거 파일을 포함하고, 구현은 하지 말고 먼저 타당성을 검토해 달라고 해. 받은 쪽의 검토가 끝나면 답변을 가져와줘. 내 화면 포커스는 유지해.

반대쪽 에이전트가 결과를 직접 가져오게 할 수도 있다.

> Herdr로 `w1:p2` 에이전트의 최근 리뷰 결과를 읽고, 이 작업에 해당하는 지적만 검토해줘. 아직 수정하지 말고 타당한 항목과 아닌 항목을 설명해줘.

새 에이전트를 시작하는 요청도 가능하다.

> Herdr로 옆 pane에 Codex 리뷰어를 띄워 현재 diff를 읽기 전용으로 검토시키고, 결과를 정리해줘. 내 화면 포커스는 유지해.

### CLI에서 읽기와 전달이 연결되는 예시

실행 위치는 Herdr 안의 일반 셸 pane이다. `w1:p2`에는 완료된 리뷰 에이전트, `w2:p1`에는 입력을 받을 준비가 된 다른 에이전트가 있다고 가정한다. 아래는 실제로 프롬프트를 보내는 예시이므로 두 ID를 현재 목록에서 확인한 값으로 바꾼다.

```bash
herdr agent get w1:p2
herdr agent get w2:p1
herdr agent read w1:p2 --source recent-unwrapped --lines 120
```

읽은 결과를 확인한 후, 전달할 내용과 다음 행동을 프롬프트로 지정한다.

```bash
herdr agent prompt w2:p1 \
  '리뷰 결과: 파일 A의 오류 처리 누락이 지적됐다. 해당 경로를 확인하고 타당성을 검토해줘. 아직 코드는 수정하지 마.' \
  --wait --timeout 120000
herdr agent read w2:p1 --source recent-unwrapped --lines 120
```

위 문장은 전달 형식을 보여주는 예시다. 원문을 옮기는 작업은 현재 대화 중인 에이전트에 맡기면 된다. CLI의 `read`가 읽은 내용을 다음 `prompt`에 자동으로 넣어주는 것은 아니다.

`--wait`는 대기만 담당한다. **답변 내용은 이후 `agent read`로 읽는다.** 기본 대기는 `idle`, `done`, `blocked` 중 하나에 도달하면 끝난다. 따라서 완료 응답인지 질문·승인 요청인지 확인해야 한다. 이미 작업 중인 에이전트에 프롬프트를 보내면 기존 작업의 완료가 대기를 끝낼 수도 있으므로, 작업 순서를 분명히 하려면 먼저 상태와 출력을 읽는다. [프롬프트와 대기의 의미](https://herdr.dev/tui/docs/cli-reference/#agents)

출력이 길어 최근 화면에 결과가 없으면 읽는 줄 수를 늘린다. 그래도 복구되지 않으면 해당 에이전트에 결과를 임시 Markdown 파일로 저장하고 경로를 알려달라고 요청한다. 전달받는 에이전트가 같은 machine의 그 경로에 접근할 수 있어야 파일을 읽을 수 있다.

## 5. Agent 제어와 일반 터미널 제어

| 목적 | 명령 |
| --- | --- |
| 에이전트 상태 확인 | `herdr agent get <대상>` |
| 에이전트 출력 읽기 | `herdr agent read <대상>` |
| 프롬프트 전달 | `herdr agent prompt <대상> '<내용>'` |
| 상태 변화 기다리기 | `herdr agent wait <대상>` |
| 일반 터미널 출력 읽기 | `herdr pane read <pane-id>` |
| 일반 셸에 명령 제출 | `herdr pane run <pane-id> '<명령>'` |
| 특정 로그 기다리기 | `herdr pane wait-output <pane-id> --match '<문자열>'` |

테스트나 서버 로그는 `pane` 명령으로 다룬다. 코딩 에이전트에 작업을 전달할 때는 현재 에이전트의 존재와 상태를 확인하는 `agent` 명령을 쓴다. `pane run`은 실제로 셸이 입력을 받을 준비가 된 pane에서 사용한다. [Pane과 agent의 구분](https://herdr.dev/tui/docs/agent-automation/#choose-the-control-surface)

상태는 다음과 같이 해석한다.

| 상태 | 의미 |
| --- | --- |
| `working` | 작업 중 |
| `blocked` | 질문이나 승인 등 입력이 필요함 |
| `done` | 작업이 끝났고 해당 클라이언트에서 아직 결과를 보지 않음 |
| `idle` | 작업을 마쳤거나 대기 중이고, 결과를 확인한 상태 |
| `unknown` | Herdr가 상태를 확실히 판별하지 못함 |

`done`은 테스트 통과나 구현 품질을 보장하지 않는다. 클라이언트마다 확인한 결과가 다르므로 TUI의 Done 표시와 CLI 상태도 다를 수 있다. [공식 상태 정의](https://herdr.dev/tui/docs/concepts/#agent)

## 6. 함께 활용할 기능

**Go To**에서는 원하는 에이전트나 터미널을 찾아 이동한다. 백그라운드 에이전트의 완료와 입력 요청은 알림으로 확인할 수 있다. [키보드 탐색](https://herdr.dev/tui/docs/keyboard/), [알림 설정](https://herdr.dev/tui/docs/config-reference/)

**저장된 SSH machine**을 사용하면 여러 서버를 한 사이드바에서 관리할 수 있다. CLI에서는 모든 대상 조회와 조작에 같은 `--machine` 선택자를 붙인다. 다음의 `build-server`는 저장된 profile의 이름이나 ID 예시다.

```bash
herdr machine list
herdr --machine build-server agent list
herdr --machine build-server agent read reviewer --lines 120
```

다른 server에는 같은 pane ID나 agent 이름이 있을 수 있다. 원격에서는 그 server의 목록에서 대상을 다시 찾는다. 로컬에서 쓰던 ID나 `--current`로 원격 pane을 지정하지 않는다. [SSH machine과 CLI](https://herdr.dev/tui/docs/cli-reference/#saved-ssh-machines)

**Detach와 재접속**은 원래 프로세스를 계속 살려 둔다. server가 재시작되면 기존 프로세스는 끝나고 workspace, 탭, pane, 디렉터리, 배치를 복원한다. integration과 세션 정보가 준비된 에이전트는 대화를 재개할 수 있다. 일반 테스트나 서버 프로세스까지 자동 재개되는 것은 아니다. [세션 복원 범위](https://herdr.dev/tui/docs/session-state/)

**Integration과 skill**은 별도다. integration은 대화 세션 복원과 일부 에이전트의 직접 상태 보고를 돕는다. 조사 시점의 Claude와 Codex integration은 설치 CLI 기준으로 최신 상태였다. 설치 상태는 `herdr integration status`로 다시 확인할 수 있다. [Integration 설명](https://herdr.dev/tui/docs/integrations/)

## 7. 사용자 설정의 단축키

사용자 prefix는 기본 문서의 `Ctrl+B` 대신 **`Ctrl+A`**다. 아래의 화살표는 prefix를 누른 뒤 다음 키를 누른다는 뜻이다.

| 단축키 | 기능 |
| --- | --- |
| `Ctrl+A` → `?` | 현재 단축키 보기 |
| `Ctrl+A` → `g` | Go To |
| `Ctrl+A` → `Shift+G` | 새 Git worktree 만들기 |
| `Alt+W` | workspace 탐색 |
| `Alt+V` | 오른쪽 분할 |
| `Ctrl+A` → `-` | 아래쪽 분할 |
| `Alt+F` | 현재 pane 확대·복귀 |
| `Alt+S` | 복사 모드 |
| `Alt+E` | scrollback 편집 |
| `Alt+T` | 현재 다음에 새 탭 |
| `Alt+C` | 현재 다음에 새 workspace |
| `Ctrl+A` → `q` | 프로세스를 유지하며 detach |

`Alt+T`와 `Alt+C`는 사용자 정의 명령에 연결돼 있다. 설정 파일은 `~/.config/herdr/config.toml`이며, `herdr --default-config`로 기본값을 비교할 수 있다.

## 8. 복습할 때 사용할 요청

다음 요청을 현재 작업에 맞게 바꿔 사용한다.

- "Herdr의 에이전트 목록을 보고 각 pane의 작업과 상태를 정리해줘. 입력은 보내지 마."
- "Herdr로 옆 pane에 리뷰어를 띄워 현재 diff를 검토하고 결과를 가져와줘."
- "리뷰어의 지적 사항을 구현 에이전트에게 전달하고, 수정 계획을 받아와줘."
- "Herdr의 별도 pane에서 테스트를 실행하고, 실패한 부분의 로그를 확인해줘."
- "이 저장소에 worktree 두 개와 workspace를 만들어 서로 다른 수정안을 병렬로 구현하고 비교해줘."
- "이미 만들어둔 이 worktree를 원래 저장소 아래에 묶이는 Herdr workspace로 열어줘."
- "Herdr에 등록된 빌드 서버의 테스트 진행 상황을 읽고 요약해줘."

CLI 문법은 설치된 바이너리의 `herdr --help`와 `herdr agent`, `herdr pane`, `herdr worktree` 도움말로 확인한다. 인자 없는 `herdr`는 도움말이 아니라 TUI 실행·접속 명령이다. 현재 웹사이트의 Herdr App과 이 문서의 Herdr TUI는 별도 제품이므로, 웹 문서는 `/tui/docs/` 경로를 참고한다.
