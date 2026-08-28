# [5주차] Claude Code Deep Dive Workshop — Chapter 5 스터디 노트

> - 자료: [whchoi98/claude-code-workshop](https://github.com/whchoi98/claude-code-workshop) — Chapter 5 (107 slides, 9 Parts + Labs 4개)

---

## 개요 — 대화형 도구에서 파이프라인의 부품으로

- Chapter 1~4가 **사람이 조타수**인 세계였다면(확인과 반복, 한 번에 한 세션, 성과는 개인 생산성에 머문다), Chapter 5는 **스크립트가 조타수**인 세계다. exit code로 판정하고, cron·CI·Routines로 병렬 반복하며, 성과가 파이프라인 자산으로 남는다.
- 두 세계를 가르는 스위치는 **`-p` 한 글자**. 그런데 Ch.4까지 만든 설정·훅·MCP 자산은 그대로 살아 있다 — `-p`는 출력 모드가 아니라 **Agent SDK 경로를 타는 단발 에이전트 실행**이기 때문이다.
- 무인의 전제는 **상한**(예산·턴)과 **판정 가능성**(exit code, JSON 계약) 둘. 이 챕터 전체가 그 두 가지의 확장이다.
- 학습 목표 4개: **플래그 정복**(6분류로 어떤 조합도 스스로) / **파이프라인**(JSON·구조화 출력으로 스크립트 통합) / **세션 자유**(이어가기·분기·웹 왕복) / **안전 자동화**(예산·턴·도구 상한 아래 CI와 스케줄).

## Part별 핵심 정리

### Part 1. claude 명령과 플래그

- **명령 구조는 `claude [플래그] [프롬프트]` 하나**. 형태는 3종 — `claude`(대화형), `claude "explain this project"`(초기 프롬프트로 시작), `claude -p "query"`(실행 후 종료). 여기에 **파이프로 stdin 입력**(`cat logs.txt | claude -p "explain"`)과 **세션 복귀**(`-c`, `-r`)가 붙는다. 서브커맨드 오타는 근접 제안이 뜬다(`claude udpate` → `Did you mean claude update?`).
- **서브커맨드 지도 1(계정·설치)**: `auth login/logout`(`--email`·`--sso`·`--console`로 구독 vs API 과금 선택) / **`auth status`**(JSON 상태, `--text`, **exit 0/1** — 스크립트 로그인 판정) / **`setup-token`**(CI용 장기 OAuth 토큰 발급, 출력만 하고 저장 안 함) / `update`·`install`(`install stable`, 버전 지정 재설치) / `doctor`(환경 진단과 자동 수정) / `project purge [path]`(프로젝트 로컬 상태 일괄 삭제, `--dry-run`·`--all`).
- **서브커맨드 지도 2(운영)**: `agents`·`attach`·`logs`(`--json`, Ch.2 백그라운드 관리) / `stop`·`respawn --all`·`rm` / `daemon status|stop`(`--keep-workers`) / `mcp login|logout`(`--no-browser` — SSH 환경 인증) / `gateway --config`(Ch.3 게이트웨이 기동) / **`ultrareview [PR]`**(비대화 심층 리뷰, `--json`·`--timeout`, **통과 0 / 발견 1**).
- **플래그 6분류(60여 개를 담는 여섯 서랍)**: ①동작 모드(`-p`, `--bg`, `--remote`, `--worktree`, `--bare`, `--safe-mode`) ②세션(`-c`, `-r`, `--from-pr`, `--fork-session`, `-n` → Part 3) ③모델과 사고(`--model`, `--effort`, `--fallback-model`, `--advisor`) ④권한과 도구(`--permission-mode`, `--tools`, `--allowed/disallowedTools`) ⑤구성과 확장(`--settings`, `--agents`, `--mcp-config`, `--plugin-dir`) ⑥출력과 진단(`--output-format`, `--json-schema`, `--verbose`, `--debug` → Part 2, 8).
- **동작 모드 플래그 상세**: `--bg`(백그라운드 기동, **세션 ID 반환**, `-p`와 병용 불가) / `--exec`(셸 명령을 PTY 잡으로) / `-w`·`--worktree`(격리 워크트리, **`#PR번호`로 분기**, `--tmux` 병용) / `--remote`·`--teleport`(웹 세션 생성/로컬 회수) / `--bare`(훅·스킬·MCP 미탐색 최소 기동) / `--init-only`·`--init`(Setup 훅만 실행 / `-p` 전 준비).
- **모델과 사고**: 별칭 4종(sonnet·opus·haiku·fable) 또는 전체 ID, `--effort low..max`(모델별 상이), **`--fallback-model sonnet,haiku`는 콤마 순차 시도**(과부하·미제공 시 폴백 체인, `fallbackModel` 키로 영구화). **우선순위는 플래그 > `ANTHROPIC_MODEL` > 설정**.
- **`--tools` vs `--allowedTools` vs `--disallowedTools`** — 가장 헷갈리는 3형제. `--allowed-tools "Bash(git log *)" "Read"`는 **규칙 문법으로 무확인 허용**. `--disallowedTools`는 지정자 유무로 의미가 갈린다 — **베어 이름은 도구 자체를 목록에서 제거**(`"Edit"` → Edit 도구 소멸, `"mcp__*"` → 전 MCP 도구 제거), **스코프 규칙은 해당 호출만 거부**(`"Bash(rm *)"`). `--tools "Bash,Edit,Read"`는 **내장 도구 한정**(MCP 미영향). `--add-dir`은 범위 확장이되 구성은 탐색하지 않는다.
- **구성과 확장**: `--settings ./ci-settings.json` 또는 **인라인 JSON**(`--settings '{"model":"haiku"}'`)으로 세션 한정 오버레이 / **`--setting-sources user,project`로 로드할 설정 소스 선별**(CI 격리 재현의 핵심) / `--agents '{...}'`, `--mcp-config ./mcp.json --strict-mcp-config`(지정 외 MCP 배제), `--plugin-dir`, `--disable-slash-commands`. **이 계열은 Ch.3 `disableSideloadFlags`의 차단 대상**이라는 점을 함께 기억.
- **시스템 프롬프트 4종의 갈림은 "정체성 유지냐 대체냐"**: `--append-system-prompt`(뒤에 텍스트 추가 — 코딩 조수 정체성 유지 + 규칙) / `--append-system-prompt-file`(긴 규칙, 버전 관리) / **`--system-prompt`(전체 대체 — 안전 지침도 함께 사라진다, 비코딩 에이전트용)** / `--system-prompt-file`(대체와 상호 배타). **영구 페르소나는 output style, 상시 규범은 CLAUDE.md**가 제자리.
- **`--bare` vs `--safe-mode` — 닮았지만 목적이 다르다**. `--bare`는 **속도를 위한 최소 기동**(훅·스킬·플러그인·MCP·CLAUDE.md 미탐색, Bash·읽기·편집은 사용 가능, `CLAUDE_CODE_SIMPLE` 설정됨) — `-p` 반복 호출의 기동 시간 절약용. `--safe-mode`는 **고장 원인 이분을 위한 무커스텀 기동**(전 커스터마이즈 비활성, 인증·권한은 정상, **managed 정책 훅은 여전히 적용**).
- **현장에서 굳어진 조합 관용구 5**: ①CI 리뷰 `claude -p --max-budget-usd 2 --allowed-tools "Read" "Grep" ...` ②빠른 배치 `claude --bare -p --model haiku "..."` ③격리 실험 `claude -w '#123' --permission-mode plan` ④세션 재현 `claude --setting-sources project --settings ./ci.json -p "..."` ⑤무인 야간 `claude -p --permission-mode dontAsk --fallback-model sonnet,haiku "..."`.

### Part 2. Headless 심화

- **`-p`의 정체 — 단순 출력 모드가 아니다**. Agent SDK 경로를 타는 **단발 에이전트 실행**이라 도구 실행·훅·MCP·권한이 대화형과 동일하게 살아 있고, 결과만 표준출력에 남는다. `--max-turns`, `--max-budget-usd`, `--json-schema` 등은 **`-p` 전용 플래그**.
- **계약 3종**: **stdout = 결과, stderr = 진단, exit code = 판정**. 파이프라인이 Claude와 맺는 인터페이스가 이 셋뿐이다.
- **입력 6경로**: ①인자(`claude -p "직접 인자"`) ②파이프(`cat error.log | claude -p "원인 분석"`) ③명령 치환(`claude -p "$(cat prompt.txt)"`) ④리다이렉트(`claude -p "요약해" < notes.md`) ⑤히어독 ⑥세션 이어받기(`claude -c -p "이어서 리팩토링"`). 관례는 **파이프 = 데이터, 인자 = 지시**. 프롬프트 안 `@경로` 파일 참조도 그대로 유효.
- **Exit code 계약**: **0 = 정상 완료, 비0 = 오류·상한 도달**. **`--max-turns` 초과도 오류로 종료되므로 그대로 게이트 재료**가 된다. `claude auth status`는 로그인 0 / 미로그인 1, `claude ultrareview`는 통과 0 / 발견 1. **`set -e` 아래에서는 의도적 분기를 따로 처리**해야 한다.
- **`--output-format json` 봉투 스키마**: `type`, `subtype`, **`result`(본문)**, `session_id`(재개 열쇠), `total_cost_usd`, `num_turns`, `duration_ms`, `usage{input_tokens, output_tokens}`, `is_error`(오류 봉투 분기). **`.result`가 본문이고 나머지는 전부 관측 메타데이터**.
- **`stream-json` 이벤트 4종**: `system`(init — 세션 시작, 모델과 도구 목록) → `assistant`(응답 메시지 단위) → `user`(tool_result — 도구 실행 결과 회신) → `result`(최종 봉투, json과 동일). `--include-partial-messages`로 토큰 단위 부분 이벤트(타자기 UI), `--include-hook-events`로 훅 수명주기 이벤트(심층 관측)를 추가한다.
- **`--json-schema` — 파싱이 아니라 계약**. JSON Schema를 넘기면 워크플로 완료 후 **검증된 JSON만 반환**한다. `"risk": {"enum": ["low","medium","high"]}`, `"block": {"type":"boolean"}`처럼 **enum과 required로 게이트 친화적으로 설계**하는 것이 요령. 정규식으로 출력을 긁던 파이프라인이 계약 기반으로 바뀐다. **`-p` 전용**이고 SDK의 Zod·Pydantic과 동형(Ch.6).
- **jq 파싱 관용구**: `jq -r '.result'`(본문) / `jq -r '.total_cost_usd'`(비용) / `jq -r '.session_id'`(재개 키) / stream-json은 `jq -c 'select(.type == "result")'`로 줄 단위 필터. **구조화 출력과 결합하면 게이트가 한 줄**이 된다 — `RISK=$(claude -p "..." --json-schema "$SCHEMA" | jq -r '.risk'); [ "$RISK" = "high" ] && exit 1`.
- **예산과 턴 상한 = 무인 실행의 안전벨트**: `--max-turns`는 에이전틱 턴 수 상한이고 **기본이 무제한이라 무인에서는 반드시 지정**한다. `--max-budget-usd`는 달러 상한(도달 시 중단). 산정 요령은 **파일럿 실측 p95의 1.5배로 시작**. **초과 종료는 실패가 아니라 신호** — 로그와 알람으로 다룬다. 조직 층 안전망은 Ch.3 게이트웨이 한도와 병행.
- **캐시 최적화**: 시스템 프롬프트의 **기기별 섹션(작업 경로, 환경 정보)이 프롬프트 캐시 재사용을 깨뜨린다**. `--exclude-dynamic-system-prompt-sections`는 그 섹션을 첫 사용자 메시지로 옮겨, **여러 사용자·여러 기계가 같은 작업을 돌릴 때 캐시 적중률을 올린다**. 기본 시스템 프롬프트일 때만 적용.
- **에러 처리 — 일시 오류와 진짜 실패를 분리**한다. 재시도 골격은 3회 백오프(`sleep $((attempt * 20))`)하되, **stderr를 `grep -qiE 'rate|overloaded|529'`로 검사해 일시 오류일 때만 재시도**하고 아니면 즉시 중단. 모델 과부하의 1차 방어는 **재시도보다 `--fallback-model`이 우아**하다.
- **다중 호출 집계**: 파일마다 `claude --bare -p ... --json-schema '...' < "$f"`를 돌리고 **`jq -s`(slurp)로 배열화**, `@csv`로 표 변환. 집계가 가능한 조건은 **구조화 출력이 전제**라는 점, 배치 비용 통제는 **`--bare` + `haiku` 조합**.
- **완성 스크립트(`nightly-deps-audit.sh`)가 이 파트의 부품을 한 몸으로 묶는다** — 재시도 함수 + 스키마 + 날짜별 리포트 축적 + `[ "$CRIT" -gt 0 ] && notify && exit 1` 게이트. 이 exit 게이트 하나로 cron과 CI 양쪽에 그대로 얹힌다(Part 4, 5).

### Part 3. 세션 제어

- **저장 구조**: 세션 트랜스크립트는 **`~/.claude/projects/` 아래 프로젝트별 JSONL**. 수명은 `cleanupPeriodDays`(기본 30일 자동 정리, Ch.4), 일괄 정리는 `claude project purge`(`--dry-run`으로 예행). 저장을 끄려면 `-p`는 `--no-session-persistence`, 전 모드는 env 변수. **훅 입력의 `transcript_path`가 이 파일을 가리킨다**(Ch.4).
- **`-c` vs `-r` — 가장 최근 하나냐, 골라잡기냐**. `-c`·`--continue`는 **현재 디렉토리의 최근 대화로 직행**(add-dir로 얹은 세션 포함, `-c -p`로 헤드리스 이어받기). `-r`·`--resume`은 **ID나 이름으로 특정**하거나 인자 없이 대화형 픽커. 픽커에는 bg 세션도 표시되고(v2.1.144+), **ID 검색은 현 프로젝트와 워크트리 한정**이다.
- **`--from-pr` — PR이 세션의 주소가 된다**. Claude가 만든 PR은 세션과 자동 링크되므로 `claude --from-pr 123` 또는 PR URL로 **그 PR을 만든 맥락 그대로 후속 수정**에 들어간다. GitHub·GitHub Enterprise·GitLab MR·Bitbucket PR URL을 모두 수용. **`-w '#123'`(코드 분기 워크트리)과는 별개**의 개념.
- **fork와 고정 좌표**: `claude --resume auth-refactor --fork-session`은 **재개하되 새 세션 ID로 분기**해 원본을 보존한다(같은 지점에서 실험 A·B 두 갈래). 반대로 스크립트가 세션을 소유하려면 **`SID=$(uuidgen)` + `--session-id "$SID"`로 좌표를 고정**하고 `--resume "$SID"`로 다단 파이프를 잇는다. 대화형 `/fork`(Ch.2)의 CLI 판.
- **이름과 내보내기**: 시작부터 `-n "payments-refactor"`, 세션 중에는 `/rename auth-hotfix`(프롬프트 바에도 표시), 복귀는 `claude -r "payments-refactor" "어제 이어서"`. `/export`는 대화를 파일·클립보드로 내보내고 **원본 JSONL은 projects 디렉토리에 그대로** 남는다. 팀 관례는 티켓 번호 명명.
- **체크포인트와 `/rewind`**: Claude의 파일 편집은 **편집 시점마다 체크포인트로 자동 추적**되고, `/rewind`로 코드와 대화를 지점 선택 복원한다. **한계는 Bash 부수효과와 외부 시스템** — 복원 범위 밖이다. **세션 내 무르기는 rewind, 이력의 본진은 git**이라는 분업.
- **웹 왕복(`--remote` / `--teleport`)**: `claude --remote "로그인 버그 수정"`으로 위임 → claude.ai 샌드박스에서 실행 → 웹·모바일 앱에서 진행 확인 → `claude --teleport`로 로컬 회수. 회의 전 위임, 자리 복귀 후 회수가 전형적 리듬.
- **Remote Control은 웹 왕복의 반대 방향** — **내 기계에서 도는 세션을 claude.ai와 모바일에서 조종**한다. 모드 2종: 대화형 세션에 `--rc`를 얹거나, `claude remote-control` 서버 모드로 로컬 화면 없이 대기. 폰에서 권한 프롬프트에 응답할 수 있고, 조직은 **`disableRemoteControl`로 통제**(Ch.3, 4 disable 패밀리).

### Part 4. 스케줄과 자동 실행

- **자동화 표면 5종 지도 — 선택이 곧 설계**: ①**세션 내**(`/loop`, cron 도구, `/goal` — 떠 있는 세션이 전제) ②**Desktop 예약**(Desktop 앱 scheduled tasks — 내 기계, GUI 관리) ③**Routines**(Anthropic 관리 클라우드 — 기계 불필요, 3종 트리거) ④**CI 파이프라인**(이벤트 구동 push·PR — Part 5) ⑤**cron + headless**(OS 스케줄러 + `-p` 스크립트 — 완전 자가 통제).
- **`/loop`와 cron 도구**: `/loop 테스트가 전부 통과할 때까지 실패를 고쳐`는 **조건 충족까지 같은 작업을 반복**. "10분마다 배포 파이프라인 상태를 확인하고 실패로 바뀌면 원인을 정리해줘"는 cron 스케줄링 도구로 반복·폴링 예약, "45분 뒤에 알려줘"는 1회성 리마인더 — **셋 다 같은 도구**다. **세션이 종료되면 함께 종료**되므로 상주가 필요하면 다른 표면으로 간다.
- **`/goal` — 완료 조건을 계약으로**. `/goal 전체 테스트 통과 + 린트 0 경고 상태 도달`을 걸면 **턴이 끝나도 조건 미충족이면 Claude가 계속 작업**하고, 충족 시 달성 보고 후 종료한다. **`/loop`가 "같은 작업의 반복 실행"이라면 `/goal`은 "상태 도달까지 수단을 바꿔가며 지속"**. 무인화 짝은 예산·턴 상한(Part 2).
- **Routines — 관리형 클라우드 자동 실행**. 내 기계 없이 Anthropic 관리 인프라에서 Claude Code를 돌린다. **트리거 3종**: 스케줄(매일 아침 의존성 감사 같은 정기 실행) / API 호출(외부 시스템이 HTTP로 발화) / GitHub 이벤트(이슈·PR에 반응). 기계와 cron 관리가 사라지고 **결과는 PR과 알림으로** 돌아온다.
- **Deep Links — URL 한 번의 클릭이 세션이 된다**: `claude-cli://open?path=~/work/payments&prompt=결제 지연 알람 원인을 조사해`. 클릭하면 해당 저장소에서 프롬프트가 채워진 세션이 열린다. **장애 런북의 알람 옆, 온보딩 문서의 첫 작업 링크**가 대표 용례. 조직 통제 키는 `disableDeepLinkRegistration`(Ch.4).
- **Slack에서 위임**: Slack에서 Claude를 멘션해 코딩 작업을 위임하면 **클라우드 세션에서 수행되고 결과는 스레드와 PR로** 돌아온다. 적합 범위는 이슈 조사·소규모 수정·상태 질의 같은 경량 작업. 채널과 권한은 조직 설정과 연동(Ch.3).
- **`--bg`와 `--exec`**: `claude --bg "flaky 테스트 원인 조사"`는 **세션 ID를 반환하고 터미널을 즉시 돌려준다**. 관측은 `claude logs <id>`, 회수는 `claude attach <id>`. `claude --bg --exec 'pytest -x'`는 셸 명령을 PTY 잡으로 띄운다. **`-p`와는 병용 불가**(v2.1.198 규칙).
- **cron + headless 레시피**: crontab에 `SHELL`과 **`PATH`를 명시**하는 것이 1번 함정(cron 환경은 빈약하다). `0 7 * * 1-5 cd /home/dev/payments && ./scripts/nightly-deps-audit.sh >> ~/logs/deps.log 2>&1` 형태로 리다이렉트가 관측 최소선. **인증 지속성**(Bedrock SSO 만료 대비 헬퍼·역할, Ch.3)이 cron의 진짜 난제이고, 기계를 벗어나고 싶으면 Routines가 대안.

### Part 5. CI/CD 통합

- **비대화 3원칙 = CI 속 Claude의 헌법**: ①**묻지 않는다**(`-p` + `--allowed-tools` 명시로 확인 프롬프트를 원천 제거 — allowlist가 곧 능력, Ch.4 dontAsk 철학) ②**넘치지 않는다**(`--max-turns`, `--max-budget-usd`, 모델 하향 기본) ③**흔적을 남긴다**(`--output-format json` 저장과 아티팩트 업로드 — 감사·디버깅 재료). 환경 전제는 러너에 설치 + 인증, 그리고 체크아웃 깊이.
- **CI 인증 전략**: 구독 조직은 **`claude setup-token`으로 장기 토큰** 발급 후 시크릿 저장소 보관 / API 조직은 `ANTHROPIC_API_KEY` 시크릿 / **AWS 표준 권장은 OIDC로 역할 인수 + Bedrock(장기 시크릿 0, Ch.3)** / 게이트웨이 조직은 `BASE_URL` + 서비스 자격. 공통 원칙은 잡 권한 최소화, 키 마스킹 확인, **포크 PR 실행 주의**.
- **GitHub Actions 기본 골격**: `permissions: { id-token: write, contents: read }` → `actions/checkout@v4` → `aws-actions/configure-aws-credentials@v4`로 역할 인수 → `curl -fsSL https://claude.ai/install.sh | bash`(설치 1줄) → `CLAUDE_CODE_USE_BEDROCK=1 ./scripts/nightly-deps-audit.sh`. **Part 2에서 만든 스크립트를 그대로 재사용**하고, 그 exit 게이트가 잡 성패에 직결된다.
- **PR 리뷰 잡**: `on: pull_request: types: [opened, synchronize]` + `fetch-depth: 0` → `git diff origin/${{ github.base_ref }}...HEAD > pr.diff` → `claude -p "pr.diff를 리뷰해" --allowed-tools "Read" "Grep" "Bash(git diff *)" --max-turns 12 --max-budget-usd 1.50 --json-schema "$(cat .ci/review-schema.json)" > review.json` → **`jq -e '.block == false' review.json`이 게이트 한 줄**. 관리형 대안으로 공식 `claude-code-action`과 `/code-review`도 있다.
- **GitLab CI는 같은 원칙 다른 문법**: `rules: - if: $CI_PIPELINE_SOURCE == "merge_request_event"`로 MR 트리거, `before_script`에서 install.sh + `export PATH="$HOME/.local/bin:$PATH"`, `script`는 **동일한 `-p` + 상한 + 스키마 스크립트**, `artifacts: { paths: [review.json], when: always }`로 흔적 보존.
- **기타 CI**: Jenkins는 sh 스텝에서 동일 스크립트 + Credentials 바인딩 / CircleCI·Buildkite는 컨테이너 + install.sh 패턴 동일(잡 캐시로 설치 가속) / **Ch.3의 조직 표준 이미지에 사전 설치하면 설치 단계 자체가 사라진다** / 자가 러너는 인증 헬퍼와 프록시를 Ch.3 그대로. **어디든 3원칙 + exit 게이트면 동작하고 플랫폼은 부차 변수**.
- **시크릿 위생 4규칙**: ①**OIDC 우선**(역할 인수로 장기 시크릿 자체를 제거) ②불가피한 토큰은 Secrets + **로그 마스킹 확인** ③**노출면 축소**(프롬프트와 결과 JSON에 시크릿 미포함 설계) ④**포크 PR에는 시크릿 미주입** 정책 유지.
- **CI 비용 다이얼 6개**: 호출 상한(`--max-budget-usd`·`--max-turns` 필수) / 모델 하향(리뷰·분류는 sonnet·haiku 기본, opus는 선별 잡만) / **발동 조건 축소가 최대 절감**(paths 필터, 라벨 조건으로 잡 자체를 줄임 — docs 변경은 스킵) / 캐시(`--exclude-dynamic...` + 설치 캐시) / 관측(`review.json`의 cost 집계 대시보드 = 월간 파이프라인 회계) / 조직 안전망(게이트웨이 한도, Budgets 알람 — Ch.3).
- **`--init` 준비 훅으로 CI 셋업을 형상화**: `.claude/settings.json`의 `Setup` 훅에 `matcher: "init"`으로 `npm ci && cp .env.ci .env`를 걸어두고 CI 스텝은 `claude -p --init "..."`. **준비 절차가 레포에 동봉**되고, 검증만 분리하려면 `claude --init-only`. 주기 정비용 `--maintenance` 매처도 있다.
- **실패 처리와 아티팩트**: 결과 JSON과 stderr 로그를 **`when: always`로 성패 무관 보존** / 실패를 **게이트 실패 vs 실행 오류 vs 상한 도달 3분류**로 구분 출력 / 요약을 PR 코멘트·잡 서머리에 게시(개발자 접점) / **일시 오류는 Part 2의 재시도 함수가 이미 흡수**.
- **완성 예시(`pr-review.sh`)**: diff 수집 → `run_claude` + 스키마 → `review.json` 저장 → `jq -r '"### Claude Review\n" + .summary' > comment.md` → `gh pr comment "$PR" --body-file comment.md` → **`jq -e '.block == false'` 최종 게이트**. Part 2와 Part 5 부품의 총결합.

### Part 6. 자동화 패턴

- **Pattern 1 / 이슈 트리아지**: `issues.opened` 트리거로 본문을 분류해 라벨·담당 팀·우선순위를 **제안**한다. 입력은 `gh issue view`, 판정 스키마는 `category`(enum)·`priority`·`team`·`duplicate_of`, 액션은 `gh issue edit --add-label`과 코멘트. **안전선은 "닫기 금지, 제안까지만" — 확정은 사람**. 비용 기본값은 haiku, 상한은 6턴. 스크립트는 `claude --bare -p --model haiku --json-schema ...` 뒤에 **gh 3연타(view·edit·comment)**.
- **Pattern 2 / 로그 분석 — 수십만 줄을 통째로 넣지 않는다**. **셸이 압축과 분할을, Claude가 해석과 상관을 맡는 분업**이 비용과 품질을 동시에 지킨다. 4단계: 전처리(`grep -E 'ERROR|FATAL'` → `sed`로 UUID를 `<id>`로 정규화 → `sort | uniq -c | sort -rn | head -20`로 **시그니처 상위 추출**) → 분할(시그니처·시간창 단위 개별 호출) → 판정(incident 여부·근본 원인 가설·심각도 스키마) → 종합(`jq -s`로 집계 후 최종 1회 호출로 인시던트 보고 초안). **개별은 haiku 4턴, 종합은 sonnet 8턴**의 2단 구성.
- **Pattern 3 / 일일 보고서**: 수집은 `git log --since` + `gh pr list --search "updated:>=$SINCE"` + `gh issue list`의 **24시간 절단**을 `digest.txt`로 결합, 서술은 `claude -p "팀 브리핑: 요약, 리스크, 오늘 볼 것 3" --max-turns 8 --max-budget-usd 0.50`, 배달은 `curl -X POST "$SLACK_WEBHOOK" -d "$(jq -n --rawfile t report.md '{text:$t}')"` 한 줄. **탑재 표면은 cron·Routines·Desktop 예약 어디든**(Part 4).
- **Pattern 4 / 문서 파이프**: `git diff --name-only HEAD~1`로 **변경된 모듈만** 문서를 재생성하고(`--allowed-tools "Read" "Grep" "Write(./docs/**)"`로 **쓰기 경로를 docs/ 로 제한**), 갱신된 문서만 골라 `claude --bare -p "기술 용어를 보존해 영어로 번역"`으로 한영 병행 배치. diff 기반이라 비용이 변경분에 비례한다. 탑재 위치는 CI post-merge.
- **Pattern 5 / 배치 마이그레이션 — 파일 1개 = 변환 + 테스트 + 커밋 한 묶음으로 원자화**한다. 관련 테스트를 즉시 실행해 통과하면 `git commit` + `done.txt` 기록, 실패하면 **`git checkout -- .`으로 원복하고 `failed.txt`에 남긴 뒤 다음 파일로** — 배치가 멈추지 않고 사람이 나중에 개입한다. 재개는 `grep -qx "$f" done.txt && continue`로 **완료 목록 대조**. 본 브랜치 보호는 `-w` 격리.
- **다섯 패턴의 공통 골격**: `run_claude` + 스키마 + exit 게이트(Part 2 부품 재사용) / **이벤트형은 CI, 정기형은 cron·Routines**(Part 4, 5 지도) / 결과 JSON의 cost를 월간 집계해 **패턴별 단가 대장** / `failed` 목록과 알림으로 **무인과 유인의 경계** 설계 / 신뢰는 **제안만 → 게이트 → 액션 순으로 축적**하고 권한도 단계 상향.
- 파트 전체를 한 문장으로: **전처리는 셸, 판정은 스키마, 액션은 CLI 도구, 실패는 큐로**. 다섯 패턴 모두 이 문장의 변주다.

### Part 7. 환경변수

- **분류 지도 7칸**: 인증·공급자(`USE_BEDROCK`, `ANTHROPIC_API_KEY`, `BASE_URL` — Ch.3) / 네트워크(`HTTPS_PROXY`, `NODE_EXTRA_CA_CERTS` — Ch.3) / 모델·사고(`ANTHROPIC_MODEL`, `MAX_THINKING_TOKENS`) / 기능 스위치(`DISABLE_*` 패밀리, `SIMPLE`, `SAFE_MODE`) / 관측(`ENABLE_TELEMETRY`, `OTEL_*` — Ch.3) / 디렉토리·기록(`PROJECT_DIR`, `SKIP_PROMPT_HISTORY`).
- **인증과 공급자**: `CLAUDE_CODE_USE_BEDROCK=1`(최상위 강제 스위치, `USE_VERTEX`·`USE_FOUNDRY` 동렬) + `AWS_REGION` / `ANTHROPIC_API_KEY`(직결 경로) / `ANTHROPIC_BASE_URL`(게이트웨이 창구) / `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`(헬퍼 갱신 주기, Ch.3 볼트 통합). **서열은 플래그 > env > 설정**이고 모델 계열도 공통.
- **네트워크**: `HTTPS_PROXY`·`HTTP_PROXY`(프록시 경유 강제) / `NO_PROXY`(사내 예외 — MCP·게이트웨이 도메인) / `NODE_EXTRA_CA_CERTS`(TLS 검사 장비 뒤의 사내 루트 CA). **`NODE_TLS_REJECT_UNAUTHORIZED=0`은 금지** — 검증 무력화. 진단은 `curl -v`로 프록시 층과 TLS 층을 분리.
- **모델과 사고**: `ANTHROPIC_MODEL=apac.anthropic.claude-sonnet-5-v1:0`으로 기본 모델 고정(배치 표준화, 플래그가 있으면 플래그 우선) / `MAX_THINKING_TOKENS=0`으로 확장 사고 차단(Anthropic API 기준, **Fable 5는 사고를 끌 수 없는 예외**, 서드파티는 thinking 파라미터 생략 방식). 짝이 되는 설정 키는 `alwaysThinkingEnabled`·`availableModels`·`fallbackModel`·`effortLevel`(Ch.4).
- **기능 스위치**: `DISABLE_AUTOUPDATER`(CI 이미지에서 갱신 차단, Ch.3) / `DISABLE_AUTO_COMPACT` / `CLAUDE_CODE_DISABLE_*` 군(AUTO_MEMORY, BUNDLED_SKILLS 등 — 설정 disable 키와 동등) / **`CLAUDE_CODE_SIMPLE`은 `--bare`가 설정하는 변수**, **`CLAUDE_CODE_SAFE_MODE`는 `--safe-mode`가 설정** — 둘 다 스크립트와 훅이 모드를 인지하는 표식 / `CLAUDE_AX_SCREEN_READER`(접근성 렌더 강제).
- **관측 6변수 세트**(Ch.3 계측 블록): `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`. 파이프라인 관점 포인트 둘 — **훅 입력의 `prompt_id`가 OTel 이벤트와 상관 연결**되고, **CI 잡 태그를 `RESOURCE_ATTRIBUTES`에 추가하면 파이프라인별 비용 절단**이 가능하다.
- **디렉토리와 기록**: `CLAUDE_PROJECT_DIR`(훅·MCP에 노출되는 프로젝트 루트 — Ch.4 플레이스홀더) / `CLAUDE_PLUGIN_ROOT`·`CLAUDE_PLUGIN_DATA` / `CLAUDE_CODE_SKIP_PROMPT_HISTORY`(트랜스크립트 기록 끄기, 전 모드) / `CLAUDE_CODE_REMOTE`(웹 환경에서 true — 환경 분기) / `CLAUDE_CODE_BRIDGE_SESSION_ID`(Remote Control 연결 중 세션 ID) / `CLAUDE_CODE_DEBUG_LOGS_DIR`.
- **점검 원라이너**: `env | grep -iE 'claude|anthropic|otel|aws_region' | sort`가 1차 일람, **유효 구성의 최종 확인은 언제나 `/status`**(공급자·모델·managed 소스·샌드박스). 충돌하면 서열을 복기 — **managed 설정 > CLI 플래그 > env > 파일 설정**이고, **env는 managed의 `env` 블록으로 배포되면 강제층으로 승격**된다.
- **보안 원칙 3문장**: 키 실값을 rc 파일에 저장 금지(**helper와 OIDC가 비밀의 제자리**, Ch.3) / **settings `env` 블록에도 비밀 금지**(Ch.4) / 로그와 CI 출력의 마스킹 확인.

### Part 8. 디버깅

- **진단 흐름 6단 — 각 단이 용의자 절반을 지운다**: ①②재현 + `--verbose`(최소 재현 명령 고정, 턴 출력 확대) → ③`--debug` 카테고리(api·hooks·mcp 선별 로그) → ④`doctor`(환경·설정 유효성 판정) → ⑤`--safe-mode`(커스텀 원인 여부 이분) → ⑥`--bare` / 새 디렉토리 재현.
- **`--debug` 카테고리**: `claude --debug "api,hooks"`로 필요한 카테고리만, **`!` 접두로 제외**(`--debug "!statsig,!file"`), `--debug-file /tmp/claude-debug.log`는 파일 기록(debug 자동 활성, **`CLAUDE_CODE_DEBUG_LOGS_DIR`보다 우선**). **CI 요령은 실패 시에만 debug-file을 아티팩트로 올리는 것**.
- **`--safe-mode` 이분법**: safe-mode에서 **증상이 사라지면 범인은 커스텀 계층** → `disableAllHooks`로 훅을 먼저 가르고, `--strict-mcp-config`로 MCP를 가르고, 절반씩 복원해 좁힌다. **safe-mode에서도 재현되면 범인은 코어·환경·네트워크** → doctor와 Ch.3 네트워크 진단표, 최근 커밋 이력 우선 의심, 버전 회귀 의심 시 `install stable`, errors 레퍼런스와 이슈 검색.
- **판정형 진단 명령**: `claude doctor`는 설치·설정 유효성과 **무효 항목의 출처까지 표시**하고 `f` 키로 자동 수정 제안을 적용한다. `claude auth status --text`는 로그인 상태·계정·조직, **`claude auth status; echo $?`는 0/1 판정**. CI 첫 스텝 관용구는 `claude auth status || { echo 'auth missing'; exit 1; }`.
- **헤드리스 전용 함정 6가지**: 조용히 오래 걸림 → **확인 대기 상태**(allow 누락) → allowed-tools 보강 또는 dontAsk / 결과가 잘림 → stream 미수집·파이프 버퍼 → **json 봉투로 수신** / turns 초과 빈발 → 상한 과소 → p95 재실측·작업 분할 / **로컬은 되는데 CI만 실패 → 설정 소스·env 차이 → `--setting-sources` 고정** / 세션 못 찾음 → 디렉토리 다름(**ID는 프로젝트 한정**) → 같은 경로·이름 재개 / 훅 미발화 → **bare 기동이 원인**.
- **도움 받기 경로**: `docs/errors`의 오류 사전에서 메시지 검색 → **재현 번들 4종**(버전, 명령, debug-file, doctor 출력) → github.com/anthropics/claude-code 이슈 **검색 먼저 후 등록** → 세션 안에서는 `/bug`.

### Part 9. Recap

- **여섯 문장 요약**: ①플래그 — 지도 2 + 서랍 6, 무인은 상한, 진단은 safe-mode ②Headless — SDK 경유 단발 에이전트, 계약은 stdout·stderr·exit ③구조화 — `--json-schema`로 파싱에서 계약으로 ④세션 — `-c`, `-r`, `--from-pr`, fork, 웹 왕복 ⑤표면 5 — loop/goal, Desktop, Routines, CI, cron ⑥운영 — 3원칙, 비용 다이얼, 실패 큐, 진단 6단.
- **FAQ 6선**: `-p`에서도 훅이 도나 → **예, 전 기능 동일하고 `--bare`만 예외** / 왜 `--max-turns`가 필수인가 → **기본 무제한, 무인 폭주 방지** / JSON을 프롬프트로 요청하면 안 되나 → `--json-schema`가 검증을 보증 / resume이 세션을 못 찾는다 → **ID 검색은 프로젝트 한정** / CI 인증은 뭘로 → **OIDC 우선, 다음이 setup-token** / `--bare`와 `--safe-mode` 차이 → **가속 vs 진단, managed 적용 여부**.
- **자가 점검 4문항**: ①6서랍에서 조합 관용구를 작성할 수 있다(Part 1) ②스키마와 게이트로 스크립트를 구성할 수 있다(Part 2) ③`--from-pr`·fork·teleport를 쓸 수 있다(Part 3) ④표면 선택과 상한, 3원칙을 적용할 수 있다(Part 4~6).
- **실무 적용 로드맵**: Week 1–2 개인 파이프(일일 보고서, 트리아지 1종 가동) → Week 3–6 팀 CI(PR 리뷰 잡, **비용 대장 시작**) → Week 7–12 확장(배치 마이그레이션, Routines 이주). **각 단계 관문은 상한 준수, 실패 큐 소화율, 월 비용 리뷰**.
- **다음 챕터 예고**: Chapter 6 — Agent SDK. `-p`가 경유하던 그 엔진을 라이브러리로 직접 쓴다(TypeScript·Python 쿼리 루프와 스트리밍, 인프로세스 MCP로 내 함수를 도구화, Zod·Pydantic 계약과 세션 영속, 컨테이너·격리·관측의 프로덕션 배포).
