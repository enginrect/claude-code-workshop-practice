## 실습 기록 (Labs)
실습은 https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch3_HandsOnLab.html#t0 으로 진행

> 환경: macOS, claude 2.1.220
> 작업 폴더는 `~/claude-lab/ch3`. Task 3·4는 시스템 경로에 배포하므로 sudo 권한 필요
> (sudo가 없다면 `~/.claude/settings.json`에 같은 JSON을 넣어 동작만 체험 가능 — 차이는 강제성뿐)

### 사전 준비 확인

```shell
 sudo -v && echo "sudo OK"
node -v && npm -v

mkdir -p ~/claude-lab/ch3/src && cd ~/claude-lab/ch3

cat > .env << 'EOF'
# 실습용 가짜 값입니다. 실제 시크릿을 넣지 마세요.
DB_PASSWORD=lab-fake-password
API_TOKEN=lab-fake-token
EOF

cat > src/app.js << 'EOF'
export function hello() { return "admin lab"; }
EOF
ls -la
Password:
sudo OK
v26.6.0
11.18.0
total 8
drwxr-xr-x@ 4 enginrect  staff  128  8월 18 09:54 .
drwxr-xr-x@ 4 enginrect  staff  128  8월 18 09:54 ..
-rw-r--r--@ 1 enginrect  staff  126  8월 18 09:54 .env
drwxr-xr-x@ 3 enginrect  staff   96  8월 18 09:54 src
```

- key point: Task 3, 4에서 시스템 경로에 파일을 배포하므로 sudo가 필요하다. `.env`는 뒤에서 deny 규칙과 DLP Hook의 검증 대상이 되니 실제 시크릿 대신 가짜 값을 넣는다.

### 사내 npm 미러 (Verdaccio)

> npmjs를 직접 열 수 없는 사내망을 가정하고 미러 경유로 설치. 한 번 캐시된 패키지는 npmjs가 두절돼도 배포 가능하다는 것(에어갭의 기초)을 확인

```shell
 npm install -g verdaccio
verdaccio --listen 4873 > /tmp/verdaccio.log 2>&1 &
sleep 3
npm ping --registry http://localhost:4873   # PONG이면 미러 정상

added 316 packages in 7s

68 packages are looking for funding
  run `npm fund` for details
[1] 19292
npm notice PING http://localhost:4873/
npm notice PONG 17ms

#==================================================================
# 2. 미러 경유로 Claude Code 설치
#    --registry 플래그라 npm registry 설정(~/.npmrc)은 오염되지 않는다.
#    단 "원복 불필요"는 registry 설정 얘기일 뿐, 전역 패키지 설치 자체는 남는다 → 4번에서 제거
 npm install -g @anthropic-ai/claude-code --registry http://localhost:4873
claude --version

added 2 packages in 2s
npm warn allow-scripts 1 package has install scripts not yet covered by allowScripts:
npm warn allow-scripts   @anthropic-ai/claude-code@2.1.234 (postinstall: node install.cjs)
npm warn allow-scripts
npm warn allow-scripts Run `npm install -g --allow-scripts=@anthropic-ai/claude-code` to allow these scripts once, or `npm config set allow-scripts=@anthropic-ai/claude-code --location=user` to allow them for all global installs.
2.1.220 (Claude Code)
# 설치된 건 2.1.234인데 출력은 2.1.220 → 셸 명령 해시 때문.
# 이 셸에서 이미 claude를 실행한 적이 있어 경로가 native(~/.local/bin/claude)로 캐싱된 상태였다.
# hash -r 후 재실행하면 npm 설치본(/opt/homebrew/bin/claude, 2.1.234)이 잡힌다.
#==================================================================
# 3. 캐시 확인 (오프라인 배포의 핵심 - 이 tgz가 사내 배포 자산)
# 실습랩과 달리 ~/.local/share 하위에 존재 (Verdaccio 6부터 XDG 규격을 따라 설정과 데이터 경로가 분리)
  ls -la ~/.local/share/verdaccio/storage/@anthropic-ai/claude-code/
total 3488
drwxr-xr-x@  4 enginrect  staff      128  8월 18 09:55 .
drwxr-xr-x@ 11 enginrect  staff      352  8월 18 09:55 ..
-rw-r--r--@  1 enginrect  staff    25252  8월 18 09:55 claude-code-2.1.234.tgz
-rw-r--r--@  1 enginrect  staff  1754278  8월 18 09:55 package.json
#==================================================================
# 4. 정리
 pkill -if verdaccio && echo "미러 종료됨"
미러 종료됨
[1]  + 19292 terminated  verdaccio --listen 4873 > /tmp/verdaccio.log 2>&1
# 가이드의 pkill -f verdaccio(소문자)는 매칭 실패한다. verdaccio 6이 프로세스명을 대문자
# Verdaccio로 바꾸기 때문. 실패하면 &&가 안 걸려 "미러 종료됨"도 안 찍히므로 -i가 필요하다.

# 미러 종료만으로는 부족 — npm 전역 설치가 남아 native 설치를 가린다
# (PATH에서 /opt/homebrew/bin이 ~/.local/bin보다 앞서고, claude doctor가 경고로 잡아냈다)
 npm uninstall -g @anthropic-ai/claude-code verdaccio
 hash -r
 claude --version   # 2.1.220 (native) 복귀 확인

(진행 후 기록)

 rm -rf ~/.config/verdaccio ~/.local/share/verdaccio /tmp/verdaccio.log   # 미러 데이터 101MB
```

> 프로덕션에서는 Verdaccio / Nexus / Artifactory에 `@anthropic-ai` 스코프를 고정(`'@anthropic-ai/*': proxy: npmjs`)하고,
> 클라이언트에 `npm config set registry http://npm.corp.local`을 MDM으로 일괄 배포. 버전 통제는 managed-settings의
> `requiredMinimumVersion` / `requiredMaximumVersion`으로 승인 범위를 강제한다.

- key point: 미러 설치 자체는 문제없었는데 가이드 명령 세 개가 현행 버전과 안 맞았다. ①캐시 경로 — verdaccio 6은 config를 `~/.config`, storage를 `~/.local/share`에 나눠 둔다. 가이드 경로엔 아무것도 없고 `config.yaml`의 `storage:` 값을 봐야 한다. ②`pkill -f verdaccio` — 프로세스명이 대문자 `Verdaccio`라 매칭이 안 된다. `-i`를 붙여야 하고, 실패해도 에러는 안 나고 `&&` 뒤 메시지만 안 찍힌다. ③`claude --version` — 2.1.234를 설치했는데 2.1.220이 나왔다. 셸이 명령 경로를 캐싱해서 그렇고 `hash -r`로 해결된다. 세 경우 다 명령은 에러 없이 끝났다.

### Bedrock + SSO 자격증명 패턴

> 장기 API Key 분배 대신 IAM Identity Center(SSO)의 **단기 자격증명**으로 Bedrock 호출.
> 장기 Key는 회전이 수동이고 퇴사자 처리가 누락되기 쉬우며 유출 시 폭발 반경이 크다 — SSO는 IdP에서 끄는 순간 접근이 끊긴다.

```shell
# 1. SSO 프로파일 작성 (AWS_CONFIG_FILE로 격리 → 기존 ~/.aws/config 안 건드림)
export AWS_CONFIG_FILE=~/claude-lab/ch3/aws-config

cat > "$AWS_CONFIG_FILE" << 'EOF'
[sso-session corp]
sso_start_url = https://my-org.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile claude-workshop]
sso_session = corp
sso_account_id = 111122223333
sso_role_name = ClaudeCodeWorkshop
region = ap-northeast-2
EOF

cat "$AWS_CONFIG_FILE"
#==================================================================
# 2. Permission Set에 부여할 IAM 정책 (최소 권한)
#{
#  "Version": "2012-10-17",
#  "Statement": [{
#    "Effect": "Allow",
#    "Action": [
#      "bedrock:InvokeModel",
#      "bedrock:InvokeModelWithResponseStream",
#      "bedrock:ListFoundationModels"
#    ],
#    "Resource": [
#      "arn:aws:bedrock:*::foundation-model/anthropic.*",
#      "arn:aws:bedrock:*:*:inference-profile/*"
#    ]
#  }]
#}
#   Action:   bedrock:InvokeModel
#             bedrock:InvokeModelWithResponseStream   ← 스트리밍 응답에 필수
#             bedrock:ListFoundationModels
#   Resource: arn:aws:bedrock:*::foundation-model/anthropic.*
#             arn:aws:bedrock:*:*:inference-profile/*   ← 교차 리전 추론(CRIS)용
#==================================================================
# 3. 선택 트랙 - 미진행
#    개인 사정으로 AWS 계정 발급이 어려워 aws sso login 검증은 생략.
#    1·2번의 SSO 프로파일 구조(sso-session + profile)와 IAM 최소 권한 정책은 이해 완료.
#    참고: 프로파일이 예시값 그대로라 aws sso login 시 RegisterClient 단계에서
#          InvalidRequestException / invalid_request 발생. 실제 조직의 포털 URL,
#          계정 ID, Permission Set 이름이 있어야 로그인이 성립한다.
#
# 자격증명 없이도 확인 가능한 것: 환경변수만으로 공급자 라우팅이 전환되는가 (CLAUDE_CODE_USE_BEDROCK=1 상태)
 claude doctor
# doctor 출력 요약 - 관찰 목적은 공급자 전환 여부다
Running: native (2.1.220) / Config install method: native

Multiple installations found
- npm-global at /opt/homebrew/bin/claude
- native at /Users/enginrect/.local/bin/claude

Remote Control
Remote Control is only available when using Claude via api.anthropic.com.
CLAUDE_CODE_USE_BEDROCK is set, so this session is using Amazon Bedrock — unset it to use Remote Control.
- Not connected to the Anthropic API (api.anthropic.com)

1 warning found
- Leftover npm global installation at /opt/homebrew/bin/claude
  Fix: Run: npm -g uninstall @anthropic-ai/claude-code
# → 환경변수 하나로 공급자가 Bedrock으로 바뀌고 Anthropic 경로 전용 기능(Remote Control)이 함께 꺼졌다
# → Task 1에서 미룬 npm 전역 설치 정리도 여기서 경고로 잡혔다
#==================================================================
# 4. 정리
 unset AWS_CONFIG_FILE CLAUDE_CODE_USE_BEDROCK AWS_PROFILE AWS_REGION
```

> Bedrock 경로에는 Claude 계정이 없어 **OTel 이벤트에 `user.email`이 비어 있다**.
> 사용자별 비용 추적이 필요하면 `OTEL_RESOURCE_ATTRIBUTES="enduser.id=user@corp.com"`처럼 신원 속성을 managed-settings로 부착.

- key point: AWS 계정 없이도 환경변수가 인증 우선순위 1순위라는 건 확인됐다. `CLAUDE_CODE_USE_BEDROCK=1` 하나로 doctor와 `/status`의 공급자 표시가 Bedrock으로 바뀌고 Remote Control이 꺼졌다. 4번 unset을 건너뛴 게 Task 3에서 문제가 됐다. 자격증명 없는 Bedrock 세션이라 deny 검증에서 `SSO session token was not found` 에러가 먼저 났다. 권한 규칙은 모델이 도구를 호출한 다음에 걸리는데 API 호출 자체가 실패해서 거기까지 가지 못했다.

### managed-settings로 조직 정책 강제

> 사용자가 덮어쓸 수 없는 Managed 스코프에 권한 규칙을 배포하고 실제 적용을 검증.
> 우선순위 최상단(Managed > CLI > Local > Project > User)을 확인한다

| OS | 경로 |
|---|---|
| Linux, WSL | `/etc/claude-code/managed-settings.json` (하이픈 포함) |
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Windows | `C:\Program Files\ClaudeCode\managed-settings.json` (구 `ProgramData` 경로는 v2.1.75부터 미지원) |

```shell
# 1. 정책 배포 (권한 규칙 + 시작 배너)
# 가이드 명령은 Linux 기준. macOS는 경로가 다르고 공백이 있어 따옴표 필수
 sudo mkdir -p "/Library/Application Support/ClaudeCode"
 sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json" > /dev/null << 'EOF'
{
  "permissions": {
    "allow": ["Read(**)", "Grep", "Glob", "Bash(npm test:*)"],
    "ask": ["Edit(**)", "Write(**)"],
    "deny": ["Bash(rm -rf:*)", "Read(./.env)", "Read(./.env.*)"]
  },
  "companyAnnouncements": [
    "[Corp Policy] Claude Code managed settings 적용됨. .env 읽기는 금지됩니다."
  ]
}
EOF
 echo "배포 완료"

배포 완료
#==================================================================
# 2. 적용 검증 A - 시작 배너와 설정 소스 확인
 cd ~/claude-lab/ch3
 claude
 ▎ [Corp Policy] Claude Code managed settings 적용됨. .env 읽기는 금지됩니다. # 배너 문구를 통해 적용 확인
❯ /status

Settings  Status   Config   Usage   Stats

Version:          2.1.220
Session name:     /rename to add a name
Session ID:       c42205a2-9ea6-4635-a03c-3834ebe2dbe5
cwd:              /Users/enginrect/claude-lab/ch3
API provider:     Amazon Bedrock
AWS region:       ap-northeast-2

Model:            opus[1m] (apac.anthropic.claude-opus-5[1m])
Setting sources:  User settings, Enterprise managed settings (file) # 적용 확인

System diagnostics
 ⚠ Leftover npm global installation at /opt/homebrew/bin/claude

#==================================================================
# 3. 적용 검증 B - deny 규칙이 .env 읽기를 차단
❯ .env 파일의 내용을 읽어서 보여주세요

  Read 1 file, listed 1 directory

.env 파일은 존재하지만(126 bytes), 권한 설정에서 읽기가 차단되어 있습니다.

File is in a directory that is denied by your permission settings.

# 이어진 안내 요약 (출력 축약)
#   cat 등으로 우회하지 않겠다고 밝히고, 대안 두 가지를 제시했다
#     1) 프롬프트에 ! cat .env 를 입력하면 이 세션에서 바로 실행된다  ← 셸 모드는 권한 검사를 안 거친다
#     2) deny 규칙이 있는 설정 파일을 찾아 수정
```

> 관리 설정은 **관대하게 파싱**되어 잘못된 항목만 버려지고 나머지는 적용된다.
> 배포 전 테스트 머신에서 `claude doctor`를 돌리면 무시된 항목을 소스 파일과 필드까지 알려준다 — fleet 배포 전 필수 습관.

- key point: 적용 확인은 배너와 `/status` 두 가지로 된다. `/status` 표시 형식은 가이드와 다르다. 가이드는 여러 줄에 경로까지 나오는 예시인데 2.1.220은 `User settings, Enterprise managed settings (file)` 한 줄이고, 괄호 안은 managed 4채널 중 어디로 들어왔는지를 뜻한다(remote / plist / HKLM / HKCU / file). deny는 예상대로 동작했다. `Read(**)`로 전체 읽기가 allow인데도 `.env`는 "File is in a directory that is denied by your permission settings."로 막혔다. Claude가 우회 방법으로 `! cat .env`를 제안한 건 기록해 둘 만하다. 셸 모드로 사용자가 직접 실행하면 권한 검사를 안 거친다.

### DLP Hook, 시크릿 유출 차단

> 권한 규칙이 **도구 단위** 통제라면 Hook은 **내용 기반** 통제(DLP).
> PreToolUse Hook은 **exit code 2로 도구 실행을 차단**하고 stderr를 Claude에 전달한다(exit 0은 통과, 그 외는 비차단 오류)

```shell
# 1. DLP 검사 스크립트 작성
 sudo mkdir -p /opt/claude
sudo tee /opt/claude/dlp.sh > /dev/null << 'EOF'
#!/bin/bash
# PreToolUse DLP: AWS Access Key ID 패턴을 도구 입력에서 차단
INPUT=$(cat)
if echo "$INPUT" | grep -qE 'AKIA[0-9A-Z]{16}'; then
  echo "Blocked by DLP: AWS Access Key ID detected in tool input" >&2
  exit 2
fi
exit 0
EOF
sudo chmod +x /opt/claude/dlp.sh
echo "DLP 스크립트 준비 완료"

DLP 스크립트 준비 완료
#==================================================================
# 2. managed-settings에 hooks 블록 추가해 재배포 (권한 규칙은 그대로 유지)
 sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json" > /dev/null << 'EOF'
{
  "permissions": {
    "allow": ["Read(**)", "Grep", "Glob", "Bash(npm test:*)"],
    "ask": ["Edit(**)", "Write(**)"],
    "deny": ["Bash(rm -rf:*)", "Read(./.env)", "Read(./.env.*)"]
  },
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Edit|Write",
      "hooks": [{ "type": "command", "command": "/opt/claude/dlp.sh" }]
    }]
  },
  "companyAnnouncements": [
    "[Corp Policy] DLP Hook 활성화됨. AWS 키는 코드에 쓸 수 없습니다."
  ]
}
EOF
echo "Hook 포함 재배포 완료"

Hook 포함 재배포 완료
#==================================================================
# 3. 차단 검증 (Hook은 설정 핫 리로드로 적용)
 cd ~/claude-lab/ch3 && claude

# 배너 문구 (dlp hook 활성화 확인)
 ▎ Message from ***s Organization: 
 ▎ [Corp Policy] DLP Hook 활성화됨. AWS 키는 코드에 쓸 수 없습니다.

#
❯ src/aws.js 파일을 만들고 그 안에 다음 줄을 넣어주세요: const key = "AKIAIOSFODNN7EXAMPLE";

  Listed 1 directory

⏺ Write(src/aws.js)
  ⎿  Error: PreToolUse:Write hook error: [/opt/claude/dlp.sh]: Blocked by DLP: AWS Access Key ID detected in tool input

⏺ 파일 생성이 DLP 훅에 의해 차단됐습니다.
# 이어진 안내 요약 (출력 축약)
#   문자열을 쪼개거나 인코딩해 우회하지 않겠다고 밝힘 - 조직 보안 통제를 무력화하는 행위라서
#   AKIAIOSFODNN7EXAMPLE이 AWS 공식 문서의 예제 키인 것도 짚었지만, 훅은 패턴만 보고 판단한다고 설명
#   대안 : 환경변수에서 읽도록 작성(process.env.AWS_ACCESS_KEY_ID) / DLP 정책 조정 / 셸에서 직접 작성

#==================================================================
# 4. 전체 원복 (반드시 수행 - 이 머신의 모든 세션에 계속 적용되므로)
 sudo rm -f "/Library/Application Support/ClaudeCode/managed-settings.json" /opt/claude/dlp.sh
sudo rmdir "/Library/Application Support/ClaudeCode" /opt/claude
echo "원복 완료"
원복 완료
```

- key point: Hook 차단은 정확히 동작했다. `PreToolUse:Write hook error: [/opt/claude/dlp.sh]: Blocked by DLP: AWS Access Key ID detected in tool input`. 권한 규칙은 어떤 도구를 쓰는지를 보고 Hook은 도구 입력 내용을 본다. 그래서 같은 Write라도 AKIA 패턴이 있을 때만 막혔다. Claude가 문자열을 쪼개거나 인코딩해서 우회하지 않겠다고 한 것, `AKIAIOSFODNN7EXAMPLE`이 AWS 공식 예제 키인데도 훅은 패턴만 본다고 짚은 것도 같이 기록. 원복은 실패했다. 배포는 macOS 경로로 했는데 삭제는 Linux 경로를 지웠고, `rm -f`는 없는 파일에도 성공하니 "원복 완료"만 찍히고 파일은 남았다.

### OpenTelemetry 텔레메트리 맛보기

> OTLP 수집기 없이 console exporter로 텔레메트리가 흐르는지 검증. SIEM 연동은 이 흐름의 목적지만 바꾼 것

```shell
# 1. console exporter로 메트릭 출력 (세션 밖 터미널)
 cd ~/claude-lab/ch3
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=console
export OTEL_METRIC_EXPORT_INTERVAL=5000

claude -p "src/app.js가 무엇을 하는지 한 줄로 설명해줘"
Permission ask rule (managed policy settings): Write(**) is not matched by file permission checks — only Edit(path) rules are. Use Edit(**) instead (Edit rules cover all file-editing tools).
{
  descriptor: {
    name: "claude_code.session.count",
    type: "COUNTER",
    description: "Count of CLI sessions started",
    unit: "",
    valueType: 1,
    advice: {},
  },
  dataPointType: 3,
  dataPoints: [
  ...
},
...
#==================================================================
# 2. 정리
 unset CLAUDE_CODE_ENABLE_TELEMETRY OTEL_METRICS_EXPORTER OTEL_METRIC_EXPORT_INTERVAL
```

> 실제 조직에서는 console 대신 `OTEL_LOGS_EXPORTER=otlp`와 `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`를 managed-settings의 `env`로 배포.
> 감사에는 `OTEL_LOG_TOOL_DETAILS=1`을 켜서 tool_decision, mcp_server_connection 등 보안 이벤트에 Bash 명령과 MCP 호출 상세를 포함시킨다.

- key point: 수집기 없이도 `claude_code.session.count` 같은 메트릭이 descriptor 단위로 출력된다. SIEM 연동은 exporter 목적지만 바꾸면 된다. 같이 나온 경고가 하나 있다. `Permission ask rule (managed policy settings): Write(**) is not matched by file permission checks — only Edit(path) rules are. Use Edit(**) instead.` 가이드 JSON의 `"ask"`에 있는 `Write(**)`가 무효 규칙이었다는 뜻이다. 관리 설정은 무효 항목만 버리고 나머지를 적용하는데(관대한 파싱), 그 사례를 직접 봤다. 그리고 이 경고가 뜬 것 자체가 정책이 아직 살아 있다는 뜻이라, 앞 단계 원복 실패를 여기서 알았다.

## 정리

- 원복이 실패했는데 성공 메시지가 찍혔다. 배포 경로만 macOS로 바꾸고 삭제 명령은 Linux 경로 그대로 둔 탓이다. `rm -f`는 없는 파일에도 exit 0이라 아무 티가 안 났다. 배포는 실패하면 배너나 `/status`로 바로 알 수 있는데 삭제는 확인할 방법이 없다. 지운 다음 배너가 사라졌는지 확인하는 것까지 해야 원복이 끝난다.
- Task 2에서 unset을 안 한 게 Task 3 검증을 막았다. 권한 규칙은 모델이 도구를 호출한 뒤에 걸리니, 그 앞의 인증이 깨지면 정책이 맞는지 틀린지 알 수가 없다. 정책이 안 먹는 것 같으면 정책 파일보다 앞단을 먼저 봐야 한다.
- 권한 규칙과 Hook은 검사 대상이 다르다. 권한은 도구와 경로, Hook은 도구 입력 내용. 그리고 둘 다 `!` 셸 모드로 사용자가 직접 실행하면 안 걸린다.
- 가이드대로 쳤는데 안 되는 곳이 많았다. verdaccio 경로, pkill 대소문자, `/status` 표시 형식, `Write(**)` 무효 규칙. 버전이 올라가며 달라진 것들이라 배포 전 `claude doctor` 검증이 필요하다.

## References

- 실습 가이드(정본): [whchoi98.github.io/ccw-hands-on-lab — Ch3 Hands-on Lab](https://whchoi98.github.io/ccw-hands-on-lab/ClaudeCode_Ch3_HandsOnLab.html)
- 워크숍 저장소(슬라이드 PDF·코드 스니펫): github.com/whchoi98/claude-code-workshop
- 공식 문서: code.claude.com/docs/ko/admin-setup, monitoring-usage
