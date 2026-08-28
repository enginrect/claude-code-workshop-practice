# [4주차] Claude Code Deep Dive Workshop — Chapter 4 스터디 노트

> - 자료: [whchoi98/claude-code-workshop](https://github.com/whchoi98/claude-code-workshop) — Chapter 4 (115 slides, 9 Parts + Labs 4개)

---

## 개요 — settings, Permissions, Hooks, MCP, Commands 완전 정복

- Chapter 3가 **조직이 강제하는 것**이었다면, Chapter 4는 그 위에서 **개인과 팀이 조율하는 것**이다. 설정 스코프와 우선순위를 설계하고, 권한 규칙과 모드를 조율하며, 훅으로 수명주기를 자동화하고, MCP·커맨드로 나만의 개발 플랫폼을 완성한다.
- **`.claude` 디렉토리 지도** — 이 챕터가 다루는 모든 것이 사는 곳: `settings.json(.local)`(설정·권한·훅·env) / `CLAUDE.md`·`rules/`(Ch.1) / `agents/`(Ch.2) / `commands/`·`skills/` / `hooks/`(핸들러 스크립트 관례 위치) / 루트의 `.mcp.json`. `~/.claude`에 사용자 스코프 쌍둥이가 있고, **`.local`만 gitignore 대상**이며 나머지는 커밋해 팀 자산화한다.
- 목표는 **클론 즉시 팀 표준이 서는 저장소**.

## Part별 핵심 정리

### Part 1. Settings 체계

- **스코프 4계층**: Managed(서버 관리·plist/레지스트리·시스템 파일 — 조직 전체, Ch.3) / User(`~/.claude/settings.json` — 나의 전 프로젝트, 비공유) / Project(`.claude/settings.json` — 협업자 전원, 커밋 공유) / Local(`.claude/settings.local.json` — 이 저장소의 나만, gitignore). **취향은 User, 팀 표준은 Project, 실험과 개인 예외는 Local**.
- **우선순위 5단**: ①Managed(무엇으로도 재정의 불가) ②CLI 인자(세션 한정 임시) ③Local ④Project ⑤User.
- **기능별 파일 지도**: Settings는 settings.json 계열이지만 **MCP만 저장 파일이 다르다** — User/Local은 `~/.claude.json`, Project는 `.mcp.json`. Subagents는 `agents/`, CLAUDE.md는 별도 경로. `~/.claude.json`은 OAuth 세션·프로젝트별 상태·캐시를 담아 **직접 편집 지양**. 설정 파일은 최근 5개까지 자동 백업된다.
- **`$schema`**: `https://json.schemastore.org/claude-code-settings.json`을 넣으면 에디터 자동완성이 붙는다 — 오타로 인한 설정 무효화를 사전 예방.
- **`env` 블록**: 셸 프로파일 대신 settings로 환경변수를 **스코프별 배포**. 프로젝트에 커밋하면 팀 전체 변수가 통일된다(OTel 관측 배선, `USE_BEDROCK`·리전 고정 등). **키·토큰은 env 블록이 아니라 볼트 헬퍼로** — No secrets 원칙.
- **키 카탈로그 1(모델과 사고)**: `alwaysThinkingEnabled` / `availableModels`(선택 가능 모델 제한 — 메인·서브에이전트·스킬 공통) / `enforceAvailableModels`(managed 짝 키) / `advisorModel` / `agent`(세션 기본 에이전트 = Ch.2 `--agent`의 설정판). **`model`은 세션 시작 시 1회만 읽으므로 중간 변경은 `/model`**.
- **키 카탈로그 2(운영)**: `autoUpdatesChannel`(stable은 약 1주 지연 — 플릿 권장) / `autoCompactEnabled`(기본 true) / `autoMemoryEnabled` / `cleanupPeriodDays`(기본 30일, 고아 worktree도 정리) / `attribution`(커밋·PR 서명 문구) / `companyAnnouncements`(시작 공지 순환).
- **키 카탈로그 3(끄기 스위치)**: `disableAllHooks`(전체 훅 + 커스텀 상태줄 정지) / `disableBundledSkills` / `disableAutoMode`(값은 `"disable"`) / `disableAgentView` / `disableRemoteControl` / `disableSideloadFlags`(`--agents` 등 우회 플래그 거부, managed 전용).
- **라이브 리로드**: 설정 파일은 감시되어 **저장 즉시 반영**(permissions, hooks, apiKeyHelper 등 대부분). 변경마다 `ConfigChange` 훅이 발화한다. **예외 2개 — `model`(시작 시 1회), `outputStyle`(`/clear`나 재시작에 재구성)**.
- **`managed-settings.d` 드롭인**(systemd 관례): `10-telemetry.json`, `20-security.json`처럼 팀별 정책 조각을 알파벳순 병합 — 스칼라는 나중 파일이 덮고, 배열은 이어붙임 + 중복 제거, 객체는 깊은 병합, 숨김 파일은 무시. **팀마다 단일 파일을 조정할 필요가 없어진다**.
- **관용 파싱 vs 엄격 파싱**: managed는 **관용적**(무효 항목만 제거하고 나머지는 강제 유지, 단 보안 키는 fail-closed로 무효 allowlist는 빈 목록이 되고 `minimumVersion`류만 fail-open) / User·Project·Local은 **엄격**(검증 실패 시 파일 전체 거부, 고치기 전까지 해당 스코프 무시). **오타 하나가 정책 전체를 죽이지 않게** 하는 설계. 배포 전 `claude doctor`로 검증.
- **`/config`**: 탭형 설정 UI로 상태 확인·토글, `/config verbose=true`처럼 **단일 키 원샷 변경**(v2.1.181+). **어디에 저장되는지 표시되므로 스코프 확인 겸용**.

### Part 2. Permissions

- **3동사와 평가 순서**: `deny`(일치 시 즉시 거부, **다른 목록보다 항상 우선**) → `allow`(프롬프트 없이 실행, 생산성의 축) → `ask`(반드시 확인, 회색 지대용) → 어느 쪽도 아니면 **현재 권한 모드의 기본 거동에 위임**. 규칙은 전 스코프에서 병합(합집합)되고 동렬 충돌은 구체적인 쪽이 우선.
- **규칙 문법은 `Tool(specifier)` 한 형식**. 도구만 쓰면 그 도구의 모든 호출(`WebFetch`). 지정자는 도구별로 의미가 다르다 — Bash는 명령 패턴, Read·Edit는 파일 경로 패턴, Agent는 서브에이전트 타입(Ch.2), `mcp__server(__tool)`은 MCP 도구. **훅의 `if` 필드도 같은 문법을 재사용**한다.
- **Bash 패턴**: `Bash(npm run lint)`는 정확 일치, `Bash(npm run test *)`는 **명령 뒤 공백 + 별표**로 임의 인자 허용, `Bash(git *)`는 하위 전부. 셸 체이닝·치환으로 우회 가능성이 있으므로 **권한은 1차 방어이고 강한 봉쇄는 sandbox(Ch.3)의 몫**.
- **파일 경로 패턴**(gitignore 스타일 글롭): `./`는 프로젝트 루트 기준(`Read(./.env)`, `Read(./.env.*)`, `Read(./secrets/**)`), `~/`는 홈 기준, **절대 경로는 `//` 접두**(`Read(//etc/passwd)`). `**`는 깊이 무제한, `*`는 한 단계.
- **특수 지정자**: `Agent(Explore)`(특정 타입 통제) / `Agent`(지정자 없이 = **위임 자체 통제**, deny 시 전 위임 봉쇄) / `mcp__github`(서버 전체) / `mcp__github__get_issue`(개별 도구).
- **권한 모드 6종(Shift+Tab 순환)**
  | 모드 | 동작 | 쓰임 |
  |---|---|---|
  | default | 위험 작업마다 확인 | 일상 기본 |
  | acceptEdits | 작업 경로 편집 자동 수락 | 반복 수정 세션 |
  | plan | 읽기 전용 탐색, 계획 산출 | 설계 단계 |
  | auto | 분류기가 명령 심사 후 자동 결정 | 신뢰 저장소 |
  | dontAsk | 확인 프롬프트 자동 거부 | 무인, 명시 allow만 |
  | bypassPermissions | 전 확인 생략 | 격리 환경 한정, 조직 차단 대상 |
- **auto 모드 커스텀**: settings의 `autoMode`에 **산문(자연어) 규칙 배열**로 `environment`, `allow`, `soft_deny`, `hard_deny`를 지정하고 `$defaults`로 내장 규칙 상속 위치를 표시. `classifyAllShell: true`면 모든 셸 명령을 분류기로 보낸다(v2.1.193+). **공유 프로젝트 설정에서는 읽지 않으며 user/managed 스코프에서만 유효**.
- **`dontAsk`의 의미론**: 프롬프트가 뜰 상황이면 그 호출을 **거부**한다 → **명시 allow 목록이 곧 능력의 전부**. bypassPermissions가 "생략"이라면 dontAsk는 "거부"로, **안전 방향으로 넘어지는(fail-safe)** 모드. 훅 게이트·cron·CI 등 응답자 없는 실행에 쓴다.
- **`additionalDirectories`**: 규칙이 아니라 **작업 범위를 넓히는 키**. 지정 경로가 작업 디렉토리처럼 접근 가능해지고 acceptEdits 자동 수락 범위에도 포함되며, **추가 경로의 `.claude/agents/`도 함께 로드**된다. CLI 동등물은 `--add-dir`.
- **조직 정책과의 관계**: 병합은 합집합이지만 **deny 우선 원칙이 managed 차단을 절대선으로** 만든다 — 내 allow는 조직 deny를 이길 수 없다. 개발자가 할 수 있는 것은 자기 allow로 마찰 줄이기, 표준 제안, `additionalDirectories` 확장, 차단되지 않은 범위의 모드 선택.
- **`/permissions`**: 현재 유효 규칙을 스코프별로 열람하고 **어느 파일에서 온 규칙인지 출처를 표시**. 확인 프롬프트에서 "항상 허용"을 고르면 `settings.local.json`에 자동 기록되고, **local에서 검증한 뒤 project로 승격**하는 것이 팀 표준화 흐름.
- **흔한 실수 5가지**: `Bash(npm run test)`가 인자 붙자 불일치(→ 명령 뒤 공백 별표) / deny에 `.env`만 넣고 `.env.local` 누락(→ `.env*`, `secrets/**` 패턴화) / 허용 과다로 ask의 의미 소실(→ 회색 지대만 ask, 명확 위험은 deny) / local에 쌓인 규칙을 팀 표준에 미반영(→ 주기적 승격) / **권한만으로 네트워크 봉쇄를 기대**(→ sandbox 도메인 병행).

### Part 3. Hooks 아키텍처

- **5가지 핸들러 타입**: `command`(stdin JSON 수신, exit·stdout으로 응답 — 고전이자 중심) / `http`(이벤트 JSON을 URL로 POST, 응답 본문이 결정) / `mcp_tool`(연결된 서버의 도구를 핸들러로 직접 호출) / `prompt`(모델 단발 판정) / `agent`(도구를 가진 서브에이전트가 확인 후 판정). 매칭된 훅은 **병렬 실행**되고 동일 핸들러는 자동 중복 제거.
- **수명주기 3 케이던스**: 세션당(SessionStart/End, Setup) / 턴당(UserPromptSubmit, Stop, StopFailure) / 도구 호출당(PreToolUse, PostToolUse 계열) + 상시 감시(FileChanged, ConfigChange, CwdChanged).
- **이벤트 카탈로그(총 30개)**
  - *코어 8*: `PreToolUse`(실행 전, 차단 가능) / `PostToolUse`(성공 후 — 포맷터) / `PostToolUseFailure` / `UserPromptSubmit`(컨텍스트 주입) / `Stop`·`StopFailure` / `SessionStart`·`SessionEnd`(재개·clear 매처).
  - *에이전트와 권한*: `SubagentStart`/`Stop`(타입 매처) / `PermissionRequest` / `PermissionDenied`(auto 분류기 거부 시 **retry 지시 가능**) / `PostToolBatch`(병렬 묶음 완료 후) / `TaskCreated`·`Completed` / `TeammateIdle`.
  - *환경과 컨텍스트*: `FileChanged`(**매처가 파일명**) / `ConfigChange`(소스 매처) / `CwdChanged` / `InstructionsLoaded` / `Pre`·`PostCompact`(manual·auto 매처) / `WorktreeCreate`·`Remove`.
- **구성 3단 중첩**: 이벤트 → 매처 그룹 → 핸들러 목록. 정의 가능 위치는 6곳(user·project·local settings + managed, 플러그인 `hooks.json`, skill/agent frontmatter). 경로는 `${CLAUDE_PROJECT_DIR}` 플레이스홀더로.
- **matcher 문법 — 문자 구성이 해석 방식을 결정**: `"*"`·`""`·생략은 전체 매칭 / 영숫자·`_`·`-`·공백·`,`·`|`만 있으면 **정확 문자열**(리스트 구분자 `|` 또는 `,`) / **그 외 문자가 섞이면 JS 정규식**(비앵커). **함정: `mcp__memory`는 정확 문자열로 해석되어 무매칭 → `mcp__memory__.*`가 필수**. `Edit.*`는 NotebookEdit도 잡으므로 `^Edit$` 권장.
- **`if` 필드**: 권한 규칙 문법 **1개**로 2차 필터(조합 문법 없음). **Bash 서브커맨드까지 검사**해서 `npm test && git push`는 `Bash(git *)`에 매칭되고, `echo $(rm -rf /)`는 `Bash(rm *)`에 매칭되며, `VAR=x git push`는 선행 대입 제거 후 매칭. 불일치면 프로세스를 스폰하지 않아 비용 절약. **파싱 불가면 fail-open이므로 강제는 권한의 몫이고 훅은 보조**.
- **입력 JSON 공통 필드**: `session_id`, **`prompt_id`(OTel 이벤트와 상관 연결)**, `transcript_path`, `cwd`, `permission_mode`, `effort`, 그리고 도구 이벤트 한정으로 `tool_name`·`tool_input`. command 훅은 stdin, http 훅은 POST 본문으로 동일 JSON을 받는다.
- **결정 출력 — 침묵은 승인이 아니다**: 권장은 JSON(`hookSpecificOutput.permissionDecision: "deny"` + `permissionDecisionReason`), 고전 방식은 `exit 2` + stderr(사유가 Claude에게 전달). **`exit 0` + 무출력은 "결정 없음"**으로 정상 권한 흐름이 계속된다. 사용자 표시는 `systemMessage`, 알림은 `terminalSequence` — **훅은 무단말 세션이라 `/dev/tty` 접근 불가**(v2.1.139+).
- **exec form vs shell form**: `args`가 있으면 셸 없이 실행 파일을 직접 스폰(인용·공백 걱정 없고 플레이스홀더가 그대로 치환 — **권장**), 없으면 셸이 문자열을 해석(파이프·`&&`·리다이렉트 가능, 플레이스홀더는 따옴표로 감싸야 함). Windows는 `.cmd` 심이 안 되므로 node + 스크립트로.
- **async와 timeout**: 기본 timeout은 command·http·mcp_tool 600초, prompt 30초, agent 60초이고 **이벤트별로 하향**된다(UserPromptSubmit 30초, MessageDisplay 10초). `async: true`는 턴을 막지 않고, `asyncRewake`는 백그라운드 + exit 2 시 Claude를 깨우며, `statusMessage`로 스피너 문구를 커스텀하고, `once`는 세션당 1회(스킬 frontmatter 전용).

### Part 4. Hooks 실전

- **HTTP 훅**: 이벤트 JSON을 POST로 전송. `headers`에 `$HOOK_TOKEN` 같은 변수를 쓰려면 **`allowedEnvVars`에 명시**해야 보간된다. **차단은 `2xx` + `permissionDecision: deny` 본문이 유일한 경로** — 비2xx·타임아웃은 비차단 오류로 계속 진행된다. 조직은 `allowedHttpHookUrls`로 목적지를 통제.
- **mcp_tool 훅**: `server`와 `tool` 2필드 지정, `input`에 `${tool_input.file_path}`처럼 **이벤트 필드를 치환 주입**. 스크립트 없이 서버 도구를 핸들러로 쓴다. **서버가 이미 연결 상태여야 하며 OAuth를 유발하지 않으므로**, SessionStart·Setup에서는 연결 전이라 첫 실행에 유의.
- **prompt / agent 훅**: `prompt`는 단발 판정(기본 fast model, timeout 30 — **haiku 권장**), `agent`는 Read·Grep 등 도구로 확인 후 판정(**실험적**). 입력 JSON은 `$ARGUMENTS` 자리에 들어간다. **정규식으로 잡기 어려운 의미 기반 판정**("마이그레이션 파일을 건드리면 deny")에 쓴다.
- **Recipe 1 / Auto Format**: `PostToolUse` + matcher `Edit|Write` → `format.sh`가 `jq -r '.tool_input.file_path'`로 파일을 뽑아 확장자별 분기(`prettier --write`, `ruff format`). 커밋해서 팀 표준 훅으로.
- **Recipe 2 / 보호 가드**: `block-rm.sh`가 명령에 `rm -rf`가 있으면 deny JSON을 사유와 함께 출력, 아니면 `exit 0`으로 무결정 통과. 등록은 matcher `Bash` + **`if "Bash(rm *)"`로 스폰 절약**, 권한 deny 규칙과 이중으로.
- **Recipe 3 / 알림**: `Notification` 이벤트를 **유형 매처**(`permission_prompt|agent_needs_input`, `idle_prompt`, `agent_completed`)로 선별해 `notify-send`, `Stop`에 http 핸들러로 Slack 웹훅. 장기 작업에 유용하다.
- **Recipe 4 / 반응형 환경**: `FileChanged` 매처에 `.envrc|.env`를 넣어(**매처 = 감시할 파일명 목록**) `direnv allow`를 async 실행, `CwdChanged`로 cd마다 환경 재정렬. 모노레포 환경 전환에 유용.
- **Recipe 5 / Setup과 컨텍스트 주입**: `claude --init-only`가 `Setup` 이벤트를 발화하므로 CI 1회 준비(`npm ci && cp .env.ci .env`)에 쓴다. `UserPromptSubmit` 훅의 **stdout은 컨텍스트로 추가**되므로 `git status --short | head -5`로 매 턴 최신 상태를 주입(30초 상한 주의).
- **`/hooks` 메뉴**: 구성된 전 훅의 **읽기 전용 브라우저** — 이벤트별 개수·매처·핸들러 상세, `[type]` 접두와 6개 소스 라벨(User, Project, Local, Plugin, Session, Built-in)로 출처 추적. 문제 시 **`disableAllHooks`로 훅 원인 여부부터 이분 판정**.

### Part 5. MCP 구성

- **3 프리미티브**: Tools(모델이 호출하는 동작, `mcp__서버__도구` 이름으로 등장) / Resources(읽을 데이터, **`@` 멘션으로 컨텍스트 첨부**) / Prompts(서버 제공 정형 명령, 슬래시로 노출). 역할 구도는 **Claude Code = 클라이언트, 연결 대상 = 서버**.
- **전송 4종**: `stdio`(로컬 프로세스 스폰, npx 배포 서버의 표준) / `http`(원격 엔드포인트, 관리형 서비스 표준) / `sse`(레거시 원격, http로 이행 추세) / `ws`(실시간성 요구). **로컬 도구는 stdio, SaaS는 http**.
- **설치**: `claude mcp add playwright -- npx -y @playwright/mcp@latest`(**`--` 뒤가 실행 명령**), 원격은 `--transport http <name> <url>`, 스코프는 `-s local|project|user`(기본 local). 관리는 `list` / `get <name>` / `remove <name>`.
- **스코프 3종과 저장 위치**: local(기본, `~/.claude.json`의 프로젝트별 항목 — 나만) / project(**`.mcp.json` 저장소 루트 — 커밋 공유, 팀 표준**) / user(`~/.claude.json` 전역 — 내 전 프로젝트). 동명 시 **local > project > user**. **project 서버는 최초 사용 시 승인 절차**가 있어 저장소발 구성으로부터 보호된다.
- **`.mcp.json`**: `${WIKI_TOKEN}` 환경변수 확장과 `${DB_HOST:-localhost}` 기본값 폴백을 지원 → **토큰 실값을 파일에 절대 넣지 않고 외부화**한 채로 커밋한다.
- **OAuth 흐름**: http 서버 add(미인증) → `/mcp`에서 서버 선택 후 Authenticate → 브라우저에서 제공자 로그인·권한 동의 → 토큰 로컬 안전 저장 및 자동 갱신. 재인증·로그아웃도 `/mcp`에서.
- **소비 문법**: 도구는 자연어로 호출(내부적으로 `mcp__github__list_issues`), 리소스는 `@corp-wiki:onboarding/backend` 멘션, 서버 프롬프트는 `/mcp__github__pr_review 123`. 권한 규칙 대상은 `mcp__github` 또는 개별 도구.
- **`/mcp` 대시보드**: 서버별 연결 상태(connected/failed), 도구·리소스 목록과 스코프 출처, OAuth 인증 관리, **세션 단위 활성 토글**. 서버가 안 보이면 여기부터.
- **claude.ai 커넥터**: 웹에서 연결한 커넥터가 CLI 세션에 자동 등장한다. `disableClaudeAiConnectors`를 프로젝트에 커밋해 거부할 수 있고, **어느 소스든 `true`면 차단**되어 `false`로 재개방 불가. managed-mcp.json 배포 시에는 기본 배제.
- **Tool Search(도구 검색)**: 서버가 많아지면 도구 스키마만으로 시작 컨텍스트가 잠식된다 → **도구를 인덱스로만 두고 검색 후 정의를 온디맨드 로드**. 도구 수가 임계를 넘으면 자동 전환되며 시작 토큰이 급감한다.
- **컨텍스트 예산**: `/context`에서 MCP tools 비중을 확인 — 잠식 신호는 시작부터 수십 퍼센트 점유, 안 쓰는 서버의 user 스코프 상주. 처방은 **프로젝트별 필요 서버만 `.mcp.json`에, 개인 상주는 최소화, 무거운 서버는 서브에이전트 `mcpServers`로(Ch.2), `/mcp` 토글로 세션 단위 오프**.

### Part 6. MCP 운영과 보안

- **인기 서버 카탈로그**: github(이슈·PR·코드 검색, http+OAuth) / playwright(브라우저 구동·E2E, stdio+npx) / sentry / slack / postgres 계열(스키마 조회·쿼리, stdio + env 자격) / notion·linear·figma.
- **사내 MCP 서버**: 위키 검색·배포 상태·티켓 CRUD·사내 API를 노출하면 **조직 컨텍스트가 도구가 된다**. 제작은 TypeScript·Python MCP SDK(Ch.6 심화), 다수 사용자 배포는 http 엔드포인트 권장, 운영은 OAuth 인증 + allowlist 등재 + 버전 관리와 결합.
- **신뢰 모델 — 서버 응답은 모델이 읽는 입력이다**: 미검증 서버의 도구 설명·응답에 지시가 삽입될 수 있고, 외부 데이터(이슈·문서)를 경유한 **간접 프롬프트 인젝션**도 가능하다. **서버 선택 자체가 보안 결정**. 방어는 공식·사내 서버 우선 채택, 프로젝트 서버 최초 승인 신중, **쓰기 도구는 ask·민감 도구는 deny**, 외부 데이터 처리 시 훅 검증 결합.
- **조직 통제(Ch.3 managed 키의 MCP 편)**: `allowedMcpServers`(**빈 배열 = 전면 봉쇄**) / `deniedMcpServers`(allowlist보다 우선, 사고 대응 즉시 배포) / `allowManagedMcpServersOnly` / `managed-mcp.json`(조직 표준 서버 배포, 커넥터 기본 배제) / `disableSideloadFlags`(`--mcp-config` 우회 거부). **적용 범위에 서브에이전트 인라인 정의가 포함**되어 Ch.2 경로의 우회도 봉쇄된다.
- **훅 결합**: matcher `mcp__.*__(write|create|delete).*`로 **쓰기 계열 정규식 가드**, `mcp__corp-wiki__.*`로 서버 전량을 async 감사 로그(`jq -c '{ts:now,tool:.tool_name}' >> ~/mcp-audit.jsonl`).
- **성능**: stdio 다중 스폰과 원격 핸드셰이크가 시작을 늦춘다 → 도구 검색·스코프로 컨텍스트 완화, **`--strict-mcp-config`로 지정 구성 외 로드 배제**(CI 재현성), **`--bare`로 MCP 등 확장 없이 최소 기동**(진단·벤치).
- **트러블슈팅**: 서버 안 보임 → 스코프·신뢰 승인 미완 / connection failed → 명령 경로·URL·네트워크(단독 실행, curl 검증) / 401 → OAuth 만료·토큰 변수 미설정 / 도구 호출 거부 → **권한 규칙 미허용인지 조직 차단인지 구분** / 느린 시작 → 서버 과다.
- **운영 모범 사례 4가지**: 팀 서버는 `.mcp.json` 커밋하고 개인 상주는 최소로 / 토큰은 `${VAR}`와 볼트, 파일에는 절대 금지 / **읽기는 넉넉히, 쓰기는 ask + 훅 가드** / 분기마다 미사용 서버 제거와 버전·인증 상태 점검.

### Part 7. Commands와 Skills

- **확장 지형 6수단 한 줄 판별**: CLAUDE.md(항상 알아야 할 지식 — 상시 컨텍스트) / Command·Skill(반복 작업의 정형 절차 — **사용자가 호출**) / Subagent(격리 컨텍스트의 전담 워커 — 위임 실행) / Hook(수명주기 자동 반응 — **이벤트 구동**) / MCP(외부 시스템 연결) / Plugin(위 전부의 배포 묶음).
- **커스텀 명령 기본**: `.claude/commands/summarize.md` 한 장이 `/summarize`가 된다. **파일명이 곧 명령 이름, 하위 폴더는 네임스페이스**(`/dir:name`). 위치는 프로젝트 `.claude/commands/`와 개인 `~/.claude/commands/`, 워처로 즉시 반영.
- **인자**: `$ARGUMENTS`(전체 문자열), `$1`·`$2`(위치 인자), `argument-hint`로 자동완성 안내(`"<이슈번호> [우선순위]"` 관례). 미전달 인자는 공백.
- **frontmatter 옵션**: `description`(명령 목록과 **모델의 자동 호출 판단**에 쓰임 — 필수 습관) / `argument-hint` / `allowed-tools`(실행 중 도구 제한 — 최소 권한) / `model`(무거운 분석은 opus) / `hooks`(명령 활성 중 스코프 훅, `once` 지원) / **`disable-model-invocation`**(모델의 자동 호출 금지 = 사용자 전용 명령).
- **컨텍스트 수집**: **`` !`cmd` ``로 인라인 실행 결과 삽입**, `@docs/review-checklist.md`로 파일 첨부. 실행 시점에 평가되므로 항상 최신 상태가 들어간다. 조직은 `disableSkillShellExecution`으로 차단 가능.
- **Skills 구조**: 폴더 단위로 `SKILL.md`(진입 본문 + frontmatter) + 동봉 리소스(`checklist.md`, `scripts/verify.sh`). **`description` 기반으로 모델이 자동 발동**하고 `/이름`으로 직접 호출도 된다.
- **스킬 고급 옵션**: `context: fork`(격리 컨텍스트에서 실행 후 요약만 반환, 메인 컨텍스트 보호) / `allowed-tools` / `hooks` + `once`(활성 중 스코프 훅, 세션 1회 — 준비 작업에 최적) / `disable-model-invocation`(위험 절차 안전판) / 번들 스킬은 `disableBundledSkills`로 조직 비활성 / 플러그인으로 묶어 유통.
- **실전 명령 1 — PR 리뷰**: `allowed-tools`에 `Agent`를 포함하고 `` !`git merge-base HEAD ${1:-main}` ``로 기준을 수집한 뒤 **code-reviewer 서브에이전트에 위임**(Ch.2 재사용), Critical/Warning/Suggestion으로 정리하고 마지막 줄에 머지 가능 여부 판정(자동화 접점). `${1:-main}`이 인자 기본값 관용구.
- **실전 명령 2 — 배포 점검**: `disable-model-invocation: true`로 **사용자 전용**으로 못 박고, 테스트·미커밋 상태를 수집해 **체크리스트 통과 시에만 배포 명령을 제시**하는 관문 로직. `allowed-tools`에 배포 명령 자체는 넣지 않고 권한 ask와 이중 관문.
- **팀 라이브러리 3단 성장**: `~/.claude/commands`에서 개인 검증 → `.claude/commands` 커밋으로 **PR 리뷰 대상화**(명령도 코드) → 다저장소 공통은 플러그인 + 마켓 배포. 명명 규칙·설명 품질·분기 정리 담당자를 둔다.
- **선택 가이드**: 짧은 정형 프롬프트 → Command 한 장 / 절차 + 동봉 자산 → Skill 폴더 / 무거운 실행 격리 → Skill `context: fork` / 역할 + 도구 제한 워커 → Subagent / 자동 반응 → Hook. **판별 축은 "누가 부르나"와 "상태 격리가 필요한가" 두 질문이면 충분**.

### Part 8. 통합과 트러블슈팅

- **`.claude` 풀스택 저장소의 모습**: `settings.json`(permissions 팀 표준, env 관측, hooks 3종) + `hooks/`(format.sh, block-rm.sh, mcp-write-guard.sh) + `commands/`·`skills/`(pr-review, deploy + deploy-check) + `.mcp.json`(corp-wiki, github, db) + `agents/`(Ch.2 워커들).
- **진단 4도구**: `/context`(컨텍스트 점유 분해 — 비대 원인 특정) / `/doctor`(환경·설정 검증과 자동 수정, **무효 항목 출처 표시**) / `/hooks`(타입·매처·소스) / `/mcp`(서버 상태·도구·인증). 보조로 `/status`, `/permissions`, `--verbose`.
- **설정이 안 먹을 때 4단계**: ①`/doctor`로 파일 유효성·오류 확인 ②`/status`·`/permissions`의 **출처 표기로 스코프 특정** ③**상위 스코프가 같은 키를 덮었는지**(범인 1순위) ④`model`·`outputStyle`은 재시작 필요한 예외. 그래도 안 되면 **새 터미널 + `--bare` 최소 기동 후 하나씩 복원**하는 이분법.
- **Hooks 트러블슈팅**: 안 발화 → 매처 불일치·이벤트 오선택 / **`mcp__server` 무매칭 → 정확 문자열로 해석되므로 `__.*` 접미 추가** / 차단이 안 됨 → **http 비2xx는 비차단**이므로 `2xx + deny 본문`으로 / 스크립트 오류 → 경로·실행 권한·jq 부재(단독 실행 재현) / 프롬프트 지연 → UserPromptSubmit이 무거움(async 또는 30초 내로) / 전부 침묵 → `disableAllHooks` 잔존.
- **커밋 전 보안 4관문**: ①비밀 없음(settings, `.mcp.json`, 훅 스크립트에 토큰 실값 금지) ②deny 기본선 유지(`.env*`, `secrets/**`, 파괴 명령) ③**훅 스크립트도 코드 리뷰 대상**(외부 전송은 사유 명시) ④`.mcp.json` 서버 출처 확인과 쓰기 도구 관문. **`.claude` 풀스택은 강력한 만큼 저장소발 실행 경로이기도 하다**.

### Part 9. Recap

- **여섯 문장 요약**: ①Settings — 4스코프 5단 우선순위, 권한은 병합, 저장 즉시 리로드 ②Permissions — `Tool(specifier)` 한 형식, deny 우선, 모드 6종 ③Hooks — 30 이벤트 3 케이던스, 5 핸들러, **침묵은 무결정** ④MCP — 3 프리미티브·4 전송·3 스코프, 도구 검색으로 규모 대응 ⑤Commands — Markdown이 `/명령`, 스킬로 승격, 플러그인 유통 ⑥Ops — 진단 4도구, 3중 방어, 커밋 전 보안 4관문.
- **FAQ 6선**: 설정을 고쳤는데 반영이 안 됨 → **`model`·`outputStyle`만 재시작, 그 외는 덮임 의심** / allow에 넣었는데 계속 물어봄 → 상위 deny 또는 ask 병합(`/permissions` 출처 확인) / 훅이 승인해 줄 수 있나 → **침묵은 무결정이고 allow 결정도 가능하나 기본은 deny 용도** / `mcp__서버` 매처가 안 걸림 → 정확 문자열 해석 함정, `__.*` 필수 / 서버가 많아 시작이 느림 → 도구 검색 + 스코프 축소 / 명령과 스킬 차이 → **한 장이면 명령, 자산 동봉이면 스킬**.
- **자가 점검 4문항**: ①스코프·우선순위·리로드 예외를 설명할 수 있다(Part 1) ②규칙 문법과 모드로 팀 표준을 작성할 수 있다(Part 2) ③이벤트·매처·결정 출력으로 레시피를 구현할 수 있다(Part 3, 4) ④MCP와 명령을 커밋해 풀스택을 구성했다(Part 5~8).
- **다음 챕터 예고**: Chapter 5 — CLI Reference. 대화형을 넘어 스크립트와 파이프라인의 세계(플래그 전집, 헤드리스와 JSON 출력 파싱, `continue`·`resume`·`fork` 세션 제어, CI·cron·Routines 통합).
