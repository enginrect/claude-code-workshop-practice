# [1주차] Claude Code Deep Dive Workshop — Chapter 1 스터디 노트

> - 자료: [whchoi98/claude-code-workshop](https://github.com/whchoi98/claude-code-workshop) — Chapter 1 (192 slides, 9 Parts + Labs 5개)

---

## 개요 — 왜 Claude Code인가

- Claude Code는 IDE 자동완성 도구가 아니라 **터미널에서 동작하는 Agentic Coding 에이전트**다. 코드를 읽고, 파일을 편집하고, 명령을 실행하며 결과를 스스로 검증한다.
- 핵심 동작 원리는 **Agentic Loop**: Gather Context → Take Action → Verify Results 반복.
- CLI 기반이라 Unix 철학 그대로 파이프/스크립트/CI에 조합할 수 있다. 같은 엔진이 IDE 확장, Desktop, Web, Slack까지 확장된다.

## Part별 핵심 정리

### Part 1. What is Claude Code

- 정의: 코드를 읽고 파일을 편집하고 명령을 실행하는 Anthropic의 Agentic Coding 도구. 자연어로 태스크를 위임하는 에이전트형 도구다.
- **Agentic Loop 3단계**
    1. Gather Context — 파일/로그 읽기, CLAUDE.md와 메모리 로드
    2. Take Action — 파일 편집, 명령 실행, 커밋 등 실제 실행
    3. Verify Results — 테스트 실행, 타입 검사, 결과 관찰 후 다음 단계 판단 (통과까지 반복, 중간에 사용자가 개입해 방향 조정 가능)
- **Code Completion과의 차이**: 자동완성은 코드 "제안"까지, Agentic Coding은 작업 단위를 위임받아 검증까지 "완수"한다.
- **4대 핵심 가치**: ①깊은 코드 이해(저장소 전체 자율 탐색) ②실제 작업 완수(수정→테스트→PR까지) ③안전한 권한 모델(allow/ask/deny + 체크포인트) ④확장 가능한 통합(Skills, Subagents, MCP, Hooks, SDK).
- 모델 패밀리(2026.07): **Fable 5**(최상위 Mythos-class, 장시간 자율 세션), **Opus 4.8**(깊은 추론·설계), **Sonnet 5**(기본 모델, 네이티브 1M 컨텍스트), **Haiku 4.5**(빠르고 저렴).
- 모델 alias: `default`(계정 권장으로 복귀), `best`(Fable 5 접근 가능하면 Fable 5, 아니면 최신 Opus), `fable`/`opus`, `sonnet`/`haiku`, `opus[1m]`(1M 컨텍스트 Opus), `opusplan`(Plan은 opus·실행은 sonnet 자동 전환). `/model`로 세션 중 전환.
- 대표 유스케이스 6가지: ①버그 원인 추적·수정 ②대규모 리팩토링(콜백→async/await 일괄 변환) ③Plan 모드로 기능 설계→구현 ④낯선 코드베이스 온보딩("이 프로젝트 구조 설명해줘") ⑤테스트 작성·커버리지 보강 ⑥DevOps/IaC(Dockerfile 최적화, terraform plan 리뷰, 로그 파이프 분석)
- 요금제: Pro/Max는 OAuth 개인, Team/Enterprise는 SSO·중앙 청구. 데이터는 신뢰 경계(Trust boundary) 안에서 로컬 실행되고 모델에 필요한 것만 API로 전송.

### Part 2. Architecture — Agentic Harness

- Claude Code = 모델 + **하네스(Harness)**: 도구, 컨텍스트 관리, 권한, 실행 환경을 제공하는 계층. 모델이 "무엇을"이라면 하네스는 "어떻게".
- Client Layer: Terminal CLI(기준점), VS Code/Cursor, JetBrains, Desktop, Web/iOS, Slack, CI — 전부 같은 코어를 공유.
- **Agent SDK**: Claude Code의 하네스를 Python/TypeScript 라이브러리로 노출. CLI와 동일한 도구·권한 체계를 내 앱에 내장 가능.
- Provider routing: Anthropic API 외에 `CLAUDE_CODE_USE_BEDROCK` / `VERTEX` / `FOUNDRY` 환경변수로 공급자 전환.
- 도구 시스템: Read/Edit/Write(파일), Bash(명령), Grep/Glob(검색), LSP(코드 인텔리전스), MCP(외부 데이터소스 연결).
- 컨텍스트 관리: 한계에 도달하면 **Auto compaction**으로 대화를 요약·압축. `/context`로 사용량 시각화.
- **Checkpointing**: 모든 파일 편집 전 스냅샷 기록 → `Esc Esc` 또는 `/rewind`로 코드/대화를 시점 복구(`/clear` 이전으로도 복귀 가능). 단 **파일 변경 한정** — DB, API 호출, 배포 같은 외부 부작용은 되돌리지 못하므로 실행 전 확인이 필요.
- 권한 2중 구조: 정적 규칙(settings.json allow/deny) + 세션 대화형 승인. 로그는 JSONL, OpenTelemetry 연동 가능.

### Part 3. Installation

- 설치 방법: **Native(권장)** curl 스크립트, Homebrew(stable/latest 캐스크), WinGet, Linux apt/dnf, Docker, Devcontainer. npm 글로벌 설치는 구식.
- 검증: `claude --version`(버전), `claude doctor`(설치 상태·설정·업데이트 종합 자가진단).
- 자동 업데이트: settings.json의 `autoUpdatesChannel`(stable/latest), 조직은 managed settings로 `minimumVersion` 강제 가능.
- 트러블슈팅: `command not found`는 대부분 PATH 문제(`~/.local/bin` 누락) → 셸 재시작.

### Part 4. Authentication

- 3가지 방법: ①**OAuth**(claude 실행 후 브라우저 로그인 — Pro/Max 개인에게 가장 간단) ②Console API Key(조직 과금, `ANTHROPIC_API_KEY`) ③클라우드 공급자(Bedrock/Vertex — 환경변수 + IAM 정책).
- `/usage`: 플랜 한도 대비 사용량, 리셋 시점, 스킬/서브에이전트/MCP별 소비 분해.
- 자격증명 저장: macOS는 Keychain. CI/스크립트 환경은 `claude setup-token`으로 1년 유효 OAuth 토큰 발급(터미널에 출력만, 저장 안 함).

### Part 5. Quick Start

- 첫 세션: `cd my-project && claude` → "이 프로젝트가 뭘 하는지 설명해줘". Claude가 Glob/Grep/Read로 탐색하며 답변.
- 프롬프트 원칙: **"Delegate, don't dictate"** — 유능한 동료에게 위임하듯. 공식 4원칙: ①구체적으로(경로·제약 명시) ②검증 기준 제공(테스트 케이스, 기대 출력) ③탐색과 분리(큰 문제는 Plan 모드로 조사 먼저) ④대화로 교정(완벽한 첫 프롬프트보다 중간 개입이 빠름).
- 상태 확인 3종 세트: `/help`(명령 카탈로그), `/status`(인증·공급자·모델·세션), `/doctor`(종합 진단, `f` 키로 자동 수정). 그 외 `/init`, `/context`, `/usage`, `/resume`, `/rewind`.
- **`/clear` vs `/compact` vs `/btw`** (컨텍스트 트리아지): 새 작업 → `/clear`(초기화, 별칭 `/new`, 이전 대화는 `/resume`에 보존) / 같은 작업 + 공간 확보 → `/compact`(요약 압축) / 흐름 깨기 싫은 곁가지 질문 → `/btw`(대화 이력에 남지 않는 사이드 질문, 컨텍스트 오염 방지).
- **권한 모드 4종 (Shift+Tab 순환)**
  | 모드 | 동작 |
  |---|---|
  | Default | 파일 편집·명령 실행 전 매번 승인 |
  | Accept Edits | 편집은 자동 승인, 그 외 질문 |
  | Plan | 파일 수정 없이 탐색·계획만(Shift+Tab 두 번 또는 `/plan`). 승인 후 실행 |
  | Auto | 백그라운드 안전 분류기가 전 행동 평가, 위험하면 확인 (Research Preview) |
    - CI 등 비대화 환경은 `--permission-mode` 플래그 + 사전 합의된 allow 규칙으로 운영.
- 세션 분기: `/branch`(대화 사본을 만들어 **내가** 갈아타서 다른 방향 시도, 원본은 `/resume`으로 복귀) vs `/fork`(사본을 **백그라운드 서브에이전트에게** 위임, 나는 원본에서 계속하고 완료되면 결과 회수).

### Part 6. Core Capabilities (도구들)

- **Read/Edit/Write**: Edit는 승인 화면에서 변경 전후 diff를 보여준다 — diff를 직접 읽는 습관이 중요.
- **Bash**: 명령별 승인, 자주 쓰는 명령은 allow 규칙으로 통과. 백그라운드 실행 + 로그 감시 위임 가능.
- **Grep/Glob**: ripgrep 기반 내용 검색 + 파일 패턴. "processPayment 호출하는 곳 전부 찾아줘".
- **LSP**: 편집 직후 타입 오류 확인 — 언어 서버 수준의 정적 이해.
- **WebSearch/WebFetch**: 최신 에러 메시지 원인 검색 등 외부 지식 보강.
- **Monitor / Agent(서브에이전트) / AskUserQuestion**: 장기 프로세스 감시, 독립 컨텍스트 위임, 사용자에게 선택지 질문.
- **Git 통합**: 변경 요약, 논리 단위 스테이징 제안, 컨벤션 맞춘 커밋 메시지, `gh` CLI로 이슈/PR까지.
- 안전장치: settings.json 권한 규칙(allow/deny 문법) + Sandboxed Bash(파일시스템·네트워크 격리 실행).

### Part 7. Interfaces

- VS Code/Cursor 확장(Cmd+Shift+X로 설치, 인라인 diff·@멘션), JetBrains 플러그인, Desktop 앱(병렬 세션 운영), Web(claude.ai/code — GitHub 연결 클라우드 세션), Slack(@Claude 멘션), Chrome(베타).
- `--teleport`: 웹/모바일에서 시작한 세션을 로컬 터미널로 가져오기.
- 필수 단축키: `Esc`(즉시 중단) / `Esc Esc`(체크포인트 되감기) — 가장 자주 쓰는 두 동작, `Shift+Tab`(권한 모드 순환, Plan은 두 번), `Ctrl+R`(전 프로젝트 명령 이력 검색 — 과거 프롬프트 재사용), `!` 접두(셸 모드 직접 실행), `Shift+Enter`(줄바꿈, `/terminal-config`로 설정).

### Part 8. CLAUDE.md & Memory

- 두 축: 내가 쓰는 **CLAUDE.md**(프로젝트 지시·규칙) + Claude가 쌓는 **Auto Memory**(빌드 명령·패턴·교정 사항을 저장소 단위 MEMORY.md에 자동 기록, 매 세션 첫 200줄/25KB까지 자동 로드).
- **로드 순서(상위 → 하위 병합)**: `/etc/claude-code/CLAUDE.md`(조직) → `~/.claude/CLAUDE.md`(사용자) → 상위 디렉토리 → 작업 디렉토리 → `CLAUDE.local.md`(개인, 마지막). 하위 디렉토리 CLAUDE.md는 해당 폴더 작업 시 온디맨드 로드.
- 관리 명령: `/memory`(로드된 메모리 목록·편집), 문장 앞 `#`(빠른 규칙 추가 — 반복해서 말하게 되는 지시는 그때 바로 등록), `@path`(다른 파일 import).
- 구조화: `.claude/rules/*.md`로 규칙 분리, frontmatter `paths:`로 특정 경로에서만 로드되는 path-scoped rule. AGENTS.md는 `@AGENTS.md` import 또는 심볼릭 링크로 호환(`/init`은 AGENTS.md, .cursorrules, .windsurfrules를 읽어 반영).
- 안티패턴 5가지: ①장문 서사 ②프로젝트와 무관한 일반 상식 ③비밀값·자격증명 기록(볼트에 두고 경로만) ④매 세션 불필요한 절차 문서 전체(다단계 절차는 skill로 분리) ⑤모순 규칙 방치 → 명령·규칙 중심의 간결한 목록, 이 저장소 고유의 사실만.

### Part 9. Workflow Patterns

- **EPC (Explore → Plan → Code)**: 가장 기본 리듬. ①탐색("아직 코드 만들지 마") ②Plan 모드로 계획·승인 ③승인된 계획대로 구현+검증. `opusplan` alias로 계획은 Opus, 실행은 Sonnet 분배.
- **TDD Loop**: 실패하는 테스트 먼저(만료 쿠폰 거부 등 케이스 명세) → 실패 확인 → 최소 구현 → 전부 통과 → 리팩토링. 기대 출력을 명세로 주는 게 핵심(mock이 아닌 실제 실행 결과로 검증).
- **/code-review**: 현재 diff 리뷰.
- **멀티에이전트**: `/fork`(곁가지 위임), `/batch`(대규모 분할 — 코드베이스를 5~30개 독립 단위로 분해, 단위별 서브에이전트가 격리된 git worktree에서 구현·테스트 후 각각 PR 오픈), `/tasks`(백그라운드 작업 현황).
- **Headless**: `claude -p "프롬프트"` — 파이프 입출력(`tail -200 app.log | claude -p "이상 징후 보고"`), `--allowed-tools "Read,Grep,Glob"`로 도구 제한, `--output-format json | jq '.result'`로 후처리, `--model haiku`로 비용 절감.
- **CI 통합**: GitHub Actions에서 PR 자동 리뷰. **Routines**: `/schedule`로 정기 실행(평일 09:00 PR 요약 → Slack) — 내 머신이 꺼져도 Anthropic 관리 인프라에서 실행, GitHub 이벤트 트리거 가능. 세션 내 반복 폴링은 `/loop`(머신 필요). **/goal**: 조건 충족까지 턴을 넘겨도 자율 반복("전체 테스트 통과 + lint 경고 0"), 각 턴 종료 시 스스로 점검.
- 컨텍스트 관리가 응답 품질에 직접 영향: 작업 단위로 `/clear`, 필요할 때만 `/compact`.