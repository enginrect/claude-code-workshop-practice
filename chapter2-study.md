# [2주차] Claude Code Deep Dive Workshop — Chapter 2 스터디 노트

> - 스터디: Claude Code Deep Dive Workshop 모각코 2주차 (8/3 ~ 8/9)
> - 자료: [whchoi98/claude-code-workshop](https://github.com/whchoi98/claude-code-workshop) — Chapter 2 (120 slides, 9 Parts + Labs 4개)
> - 방식: 실습 중심. 각 파트 핵심만 요약하고, 랩은 내 터미널 세션 로그로 기록한다.

---

## 개요 — 위임과 병렬화로 컨텍스트를 지키는 기술

- **Subagent는 자기만의 컨텍스트 윈도우를 가진 격리된 Claude 인스턴스**다. 자체 시스템 프롬프트, 별도 도구 권한으로 독립 작업한 뒤 **요약만 메인에 돌려준다**.
- 위임하는 이유는 성능이 아니라 **컨텍스트 오염 방지**다. 검색 결과·로그·파일 내용이 메인 대화를 채우면 모델의 주의가 분산되고 응답 품질이 떨어진다. 다시 참조하지 않을 출력이 특히 낭비.
- 정의는 **Markdown 한 장**(YAML frontmatter + 시스템 프롬프트 본문). 마크다운 파일 하나로 역할이 고정되고, 버전 관리로 팀과 공유된다.

## Part별 핵심 정리

### Part 1. Subagents 개념과 원리

- **위임 없이 vs 위임하면**: 30개 파일 읽기가 그대로 이력에 축적 → 자식 컨텍스트에서 소화 / 테스트 로그 수천 줄이 윈도우 점유 → 메인에는 실패 테스트 요약만 도착 / Compaction이 빨리 오고 초기 지시 소실 → 본 대화는 결정과 방향에만 사용.
- **Main vs Subagent**: 컨텍스트(전체 이력 ↔ 위임 메시지로 새 출발), 시스템 프롬프트(Claude Code 전체 ↔ 정의 파일 본문 + 환경 정보), 도구(세션 전체 ↔ `tools`로 좁힌 집합), 산출물(사용자와의 대화 ↔ 요약 한 건), 수명(세션과 함께 ↔ 작업 완료 시 종료, resume 가능).
- **Subagent 초기 컨텍스트 5구성**: ①System Prompt(자신의 프롬프트 + 환경 정보, Claude Code 전체 프롬프트는 **미포함**) ②Task Message(메인이 작성한 위임 지시문, 대화 이력은 안 옴) ③CLAUDE.md + Memory(메모리 계층 전체, 단 Explore·Plan은 생략) ④Git Status(부모 세션 시작 시점 스냅샷) ⑤Preloaded Skills(`skills` 필드 지정 스킬 본문 전체).
- **위임의 5대 가치**: Preserve Context(출력 격리) / Enforce Constraints(도구 제한으로 안전선 강제) / Reuse Configs(사용자 레벨 정의로 전 프로젝트 재사용) / Specialize(도메인 특화 프롬프트로 성공률 상승) / Control Costs(haiku 등 경량 모델 라우팅) + 팀 자산화(프로젝트 정의를 버전 관리로 공유).
- **Built-in 에이전트**
  | 이름 | 특성 |
  |---|---|
  | Explore | 읽기 전용 탐색 전문. Write/Edit 거부, `quick`/`medium`/`very thorough` 3단계 강도. CLAUDE.md와 git 상태를 생략해 빠르고 저렴. v2.1.198부터 메인 모델 상속(Claude API는 Opus 상한) |
  | Plan | Plan 모드의 조사 담당, 읽기 전용, 모델 상속 |
  | general-purpose | 전체 도구, 모델 상속. 탐색과 수정 병행하는 복잡한 다단계 작업 |
  | statusline-setup | Sonnet 고정, `/statusline` 실행 시 자동 |
  | claude-code-guide | Haiku 고정, Claude Code 기능 질문 시 자동 |
  - 비활성화: `permissions.deny`에 `Agent(Explore)` 등을 넣거나, `DISABLE_EXPLORE_PLAN_AGENTS=1`. Agent 도구 자체를 deny하면 모든 위임이 봉쇄.
- **실행 모델**: v2.1.198부터 **백그라운드가 기본값**. 결과가 다음 판단에 즉시 필요할 때만 포그라운드. 백그라운드의 권한 요청은 요청 에이전트 이름과 함께 **메인 세션에 떠오른다**. `Ctrl+B`로 실행 중 작업을 백그라운드 전환, `DISABLE_BACKGROUND_TASKS=1`로 전체 비활성.
- **상태 모델**: 호출마다 새 인스턴스로 시작하지만 대화 기록은 파일로 남아 **재개 가능**. 완료 시 메인이 agent ID 수신 → 그 ID로 특정 인스턴스 지목. 트랜스크립트는 세션 폴더 `subagents/` 아래 JSONL(`agent-{id}`), 30일 자동 정리(`cleanupPeriodDays`). **Explore와 Plan만 일회성**(resume 불가).
- **모델 해석 우선순위**: ①환경변수 `CLAUDE_CODE_SUBAGENT_MODEL` ②Agent 도구 호출 파라미터의 `model` ③frontmatter `model` 필드 ④메인 모델 상속(기본값).
- **4가지 병렬화 수단의 경계**
  | 수단 | 성격 | 적합 |
  |---|---|---|
  | Subagent | 한 세션 안의 워커, 새 컨텍스트, 요약 회수 | 곁가지 격리, 병렬 리서치 |
  | Fork | 대화 전체를 물려받은 서브에이전트 | 설명 없이 맡기는 곁가지 |
  | Background agents | 독립 세션 여러 개를 한 화면에서 관찰 | 무관한 작업들의 병렬 운영 |
  | Agent Teams | 세션들끼리 메시지로 협업, 더 무겁고 비쌈 | 지속 병렬, 컨텍스트 초과 작업 |
  - 판단 기준: **소통이 필요 없으면 Subagent, 필요하면 Teams**.
- **6대 사용 사례**: ①고볼륨 격리(테스트 스위트, 로그 처리) ②병렬 리서치 ③제2의 시선(구현과 분리된 신선한 컨텍스트로 리뷰) ④권한 강제 ⑤문서 조회(외부 문서 다량 페치를 자식에서 소화) ⑥같은 지시를 반복하는 워커를 정의 파일로 고정.
- **비용/지연 고려**: 새 컨텍스트 출발이라 **기동 오버헤드**가 있고 결과 회수도 토큰을 쓴다. 다수 에이전트의 상세 결과가 메인을 다시 채울 수 있으므로 회수 형태를 지시로 통제. 탐색성 워커는 haiku, **fork는 메인의 프롬프트 캐시를 공유해 신규 스폰보다 저렴**.
- **안티패턴 5가지**: ①잦은 왕복이 필요한 대화형 작업 위임 → 메인에서 직접 ②계획·구현·테스트처럼 맥락 공유하는 일 분리 → 한 흐름 유지 ③한 줄 수정 같은 초단발 위임 → `/btw` ④전 단계 결과에 의존하는 조사들의 병렬화 → 체이닝으로 순차 ⑤만능 에이전트 하나 → 역할별 단일 책임으로 분리.

### Part 2. Subagent 정의 방법

- 정의 파일 = **YAML Frontmatter(메타데이터·설정) + 본문(시스템 프롬프트)**. 필수는 `name`, `description` 둘뿐. 파일명은 자유이고 **정체성은 오직 `name` 필드**.
- **스코프 5계층(위가 이김)**: ①Managed(조직 관리 설정 디렉토리, 관리자 배포) ②`--agents` 플래그(JSON 주입, 세션 한정, 디스크 저장 없음) ③Project `.claude/agents/`(버전 관리로 팀 공유, **실무의 중심**) ④User `~/.claude/agents/` ⑤Plugin(`my-plugin:name`으로 등록).
- **배치 규칙**: agents 디렉토리는 재귀 스캔(하위 분류 폴더 가능). 동일 스코프 중복 `name`은 하나만 로드되고 `/doctor`가 중복 보고(v2.1.196+). 중첩 `.claude` 간 동명 충돌 시 **작업 위치에 가까운 정의 승리**(v2.1.178+). `--add-dir`로 붙인 경로의 agents도 함께 로드.
- **파일 워처**: 파일을 추가·수정하면 몇 초 안에 감지되어 **재시작 없이** 다음 위임부터 적용. 예외 두 가지 — 세션 시작 시 없던 agents 폴더를 새로 만들면 재시작 필요, `--disable-slash-commands` 세션은 감시 자체가 꺼짐.
- **생성 방법**: v2.1.198부터 `/agents` 위저드는 **제거**. Claude에게 자연어로 요구사항을 주고 파일을 쓰게 하는 것이 공식 경로("`~/.claude/agents/`에 code-improver 서브에이전트를 만들어줘. 읽기 전용, 모델은 sonnet").
- **`name` / `description`** — 자동 위임을 좌우하는 필수 2필드
  - `name`: 소문자와 하이픈, 고유 식별자. 훅에는 `agent_type`으로 전달됨.
  - `description`: **무엇을 하는지보다 "언제 쓰는지"**를 담아야 위임 판단이 가능. `use proactively`, `MUST BE USED`가 자동 위임을 강화.
  - 무시되는 설명("코드를 리뷰하는 에이전트", "보안 전문가") vs 위임되는 설명(`Use proactively after writing or modifying code`, `MUST BE USED when tests fail or coverage drops`, `Use before commits touching auth, payments, or user data`) — **행동 트리거가 문장에 내장되어야 매칭 조건이 곧 자동 위임 규칙이 된다**.
- **`tools`(허용 목록)**: 생략 시 메인의 전체 도구를 상속(MCP 포함), 지정 시 나열한 도구만. `AskUserQuestion`, `EnterPlanMode`, `ScheduleWakeup`, `WaitForMcpServers`는 UI·세션 상태 의존이라 **서브에이전트에서 원천 불가**. `Agent`를 포함시키면 중첩 스폰 허용.
- **`disallowedTools`(차단 목록)**: 상속에서 빼기. MCP는 서버 단위 패턴 지원 — `mcp__github`(= `mcp__github__*`), `mcp__*`(모든 MCP, deny 전용). 둘 다 지정 시 **disallowed를 먼저 제거한 뒤 tools 해석**.
- **`model`**: alias(`sonnet`/`opus`/`haiku`/`fable`) 또는 전체 ID(버전 고정) 또는 `inherit`(생략 시 기본). 조직 `availableModels` 허용 목록을 통과해야 하며 제외 모델은 상속으로 폴백. 확장 사고는 메인 설정 상속(에이전트별 개별 설정 없음).
- **`permissionMode`**: `default` / `acceptEdits`(반복 수정 워커) / `auto`(분류기 검토, 신뢰 저장소) / `dontAsk`(확인 프롬프트 자동 거부 — 무인·읽기 중심) / `plan`(조사 전용). **부모가 bypass·acceptEdits·auto면 부모 우선**이고 frontmatter는 무시된다.
- **`skills` 프리로드**: 나열한 스킬의 **본문 전체**가 시작 컨텍스트에 주입 → 탐색 단계 생략. 미나열 스킬도 Skill 도구로 실행 중 호출은 가능하며, 막으려면 `tools`에서 Skill 제외.
- **`mcpServers` 스코프**: 인라인 정의(이 에이전트 전용, 시작 시 연결·종료 시 해제) 또는 문자열로 기존 서버 참조. 메인 컨텍스트에 정의가 노출되지 않아 절약되고, managed 정책은 동일 적용.
- **`memory`(영속 메모리)**: `user` / `project`(권장) / `local`. `.claude/agent-memory/` 아래 MEMORY.md에 축적되고 **첫 200줄 또는 25KB**가 프롬프트에 포함. 관리용 Read/Write/Edit 도구가 자동 부여된다.
- **고급 필드**: `maxTurns`(턴 수 상한 — 폭주 방지) / `effort`(low~max, 노력 수준 오버라이드) / `isolation: worktree`(임시 git worktree에서 실행) / `background: true`(항상 백그라운드) / `color`(작업 목록 표시 8색) / `initialPrompt`(`--agent` 메인 실행 시 첫 턴 자동 제출).
- **`hooks`(프론트매터 훅)**: 그 에이전트가 도는 동안만 유효한 검증. 예 — `db-reader`에 `PreToolUse` matcher `Bash` + 검증 스크립트, 스크립트가 INSERT/UPDATE 감지 시 **exit 2로 차단하고 사유 회신**. Stop은 SubagentStop으로 변환되고, 플러그인 정의에서는 보안상 무시되는 필드.

### Part 3. Agent 도구와 디스패치

- **Agent 도구가 위임의 프리미티브**. v2.1.63에서 Task → Agent로 개명(기존 `Task(...)` 표기는 별칭으로 유효). 메인이 작업 요약과 `subagent_type`을 지정해 호출하고, 완료 시 요약 + agent ID가 귀환. 권한 규칙 문법은 `Agent(name)`.
- **자동 위임의 판단 재료 3가지**: 요청의 성격 + 각 에이전트의 `description` + 현재 컨텍스트. description의 능동 문구가 위임 여부를 좌우한다.
- **명시 호출 3단계 사다리**
  | 레벨 | 수단 | 강도 |
  |---|---|---|
  | 1 | 자연어("test-runner 서브에이전트로 고쳐줘") | 제안 — Claude가 판단 |
  | 2 | `@agent-이름` 멘션 | 이번 작업은 반드시 그 에이전트 |
  | 3 | `claude --agent <name>` | 세션 자체가 그 에이전트로 실행 |
  - `@멘션`: `@` 입력 후 타입어헤드에서 선택(실행 중인 백그라운드 에이전트도 상태와 함께 표시). 수동 표기는 `@agent-code-reviewer`, 플러그인은 `@agent-my-plugin:review:security`. **멘션은 어떤 에이전트가 돌지를 고정할 뿐, 지시문 작성은 여전히 메인 Claude의 몫**.
  - `--agent`: 기본 Claude Code 프롬프트를 완전 대체하되 **CLAUDE.md와 프로젝트 메모리는 그대로 로드**. 시작 헤더에 `@이름` 표시, resume 시에도 유지. 프로젝트 기본값은 `.claude/settings.json`의 `{"agent": "code-reviewer"}`(CLI 플래그가 우선).
- **패턴 3종**
  - **병렬 리서치**: 독립 에이전트 여러 개가 동시 탐색 → 완료되는 대로 요약 도착 → 메인이 종합. 조건은 **조사 경로가 서로 의존하지 않을 것**.
  - **고볼륨 격리**: "서브에이전트로 전체 테스트 스위트를 돌리고 실패한 테스트와 에러 메시지만 보고해줘" — 수천 줄 로그는 자식이 소화, 실패만 귀환.
  - **체이닝**: "code-reviewer로 성능 이슈를 찾고, 그다음 optimizer로 고쳐줘" — 앞 결과를 메인이 받아 다음 지시문에 반영. **에이전트 간 직통은 없고 메인이 항상 중계**.
- **백그라운드 운용**: 프롬프트 아래 패널에 행 단위 표시, 화살표로 이동하고 `Enter`로 트랜스크립트 열람(열람 중 후속 지시를 보내 방향 교정 가능). `Ctrl+B` 백그라운드 전환, `x` 중지·정리, `Esc` 복귀, `/tasks` 현황.
- **Resume과 SendMessage**: 완료 후 "그 리뷰 이어서 이번엔 인가 로직을 분석해줘"라고 하면 Claude가 **SendMessage로 해당 에이전트를 재개** — 이전 도구 호출과 추론을 전부 가진 채 계속. 중지된 에이전트는 메시지 수신 시 자동 재개, 이름 재사용 충돌 시 전송 거부 후 대상 안내(v2.1.199). Explore·Plan은 재개 불가.
- **중첩 서브에이전트**(v2.1.172+): 서브에이전트도 자기 서브에이전트를 만들 수 있다. **깊이 5 고정 상한**(조정 불가), 해당 에이전트 `tools`에 `Agent` 포함이 활성 조건. 패널에 하위 개수 `(+N)` 표시. 최상위 서브에이전트의 요약만 메인으로 귀환("요약의 요약"). **fork는 fork 불가**.
- **`/fork` 상세**(v2.1.161+ 기본 활성): 지시문 첫 단어들로 이름이 자동 부여되고 패널에 행이 추가되어 백그라운드 진행. 열람 중 `/model`, `/fast`는 메인 대상임을 안내.
- **Fork vs Named Subagent**
  | 항목 | Fork | Named |
  |---|---|---|
  | 컨텍스트 | 전체 대화 이력 | 지시문으로 새 출발 |
  | 프롬프트·도구 | 메인과 동일 | 정의 파일의 것 |
  | 모델 | 메인과 동일 | `model` 필드 |
  | 프롬프트 캐시 | 메인과 공유 | 별도 캐시 |
  | 선택 기준 | 배경 설명이 긴 곁가지 | 역할이 정형화된 작업 |
- **에러 처리**(v2.1.199+): API 오류로 끊긴 에이전트가 **오류 텍스트를 결과인 척 반환하지 않고 실패로 정확히 보고**하며 부분 산출물은 보존. 복구는 원인 해소 후 재시도 또는 resume. 사전 안전장치는 `maxTurns` 상한과 훅 검증.
- **관측과 디버깅**: 트랜스크립트 `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`, 압축 이벤트도 `compact_boundary`로 기록(`preTokens`). settings.json의 `SubagentStart`/`SubagentStop` 훅은 **에이전트명 매처** 지원.
- **선택 가이드**: 잦은 왕복·맥락 공유 단계 → Main 대화 / 대량 출력·권한 제약 → Subagent / 대화 맥락 빠른 질문 → `/btw` / 재사용할 프롬프트·워크플로 → Skill(메인 컨텍스트에서 실행).

### Part 4. Pattern 1 — Code Reviewer

- 시나리오: **커밋 전 셀프 리뷰 표준화**. 사람 리뷰어에게 가기 전 명백한 결함·보안 이슈를 걸러 리뷰 왕복을 줄인다. 발동은 코드 수정 직후·커밋/PR 생성 전, 범위는 **git diff 변경 파일 중심**(저장소 전수 아님), 산출은 3단계 우선순위, 원칙은 **읽기 전용(고치지 않고 보고만)**.
- 정의 골격: `tools: Read, Grep, Glob, Bash`(**Edit 없음** = 읽기 전용 강제, Bash는 git diff 실행용), `model: inherit`, `memory: project`, description에 `Proactively reviews code. Use immediately after writing or modifying code.`
- 본문(시스템 프롬프트)에 고정할 것: **체크리스트**(명확성 / 안전성 — 에러 처리·입력 검증·시크릿 노출 / 품질 — 커버리지·성능), **출력 형식**(Critical → Warnings → Suggestions, 각 항목에 `파일:라인` + 수정 예시), 결론에 **머지 가능 여부 한 줄 판정**.
- **영속 메모리 결합**: 본문에 "발견 패턴을 메모리에 기록하라" 지시 + 호출 시 "메모리의 과거 패턴 먼저 확인하고 시작해" 요청 → 리뷰할수록 저장소 컨벤션이 축적. `.claude/agent-memory/`를 커밋하면 **학습 자체가 팀 자산**이 되고, 상한 초과 시 에이전트가 스스로 큐레이션.
- **호출 3경로**: ①대화형(자동 위임 + `@멘션` 보장 + Critical만 debugger로 넘기는 체이닝) ②헤드리스(`claude -p`로 "Critical 개수를 마지막 줄에 N건 형식으로" → pre-push 훅에서 `grep -q "0건" || exit 1` 게이트) ③GitHub Actions(`on: [pull_request]`, 결과를 review.md로 저장해 코멘트 게시).
  - 헤드리스·CI 모두 `--allowed-tools`에 **`Agent`를 반드시 포함**해야 위임이 성립한다.
- **내장 `/code-review`와의 관계는 대체가 아니라 보완**: 즉석 점검·마무리 정리는 내장 명령(정의 불필요, fix·comment·ultra 옵션), 팀 체크리스트와 출력 형식 고정·영속 메모리·훅/CI 참여는 맞춤 에이전트.
- 확장 변형: security-reviewer / perf-reviewer(N+1, 캐시, 복잡도) / api-reviewer(계약 호환성) / test-reviewer / 중첩 구성(발견 항목별 검증 워커 스폰) / 아키텍처 판단이 무거우면 `model: opus`.
- **자주 만나는 함정**: description에 시점이 없어 자동 위임 불발 / `Bash` 누락으로 git diff 실행 불가 / 출력 형식 미지정으로 리뷰가 산문으로 흩어짐 / 저장소 전수 검사로 토큰 폭증 / **CI에서 Agent 도구 미허용으로 위임 실패**.

### Part 5. Pattern 2 — Tester

- 정의 골격: `tools: Read, Write, Edit, Bash, Grep, Glob`(**Write 포함** = 실행형), `model: sonnet`, `isolation: worktree`, `maxTurns: 40`. description은 `Use proactively when new code lacks tests or coverage drops.`
- **커버리지 확장 5단계**: ①측정(리포트 실행 — 감이 아닌 측정부터) ②갭 선정(미커버 분기와 위험 모듈 우선, 결제·인증) ③작성(프로젝트 컨벤션 준수) ④실행(신규 포함 전체) ⑤수정(**의도 보존하며** 통과까지). 최종 보고는 커버리지 Before/After와 신규 케이스 목록.
- **엣지 케이스 발굴 4축**(본문에 명시하면 놓치는 축이 줄어든다): ①경계값(0, 음수, 최대치, 빈 문자열/배열) ②예외 경로(타임아웃, 재시도, 부분 실패, 예외 전파) ③동시성(중복 요청, 경합, 순서 역전) ④시간과 로캘(시간대, 월말, 윤년, 다국어).
- 호출 시 **완료 기준을 수치로**: "src/payments 커버리지를 80% 이상으로. 만료 카드와 부분 환불 경로 필수 포함, 끝나면 전후 수치로 보고" → 회수 보고 "62% → 84%, 신규 12 케이스(만료 카드 3, 부분 환불 4, 경계값 5), 전체 스위트 통과".
- 프레임워크 대응: JS/TS(Jest, Vitest, Playwright — 기존 설정 파일 우선), Python(pytest, fixture/parametrize, conftest 컨벤션), Java/Kotlin(JUnit 5, Mockito). **Mock 원칙은 외부 IO만 mock, 내부 로직은 실물** — 과도한 mock은 검증력을 훼손. 픽스처는 기존 팩토리·헬퍼 재사용 우선.
- **`isolation: worktree`**: 기본 브랜치에서 분기한 임시 worktree에서 작업 → 내 미커밋 변경과 무관하고 편집 중인 파일과 절대 겹치지 않음. 결과는 diff나 브랜치로 검토 후 선택적 병합, **무변경 종료 시 자동 삭제**. 테스터 다중 실행도 안전하며 `/batch`와 동일한 격리 기술.
- CI 통합: 주간 크론으로 "커버리지 최하위 모듈 1개를 보강하고 PR 브랜치 생성" — **회당 범위를 제한**하고 PR로 사람 리뷰 게이트를 둔다(관리형 대안은 Routines).
- 테스트 품질 지표: 커버리지는 절대값보다 **하락 감지**(모듈별 최저선), 수정 시 **assertion 약화 여부**(의도 보존율), 플레이키 비율(간헐 실패 격리), 실행 시간 추세(병렬화·분할 시점 판단).

### Part 6. Pattern 3 — Security Scanner

- 정의 골격: `tools: Read, Grep, Glob, Bash`, `model: opus`(판단 난도 상향), `memory: project`, `permissionMode: dontAsk`(무인 자동 거부), description은 `MUST BE USED before commits touching auth, payments, or user data.`
- **이중 잠금**: tools 허용 목록(1차) + `PreToolUse` 훅 검증(2차). `ro-guard.sh`는 stdin JSON에서 `.tool_input.command`를 뽑아 조회 계열(`git diff|log|show|status`, `grep`, `rg`, `cat`, `head`, `ls`, `find`, `npm audit`, `pip-audit`, `trivy`)만 통과시키고 나머지는 **exit 2로 차단 + 사유 회신**.
- **탐지 6관점**(본문에 명시): ①인젝션(SQL·커맨드·템플릿, 이스케이프 누락) ②인증/인가(검증 우회, 권한 상승, 세션 고정) ③민감 데이터(시크릿 하드코딩, 로그 노출, 평문 저장) ④입력 검증(경계 미검증, 역직렬화, 경로 조작) ⑤설정 결함(CORS 과개방, 디버그 모드, 기본 자격증명) ⑥의존성(CVE, 유지보수 중단 패키지).
- 보고 형식은 **파일:라인 + 왜 위험한지(공격 시나리오)**. 예 — "Critical 1: 웹훅 서명 검증이 타임스탬프 미확인, 재전송 공격 가능 (src/webhooks/pay.ts:42)".
- 코드 밖의 두 전선: **시크릿 탐지**(키·토큰 패턴, 엔트로피 높은 문자열, `.env` 예시와 실파일 구분, 이력 유출 시 **회전 절차 안내까지**) / **의존성 스캔**(`npm audit`·`pip-audit` 실행과 해석, 심각도 분류, 직접·전이 의존 구분, 락파일 기준 재현 가능).
- **pre-commit 게이트**: 스테이징 파일에서 `auth|payment|user` 경로 필터 → 없으면 통과, 있으면 스캐너 실행 후 "마지막 줄에 Critical N건" 판정 규격으로 `grep -q "Critical 0건" || exit 1`. `--no-verify` 우회 금지를 팀 정책으로 합의.
- **오탐 처리 순환이 게이트의 신뢰를 지킨다**: 발견 항목을 사람이 확정 → 확인된 오탐을 사유와 함께 메모리에 기록 → 다음 스캔 전 오탐 목록을 먼저 대조 → 근거 링크·날짜를 남기고 분기별 재검토. 오탐이 반복되면 게이트는 무시된다(늑대 소년).
- 컴플라이언스 연계: 스캔 보고를 날짜별 파일로 보존(SubagentStop 훅 자동화), 발견 항목에 **OWASP 분류 태깅**, 민감 경로·발동 조건 문서화, 오탐·수용 위험의 사유와 승인자를 메모리에 대장으로, 주간 전체 스캔은 CI 크론 또는 Routines.

### Part 7. Pattern 4 — Docs Writer

- 정의 골격: `tools: Read, Write, Edit, Grep, Glob, Bash`, `model: sonnet`, `skills: [docs-style-guide, api-doc-template]`. description은 `Use proactively when public APIs change or docs drift from code.` 본문 원칙은 **"코드를 먼저 읽고, 실제로 하는 일을 문서화하고, 프리로드된 스타일 가이드를 정확히 따른다"**.
- **README 자동 생성**: 실제 `package.json` 스크립트와 `.env.example` 기준으로 설치·실행 절차를 **직접 실행해 검증한 뒤 기재**. 산출 구조는 스킬 템플릿이 고정(개요·아키텍처 한 단락 / Quick Start / 환경변수 표 / 개발 절차).
- **API 문서**: 요청·응답 스키마를 **타입 정의에서 추출**하고 엔드포인트별로 메서드·경로·인증 요건, 파라미터/바디 스키마, 성공·실패 응답 예시 쌍, **에러 코드와 의미 표**까지. OpenAPI로 확장 가능.
- **ADR**: 결정 직후가 작성 타이밍. 골격은 Status(Proposed/Accepted/Superseded) / Context(문제와 제약) / Decision(선택과 근거) / **Alternatives(검토했으나 기각한 안과 이유 — 재논쟁 방지)** / Consequences(트레이드오프와 후속 작업). `docs/adr/`에 번호 연번 관리.
- **Changelog**: 태그 기준으로 범위를 산정하고 커밋 메시지를 **사용자 관점 문장으로 번역**. 분류는 Added/Changed/Fixed/Deprecated/Removed, **Breaking Changes는 최상단 + 마이그레이션 안내 필수**, 내부 리팩토링은 제외. conventional commits면 분류 정확도 상승.
- **skills 프리로드의 메커니즘**: 스타일 가이드 본문 **전체**가 시작 컨텍스트에 주입되므로 에이전트는 규칙을 **찾는 게 아니라 이미 아는 상태로 출발**한다. 스킬은 팀 규범, 메모리는 이 저장소의 발견 사항으로 **분업**. 스킬 파일 수정은 다음 호출부터 즉시 반영되고, 메인 대화 컨텍스트는 소비하지 않는다.
- CI 문서 부패 방지: `on: pull_request, paths: ["src/api/**"]`로 **API 변경 시에만 발동**, "변경 없으면 무동작"을 지시해 불필요 커밋 방지. 코드와 문서가 **동일 PR에 동행**하면 괴리가 원천 차단된다.

### Part 8. Pattern 5 — Migration Bot

- 정의 골격: `tools: Read, Write, Edit, Bash, Grep, Glob`, `model: sonnet`, `isolation: worktree`, `maxTurns: 60`, `memory: project`. 본문 원칙은 **"작고 검증 가능한 배치로 마이그레이션하고, 배치마다 빌드·테스트를 돌리고, 진행 상황을 메모리에 기록하고, 임계치에서 멈춘다"**.
- **시나리오 3종**
  - **라이브러리 업그레이드**: "lodash 4 제거 → 네이티브 ES 메서드 전환, 대체 불가 유틸은 자체 구현, **10파일 단위로 빌드·테스트 검증**" → 배치마다 메모리 기록(`batch 3/9: 28 files done, tests green`), 치환 불가 건은 별도 처리 방침.
  - **Import 경로 변경**: 모듈 재배치 + 배럴 재수출 정리. **`tsc --noEmit` → lint → test 3관문**을 통과한 배치만 다음으로. 규칙 고정(별칭 기준, 상대 경로 신규 생성 금지), **순환 참조 발견 시 목록화 후 중단하고 사람 판단 요청**.
  - **Deprecated API 교체**: 시그니처 차이는 마이그레이션 가이드 기준으로 기계 교체(143 call sites), **의미가 바뀌는 케이스는 목록만** 만들고 사유 첨부. 회귀 방지로 v1 호출 재유입 차단 lint 규칙 추가.
- **안전장치 4겹 방어**: ①worktree 격리(내 체크아웃과 분리, 무변경 시 자동 정리) ②상한과 차단기(`maxTurns`, 실패율 초과 시 자진 중단 = 서킷 브레이커) ③배치 검증 절차(빌드·테스트 관문을 배치마다) ④리뷰 회수(**브랜치와 PR로만 본선 합류**, 사람 승인 게이트). 메모리의 진행 기록이 중단 후 재개 근거.
- **비용 효율 / 모델 배분**: 탐색 단계는 haiku 워커(대상 호출부 전수 수집, 영향도 분류, 치환 가능/판단 필요 선별 → 작업 목록과 배치 계획), 변환 단계는 sonnet 본대(정밀 치환, 관문 검증, 실패 수습 → 브랜치와 완료 보고). 단계별로 모델을 나누면 **품질 저하 없이 비용이 내려간다**.
- **`/batch`와의 분업**: `/batch`는 조사·분해·병렬 스폰·단위별 PR까지 자동인 내장 오케스트레이터(5~30 워커, 대규모 **균질** 변경에 최적). 맞춤 봇은 배치 크기·관문·규칙을 세밀 통제하고 메모리로 회차 간 연속성을 가지며 판단 분리 같은 도메인 규칙을 내장 — **중간 규모, 반복 마이그레이션에 최적**.

### Part 9. Recap

- **여섯 문장 요약**: ①개념 — 격리 컨텍스트에서 일하고 요약만 돌려주는 세션 내 워커 ②정의 — Markdown 한 장, 필수는 name·description, 스코프 5계층 ③필드 — tools/model/memory/skills/hooks/isolation의 조합 설계 ④디스패치 — 자연어·@멘션·`--agent` 사다리와 병렬·체이닝·resume ⑤고급 — fork의 전체 상속과 캐시 공유, 중첩 깊이 5, SendMessage ⑥패턴 — 리뷰어·테스터·스캐너·문서·마이그레이션의 검증된 골격.
- **FAQ 6선**: 서브에이전트끼리 대화는 **불가**(메인이 항상 중계, 필요하면 Agent Teams) / 정의 수정은 파일 워처가 수 초 내 반영(새 폴더 첫 생성만 예외) / `/agents` 위저드는 v2.1.198에서 제거(Claude에게 생성 위임) / 비용은 haiku 배분과 fork 캐시로 통제 / 폭주는 maxTurns·훅·worktree로 방어 / 팀 표준은 `.claude/agents` 커밋 + managed 스코프로 배포.
- **자가 점검 4문항**: ①격리 모델과 fork·Teams의 경계를 설명할 수 있다(Part 1, 3) ②프론트매터로 역할별 에이전트를 정의할 수 있다(Part 2) ③멘션·병렬·체이닝·resume을 운용할 수 있다(Part 3) ④5패턴 중 둘 이상을 우리 저장소에 배포했다(Part 4~8).
- **다음 챕터 예고**: Chapter 3 — Admin Setup. 개인의 도구를 조직의 플랫폼으로(대규모 배포·버전 통제·managed settings, SSO/Bedrock IAM/키 볼트, 프록시·에어갭, 정책 계층·감사 로그·OTel 비용 관측).
