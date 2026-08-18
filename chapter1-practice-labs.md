## 실습 기록 (Labs)
실습은 https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch1_HandsOnLab.html#t0 으로 진행

> 환경: macOS, claude 2.1.220

### 설치 확인

```shell
# 이미 설치 완료된 환경에서 진행
 claude --version   
2.1.220 (Claude Code)

 claude doctor   
Claude Code doctor

Running: native (2.1.220)
Commit: 4073f59596e2
Platform: darwin-arm64
Path: /Users/enginrect/.local/share/claude/versions/2.1.220
Config install method: native
Search: OK (bundled)
Auto-updates: disabled (set by env: DISABLE_AUTOUPDATER)
Auto-update channel: latest
Last update attempt: none recorded

Remote Control
Control this session from claude.ai/code or the Claude mobile app

No installation issues found.

For a full setup checkup that can also fix issues, run /doctor in a Claude Code session.
```

- key point: doctor는 설치 상태뿐 아니라 설정 파일, 업데이트 채널까지 종합 진단한다. 세션 안에서는 `/doctor`로 문제 자동 수정까지 가능.

### 실습 프로젝트 준비 + 첫 세션
```shell
# 실습 랩 파일 생성 완료 후 cd ~/claude-lab/ch1
 claude
▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
 Accessing workspace:

 /Users/enginrect/claude-lab/ch1

 Quick safety check: Is this a project you created or one you trust? (Like your own code, a well-known open source project, or work from your team). If not, take a moment to review what's in this folder first.

 Claude Code'll be able to read, edit, and execute files here.

 Security guide

 ❯ 1. Yes, I trust this folder
   2. No, exit

 Enter to confirm · Esc to cancel
#=======================================================================================================================
❯ 이 프로젝트의 구조와 각 파일의 역할을 간단히 설명해 주세요

  Searched for 1 pattern, read 5 files

프로젝트 구조

이 프로젝트는 사용자 정보 조회 및 요금제 확인 기능을 제공하는 간단한 Node.js 모듈입니다:

📁 파일별 역할

package.json
- 프로젝트 메타데이터 (이름, 버전, 설명)
- "type": "module" — ES Modules 사용 지정
- npm test 커맨드 정의

src/users.js
- 사용자 데이터 저장소 (메모리 기반)
- 3명의 사용자 정보 포함
- ⚠️ 주의: ID 3번 사용자는 profile 필드가 없음 (마이그레이션 미완료 상황 시뮬레이션)

src/userService.js
- 사용자 관련 비즈니스 로직
- getUser(id) — ID로 사용자 조회
- getUserPlan(id) — 사용자의 요금제 조회

test.js
- 단위 테스트 파일
- 3가지 테스트 케이스:
  a. Pro 플랜 사용자 조회
  b. Free 플랜 사용자 조회
  c. Profile 없는 사용자 조회 (현재 실패 상황)

🔴 현재 문제

getUserPlan(3) 호출 시 Cannot read property 'plan' of undefined 에러 발생 — profile이 없는 사용자를 처리하지 못함.

테스트는 npm test 커맨드로 실행할 수 있습니다.
```

- key point: 시작 전에 작업 폴더 신뢰 여부를 묻는다(Trust boundary). 구조 설명만 요청했는데 탐색 단계(Read 5 files)에서 의도된 버그(ID 3번 사용자에 profile 없음)까지 같이 찾아냈다.

### CLAUDE.md 작성

```shell
# /init으로 자동 생성
❯ /init

/init

I'll analyze this codebase to create a comprehensive CLAUDE.md file. Let me start by exploring the structure and understanding what we're working with.

  Searched for 2 patterns, read 5 files, listed 1 directory, ran 1 shell command

Now I'll create the CLAUDE.md file based on my analysis:

⏺ Write(CLAUDE.md)
Wrote 32 lines to CLAUDE.md
#===========================
# 기존 CLAUDE.md 수정...
#===========================

# 생성 결과 (출력 축약)
#   Commands     : npm test
#   Architecture : users.js(의도적으로 스키마가 어긋난 데이터) / userService.js(조회 함수) / test.js
#   Key Notes    : profile 없는 사용자 엣지 케이스, ES module 사용
#=======================================================================================================================

# 동작 검증
❯ 이 프로젝트에서 테스트는 어떻게 실행하나요? 그리고 이 프로젝트의 코딩 규칙을 요약해 주세요

프로젝트 정보는 CLAUDE.md에 잘 정리되어 있습니다.

테스트 실행

npm test

코딩 규칙 요약

필수 사항:
- ES Modules만 사용 — import/export 사용, CommonJS(require) 금지
- 존재하지 않는 데이터 처리 — 예외를 던지지 않고 기본값을 반환
- 모든 수정은 테스트와 함께 — test.js 통과가 필수

금지 사항:
- src/users.js의 데이터 구조를 임의로 변경하지 않기
- 새로운 npm 패키지 추가하지 않기 (외부 의존성 없음)

이 프로젝트는 Node.js 18+ ES Modules 환경에서 사용자 서비스 모듈(사용자 조회, 요금제 확인)을 제공하는 실습용 프로젝트입니다.
```
### 디버깅 시나리오와 Plan 모드

```shell
# 버그 재현
 npm test

> user-service-lab@1.0.0 test
> node test.js

Test 1 - pro 플랜 사용자: PASS
Test 2 - free 플랜 사용자: PASS
file:///Users/enginrect/claude-lab/ch1/src/userService.js:9
  return user.profile.plan;
      ^

TypeError: Cannot read properties of undefined (reading 'plan')
    at getUserPlan (file:///Users/enginrect/claude-lab/ch1/src/userService.js:9:23)
    at file:///Users/enginrect/claude-lab/ch1/test.js:10:40
    at check (file:///Users/enginrect/claude-lab/ch1/test.js:4:18)
    at file:///Users/enginrect/claude-lab/ch1/test.js:10:1
    at ModuleJob.run (node:internal/modules/esm/module_job:569:25)
    at async node:internal/modules/esm/loader:650:26
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:101:5)

Node.js v26.6.0

#=======================================================================================================================
# 에러 전달 (참고 shift + tab 으로 accept edits on)
❯ npm test 실행 시 다음 에러가 발생합니다. 원인을 찾고 수정해 주세요.

  TypeError: Cannot read properties of undefined (reading 'plan')
      at getUserPlan (src/userService.js:9)

  기대 동작: profile이 없는 사용자는 "unknown"을 반환해야 합니다.
  수정 후 npm test로 3개 테스트가 모두 PASS 하는지 직접 확인해 주세요.
# 자율 작업 흐름 관찰
에러를 확인하고 수정하겠습니다. 먼저 현재 코드 상태를 확인하겠습니다.

  Read 3 files

문제를 찾았습니다. getUserPlan 함수에서 user.profile이 undefined일 때 .plan에 접근하려고 해서 에러가 발생합니다. 데이터를 보니 id=3인 사용자(Park)는 profile 필드가 없습니다.

CLAUDE.md 규칙에 따라 존재하지 않는 데이터는 기본값을 반환해야 하므로, 수정하겠습니다.

⏺ Update(src/userService.js)
Added 1 line, removed 1 line

export function getUserPlan(id) {
  const user = getUser(id);
  return user.profile.plan;
  return user?.profile?.plan || "unknown";
}

이제 테스트를 실행해서 모든 테스트가 통과하는지 확인하겠습니다.

  Ran 1 shell command

모든 테스트 통과
# 수정 요약 (출력 축약)
#   원인 : id=3(Park)에 profile이 없어 user.profile.plan이 undefined.plan 접근이 됨
#   수정 : return user?.profile?.plan || "unknown"
#   근거 : CLAUDE.md의 "예외를 던지지 말고 기본값 반환" 규칙
#   결과 : Test 1~3 전부 PASS

# 수정 사항 커밋은 skip

# Plan 모드로 복잡한 작업 요청
❯ 사용자 soft delete 기능을 추가해 주세요.
  - users 데이터에 deletedAt 필드 (기본 null)
  - removeUser(id): deletedAt에 현재 시각 기록
  - getUser / getUserPlan은 삭제된 사용자를 제외해야 함
  - test.js에 삭제 시나리오 테스트 추가
  구현 전에 계획을 먼저 보여 주세요.

Soft Delete 기능 추가

Context

현재 users.js의 사용자 데이터는 완전삭제만 가능한 구조이며, getUser/getUserPlan은 삭제 개념이 없다. 사용자 요청에 따라 소프트 삭제(soft delete)를 도입해 삭제된 사용자를 실제 데이터에서 제거하지 않고 deletedAt 타임스탬프로 표시하며, 조회 함수들이 삭제된 사용자를 자동으로 제외하도록 한다.

변경 파일

1. src/users.js

각 사용자 객체에 deletedAt: null 필드 추가 (CLAUDE.md 규칙상 데이터 구조를 "임의로" 바꾸는 게 아니라 요청된 필드 추가이므로 허용 범위):
export const users = [
  { id: 1, name: "Kim", profile: { email: "kim@example.com", plan: "pro" }, deletedAt: null }, 
  { id: 2, name: "Lee", profile: { email: "lee@example.com", plan: "free" }, deletedAt: null },
  { id: 3, name: "Park", deletedAt: null }     
];  

2. src/userService.js

- getUser(id): 삭제된 사용자(deletedAt이 null이 아닌 경우)를 조회 대상에서 제외.
export function getUser(id) {
  return users.find((u) => u.id === id && !u.deletedAt);
}
- getUserPlan(id): 변경 없음 — 이미 getUser를 통해 조회하므로 삭제된 사용자는 자동으로 "unknown" 처리됨 (기존 optional chaining 로직 그대로 재사용).
- removeUser(id): 신규 함수. 원본 users 배열에서 (삭제 여부와 무관하게) id로 사용자를 찾아 deletedAt에 현재 시각(ISO 문자열)을 기록. 존재하지 않는 id는 예외를 던지지 않고 조용히 무시(CLAUDE.md 컨벤션 준수).
export function removeUser(id) {
  const user = users.find((u) => u.id === id); 
  if (user) {
    user.deletedAt = new Date().toISOString(); 
  }
}
- (내부에서 users를 직접 참조해 삭제 여부와 관계없이 찾도록 getUser가 아닌 원본 배열 검색 사용)

3. test.js

removeUser import 추가 후 삭제 시나리오 테스트 1개 추가:       
import { getUserPlan, removeUser } from "./src/userService.js";
...
removeUser(1);
check("Test 4 - 삭제된 사용자", () => getUserPlan(1), "unknown");

검증

- npm test 실행 → 기존 3개 + 신규 1개, 총 4개 테스트 모두 PASS 확인.

# 승인 후 진행
❯ plan 대로 진행해

계획대로 구현을 진행하겠습니다.
...
완성 - 4개 테스트 모두 PASS
# 구현 결과 (계획과 동일, 출력 축약)
#   users.js       : 각 사용자에 deletedAt: null 추가
#   userService.js : getUser에 && !u.deletedAt 조건 추가, removeUser(id) 신규
#   test.js        : removeUser import + "Test 4 - 삭제된 사용자" 추가
# 터미널에서 회귀 확인
 npm test

> user-service-lab@1.0.0 test
> node test.js

Test 1 - pro 플랜 사용자: PASS
Test 2 - free 플랜 사용자: PASS
Test 3 - profile 없는 사용자: PASS
# (기록 확인 필요: soft delete 구현 후라면 "Test 4 - 삭제된 사용자: PASS"까지 4줄이 나와야 함 — 구현 전 출력이 붙은 것으로 보임)
```

### Headless 자동화

```shell
# 기본 Headless 호출
 claude -p "이 프로젝트의 구조와 목적을 3줄로 요약해 주세요" --output-format text
프로젝트 구조와 목적:

1. **목적**: Claude Code 워크샵 실습용 사용자 서비스 모듈로, 사용자 조회와 요금제 확인 기능을 제공합니다.

2. **기술**: Node.js 18+ ES Modules 기반의 순수 자바스크립트 프로젝트로, 외부 의존성이 없습니다.

3. **구조**: `src/userService.js`와 `src/users.js`로 구성되어 있으며, `test.js`로 테스트하고 모든 수정은 테스트 통과를 동반해야 합니다.
 
# 쉘 파이프 통합
 git log --oneline | claude -p "이 커밋 히스토리를 한 문단으로 요약해 주세요"
이 저장소는 초기 설정 커밋(`b8924c1`)으로 시작되었으며, 사용자 조회와 요금제 확인 기능을 제공하는 Node.js 기반의 사용자 서비스 모듈의 기본 골격이 구성되었습니다.

# JSON 출력으로 후처리
 claude -p "src 디렉토리의 코드 품질 이슈를 분석해 주세요" \
  --allowed-tools "Read,Grep,Glob" \
  --output-format json > report.json

# jq가 있다면
jq '.' report.json | head -30
{
  "is_error": false,
  "duration_api_ms": 20022,
  "num_turns": 6,
  "stop_reason": "end_turn",
  "session_id": "920a620e-6c87-498e-aedf-205dce601406",
  "total_cost_usd": 0.0699183,
  "usage": {
    "input_tokens": 26,
    "cache_creation_input_tokens": 27765,
    "cache_read_input_tokens": 52443,
    "output_tokens": 1699,
    "server_tool_use": {
      "web_search_requests": 0,
      "web_fetch_requests": 0
    },
    "service_tier": "standard",
    "cache_creation": {
      "ephemeral_1h_input_tokens": 27765,
      "ephemeral_5m_input_tokens": 0
    },
    "inference_geo": "not_available",
    "iterations": [
      {
        "input_tokens": 8,
        "output_tokens": 1098,
        "cache_read_input_tokens": 26459,
        "cache_creation_input_tokens": 1306,
        "cache_creation": {
          "ephemeral_5m_input_tokens": 0,

# 자동화 스크립트 생성
 cat > review.sh << 'EOF'
#!/bin/bash
DIFF=$(git diff HEAD~1)
echo "$DIFF" | claude -p "이 diff를 코드 리뷰해 주세요. 심각도별로 정리해 주세요." \
  --allowed-tools "" --output-format text > review.md
EOF
 
 chmod +x review.sh
 
 ./review.sh
fatal: 애매한 인자 'HEAD~1': 알 수 없는 리비전 또는 작업 폴더에 없는 경로.
경로와 리비전을 구분하려면 다음과 같이 '--'를 사용하십시오:
'git <명령> [<리비전>...] -- [<파일>...]'
# 실패 원인: 앞에서 수정 사항 커밋을 skip했기 때문에 저장소에 초기 커밋 1개뿐 → HEAD~1 리비전이 존재하지 않음.
# 커밋을 했다면 정상 동작.

# review 결과 확인 (diff가 비었는데도 review.md에 상세 리뷰가 생긴 것은 Claude가 작업 트리 파일을 직접 읽었기 때문)
 cat review.md
# review.md 내용 요약 (출력 축약)
#   Critical : users.js에 deletedAt을 추가한 것이 CLAUDE.md의 "데이터 구조 임의 변경 금지"와 충돌
#              → removeUser 안에서만 필드를 붙이거나 CLAUDE.md를 갱신해야 한다는 지적
#   Major    : removeUser가 이미 삭제된 사용자도 다시 찾아 idempotency가 깨짐.
#              getUser는 !u.deletedAt로 거르는데 removeUser는 안 걸러 일관성도 없음
#   긍정     : 기본값 반환 규칙 준수, optional chaining 적절, 테스트 추가와 통과, ES Modules
```

## 정리

- CLAUDE.md에 넣은 규칙이 실제 코드 수정 방향을 결정했다. 디버깅 때 Claude가 "CLAUDE.md 규칙에 따라 존재하지 않는 데이터는 기본값을 반환해야 하므로"라며 `?.` + `"unknown"`을 선택했다.
- Plan 모드는 구현 전에 설계를 검토할 수 있다. soft delete 계획에서 removeUser가 getUser 대신 원본 배열을 검색하는 이유, getUserPlan은 변경 불필요라는 판단을 승인 전에 확인했다.
- 커밋을 건너뛰어서 review.sh의 `git diff HEAD~1`이 fatal로 실패했다. 초기 커밋 하나뿐이라 비교 대상이 없었다. 헤드리스 자동화는 작업 단위로 커밋이 돼 있어야 돌아간다.
- 헤드리스 리뷰가 승인했던 계획의 문제를 잡았다. deletedAt 필드 추가가 CLAUDE.md의 데이터 구조 변경 금지와 충돌한다는 점, removeUser의 idempotency 불일치. 계획 승인과 별개로 리뷰 단계가 필요하다.

## References

- 실습 가이드(정본): [whchoi98.github.io/ccw-hands-on-lab — Ch1 Hands-on Lab](https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch1_HandsOnLab.html)
- 워크숍 저장소(슬라이드 PDF·코드 스니펫): github.com/whchoi98/claude-code-workshop
- 공식 문서: code.claude.com/docs (Best Practices 포함)
- Claude Code GitHub: github.com/anthropics/claude-code
- MCP 표준: modelcontextprotocol.io
