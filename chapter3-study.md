# [3주차] Claude Code Deep Dive Workshop — Chapter 3 스터디 노트

> - 스터디: Claude Code Deep Dive Workshop 모각코 3주차 (8/10 ~ 8/16)
> - 자료: [whchoi98/claude-code-workshop](https://github.com/whchoi98/claude-code-workshop) — Chapter 3 (112 slides, 9 Parts + Labs 3개)
> - 방식: 실습 중심. 각 파트 핵심만 요약하고, 랩은 내 터미널 세션 로그로 기록한다.

---

## 개요 — 개인의 도구를 조직의 플랫폼으로

- Chapter 1~2가 **개발자의 세계**(원라이너 설치, 개인 구독 로그인, 내 settings.json, 비용은 내 플랜 한도)였다면, Chapter 3는 **관리자의 질문**에 답한다 — 500명에게 어떻게 배포하고 갱신하나 / 키를 나눠주지 않고 인증하려면 / 금지 명령과 도메인을 어떻게 강제하나 / 누가 얼마나 쓰는지, 감사는 어떻게.
- 네 개의 키워드로 요약되는 챕터: **Scale**(수백 명 규모) / **No keys**(키 미배포 원칙) / **Enforce**(정책은 강제로) / **Audit**(증적 가능한 운영).
- 전 파트를 관통하는 검증 도구는 **`/status` 한 줄** — 공급자, 모델, managed 정책 소스, 샌드박스 활성 여부가 모두 여기서 확인된다.

## Part별 핵심 정리

### Part 1. 배포 전략 — 500대에 설치하고 갱신하는 법

- **설치 채널 4계열(조직 관점)**: Native 스크립트(단일 바이너리 + 자동 갱신, **표준 권장**, MDM 스크립트화) / OS 패키지 저장소(apt·dnf·apk 공식, GPG 서명 — 리눅스 서버 플릿) / 패키지 매니저(Homebrew cask, WinGet — 개발자 셀프서비스) / 컨테이너(표준 이미지, devcontainer — CI 러너, 통제 환경). **npm은 레거시**로 신규 배포 비권장.
- **Linux 플릿 표준 절차**: `/etc/apt/keyrings`에 GPG 키 설치 → **지문 대조**(`31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE`) → `signed-by=` 옵션으로 sources.list 등록 → `apt install claude-code`. 신뢰하기 전에 지문을 반드시 대조하고, 절차는 Ansible로 코드화한다.
- **사내 미러**(Artifactory, Nexus): 외부 다운로드가 막힌 망·대역폭·감사 요구가 있는 조직은 공식 저장소를 remote repo로 상류 캐시. 클라이언트의 sources.list와 설치 스크립트 URL을 사내 주소로 치환하되 **GPG 검증은 그대로 유지**(미러도 서명 파일을 함께 제공).
- **Windows 대량 배포**: `winget install Anthropic.ClaudeCode --silent` 를 Intune·SCCM 스크립트로. 관리 정책은 `HKLM\SOFTWARE\Policies\ClaudeCode`(관리자 권한 필요 = 변조 저항). WSL 병행 조직은 **`wslInheritsWindowsSettings: true`로 Windows 정책을 WSL까지 상속**시켜 정책을 일원화.
- **표준 컨테이너 이미지 3원칙**: 버전 고정(`install.sh | bash -s 2.1.201`) + 자동갱신 차단(`DISABLE_AUTOUPDATER=1`) + 정책 내장(`COPY managed-settings.json /etc/claude-code/`). 이미지 불변성이 재현 가능성을 만든다.
- **버전 통제 두 다이얼**: `autoUpdatesChannel`로 갱신 속도(**stable이 플릿 기본**, latest는 얼리어답터 링만), `minimumVersion`으로 보안 픽스 하한선(미달 버전은 강제 갱신 유도). 둘 다 managed settings에 넣어 개인이 못 바꾸게 한다. 완전 고정(`DISABLE_AUTOUPDATER=1` + 버전 지정 설치)은 **CI 전용**.
- **점진 롤아웃 4링**: Ring 0 챔피언 10명(latest 채널, 피드백 루프) → Ring 1 파일럿 팀 50(stable, managed 정책 초판) → Ring 2 부문 200(교육 자료, 지표 관측) → Ring 3 전사(표준 온보딩, 지원 체계). 정책과 버전은 링별로 독립 조정.
- **에어갭 현실**: 설치는 미러로 완전 내재화 가능하지만 **추론 트래픽은 모델 엔드포인트가 필요** → Bedrock + PrivateLink 등 클라우드 프라이빗 연결이 현실적인 답. 웹 검색 등 외부 의존 기능은 도메인 정책으로 차단을 명시.
- **멀티 플랫폼 표준 조합**: macOS(Native + MDM, plist 정책 — Jamf/Kandji) / Windows(WinGet + Intune, HKLM + WSL 상속) / Linux(apt·dnf + `/etc/claude-code` 정책, Ansible) / 컨테이너(표준 이미지, 정책 내장, 갱신 차단). **검증은 전 플랫폼 `/status`로 정책 소스 확인**.
- **용량 예측**: 좌석은 활성 개발자 기준(파일럿 실측으로 배수 보정), 토큰은 1인 1일 세션 수와 평균 규모를 파일럿에서 실측, **opus 비중이 비용의 지배 변수**, 클라우드 경로는 리전 쿼터와 스로틀 한도를 사전 확인.

### Part 2. 공급자와 자격증명 — 키를 나눠주지 않는 인증 설계

- **공급자 결정표**: Teams/Enterprise(좌석제, 인프라 불필요 — **기본 권장**) / Claude Console(API 우선, 종량 과금 — 파이프라인 중심) / Amazon Bedrock(AWS 컴플라이언스·과금 상속) / Google Vertex AI(GCP 통제 상속) / Microsoft Foundry(Azure 통제 상속) / 혼합(LLM gateway로 단일 엔드포인트 — 중앙 로깅 요구 시).
- **인증 우선순위(충돌 시 위가 이김)**: ①`CLAUDE_CODE_USE_BEDROCK`/`USE_VERTEX`/`USE_FOUNDRY` 강제 스위치 ②`ANTHROPIC_API_KEY` 환경변수 ③`apiKeyHelper`(볼트 연동 지점) ④OAuth 토큰(setup-token, claude.ai 세션).
- **Bedrock 조직 표준 구성**: `/etc/profile.d/claude.sh`에 `CLAUDE_CODE_USE_BEDROCK=1` + `AWS_REGION`. 모델 alias는 리전 가용성에 맞춰 해석되며, 필요 시 `ANTHROPIC_MODEL`로 인퍼런스 프로파일 ID 고정. **IAM 최소 권한**은 `bedrock:InvokeModel` + `InvokeModelWithResponseStream` 2종에 리소스를 `foundation-model/anthropic.*`로 스코프.
- **Bedrock + Identity Center SSO 흐름**: `aws sso login` 브라우저 인증 → 그룹을 Permission Set(IAM 정책)에 매핑 → STS 단기 토큰 자동 발급·갱신 → Claude Code가 SDK 체인으로 사용. **퇴사자는 IdP에서 끊는 순간 접근이 사라진다**.
- **Claude Platform on AWS(네 번째 경로)**: 엔드포인트는 Anthropic이 운영하되 인증은 AWS IAM, 과금은 Marketplace로 AWS 청구서에 통합. **최신 기능 속도와 AWS 거버넌스를 동시에** 원할 때의 선택지(정적 키 불필요).
- **정적 키 분배 금지**: 커밋·로그·채팅으로 유출 경로가 다수이고, 퇴사 시 회수 불가로 전수 회전이 필요하며, 누가 썼는지 귀속이 안 되어 감사 증적에 공백이 생긴다. → **사람은 SSO 단기 토큰, 기계는 Secrets Manager + apiKeyHelper, CI는 OIDC 페더레이션**(setup-token 최소화), 환경변수 하드코딩은 전면 금지.
- **apiKeyHelper + 볼트**: settings.json에 `apiKeyHelper` 경로와 `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`(예: 300000 = 5분)를 지정하면 헬퍼 스크립트가 `aws secretsmanager get-secret-value`로 키를 동적 공급 → **TTL마다 재호출되므로 회전이 클라이언트에 자동 반영**(무중단).
- **회전 4단계**: 이중화(신 키 발급, 볼트 병행 저장) → 전환(볼트 포인터 스위치) → 관찰(TTL 경과 후 구 키 호출 0 확인) → 폐기(비활성화, 감사 기록). **SSO 경로는 회전 자체가 불필요**하며 이 절차는 헬퍼 기반 키에만 해당.
- **PrivateLink / VPC Endpoint**: `bedrock-runtime` 인터페이스 엔드포인트 생성 + **Private DNS 활성으로 SDK 무수정 전환**. 엔드포인트 정책으로 호출 가능 모델·주체를 제한하고, VPC Flow Logs로 경로 증적을 남긴다.
- **주체별 표준 경로**: 개발자(Bedrock + Identity Center SSO — 정적 키 0) / CI(OIDC 역할 인수) / 공유 서비스(Secrets Manager + apiKeyHelper, TTL 회전) / 폐쇄망(PrivateLink + SSO) / 예외 승인(기간 한정, 사유 기록, 자동 만료 + 감사 대장).

### Part 3. Claude apps gateway — 자체 호스팅 SSO 게이트웨이

- **왜 필요한가**: Bedrock 직결은 인증과 과금은 상속하지만 **개발자별 실시간 지출 한도, IdP 그룹별 모델 접근 차등, 조직 표준 로그인 UX, 요청 단위 중앙 텔레메트리**가 비어 있었다. 게이트웨이가 이 네 조각을 채운다.
- **아키텍처 5층**: Listener + TLS(조직 도메인) → OIDC + Session(IdP 사인인, 세션은 Postgres) → Policy(managed 정책, 그룹 매핑, 지출 한도 판정) → Model Routing(그룹 규칙에 따라 허용·치환) → Upstream(Bedrock, Agent Platform, Foundry 전달).
- **개발자 경험**: 클라이언트 설정은 `ANTHROPIC_BASE_URL=https://claude-gw.corp.example` **하나뿐**. `claude` 실행 시 브라우저가 열려 조직 IdP(OIDC) 로그인 → 이후 상류 인증은 게이트웨이가 대행. **AWS 프로파일, 정적 키, 리전 설정이 전부 사라진다**.
- **그룹별 모델 라우팅**: `gateway.yaml`의 `routing.groups`에서 IdP 그룹 클레임 기준으로 `allowedModels` 차등(예: eng-platform은 opus·sonnet·haiku, eng-default는 sonnet·haiku + `rewrite: {opus: sonnet}`으로 상위 요청 하향 치환, contractors는 haiku만). 위반 요청은 거부 또는 치환.
- **지출 한도 Admin API**: `PUT /admin/limits`로 `{subject, period(day/week/month), usd}` 설정 → 게이트웨이가 요청마다 누적 지출을 판정하고 한도 도달 시 **사유와 함께 요청 거부**. 그룹 기본값 + 개인 예외의 2단 구성 가능.
- **배포와 운영**: 컨테이너 하나 + Postgres가 전부(K8s, Cloud Run 표준 경로). IdP에 OIDC 클라이언트·리다이렉트 URI 사전 등록, 헬스체크·시크릿 회전·무중단 업그레이드 절차는 공식 문서 제공. **무상태 수평 확장**(세션은 Postgres 담당).
- **OTLP 텔레메트리**: 전 요청의 메트릭·트레이스를 표준 송출하며 차원은 사용자·그룹·모델·토큰·**판정 결과 태그**(한도, 라우팅). CloudWatch·Datadog 등 OTel 호환 어디로든 보내고 Part 6의 클라이언트 OTel과 상호 보완.
- **일반 LLM gateway(LiteLLM 등)와의 차이**: 범용 프록시는 다중 벤더 추상화가 목적이라 Claude Code 정책 모델을 인지하지 못하고 한도·라우팅을 직접 구현해야 한다. apps gateway는 **Claude Code 전용 설계 완제품**으로 managed 정책과 자연 결합하고 공식 배포·운영 문서가 따라온다.

### Part 4. 네트워크와 보안 — 사내망에서 안전하게 통과시키기

- **필수 아웃바운드 도메인**(방화벽 화이트리스트 기준선): `api.anthropic.com`(추론 API — Bedrock 경로는 불필요), `claude.ai`(구독 로그인, **서버 관리 설정 수신**), `downloads.claude.ai`(설치 자산·자동 갱신 — 미러 운영 시 대체), statsig 계열(기능 플래그, 차단 시 기본값 동작), sentry 계열(오류 리포트, 차단 가능하나 진단 저하), 클라우드 엔드포인트(`bedrock-runtime.<region>.amazonaws.com` — PrivateLink로 대체 가능).
- **프록시 3변수**: `HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`(사내 MCP·게이트웨이 도메인 포함). `/etc/profile.d` 또는 MDM 프로파일로 조직 배포. 실패 시 `curl -v https://api.anthropic.com/`로 **프록시 계층과 TLS 계층을 분리 진단**.
- **인증 프록시**: `http://user:pass@proxy` **URL 삽입은 지양**(평문 자격증명이 환경변수에 남음). Px·Cntlm 등 **로컬 중계**를 세우고 `HTTPS_PROXY`는 localhost 중계 포트만 가리키게 한다. 구조적 대안은 ZTNA로 인증 프록시 자체를 대체.
- **사내 CA**: `SELF_SIGNED_CERT_IN_CHAIN` 증상 → `NODE_EXTRA_CA_CERTS=/etc/ssl/corp/corp-root-ca.pem`. curl 등 다른 도구를 위해 OS 신뢰 저장소에도 병행 등록(Debian `update-ca-certificates`, RHEL `update-ca-trust`). **`NODE_TLS_REJECT_UNAUTHORIZED=0`은 검증 무력화이므로 금지**.
- **sandbox 네트워크로 curl 갭 봉쇄**: WebFetch를 deny해도 Bash가 허용이면 curl·wget으로 임의 URL 접근이 가능하다 — **권한 규칙이 못 막는 층을 OS가 막는다**. managed-settings의 `sandbox.enabled: true` + `network.allowedDomains`로 아웃바운드 허용 목록을 강제(공식 admin-setup 권고).
- **VPN/ZTNA 점검**: 스플릿 터널 시 API 도메인 경로 확인, MTU 문제로 인한 스트리밍 끊김, DNS의 사내 리졸버 경유 여부, 터널 밖 예외와 `NO_PROXY` 정합성 / ZTNA는 앱 단위 정책 등록, 커넥터 정책의 도메인 허용, 디바이스 포스처 요건과 CLI 호환.
- **DLP 훅**: `PreToolUse`에서 명령을 검사해 **외부 전송 명령(curl·wget·nc·scp) + 민감 패턴(`sk-ant-`, `AKIA[0-9A-Z]{16}`, `BEGIN...PRIVATE KEY`)이 결합될 때** exit 2로 차단하고 사유를 회신. managed 배포로 조직 강제.
- **사내 MCP 통합**: 네트워크 준비(`NO_PROXY`와 sandbox `allowedDomains`에 내부 도메인 등록) + 정책 통제(`allowedMcpServers`로 승인 서버만) 두 층이 필요. 인증은 OAuth 지원 서버 권장, 노출은 서브에이전트 `mcpServers`로 필요한 곳에만 최소화.
- **네트워크 진단표**: 연결 타임아웃 → 프록시 변수 누락·방화벽(`curl -v` 계층 분리) / TLS 체인 오류 → 사내 CA 미등록 / 407 → 프록시 인증 실패(로컬 중계) / 스트리밍 끊김 → MTU·검사 장비 버퍼링 / 일부 기능만 실패 → 도메인 부분 차단 / 사내 MCP 불통 → `NO_PROXY`·sandbox **양쪽 목록 동시 확인**.

### Part 5. 거버넌스와 정책 — 개인이 못 바꾸는 것을 설계한다

- **managed settings 4채널(우선순위순)**: ①**Server-managed**(claude.ai 어드민 콘솔 배포, 전 플랫폼, Teams/Enterprise 전용) ②**plist / HKLM**(`com.anthropic.claudecode`, macOS·Windows, 변조 저항) ③**파일 기반**(`/etc/claude-code/managed-settings.json`, 전 플랫폼, 형상 관리 배포) ④**HKCU**(사용자 레지스트리, 권한 상승 불필요, Windows 편의용 = 비강제). **기기에서 처음 발견되는 채널 하나만 사용**되고 `/status`가 소스를 표기한다.
- **Server-managed 상세**: 인증 시점에 기기로 내려오고 활성 세션 중 **매시간 갱신**. MDM·파일 배포 파이프라인 없이 콘솔에서 완결되지만 Teams/Enterprise 전용이므로, **Bedrock 등 타 공급자 사용자는 미수신** → 혼합 조직은 파일·plist 폴백을 함께 배포.
- **조직 기준선 정책 한 장**(managed-settings.json): `permissions.deny`(`Read(./.env*)`, `Bash(rm -rf:*)`, `Bash(curl * | bash:*)`) + `disableBypassPermissionsMode: "disable"` + `sandbox`(도메인 통제) + `allowedMcpServers` + `minimumVersion` + `env`로 공급자 스위치 강제.
- **병합 규칙**: **스칼라는 덮어쓰기**(model, minimumVersion 등 — managed > 사용자 > 프로젝트, 개인 설정은 조용히 무시되므로 배포 공지 필요), **배열은 병합**(`permissions.allow`·`deny`는 전 소스 합집합 — 개발자가 자기 allow를 추가할 수는 있어도 **managed deny는 누구도 제거 불가**).
- **잠금 키 6종**: `allowManagedPermissionRulesOnly`(개인 allow 무시, 고통제 환경) / `disableBypassPermissionsMode`(`--dangerously-skip` 봉쇄 — **전사 기본 권장**) / `allowManagedHooksOnly`(훅 주입 차단) / `allowedHttpHookUrls`(웹훅 유출 통제) / `strictKnownMarketplaces`·`blockedMarketplaces`(플러그인 공급망 방어) / `minimumVersion`(보안 픽스 하한).
- **MCP·마켓플레이스 통제**: `allowedMcpServers`(승인 목록) / `deniedMcpServers`(사고 대응 시 즉시 배포) / `allowManagedMcpServersOnly`(최고 수준) / `strictKnownMarketplaces`·`blockedMarketplaces`. **v2.1.153+부터 서브에이전트 인라인 정의에도 일관 강제**된다.
- **조직 CLAUDE.md**: 관리 정책 경로에 둔 CLAUDE.md는 전 세션에 로드되고 **사용자가 제외할 수 없다**. 개인·프로젝트 메모리보다 앞서 로드되므로 보안 수칙·금지 관행·사내 표준 링크 같은 **행동 지침 채널**로 쓰되, 긴 문서 전문이나 프로젝트별 세부는 각 계층으로 내리고 컨텍스트 예산을 위해 간결하게 유지.
- **auto-mode 조직 구성**: `autoMode`에 `trustedRepositories`·`trustedBuckets`·`trustedDomains`(신뢰 3종)와 `blockOverrides`·`allowOverrides`로 기본 규칙을 가감. **`claude auto-mode show`로 유효 구성 확인, `claude auto-mode test "aws iam create-user x"`로 배포 전에 분류기 판정을 시뮬레이션**.
- **권한 패턴 설계 전략 — deny는 좁고 단단하게, allow는 넓고 명시적으로**: managed deny에는 논쟁 없는 위험만 최소로 고정(파괴 명령, 자격증명 파일 읽기, `curl | bash` 파이프 실행, IAM 변조), 빌드·테스트·린트와 git 조회 계열은 넉넉히 allow, push·배포는 ask로 확인 유지, 나머지 회색 지대는 auto 분류기 + 신뢰 경계에 위임.
- **감사 훅**: `allowManagedHooksOnly: true`와 함께 `PreToolUse` matcher `Bash|Write|Edit`에 audit 스크립트를 걸어 **누가·언제·어떤 도구로·무엇을(4필드)** 사내 로그로 송신. **`exit 0`을 유지해 기록만 하고 차단하지 않는 관찰 전용** 원칙.
- **적용 검증**: `/status` 출력의 `Enterprise managed settings (file)` 한 줄 — 괄호 안 소스가 `remote | plist | HKLM | HKCU | file` 중 무엇인지 확인. Provider(env 강제 반영), Model, Sandbox 활성도 함께 보고, 파일럿 기기 전수에서 소스 표기 스크린샷을 수집한다.
- **Policy as Code**: 정책 저장소에 변경 PR(사유 기록) → `auto-mode test`·스테이징 검증 → 링별 배포 → 차단 로그·문의량 관측 → 롤백 태그 유지하며 전사 확정. **정책 저장소 + PR 리뷰 + 링 배포**가 공식.

### Part 6. 모니터링과 비용 — 쓰임을 보고, 비용을 귀속시키기

- **관측 3축**: Usage monitoring(OTel로 세션·도구·토큰 송출 — **전 공급자 지원**) / Analytics 대시보드(`claude.ai/analytics/claude-code`, 사용자별 지표와 기여 추적 — Anthropic 경로 전용) / Cost tracking(지출 한도, 사용 귀속 — Anthropic 경로 전용). 클라우드 경로는 Cost Explorer·GCP Billing·Azure CM의 과금 데이터를 직접 활용.
- **OTel 전사 계측**: managed-settings의 `env`로 배포 — `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER=otlp`, `OTEL_EXPORTER_OTLP_PROTOCOL=grpc`, `OTEL_EXPORTER_OTLP_ENDPOINT`(사내 Collector), **`OTEL_RESOURCE_ATTRIBUTES=department=platform,team=payments`로 부서 태깅**.
- **메트릭 카탈로그**: 세션(수, 활성 시간 — 채택률) / 토큰(입력·출력·캐시 읽기·생성 — 비용 근사 원천) / 비용(추정 카운터) / 도구(호출 수, **승인·거부 = 정책 마찰 신호**) / 코드 변화(수정 라인, 커밋, PR — 기여 추적) / 이벤트(프롬프트 제출 등 — SIEM 연계 소스).
- **SIEM 연계**: Collector에서 Splunk·OpenSearch로 분기 → Part 5 감사 훅 로그와 사용자 키로 조인 → 탐지 규칙 3종(**거부 급증, 심야 대량 토큰, 민감 경로 접근**) → 티켓 자동 발부와 `deniedMcpServers` 긴급 배포로 대응 연결.
- **클라우드 비용 가시화**: Cost Explorer에서 Bedrock 서비스·모델 차원 필터, **애플리케이션 인퍼런스 프로파일에 비용 태그 부여**, 태그·계정 분리로 차지백 리포트, 일별 그래프로 추세와 급변 감시.
- **사용자별 비용 귀속**: OTel의 `user.id`·부서 리소스 속성 + **SSO 역할 세션 이름이 CloudTrail에 남는 사용자 단서**를 결합. apps gateway 경유라면 사용자·그룹 태그가 기본 제공되어 부서·팀·개인 3단 리포트가 자동 생성.
- **예산 알람**: `aws budgets create-budget`로 Bedrock 서비스 한정 월 예산 설정 + **ACTUAL 80% 문턱 SNS 알림**, FORECASTED 예측 경보 병행.
- **비용 최적화 6손잡이**: ①모델 배분(탐색 haiku, 본대 sonnet, opus 선별) ②캐시 활용(프롬프트 캐시 적중, fork의 캐시 공유) ③컨텍스트 위생(`/clear` 습관, compact 임계 조정) ④서브에이전트로 고볼륨 출력 격리 ⑤무인 작업 야간 예약으로 피크 회피 ⑥그룹 기본 + 개인 예외의 지출 캡.
- **이상 사용 감지 4신호**: 개인 일 사용량이 **이동평균 3배 초과** / 비업무 시간대 지속 호출 / 정책 거부 이벤트 단기 폭증 / 허용 밖 도메인·MCP 시도 반복 → 자동 티켓 연결.
- **분기 운영 리뷰 5축**: 채택(활성 사용자·세션 추세 → 링 확산 판단), 비용(부서 절단·모델 믹스 → 배분 조정), 정책(거부 상위 규칙·문의량 → 완화/강화), 안전(이상 신호·DLP 적중 → 탐지 규칙 갱신), 성과(기여 지표·만족도 → 경영 보고).

### Part 7. 신원, 데이터, 컴플라이언스 — 감사관의 질문에 답하는 체계

- **신원 통합의 두 레벨(가장 혼동이 잦은 지점)**: **Claude 계정 레벨**(SSO·SCIM 프로비저닝·좌석 배정, claude.ai 콘솔, Teams/Enterprise 대상)과 **클라우드 IAM 레벨**(Bedrock 호출 권한과 그룹 매핑, Identity Center·Workforce ID·Entra, **Permission Set이 실권한**)은 별개다. 게이트웨이 경유 시 OIDC가 이 층을 흡수.
- **IdP 연동 지도**: IAM Identity Center(AWS 표준, Bedrock 경로 1순위) / Okta(SAML·OIDC, Identity Center 상류로도 — 혼합 조직 허브) / Microsoft Entra ID(Foundry 자연 결합, SCIM) / Google Workspace(Workforce Identity). **신규는 OIDC 권장**(게이트웨이는 OIDC), SAML은 기존 호환용.
- **그룹 매핑 전략 — 단일 원천**: IdP 그룹 하나를 원천으로 클라우드 층은 Permission Set(`idp:eng-platform → PS-Claude-Full`), 게이트웨이 층은 동일 그룹명으로 라우팅 규칙(opus 접근·지출 한도 차등). 겸직은 상위 그룹 우선. **그룹 변경이 곧 권한 변경**.
- **오프보딩 설계**: IdP 계정 비활성이 곧 신규 인증 불가(**원천 절단은 단일 지점**). 남는 것은 세션 수명과 잔여 토큰의 시간창뿐이므로 SSO 세션·STS 만료를 업무 리듬에 맞게 단축하고, setup-token 등 장기 토큰은 **발급 대장 기반으로 회수**하며, 비활성 시각과 마지막 호출 시각의 대조 리포트를 증적으로 남긴다.
- **데이터 정책**(감사관의 첫 질문): 학습 사용 — Team·Enterprise·API·클라우드 경로 **미사용** / 보존 — 경로별 retention, 공급자 정책 상속 / **ZDR** — 요청 완료 후 무저장, Enterprise 제공 / 요청 단위 감사 — 필요 시 LLM gateway로 중앙 로깅 / 로컬 데이터 — 세션 트랜스크립트 **30일 기본 정리**(`cleanupPeriodDays`).
- **CloudTrail 통합**: `InvokeModel`은 **데이터 이벤트라 트레일에 명시 활성이 필요**하다(`put-event-selectors`로 `eventCategory=Data`, `resources.type=AWS::Bedrock::Model`). 기록되는 것은 시각·주체(역할 세션)·모델 ARN·소스 IP.
- **GDPR 4질문**: 처리 근거(업무 도구 사용의 적법 근거와 고지 문서화) / 이전 경로(리전 선택과 데이터 경로를 계약 문서와 정렬) / 보존·삭제(ZDR 또는 retention, 로컬 정리 주기 명시) / 주체 권리(로그 내 개인 식별자 대응 절차).
- **SOC 2 / ISO 27001 매핑표**: 접근 통제 ← SSO·Permission Set·그룹 매핑(Part 2,7) / 변경 관리 ← Policy as Code·PR 리뷰·링 배포(Part 5) / 로깅·감시 ← CloudTrail·OTel·감사 훅·SIEM(Part 5,6,7) / 공급망 ← GPG 서명·마켓플레이스 통제(Part 1,5) / 데이터 보호 ← 학습 미사용·ZDR·암호화 전송(Part 7) / 가용성 ← 게이트웨이 수평 확장·버전 하한(Part 1,3).
- **로그 보존**: CloudTrail(S3 장기, 불변 저장 옵션) / OTel 메트릭(원시 90일 + 집계 장기) / 감사 훅 로그(SIEM 인덱스, 접근 통제 필수) / 로컬 트랜스크립트(`cleanupPeriodDays` 기본 30일) / 게이트웨이 로그(요청 원장과 한도 판정 이력, Postgres 백업 정책).
- **감사 증적 패키지(상시 갱신)**: ①정책 원본(managed-settings 이력, PR 기록, 배포 태그) ②적용 증명(`/status` 소스 표기 수집본, 링별 검증 기록) ③행위 원장(CloudTrail·감사 훅·SIEM 조회 절차서) ④데이터 근거(학습 미사용·ZDR·retention 공식 문서 스냅샷). **감사 통지 후 모으기 시작하면 늦다** — 분기 리뷰와 동기화.

### Part 8. 트러블슈팅 — 관리자에게 오는 문의의 지도

- **인증 진단표**: 로그인 루프·계정 혼선 → **`/logout` 후 `/login` 재시도(공식 1순위 처방)** / Enterprise 옵션 안 보임 → `claude update` 후 터미널 재시작(구버전 증상) / 조직 미배정 메시지 → 좌석에 Code 권한 미포함, 어드민 콘솔에서 갱신 / Bedrock 자격 실패 → `aws sso login` 만료·프로파일 확인(`sts get-caller-identity`) / 헬퍼 키 오류 → 볼트 권한·TTL 캐시, 스크립트 단독 실행 검증.
- **네트워크 진단표**(Part 4 요약판): 타임아웃 → 프록시 3변수·방화벽 / TLS 오류 → 사내 CA / 407 → 로컬 중계 / 부분 기능 실패 → 도메인 부분 차단 / 샌드박스 차단 → `allowedDomains` 누락.
- **성능 문제 4갈래 분해**: 경로 지연(프록시·검사 장비 홉 측정, 리전 지연) / 모델 선택(작업 대비 과사양) / 컨텍스트 비대(`/context` 확인과 `/clear` 안내) / 쿼터 스로틀(리전 TPM 한도, **429 비율 관측** → 지속 시 쿼터 증설).
- **진단 수집 표준 세트**: `claude doctor`(1차 자가 진단, `f` 키 자동 수정) + `claude --version` + `/status`(구성 스냅샷) + `/context` + `claude --verbose 2> claude-debug.log`(재현 로그) + `curl -v https://api.anthropic.com/`, 여기에 시각·사용자·저장소 경로를 함께 접수. **접수 양식을 고정하는 것이 속도의 핵심**.
- **에스컬레이션 3선**: 1선 헬프데스크(진단표 매칭, 계정·좌석) → 2선 플랫폼팀(정책, 네트워크, 게이트웨이) → 3선(AWS Support는 Bedrock 쿼터·서비스 이슈, Anthropic은 제품 결함·문서 괴리). **진단 번들이 각 선을 잇는 공용어**.

### Part 9. Recap

- **여섯 문장 요약**: ①배포 — 플랫폼별 표준 조합, stable + minimumVersion, 링 롤아웃 ②자격증명 — 사람은 SSO, 기계는 볼트 헬퍼, CI는 OIDC, 키 배포 0 ③게이트웨이 — SSO 로그인, 그룹 모델 라우팅, 지출 한도, OTLP ④네트워크 — 프록시·CA로 열고 sandbox 도메인과 DLP로 좁힘 ⑤거버넌스 — managed 4채널, 배열 병합, 잠금 키, Policy as Code ⑥관측 — OTel 전사 계측, 귀속 태그, 예산 알람, 분기 리뷰.
- **FAQ 6선**: 개인이 정책을 우회할 수 있나 → **managed deny는 제거 불가, bypass 차단** / Bedrock인데 서버 관리 설정이 되나 → 좌석제 전용이라 파일 채널 폴백 / 개인별 지출 캡 → apps gateway 한도 API / 코드가 학습에 쓰이나 → 상업 경로 미사용, ZDR 옵션 / curl 우회 차단 → sandbox `allowedDomains` / 적용 확인 → `/status` 소스 표기.
- **자가 점검 4문항**: ①플랫폼 조합과 버전 통제를 설명할 수 있다(Part 1) ②SSO 흐름과 헬퍼·게이트웨이를 구성할 수 있다(Part 2, 3) ③managed 채널·잠금 키·병합 규칙으로 정책을 강제할 수 있다(Part 4, 5) ④OTel 배포와 귀속, 증적 패키지를 갖췄다(Part 6, 7).
- **실행 로드맵 30/60/90**: Day 0–30 기반(공급자 확정, 파일럿 링, 정책 초판) → Day 31–60 통제(managed 전 채널, 샌드박스, OTel) → Day 61–90 규모화(전사 링, 게이트웨이, 분기 리뷰 1회차). **각 구간 말미에 `/status` 전수 검증과 경영 보고를 고정 이벤트로**.
- **다음 챕터 예고**: Chapter 4 — Settings. 설정 파일 체계, 권한 문법, MCP, 명령어 등 5가지 축의 상세.
