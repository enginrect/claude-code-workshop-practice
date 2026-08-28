## 실습 기록 (Labs)
실습은 https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch5_HandsOnLab.html#t0 으로 진행

> 환경: macOS, claude 2.1.234

### 사전 준비 확인

> 가이드 t0 · 헤드리스 호출은 실제 API를 태우므로 **인증 상태를 먼저 확정**하고 시작한다.
> 리뷰 대상이 될 커밋 2개짜리 저장소를 만드는데, 두 번째 커밋에 **의도적으로 검증 없는 코드**를 넣어 Task 2의 리뷰가 지적할 거리를 심어둔다.

```shell
# 1. 인증 확인 - 이미 인증은 되어있으니 생략
# claude auth status || echo "로그인 필요: claude 실행 후 인증 (Chapter 1 참고)"

# 2. jq 확인 - 상동
# jq --version

# 3. 실습 저장소 생성
#    package.json - Task 1 패턴 4(파이프 입력)의 재료. 구버전 의존성을 일부러 넣어 지적거리를 만든다
mkdir -p ~/claude-lab/ch5/src && cd ~/claude-lab/ch5

cat > package.json << 'EOF'
{
  "name": "cli-lab",
  "version": "1.0.0",
  "dependencies": { "express": "4.17.1", "lodash": "4.17.20" }
}
EOF

cat > src/pay.js << 'EOF'
export function charge(amount) {
  return { ok: true, amount };
}
EOF

git init -q
git config user.name  >/dev/null 2>&1 || git config user.name "lab"
git config user.email >/dev/null 2>&1 || git config user.email "lab@example.com"
git add -A && git commit -q -m "chore: cli lab scaffold"

# 4. 두 번째 커밋 - 입력 검증 없이 청구하는 코드.
#    Task 2의 my-review.sh가 HEAD~1..HEAD로 잡아낼 diff가 바로 이것이다
cat > src/pay.js << 'EOF'
export function charge(amount) {
  // TODO: 입력 검증 없이 그대로 청구
  return { ok: true, amount, currency: "KRW" };
}
EOF
git commit -qam "feat: add currency to charge"
git log --oneline

# 결과 (커밋 2개 확인)
da2548d (HEAD -> master) feat: add currency to charge
111fef8 chore: cli lab scaffold
```

### 헤드리스 5패턴, 자동화의 어휘

> 가이드 t1 · 앞으로 모든 자동화에서 반복될 5가지 기본형을 손에 익힌다.
> 각 패턴이 답하는 질문이 다르다 — ①어떻게 부르나 ②본문을 어떻게 꺼내나 ③비용을 어떻게 재나 ④데이터를 어떻게 넣나 ⑤맥락을 어떻게 잇나.

```shell
# 패턴 1. 단순 호출 - 결과가 stdout에 텍스트로 남고 세션은 바로 종료된다
#         -p는 출력 모드가 아니라 SDK 경로를 타는 단발 에이전트 실행이라, 훅과 MCP도 그대로 산다
claude -p "리스트와 튜플의 차이를 정확히 3줄로"

# 결과
리스트는 가변(mutable)이라 생성 후에도 요소를 추가·삭제·수정할 수 있지만, 튜플은 불변(immutable)이라 생성 시점의 내용이 고정됩니다.
이 불변성 덕분에 튜플은 해시 가능해서 딕셔너리 키나 집합 원소로 쓸 수 있고, 메모리도 더 적게 쓰며 생성 속도가 빠릅니다.
그래서 관례적으로 리스트는 동질적인 데이터의 가변 컬렉션에, 튜플은 위치마다 의미가 다른 고정된 레코드(좌표, 함수의 다중 반환값 등)에 씁니다.
#==================================================================
# 패턴 2. JSON 출력과 result 추출
claude -p "정렬 알고리즘 3가지 이름만" --output-format json | jq -r '.result'

# 결과
퀵 정렬, 병합 정렬, 버블 정렬
#==================================================================
# 패턴 3. 토큰과 비용 추적 - 자동화의 비용 대장이 여기서 나온다
#         가이드 예시는 in=4213인데 실제로는 in=2가 나왔다 (캐시에 걸린 분량은 cache_* 필드로 빠진다)
claude -p "OK 한 단어만" --output-format json | \
  jq '{in: .usage.input_tokens, out: .usage.output_tokens, usd: .total_cost_usd, ms: .duration_ms}'

# 결과
  jq '{in: .usage.input_tokens, out: .usage.output_tokens, usd: .total_cost_usd, ms: .duration_ms}'
{
  "in": 2,
  "out": 4,
  "usd": 0.10511899999999999,
  "ms": 3312
}
#==================================================================
# 패턴 4. 파이프 입력 - stdin이 데이터, 인자가 지시라는 역할 분담
cat package.json | claude -p "이 의존성 목록에서 눈에 띄는 위험 하나만 한 줄로"

# 결과
lodash 4.17.20에는 커맨드 인젝션 취약점(CVE-2021-23337, `_.template` 경유)이 있어 4.17.21로 올려야 합니다 — 한 버전 차이로 해결되는 가장 시급한 위험입니다.
#==================================================================
# 패턴 5. 세션 이어가기 - 봉투의 session_id를 열쇠로 다음 호출이 맥락을 물려받는다
#         스크립트가 다단 파이프를 설계할 때 쓰는 구조 (1단계 스캔 → 2단계 수정)
S=$(claude -p "정렬 알고리즘 3가지 이름만" --output-format json | jq -r '.session_id')
echo "session: $S"
claude -p --resume "$S" "그 중 첫 번째 것만 파이썬 코드로" --output-format json | jq -r '.result'

# 결과
session: 74a081b4-2051-4907-81a1-aae042b12a6a
# ```markdown 코드블럭 형식이 깨져서 ''' 로 대체 (기록용)
'''python 
def quick_sort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]
    left = [x for x in arr if x < pivot]
    mid = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quick_sort(left) + mid + quick_sort(right)
'''
```

> 구자료 주의 — 세션 지정은 `--resume`이다.
> 구버전 자료의 `claude --continue "$SESSION_ID"`는 현행과 다르다. `-c / --continue`는 **ID 없이** 현재 디렉토리의 최근 대화를 잇고,
> 특정 세션을 ID나 이름으로 지정할 때는 `-r / --resume`을 쓴다. 분기해서 실험하려면 `--resume "$S" --fork-session`으로 새 세션 ID를 만든다.

> 스크립트 가속 — `--bare`.
> 훅·스킬·MCP·CLAUDE.md 탐색이 필요 없는 순수 텍스트 작업이라면 `claude --bare -p "..."`로 기동 오버헤드를 줄일 수 있다. 반복 호출 파이프라인에서 체감이 크다.

- key point: 다섯 패턴 모두 성공. `--resume`은 세션이 이어져서 "그 중 첫 번째"만으로 퀵 정렬 코드가 나왔다. 봉투 숫자는 가이드 예시와 달랐다 — 패턴 3이 `in: 4213` 대신 `in: 2`, 비용은 `$0.013` 대신 `$0.105`. 체크포인트는 숫자 일치가 아니라 구조로 본다.

### 자동화 스크립트, 로컬 리뷰 봇

> 가이드 t2 · 커밋 범위를 인자로 받아 diff를 리뷰하고, 결과를 파일로 남기며, **심각도에 따라 exit code로 신호**하는 스크립트를 만든다.
> 이 exit code가 Task 4에서 그대로 CI 게이트가 된다 — 스크립트가 CI보다 먼저 완성되어야 하는 이유.

```shell
# 1. my-review.sh 작성
#    설계 포인트 3가지
#      set -euo pipefail        : 중간 실패를 삼키지 않고 즉시 드러낸다
#      --max-turns 5            : 에이전트 폭주 방지 (기본이 무제한이라 무인에서는 필수)
#      --allowed-tools Read/Grep/Glob : 읽기 전용으로 제한 = 리뷰 봇이 코드를 고치지 못하게 한다
#                                (공백 구분 다중 인자 형식)
cd ~/claude-lab/ch5
cat > my-review.sh << 'EOF'
#!/bin/bash
# my-review.sh <커밋범위> - 로컬 코드 리뷰 봇
set -euo pipefail
RANGE="${1:?Usage: ./my-review.sh <commit-range>  예: HEAD~1..HEAD}"

echo "1) diff 수집: $RANGE"
DIFF=$(git diff "$RANGE")
[ -z "$DIFF" ] && { echo "diff 없음"; exit 0; }

echo "2) Claude 리뷰 실행..."
RESULT=$(claude -p "$(cat <<PROMPT
다음 diff를 리뷰하세요.
관점: 1) 버그 위험 2) 보안 3) 테스트 필요성
각 지적은 "- [심각도] 파일: 내용" 형식, 심각도는 CRITICAL/WARN/INFO 중 하나.
심각한 문제가 없으면 "- [INFO] 특이사항 없음" 한 줄만.

$DIFF
PROMPT
)" --output-format json --max-turns 5 \
  --allowed-tools "Read" "Grep" "Glob")

echo "$RESULT" | jq -r '.result' > review.md
COST=$(echo "$RESULT" | jq -r '.total_cost_usd')
echo "3) 저장: review.md (비용: \$$COST)"

if grep -q "CRITICAL" review.md; then
  echo "4) CRITICAL 발견 → exit 1 (게이트 차단 신호)"
  exit 1
fi
echo "4) 통과"
EOF
chmod +x my-review.sh
bash -n my-review.sh && echo "문법 OK"

# 결과
문법 OK
#==================================================================
# 2. 실행과 exit code 확인
#    $? 를 같이 찍는 것이 핵심 - 사람은 review.md를 읽지만 CI는 이 숫자만 본다
./my-review.sh HEAD~1..HEAD; echo "exit code: $?"
cat review.md

# 결과
1) diff 수집: HEAD~1..HEAD
2) Claude 리뷰 실행...
3) 저장: review.md (비용: $0.598566)
4) CRITICAL 발견 → exit 1 (게이트 차단 신호)
exit code: 1
diff 본문이 프롬프트에 안 실려 왔습니다 (`각 지적은 -` 에서 잘림). `my-review.sh`의 기본 대상인 `HEAD~1..HEAD` (da2548d, `src/pay.js`)를 리뷰했습니다.
# ``` 코드블럭 ''' 로 대체
'''
export function charge(amount) {
-  return { ok: true, amount };
+  // TODO: 입력 검증 없이 그대로 청구
+  return { ok: true, amount, currency: "KRW" };
   }
'''

## 리뷰 결과

- **[CRITICAL] src/pay.js:1** — `amount` 검증이 전혀 없음. 음수(`charge(-10000)`), `0`, `NaN`, `undefined`, 문자열(`"1e9"`), `Infinity`가 모두 `ok: true`로 통과합니다. 결제 함수에서 음수 금액은 환불/크레딧 발급 악용 경로가 되고, `NaN`은 하위 PG 연동에서 조용히 0원 또는 파싱 에러로 갈라집니다. 진입점에서 `Number.isInteger(amount) && amount > 0` 검증 후 실패 시 throw 또는 `{ ok: false, reason }` 반환이 필요합니다.
- **[CRITICAL] src/pay.js:2** — 검증 부재를 인지한 상태로 `// TODO`만 남기고 머지됨. 코멘트가 방어 코드를 대신하지 못하며, 이슈 번호도 없어 추적되지 않습니다. 이 TODO는 커밋 전에 해소하거나 최소한 티켓 링크를 달아야 합니다.
- **[WARN] src/pay.js:3** — 커밋 메시지는 "add currency"인데 `currency`가 `"KRW"` 하드코딩입니다. 호출자가 통화를 지정할 방법이 없어, 다통화 호출이 들어오면 **금액은 그대로인데 통화 라벨만 KRW로 잘못 붙는** 무결성 사고가 납니다. `charge(amount, currency = "KRW")` + ISO-4217 화이트리스트 검증을 권합니다.
- **[WARN] src/pay.js:3** — 금액 단위 계약이 없음. `KRW`는 소수 단위가 없지만 위 통화 파라미터화 이후 `USD` 등이 들어오면 부동소수 `amount`가 곧바로 반올림 오차(`0.1 + 0.2`)로 이어집니다. 지금 최소 단위 정수(minor unit)로 계약을 못박아 두는 게 비용이 가장 쌉니다.
- **[WARN] src/pay.js:3** — `ok`가 항상 `true`인 상수. 호출자는 이를 성공 신호로 신뢰하게 되는데 실제 실패 경로가 없어, 나중에 실패 케이스가 생겨도 기존 호출자는 계속 성공으로 처리합니다. 반환 타입에 실패 분기를 먼저 만들어 두세요.
- **[WARN] src/pay.js** — 테스트 0건. `package.json`에 `scripts.test`도 테스트 파일도 없습니다. 최소 케이스: ①음수/0/`NaN`/`undefined`/문자열/`Infinity` 거부 ②정상 금액의 반환 shape(`ok`, `amount`, `currency`) ③통화 기본값 `"KRW"` ④(파라미터화 후) 미지원 통화 코드 거부. 위 CRITICAL은 ①번 테스트 하나로 회귀 방지가 됩니다.
- **[INFO] package.json** *(diff 범위 밖)* — `lodash 4.17.20`은 CVE-2021-23337(`_.template` 명령 주입) 영향 버전이라 `4.17.21` 이상이 필요하고, `express 4.17.1`도 `body-parser`/`path-to-regexp` ReDoS 계열 권고가 누적된 구버전입니다. 결제 코드가 있는 저장소이니 별도 커밋으로 올리는 걸 권합니다.

## 참고: `my-review.sh` 게이트

위 리뷰 결과에는 `CRITICAL`이 있으므로 스크립트는 의도대로 `exit 1` 합니다. 다만 한 가지 구멍이 있습니다 — `claude`가 `--max-turns 5` 초과 등으로 에러 subtype을 반환하면 `jq -r '.result'`가 `null`을 뱉고, `review.md`에 `"null"`만 남아 `grep -q CRITICAL`이 실패해 **조용히 게이트를 통과**합니다. `.is_error`/`.subtype`을 먼저 확인하고 비정상이면 `exit 2`로 빠지게 하시는 걸 권합니다.
```

> 예산 상한이 더 필요하면 `--max-budget-usd 0.50`을 추가한다. 턴 상한이 "몇 번 도느냐"라면 예산 상한은 "얼마까지 쓰느냐"로, 축이 다르므로 같이 거는 편이 안전하다.

- key point: 스크립트는 돌았고 `CRITICAL`을 잡아 의도대로 `exit 1`이 나왔다. 다만 리뷰 결과가 짚었듯 **diff가 프롬프트에 실리지 않았다** — 가이드 스크립트의 `claude -p "$(cat <<PROMPT ...)"`가 macOS 기본 `/bin/bash`(3.2)에서 히어독 안의 `)`에 걸려 잘린다. 그런데도 `--allowed-tools`의 `Read`·`Grep`·`Glob` 덕에 모델이 저장소를 직접 읽어 리뷰를 만들어냈다. **입력이 비었는데 결과와 exit code는 정상으로 보였다.**

### 파싱 파이프라인, JSON에서 대시보드까지

> 가이드 t3 · 여러 호출의 JSON을 모아 **CSV로 변환하고 통계를 집계**한 뒤 HTML 대시보드로 만든다.
> 조직의 사용량·비용 리포트 자동화가 이 구조의 확장판이고, topic 자리에 파일 목록이나 이슈 목록이 들어가면 그대로 대량 처리 패턴이 된다.

```shell
# 1. 데이터 수집, 3회 호출
#    --model haiku - 수집은 판단이 아니라 양이므로 저비용 모델으로 진행
cd ~/claude-lab/ch5 && mkdir -p data
for topic in Python Go Rust; do
  echo "수집: $topic"
  claude -p "$topic 언어의 장점을 정확히 2가지, 각 한 줄로" \
    --model haiku --output-format json > "data/$topic.json"
done
ls -la data/

# 결과
수집: Python
수집: Go
수집: Rust
total 24
drwxr-xr-x@ 5 enginrect  staff   160  8월 28 13:29 .
drwxr-xr-x@ 8 enginrect  staff   256  8월 28 13:28 ..
-rw-r--r--@ 1 enginrect  staff  1881  8월 28 13:29 Go.json
-rw-r--r--@ 1 enginrect  staff  1765  8월 28 13:28 Python.json
-rw-r--r--@ 1 enginrect  staff  1730  8월 28 13:29 Rust.json
#==================================================================
# 2. JSON → CSV 변환
#    input_filename으로 파일명에서 topic을 복원한다 (봉투 안에는 topic이 없으므로)
#    -s 없이 파일별로 처리하는 것이 핵심 (아래 주의 참고)
{
  echo "topic,duration_ms,input_tokens,output_tokens,cost_usd"
  jq -r '[
    (input_filename | sub("data/"; "") | sub("\\.json$"; "")),
    .duration_ms, .usage.input_tokens, .usage.output_tokens, .total_cost_usd
  ] | @csv' data/*.json
} > stats.csv
cat stats.csv

# 결과
topic,duration_ms,input_tokens,output_tokens,cost_usd
"Go",3834,10,340,0.011121800000000001
"Python",3891,10,359,0.014045900000000002
"Rust",4875,10,431,0.011584800000000001
#==================================================================
# 3. 통계 집계
#    여기서는 -s(slurp)가 맞다 - 전체를 하나의 배열로 봐야 length와 add가 성립하기 때문
jq -rs '{
  total_calls: length,
  avg_duration_ms: ([.[].duration_ms] | add / length | floor),
  total_input: ([.[].usage.input_tokens] | add),
  total_output: ([.[].usage.output_tokens] | add),
  total_cost_usd: ([.[].total_cost_usd] | add)
}' data/*.json

# 결과
{
  "total_calls": 3,
  "avg_duration_ms": 4200,
  "total_input": 30,
  "total_output": 1130,
  "total_cost_usd": 0.0367525
}
#==================================================================
# 4. HTML 대시보드 생성
#    Claude를 부르지 않고 awk로 CSV를 표로 렌더한다 - 형식 변환에 모델을 쓸 이유가 없다
#    (따옴표 없는 HTML 히어독이라 $(date)와 $(awk ...)가 생성 시점에 치환된다)
cat << HTML > dashboard.html
<!DOCTYPE html>
<html><head><meta charset="utf-8"><title>Claude Usage</title>
<style>
  body { font-family: sans-serif; max-width: 760px; margin: 40px auto; }
  table { border-collapse: collapse; width: 100%; }
  th, td { padding: 8px 10px; border-bottom: 1px solid #ddd; text-align: left; }
  th { background: #f0f0f0; }
</style></head><body>
<h1>Claude 호출 통계</h1>
<p>생성: $(date)</p>
$(awk -F, 'BEGIN{print "<table>"}
NR==1{printf "<tr>"; for(i=1;i<=NF;i++) printf "<th>%s</th>", $i; print "</tr>"; next}
{gsub(/"/,""); printf "<tr>"; for(i=1;i<=NF;i++) printf "<td>%s</td>", $i; print "</tr>"}
END{print "</table>"}' stats.csv)
</body></html>
HTML
echo "생성 완료: dashboard.html (브라우저로 열어 확인)"
open dashboard.html

# 결과
생성 완료: dashboard.html (브라우저로 열어 확인)
# 브라우저 결과
#Claude 호출 통계
#생성: 2026년 8월 28일 금요일 13시 30분 25초 KST

#topic	duration_ms	input_tokens	output_tokens	cost_usd
#Go	3834	10	340	0.011121800000000001
#Python	3891	10	359	0.014045900000000002
#Rust	4875	10	431	0.011584800000000001
```

> 구자료 주의 — 슬러프(`-s`)와 `input_filename`은 함께 쓰면 안 된다.
> 구버전 자료의 `jq -rs '.[] | [(input_filename | ...)]'` 패턴은 슬러프가 모든 파일을 먼저 읽어버려 **모든 행에 마지막 파일명**이 찍힌다.
> 파일당 JSON이 하나라면 **`-s` 없이 파일별 처리**가 맞고, 반대로 통계 집계는 전체를 하나의 배열로 봐야 하므로 `-s`가 맞다. 같은 데이터에 두 방식이 공존한다.

> 실전 확장 — 수백 건 규모라면 개별 스크립트에 `--max-budget-usd`로 상한을 걸고,
> 조직 관점에서는 Chapter 3의 OTel 텔레메트리가 이 수집 자체를 대체한다. 스트리밍 이벤트 단위 처리가 필요하면 `--output-format stream-json`.

- key point: 수집 3회 → CSV → 집계 → HTML 대시보드까지 전 단계 성공. 다만 CSV의 `input_tokens`는 캐시에 안 걸린 분량만 담아서 `total_input: 30`은 실제 입력량이 아니다. 비용 집계는 `total_cost_usd`를 기준으로 해야 한다.

### CI 통합, GitHub Actions 워크플로

> 가이드 t4 · Task 2의 리뷰 봇 개념을 CI로 올린다.
> 워크플로 작성과 문법 검증은 전원 진행하고, 실제 push 검증은 GitHub 저장소 보유자용 선택 트랙.

```shell
# 1. 워크플로 작성
#    로컬 스크립트와 달라지는 지점 3가지
#      fetch-depth: 0  - 얕은 체크아웃이면 base_ref와 비교할 이력이 없어 diff가 실패한다
#      secrets 주입    - 인증을 env로 넘긴다 (아래 3경로 표 참고)
#      3중 잠금        - timeout-minutes / --max-turns / --max-budget-usd
cd ~/claude-lab/ch5
mkdir -p .github/workflows
cat > .github/workflows/claude-check.yml << 'EOF'
name: Claude Check

on:
  pull_request:
    types: [opened, synchronize]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  claude-check:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Review diff
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/${{ github.base_ref }}...HEAD | \
            claude -p "이 diff에서 CRITICAL 수준 문제만 지적, 없으면 PASS 한 단어" \
            --output-format json --max-turns 3 --max-budget-usd 0.50 | \
            jq -r '.result' | tee check.md
          grep -q "CRITICAL" check.md && exit 1 || echo "게이트 통과"
EOF

# 2. YAML 문법 검증 - push 하고 Actions 탭에서 실패를 확인하는 왕복을 줄인다
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/claude-check.yml')); print('YAML 문법 OK')"

# 결과
YAML 문법 OK
```

**인증 준비, 3가지 경로**

| 경로 | 방법 | 적합 |
|---|---|---|
| API Key | 저장소 Settings → Secrets → `ANTHROPIC_API_KEY` 등록 | Console 계정 팀 |
| 구독 토큰 | 로컬에서 `claude setup-token`으로 장수명 토큰 발급 후 Secret 등록 | Pro/Max 구독 CI |
| Bedrock | OIDC로 AWS 역할 Assume + `CLAUDE_CODE_USE_BEDROCK=1` | 키 없는 조직 표준 (Ch.3) |

```shell
# 3. 선택 트랙, 실제 실행 (GitHub 저장소 보유자)
#    관찰 포인트 - Actions 탭에서 잡이 돌고, CRITICAL 유무에 따라 잡 성패가 갈리는지
git checkout -b test-claude-ci
git add .github && git commit -m "ci: add claude check"
git push -u origin test-claude-ci
gh pr create --title "Test Claude CI" --body "Testing" --fill 2>/dev/null || true
gh run watch

# 결과
새로 만든 'test-claude-ci' 브랜치로 전환합니다
[test-claude-ci b6d7025] ci: add claude check
 1 file changed, 32 insertions(+)
 create mode 100644 .github/workflows/claude-check.yml
fatal: 'origin' does not appear to be a git repository
fatal: 리모트 저장소에서 읽을 수 없습니다

올바른 접근 권한이 있는지, 그리고 저장소가 있는지
확인하십시오.
failed to determine base repo: no git remotes found
```

> CI 비용과 안전의 3중 잠금 — `timeout-minutes`(작업 시간 상한), `--max-turns`(에이전트 턴 상한), `--max-budget-usd`(지출 상한)를 항상 함께 건다. 축이 셋 다 달라서 하나로는 못 막는다.
> `permissions`는 최소로 시작하고 PR 코멘트가 필요할 때만 `pull-requests: write`를 추가한다. 시크릿은 organization secret으로 중앙 관리하는 편이 회전에 유리하다.

> 선택 트랙은 `~/claude-lab/ch5`가 로컬 전용 저장소라 원격이 없어서 `push` 단계에서 멈췄다. 워크플로 작성과 YAML 검증까지가 전원 공통 범위이고, 실제 잡 실행은 확인하지 않았다.

- key point: 워크플로 작성과 YAML 문법 검증까지 성공. 실제 실행은 원격이 없어 `fatal: 'origin' does not appear to be a git repository`로 멈췄다. 로컬 스크립트와 달라지는 지점은 `fetch-depth: 0`(얕은 체크아웃이면 `base_ref` 비교 이력이 없어 diff가 빈다), 인증을 env로 넘기는 것, 3중 잠금 세 가지다.

### 마무리, 학습 목표 점검

| 목표 | 확인 질문 | 관련 |
|---|---|---|
| Headless | `-p` / JSON / 파이프 / `--resume` 5패턴을 손으로 실행했는가 | Task 1 |
| Scripting | diff 주입, 도구 제한, exit code 게이트 스크립트를 만들었는가 | Task 2 |
| Parsing | 다중 호출 JSON을 CSV와 대시보드로 집계했는가 | Task 3 |
| CI/CD | 3중 잠금이 걸린 워크플로를 작성하고 검증했는가 | Task 4 |

**자동화 5패턴 지도 — 오늘 만든 골격이 꽂히는 곳**

| 패턴 | 구조 | 오늘 만든 재료 |
|---|---|---|
| 1 PR 리뷰 봇 | diff 주입 → 리뷰 → 코멘트/게이트 | Task 2 + Task 4 그대로 |
| 2 이슈 트리아지 | 이슈 본문 파이프 → 분류 JSON → 라벨링 | Task 1 패턴 4 + jq |
| 3 코드 마이그레이션 | 파일 루프 → 변환 → 실패 수집 | Task 3 루프 구조 |
| 4 보안 감사 | 주기 실행 → CRITICAL 게이트 → 알림 | Task 2 exit code + cron |
| 5 일일 보고서 | 다중 수집 → 집계 → HTML 발행 | Task 3 파이프라인 그대로 |

```shell
# 실습 정리 - 랩 저장소와 산출물 제거 (계속 쓸 거면 건너뛴다)
# rm -rf ~/claude-lab/ch5
```

## 정리

- `-p`는 출력 모드가 아니라 단발 에이전트 실행이다. 도구·훅·권한이 대화형과 똑같이 살아 있어서, Task 2에서 프롬프트가 비었는데도 `--allowed-tools`로 준 `Read`·`Grep`·`Glob`으로 저장소를 직접 읽어 리뷰를 만들어냈다. 도구를 열어준 만큼 모델이 자율적으로 움직인다.
- 자동화가 Claude와 주고받는 것은 stdout·exit code·JSON 봉투 셋뿐이다. 사람은 review.md를 읽지만 게이트는 숫자만 본다. Task 2에서 만든 `exit 1`이 Task 4에서 그대로 CI 잡의 성패가 됐다 — 스크립트가 먼저 완성돼야 CI가 그 위에 얹힌다.
- 봉투에서 `.result`가 본문이고 나머지는 관측 메타데이터다. `session_id`는 재개 열쇠고(패턴 5의 `--resume`으로 "그 중 첫 번째"만으로 코드가 나왔다), `total_cost_usd`·`duration_ms`·`usage`는 비용 대장의 재료다. 사용량 리포트라는 게 결국 봉투를 파일로 쌓아 집계하는 일이다.
- 셸과 모델의 분업이 파이프라인 설계의 핵심이다. diff 수집은 git이, CSV를 표로 바꾸는 건 awk가 하고 모델은 판단에만 쓴다. 형식 변환에 모델을 부르면 비용도 들고 결과도 흔들린다.
- 무인 실행의 상한은 축이 셋이다 — `--max-turns`(기본 무제한이라 반드시 지정), `--max-budget-usd`, CI의 `timeout-minutes`. 다만 상한은 폭주를 막을 뿐이라, 입력이 비거나 에러 봉투가 와서 조용히 통과하는 경우는 따로 검사해야 한다.
- 비용은 토큰 수가 아니라 `total_cost_usd`로 읽는다. `-p`는 호출마다 새 세션이라 시스템 프롬프트를 매번 캐시에 쓰고, `input_tokens`에는 캐시에 안 걸린 분량만 남는다. `--model`을 안 주면 설정된 기본 모델 단가가 그대로 붙어 haiku 지정 호출과 10배 차이가 났다.

## References

- 실습 가이드(정본): [whchoi98.github.io/ccw-hands-on-lab — Ch5 Hands-on Lab](https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch5_HandsOnLab.html)
- 워크숍 저장소(슬라이드 PDF·코드 스니펫): github.com/whchoi98/claude-code-workshop
- 공식 문서: code.claude.com/docs/ko/cli-reference, headless, sessions, github-actions
