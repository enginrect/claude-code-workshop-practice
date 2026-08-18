## 실습 기록 (Labs)
실습은 https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch2_HandsOnLab.html#t0 으로 진행

> 환경: macOS, claude 2.1.220
> Chapter 1에서 만든 `~/claude-lab/ch1` 프로젝트를 이어서 사용 (모든 상대 경로는 이 폴더 기준)

### 사전 준비 확인

```shell
# 프로젝트 상태 확인 (npm test 3개 PASS, git log 커밋 2개 이상)
  cd ~/claude-lab/ch1
 npm test

> user-service-lab@1.0.0 test
> node test.js

Test 1 - pro 플랜 사용자: PASS
Test 2 - free 플랜 사용자: PASS
Test 3 - profile 없는 사용자: PASS

  git log --oneline
5b7edbb (HEAD -> master) fix: handle users without profile
9eeea63 chore: initial lab project (with bug)
b8924c1 chore: initial lab project

# jq 설치 확인 (Task 5의 JSON 후처리에 필요)
 jq --version || sudo dnf install -y jq
jq-1.7.1-apple
```

- key point: 첫 리뷰 대상이 `HEAD~1` diff이므로 버그 커밋과 수정 커밋이 분리돼 있어야 한다. jq는 Task 5의 JSON 후처리에서만 쓰이지만 여기서 미리 확인해 둔다.

### 첫 Subagent, code-reviewer 만들기

> 정의 파일 작성 → 등록 확인 → Chapter 1의 버그 수정 커밋을 첫 리뷰 대상으로 호출

```shell
# 1. 에이전트 정의 파일 작성 (가이드 Task 1의 heredoc 블록 실행)
cd ~/claude-lab/ch1
mkdir -p .claude/agents

cat > .claude/agents/code-reviewer.md << 'EOF'
---
name: code-reviewer
description: |
  PR diff를 검토하여 가독성, 안전성, 성능, 테스트 커버리지 관점에서
  개선 사항을 제안하는 시니어 코드 리뷰어. 사용 시점: 코드 리뷰가
  필요하다고 사용자가 요청하거나 git diff 검토가 필요한 경우.
tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*)
model: sonnet
---
당신은 10년 경력의 시니어 소프트웨어 엔지니어입니다.

# 검토 우선순위
1. 안전성: null 참조, race condition, 입력 검증 누락
2. 가독성: 명명, 함수 크기, 주석 필요성
3. 성능: 알고리즘 복잡도, 불필요한 I/O
4. 테스트: 새 코드의 테스트 커버리지

# 출력 형식
- Severity: high | medium | low
- 파일:라인, 문제 설명, 권장 수정안
EOF
#==================================================================
# 2. 등록 확인 (/agents 위저드는 v2.1.198+에서 제거, @ typeahead 또는 질문으로 확인)
❯ 현재 이 세션에서 사용할 수 있는 서브에이전트 목록을 알려주세요

# 출력 요약 - 프로젝트 정의 1종 + 내장 7종이 함께 나온다
#   code-reviewer     : 방금 만든 것 (.claude/agents/)
#   claude / claude-code-guide / Explore / general-purpose / Plan / statusline-setup : 내장
# 각 항목 설명이 내가 쓴 description 문장 그대로 노출된다

# code-reviewer 에이전트 존재 확인 (@ typeahead 자동 완성 기능)
  + /Users/enginrect/claude-lab/ch1/.claude/agents/code-reviewer.md
  ◇ claude.ai Notion:notion://doc… – Complete specification for Notion's enhanced Markdown forma…
▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
❯@codere
#==================================================================
# 3. 첫 호출 (버그 수정 커밋 리뷰)
❯ code-reviewer 에이전트를 사용해서 최근 커밋의 변경분(git diff HEAD~1)을 검토해 주세요

  Ran 1 shell command

⏺ 이제 code-reviewer 에이전트로 이 변경분을 검토하겠습니다:

⏺ code-reviewer(Review recent commit changes)
  ⎿ Backgrounded agent (↓ to manage · ctrl+o to expand)

⏺ code-reviewer 에이전트가 최근 커밋의 변경분을 검토하고 있습니다. 곧 결과를 받게 될 것 같습니다.

  변경분을 보면:
  - getUserPlan() 함수에서 optional chaining (?.)과 nullish coalescing (??)을 사용하여 안전하게 처리하는 것으로 보입니다
  - 기존에는 user.profile.plan에 직접 접근했는데, 이제 user?.profile?.plan ?? "unknown"로 변경되었네요

  에이전트의 상세한 검토 결과를 기다리고 있습니다. 완료되면 자동으로 알림을 받게 됩니다. ⏳

✻ Waiting for 1 background agent to finish

⏺ Agent "Review recent commit changes" finished · 31s

⏺ 코드 리뷰 완료
# 결과 요약 (출력 축약) - 시스템 프롬프트에 지정한 형식을 그대로 따랐다
#   종합    : Blocking 없음, 테스트 3개 PASS, 컨벤션 준수
#   Low 3건 : userService.js:9 의도 주석 / test.js에 존재하지 않는 id(999) 케이스 /
#             npm test 실패 시 exit code 1 설정 (CI 감지용, 이번 diff 범위 밖)
#   결론    : 머지 가능
```

| 필드 | 역할 | 이 예시의 선택 |
|---|---|---|
| `name` | 고유 식별자, kebab-case | code-reviewer |
| `description` | **자동 디스패치의 판단 기준**, 사용 시점까지 명시 | 리뷰 요청 / git diff 검토 시 |
| `tools` | 도구 화이트리스트, Bash는 명령 패턴으로 제한 | 읽기 전용 + git diff/log |
| `model` | sonnet/opus/haiku/전체 ID/inherit (생략 시 inherit) | sonnet |

- key point: `/agents` 위저드가 없어도 "사용 가능한 서브에이전트를 알려줘" 질문과 `@` typeahead 두 경로로 등록을 확인할 수 있다. 호출된 리뷰어는 백그라운드로 31초 돌았고, 그 사이 메인이 먼저 자체 diff 해석을 내놨다. 결과는 시스템 프롬프트에 지정한 Severity / 파일:라인 / 머지 가능 여부 형식을 그대로 따랐다.

### 도구 권한 격리 체험

> 읽기 전용 에이전트에 일부러 권한 밖 요청 → 권한 추가 → 새 위임으로 차이 확인

```shell
# 1. 읽기 전용 테스트 에이전트 작성
cat > .claude/agents/dangerous-test.md << 'EOF'
---
name: dangerous-test
description: 의도적 권한 테스트용 에이전트. 사용 시점: 사용자가 이름으로 직접 지정할 때만.
tools: Read, Grep
model: haiku
---
당신은 테스트 에이전트입니다.
파일을 분석하고 결과를 보고하세요.
EOF
#==================================================================
# 2. 일부러 권한을 벗어난 요청 (파일 워처가 몇 초 안에 자동 인식 → 재시작 불필요)
❯ dangerous-test 에이전트를 사용해서 src/users.js의 Park 사용자에게 profile 필드를 추가해 주세요

프로젝트 규칙 확인이 필요합니다.

CLAUDE.md에서 **"src/users.js의 데이터 구조를 임의로 변경하지 않기"**라고 명확히 명시되어 있습니다. Park 사용자에게 profile 필드를 추가하는 것은 이 규칙을 위반합니다.

혹시 다음 중 하나를 의도하신 건가요?

1. 규칙을 일시적으로 무시하고 진행 → 사용자 명시 필요
2. 대신 다른 방식으로 처리 → 예를 들어, profile 필드가 필요할 때 기본값을 반환하는 로직을 추가 (규칙에 부합)
3. 테스트 코드에서만 profile 추가 → users.js는 변경하지 않고 test.js에서만 mock 처리

어떻게 진행하길 원하시나요?
#==================================================================
# 3. 권한을 명시적으로 부여
⏺ Agent "Park 사용자에게 profile 필드 추가" finished · 1m 43s

dangerous-test 에이전트가 작업을 완료했습니다. 이제 실제로 파일을 확인하고 수정하겠습니다.

  Read 1 file

⏺ Update(src/users.js)
Added 1 line, removed 1 line
export const users = [
  { id: 1, name: "Kim", profile: { email: "kim@example.com", plan: "pro" } },
  { id: 2, name: "Lee", profile: { email: "lee@example.com", plan: "free" } },
  { id: 3, name: "Park" }
  { id: 3, name: "Park", profile: { bio: '', avatar: null } }
];

✅ Park 사용자에게 profile 필드가 추가되었습니다.

{ id: 3, name: "Park", profile: { bio: '', avatar: null } }

이제 테스트가 여전히 통과하는지 확인하겠습니다:

  Ran 1 shell command

완료되었습니다! ✨

Park 사용자에게 profile 필드({ bio: '', avatar: null })가 정상적으로 추가되었으며, 모든 테스트가 통과했습니다.

- Test 1 (pro 플랜): ✅ PASS
- Test 2 (free 플랜): ✅ PASS
- Test 3 (profile 없는 사용자): ✅ PASS
#==================================================================
# 4. /exit 후 재진입해서 새 위임으로 재시도
# 이미 ! 를 통해 shell 명령을 진행했음으로 생략
#==================================================================
# 5. 실습 프로젝트 원상 복구
 git checkout -- src/users.js
 rm -f .claude/agents/dangerous-test.md .claude/agents/dangerous-test.md.bak
```

> `tools`를 아예 생략하면 **메인의 모든 도구를 상속**(지양). 대부분 상속하되 일부만 빼려면 `disallowedTools: Write, Edit`.

- key point: 가이드 시나리오와 다르게 거부 사유가 tools 화이트리스트가 아니라 CLAUDE.md의 "src/users.js 데이터 구조를 임의로 변경하지 않기" 규칙이었다. 서브에이전트가 메인의 CLAUDE.md와 메모리를 상속한다. 파일을 실제로 고친 것도 dangerous-test가 아니라 메인이었다("에이전트가 작업을 완료했습니다. 이제 실제로 파일을 확인하고 수정하겠습니다"). 권한 없는 워커는 보고까지만 하고 쓰기는 권한 있는 쪽이 한다.

### 에이전트 3종 완성 (test-writer, docs-writer)

> Task 4의 디스패치 실습을 위해 2종 추가. 두 description의 **영역이 겹치지 않게** 작성된 점에 주목

```shell
# 1. 두 에이전트 한 번에 생성
 cat > .claude/agents/test-writer.md << 'EOF'
---
name: test-writer
description: |
  기존 코드의 테스트 커버리지를 분석하고 누락된 엣지 케이스, 분기,
  예외 경로에 대한 테스트를 작성. 사용 시점: 커버리지 부족이 확인되거나
  새 함수에 테스트가 필요한 경우.
tools: Read, Grep, Glob, Edit, Write, Bash(npm test:*)
model: sonnet
---
당신은 테스트 자동화 전문가입니다.

# 작업 절차
1. 대상 함수의 시그니처와 분기 구조 분석
2. 기존 테스트 파일이 있다면 스타일 일치
3. 정상 케이스, 엣지 케이스, 예외 케이스 모두 커버
4. 테스트 작성 후 실제로 실행하여 통과 확인

# 작성 원칙
- 테스트 하나가 한 가지만 검증
- 의미 있는 테스트 이름
- AAA 패턴 (Arrange Act Assert)
EOF

cat > .claude/agents/docs-writer.md << 'EOF'
---
name: docs-writer
description: |
  README, API 문서, 아키텍처 결정 문서(ADR)를 코드와 일치하도록 작성하거나
  업데이트. 사용 시점: 신규 기능 추가 후 문서 작성이 필요하거나 기존 문서가
  코드와 불일치할 때.
tools: Read, Grep, Glob, Edit, Write, Bash(git log:*)
model: sonnet
---
당신은 기술 문서 작성가입니다.

# 문서 종류별 가이드
- README: 프로젝트 개요, 설치, 사용법, 예시
- API 문서: 함수 시그니처, 파라미터, 반환값, 예시
- ADR: Context, Decision, Consequences 형식

# 작성 원칙
- 코드와 실제 동작이 일치해야 함
- 예시는 실제로 실행 가능한 것
- 한국어와 영어 혼용 가능, 일관성 유지
EOF

ls .claude/agents/
code-reviewer.md docs-writer.md   test-writer.md
#==================================================================
# 2. 3종 등록 확인
❯ 사용 가능한 서브에이전트를 모두 알려주세요

# 출력 요약 - 프로젝트 3종 + 내장 6종, 총 9개
#   code-reviewer : PR diff 검토               (Read, Grep, Glob, Bash git)
#   test-writer   : 커버리지 분석과 테스트 작성 (Read, Grep, Glob, Edit, Write, npm test)
#   docs-writer   : README/API 문서/ADR         (Read, Grep, Glob, Edit, Write, git log)
#   나머지는 내장 - claude, claude-code-guide, Explore, general-purpose, Plan, statusline-setup
# 도구 목록까지 표시되므로 tools 필드가 그대로 드러난다

# typeahead 로 확인 (docs-writer 는 생략)
+ .claude/agents/test-writer.md
* test-writer (agent) – 기존 코드의 테스트 커버리지를 분석하고 누락된 엣지 케이스, …
+ /Users/enginrect/claude-lab/ch1/.claude/agents/test-writer.md
▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
❯@test-writer
```

> **description이 디스패치 정확도를 결정한다** — "코드를 분석하는 도우미"처럼 모호하거나
> "모든 개발 작업을 돕는 만능"처럼 광범위하면 잘못된 디스패치가 발생. 구체적 역할 + **사용 시점** 명시.
>
> `.claude/agents/`는 프로젝트 단위(팀 공유, Git 커밋 대상), `~/.claude/agents/`는 개인 전역.

- key point: 목록에는 내가 만든 3종 외에 내장 6종(claude, claude-code-guide, Explore, general-purpose, Plan, statusline-setup)까지 함께 나온다. 각 항목이 내가 쓴 description 문장 그대로 요약되고 typeahead 미리보기에도 같은 문장이 보인다.

### 디스패치 3패턴

> 자동 / 명시 / 병렬 3가지 호출 방식 비교 + `/cost`로 비용 확인
> (패턴 사이에 `/exit` 재진입 또는 `/clear`로 컨텍스트를 정리하면 관찰이 깨끗해진다)
>
> 참고: 위임 내부 도구가 **Task → Agent로 개명**(구 표기는 별칭). 서브에이전트는 **기본 백그라운드 실행**이며
> 진행 상황은 프롬프트 아래 패널과 `/tasks`, 권한 요청은 메인 세션에 표시, `Ctrl+B`로 백그라운드 전환.

```shell
# 1. 패턴 A - 자동 디스패치 (에이전트를 지정하지 않음)
❯ 이 프로젝트의 최근 변경 사항을 코드 리뷰해 주세요

  Ran 3 shell commands

⏺ code-reviewer(Review latest commit changes)
Backgrounded agent (↓ to manage · ctrl+o to expand)

최근 커밋을 분석하고 있습니다. 코드 리뷰어 에이전트가 변경사항을 검토하고 있으니 잠시만 기다려 주세요. 곧 결과를 알려드리겠습니다. 🔍

✻ Waiting for 1 background agent to finish

⏺ Agent "Review latest commit changes" finished · 58s

코드 리뷰 결과: fix: handle users without profile

# 결과 요약 (출력 축약)
#   잘한 점  : ?. + ?? 로 profile 없는 사용자 방어, CLAUDE.md의 "기본값 반환" 규칙 준수,
#              users.js 구조는 그대로 두고 사용처에서만 방어
#   medium 1 : test.js가 실패해도 exit code 0 → CI에서 실패 감지 불가
#   low 3    : 존재하지 않는 id 테스트 부재 / "unknown" 매직 스트링 /
#              사용자 없음과 plan 없음이 같은 값이라 구분 불가
#==================================================================
# 2. 패턴 B - 명시적 호출
❯ test-writer 에이전트를 사용해서 src/userService.js의 테스트 커버리지를 분석하고, 존재하지 않는 사용자 id 같은 누락된 엣지 케이스 테스트를 추가해 주세요

  Read 3 files

⏺ test-writer(Add missing edge case tests for userService)
  ⎿  Backgrounded agent (↓ to manage · ctrl+o to expand)

⏺ test-writer 에이전트가 테스트 커버리지를 분석하고 누락된 엣지 케이스를 추가하고 있습니다. 곧 결과를 알려드리겠습니다. 🧪

✻ Waiting for 1 background agent to finish

⏺ Agent "Add missing edge case tests for userService" finished · 1m 12s

⏺ 완료 - test-writer가 커버리지를 분석하고 엣지 케이스를 추가했다
# 결과 요약 (출력 축약)
#   커버리지 : 기존 3개 → 21개 테스트, 전부 PASS
#   추가분   : getUserPlan 8개(999, 0, 음수, 문자열, undefined, null, 타입 검증)
#              getUser 10개(정상 조회, profile 없는 사용자, 잘못된 id 계열, 타입 검증)
#   부수 수정: check()에 Object.is 비교와 checkType() 헬퍼 추가,
#              실패 시 process.exitCode = 1 → 패턴 A 리뷰가 지적한 medium 이슈를 그대로 해결
#==================================================================
# 3. 패턴 C - 병렬 호출 (총 시간은 합산이 아니라 max)
❯ 이 코드베이스를 종합 점검해 주세요. 다음 세 가지를 병렬로 진행해 주세요.
  1) 코드 품질 리뷰
  2) 테스트 커버리지 분석
  3) README 문서 초안 작성

⏺ 2 background agents launched (↓ to manage)
   ├ code-reviewer (Comprehensive code quality review)
   └ test-writer (Comprehensive test coverage analysis)

⏺ 3개의 에이전트가 병렬로 작동 중입니다! 🚀

  - 코드 리뷰어: 전체 코드 품질 및 아키텍처 평가 중...
  - 테스트 분석가: 커버리지 및 테스트 품질 분석 중...
  - 문서 작성가: README 작성 중...

  잠시만 기다려주세요. 곧 세 가지 종합 점검 결과를 한 번에 알려드리겠습니다. 📋

✻ Waiting for 3 background agents to finish

⏺ Agent "Comprehensive code quality review" finished · 1m 26s

⏺ 완료! 코드 리뷰 결과가 도착했습니다. ✅ 다른 2개 결과도 곧 올 것 같습니다. 🔄

✻ Waiting for 2 background agents to finish

⏺ Agent "Create comprehensive README documentation" finished · 2m 7s

⏺ README 작성 완료! ✅

  - README.md가 프로젝트 루트에 생성되었습니다
  - 모든 코드 예제가 실제 실행으로 검증됨
  - 개발 컨벤션 문서화 완료

  마지막 에이전트(테스트 커버리지 분석) 결과를 기다리는 중... ⏳

✻ Waiting for 1 background agent to finish

⏺ Agent "Comprehensive test coverage analysis" finished · 3m 37s

⏺ 세 가지 종합 점검 완료
# 결과 요약 (출력 축약)
#  1) 코드 품질 : Critical/High 없음. medium 2건 - getUser()가 원본 객체 참조를 반환해
#                 호출자가 내부 데이터를 변조할 수 있음, check()에 try/catch 없음
#  2) 커버리지  : 라인/함수/브랜치 100%인데 뮤테이션 커버리지는 약 85%.
#                 ?? 를 || 로 바꿔도 테스트가 통과한다 - 실제 데이터가 전부 truthy라 검증 불가.
#                 falsy-but-valid 값("", 0) 테스트가 빠져 있다
#  3) README    : 프로젝트 루트에 생성, 코드 예제 실행 검증까지 완료
#  액션 : P1 check()에 try/catch, P2 방어적 복사 또는 Object.freeze + resolvePlan() 분리
#==================================================================
# 4. 비용 확인 (/cost는 현재 /usage의 별칭)
❯ /usage
 
Subagents               % of usage
   test-writer                     1%
   code-reviewer                   1%
# 나머지 sub agent들은 생략 된듯 해 보인다.
```

> 병렬 실행은 시간이 max로 줄지만 비용은 합산된다.
> 3개 병렬은 약 7배까지 증가 가능. 반복적 단순 작업은 `model: haiku`, Opus는 심층 분석에만.

- key point: 패턴 A에서 에이전트를 지정하지 않았는데 code-reviewer가 선택됐다. description 매칭이 동작한다. 패턴 B의 test-writer는 절차 4번대로 `npm test`를 직접 돌려 21개 통과를 확인했고, 패턴 A 리뷰가 지적한 exit code 이슈까지 같이 고쳤다. 체이닝을 명시하지 않아도 메인 컨텍스트가 앞 결과를 유지한다. 패턴 C는 패널에 "2 background agents"로 떴지만 실제로는 3개가 돌았고, 완료가 1m26s / 2m7s / 3m37s라 총 시간은 합산이 아니라 max였다. `/usage`에는 test-writer와 code-reviewer만 1%씩 잡히고 docs-writer는 안 나왔다.

### Headless 통합과 CI 맛보기

> 대화 없이 스크립트에서 서브에이전트 호출 → JSON 후처리 → CI 패턴 확인 (세션 밖 터미널에서 실행)

```shell
# 1. 헤드리스로 에이전트 호출
 git diff HEAD~1 | claude -p \
  "code-reviewer 에이전트로 이 diff를 검토해 주세요" \
  --output-format json > review-result.json

head -c 400 review-result.json
{"is_error":false,"duration_api_ms":158163,"num_turns":2,"stop_reason":"end_turn","session_id":"1d3f3522-06d3-4e95-9288-3cc63fdebf37","total_cost_usd":0.44302530000000007,"usage":{"input_tokens":4,"cache_creation_input_tokens":13425,"cache_read_input_tokens":39771,"output_tokens":3327,"server_tool_use":{"web_search_requests":0,"web_fetch_requests":0},"service_tier":"standard","cache_creation":{"ep%
#==================================================================
# 2. JSON 후처리 (jq가 있다면: 요약과 high severity만 추출)
jq '.summary' review-result.json
jq '.issues | map(select(.severity == "high"))' review-result.json 2>/dev/null
null
```

> **CI 안전장치**: API Key는 반드시 Secrets로 주입, 읽기 전용 검사라면 `--allowed-tools "Read,Grep,Glob"`으로 도구 제한.
> (헤드리스에서 위임을 쓰려면 `--allowed-tools`에 **`Agent` 포함 필수**)

- key point: `jq '.summary'`와 `.issues`가 전부 null이었다. `--output-format json`의 최상위에는 `is_error`, `duration_api_ms`, `total_cost_usd`, `usage` 같은 실행 메타데이터가 오고 리뷰 본문은 `.result`에 담긴다. 가이드의 예시 필터를 그대로 쓰지 말고 `head`로 스키마를 확인한 뒤 1주차처럼 `jq -r '.result'`로 꺼내야 한다. 이번 헤드리스 1회 호출 비용은 `total_cost_usd: 0.443`, 소요 158초였다.

## 정리

- description은 등록 목록, `@` typeahead 미리보기, 자동 디스패치 판단에 전부 그대로 쓰인다. "사용 시점"을 문장에 넣어서 패턴 A에서 에이전트를 지정하지 않고도 code-reviewer가 선택됐다. 시스템 프롬프트의 출력 형식도 리뷰 결과에 그대로 반영됐다. 정의 파일이 결과물의 스펙 역할을 한다.
- Task 2에서 막은 건 tools 화이트리스트가 아니라 CLAUDE.md 규칙이었고, 실제 쓰기는 메인이 했다. 서브에이전트는 메모리를 상속하고, 권한 밖 작업은 실패가 아니라 보고로 끝나고 상위로 넘어온다.
- 병렬 3개 실행에서 총 시간은 max(3m37s)였다. 다만 패널 표기는 2개로 나왔고 `/usage`에도 docs-writer가 빠져 있었다. 진행 상황과 비용을 화면 표시만으로 판단하기는 어렵다.
- 가이드의 `.summary`/`.issues` 필터가 null을 반환했다. JSON 스키마를 확인하지 않고 파이프라인을 짜면 빈 결과가 나온다. CI에 넣기 전 `head`나 `jq keys`로 최상위 필드부터 확인한다.

## References

- 실습 가이드(정본): [whchoi98.github.io/ccw-hands-on-lab — Ch2 Hands-on Lab](https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch2_HandsOnLab.html)
- 워크숍 저장소(슬라이드 PDF·코드 스니펫): github.com/whchoi98/claude-code-workshop
- 공식 문서: code.claude.com/docs/ko/sub-agents
