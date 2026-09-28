# Temporal `ui-server` 전수조사 & 활용 전략 정리

> 이 문서는 `bmshin94/ui-server` 레포지토리(= `temporalio/ui-server` 포크)를 전수조사한 결과와,
> 설치/사용법 · 기술 분류 · 로컬 AI 에이전트 활용 · 수익화 전략까지 정리한 기록입니다.
>
> - 작성일: 2026-09-28
> - 작성 브랜치: `claude/jolly-dijkstra-apr2a0`
> - 분석 범위: 파일 512개, Go 코드 약 7,300줄, 임베드 UI 에셋 5.9MB

---

## 📌 목차

1. [관련 GitHub 주소](#1-관련-github-주소)
2. [이 레포는 무엇인가](#2-이-레포는-무엇인가)
3. [폴더별 전수조사 결과](#3-폴더별-전수조사-결과)
4. [설정 옵션 전체 정리](#4-설정-옵션-전체-정리)
5. [동작 흐름 (요청 라이프사이클)](#5-동작-흐름-요청-라이프사이클)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [기술 분류: 플러그인 / 스킬 / MCP 아님](#7-기술-분류-플러그인--스킬--mcp-아님)
8. [API 토큰이 필요한가](#8-api-토큰이-필요한가)
9. [왜 GitHub에서 유명한가](#9-왜-github에서-유명한가)
10. [로컬 AI 에이전트 구축에 도움이 되는가](#10-로컬-ai-에이전트-구축에-도움이-되는가)
11. [React / PHP로 만들 수 있는가](#11-react--php로-만들-수-있는가)
12. [수익화 아이디어 7선](#12-수익화-아이디어-7선)
13. [추천 실행 전략 (3단계)](#13-추천-실행-전략-3단계)
14. [라이선스 및 법적 체크리스트](#14-라이선스-및-법적-체크리스트)
15. [코드에서 배울 만한 패턴 Top 3](#15-코드에서-배울-만한-패턴-top-3)

---

## 1. 관련 GitHub 주소

| 구분 | 주소 | 설명 |
|---|---|---|
| **현재 작업 레포 (포크)** | https://github.com/bmshin94/ui-server | 지금 분석 중인 레포 |
| **원본 (배포용 미러)** | https://github.com/temporalio/ui-server | Docker/바이너리 릴리즈 대상 |
| **개발 원본 ⭐** | https://github.com/temporalio/ui | 실제 개발은 여기서 (`server/` 폴더가 위로 동기화됨) |
| Temporal 엔진 본체 | https://github.com/temporalio/temporal | 워크플로 오케스트레이션 엔진 |
| docker-compose 세트 | https://github.com/temporalio/docker-compose | Temporal + DB + UI 한번에 |
| Temporal PHP SDK | https://github.com/temporalio/sdk-php | PHP로 Workflow/Activity 작성 |
| Docker Hub 이미지 | https://hub.docker.com/r/temporalio/ui | `temporalio/ui:latest` |
| Temporal API proto | https://github.com/temporalio/api | gRPC 서비스 정의 |
| 공식 문서 (설정) | https://docs.temporal.io/references/web-ui-configuration | 환경변수/설정 레퍼런스 |
| Air (핫리로드) | https://github.com/air-verse/air | 개발 서버 핫리로드 |

> ⚠️ **중요**: 이 레포는 `temporalio/ui`의 `server/` 디렉터리를 **자동 미러링**하는 읽기 전용 성격입니다.
> 원본에 기여하려면 `temporalio/ui` 쪽에 PR을 보내야 하며, 이 레포에 직접 커밋하면 다음 동기화 시 덮어써질 수 있습니다.

---

## 2. 이 레포는 무엇인가

### 한 문장 정의

> **Temporal 워크플로 엔진의 웹 대시보드를 서빙하고, 브라우저의 REST 요청을 Temporal의 gRPC로 통역하며,
> 인증·보안까지 담당하는 Go 단일 실행파일 웹 서버.**

### 정체 확인 근거

| 단서 | 내용 |
|---|---|
| `go.mod` | `module github.com/temporalio/ui-server/v2` (Go 1.26.5) |
| `LICENSE` | MIT, Copyright (c) 2022 Temporal Technologies Inc. |
| `server/version/version.go` | `UIVersion = "2.54.1"` |
| 커밋 로그 | `Sync from UI commit 75a20a4...` (자동 동기화 봇 커밋) |
| `README.md` | "`ui-server`는 `temporalio/ui/tree/main/server`를 자동 미러링한다" |

### Temporal이 뭔지 (배경)

Temporal은 **Durable Execution(내구성 있는 실행)** 엔진입니다.
함수 실행의 모든 상태를 이벤트로 DB에 기록해서, 프로세스가 죽거나 재배포되거나 며칠이 지나도
**중단된 지점부터 그대로 재개**됩니다. Uber의 Cadence 개발자들이 창업한 회사에서 만들었습니다.

주요 활용: 결제/주문 파이프라인, 데이터 ETL, 프로비저닝 자동화, **장기 실행 AI 에이전트**.

### 역할 구분 (헷갈리기 쉬움)

| 레포 | 역할 |
|---|---|
| `temporalio/temporal` | 워크플로 엔진 본체 (gRPC :7233) |
| `temporalio/ui` | 프론트엔드(SvelteKit) 소스 + `server/` 폴더 = **개발 원본** |
| `temporalio/ui-server` | 위의 `server/` + 빌드된 프론트 에셋 = **배포용 미러** (지금 이 레포) |

---

## 3. 폴더별 전수조사 결과

### `cmd/server/main.go` — CLI 진입점
- `urfave/cli`로 구성, 명령어는 `start` 단 하나.
- 플래그: `--root`(`TEMPORAL_ROOT`), `--config`(`TEMPORAL_CONFIG_DIR`), `--env`(`TEMPORAL_ENV`).
- `fs_config_provider`로 설정을 읽고 `server.NewServer(opts...)` 실행.

### `server/server.go` — 심장부
Echo v4 미들웨어 체인을 순서대로 구성:

```
Logger → Recover → Gzip → CORS → Secure(보안 헤더)
  → CSRF 쿠키 발급(EnsureTokenCookie) → CSRF 검증
  → PublicPath(Pre) → /health → /api/v1/* → /auth/* → UI 정적 파일 + /render
```

- `EnableUI`가 true면 `uiAssetPath`(외부 폴더) 또는 임베드 에셋을 서빙.
- `cloudUi` 플래그로 `local` / `cloud` 두 종류 UI 번들 선택.
- `uiServerTLS.certFile/keyFile`이 있으면 `StartTLS`로 HTTPS 기동.

### `server/api/` — 핵심 (REST ↔ gRPC 변환)
| 파일 | 역할 |
|---|---|
| `handler.go` | `grpc-gateway` ServeMux에 `WorkflowService` + `OperatorService` 등록 → REST가 자동으로 gRPC로 번역 |
| `handler.go` (`GetSettings`) | `/api/v1/settings` — UI 부트스트랩용 feature flag 덩어리 JSON |
| `handler.go` (`GetUIExtensions`) | `/api/v1/ui-extensions` — 커스텀 iframe 확장 목록 |
| `handler.go` (`TemporalAccessCheck`) | 확장 목록 응답 전, Temporal에 `system-info`를 실제로 호출해 권한 검증 |
| `marshaler.go` | Temporal 전용 protobuf JSON 마샬러 (indent 적용) |
| `rawhistory.go` | 워크플로 원본 히스토리 다운로드 핸들러 |
| `middleware.go` | gRPC-gateway ServeMuxOption 형태의 미들웨어 타입 |

### `server/config/` + `plugins/fs_config_provider/` — 설정 엔진
- `config.go`: 40개 이상의 설정 필드를 담은 `Config` 구조체 + `Validate()`.
- `config_provider_with_refresh.go`: `refreshInterval` 주기로 **재시작 없이 설정 리로드**.
- `loader.go`: `base.yaml` + `{env}.yaml` **머지**, 파일 첫 줄에 `# enable-template` 주석이 있으면
  **Go text/template + Sprig 함수** 실행 → `{{ env "TEMPORAL_ADDRESS" | default "127.0.0.1:7233" }}` 가능.
- `custom_ui.go`: iframe 확장 검증 + CSP `frame-src` 목록 생성.

### `server/auth/` + `server/route/auth.go` — OIDC 인증
- OIDC 단일 프로바이더 지원 (Auth0 / Google / Okta / Keycloak 등).
- 라우트: `/auth/sso` → `/auth/sso/callback` → `/auth/logout`, 토큰 갱신 엔드포인트.
- 토큰을 `gorilla/securecookie`로 **암호화 쿠키에 저장**, 용량 초과 시 여러 쿠키로 분할(`splitCookie`).
- `maxSessionDuration`: 토큰이 유효해도 강제 재로그인시키는 세션 상한.
- `validateReturnURL`, `randNonce`: 오픈 리다이렉트 및 CSRF 방어.
- `useIdTokenAsBearer`: access token 대신 ID token을 Bearer로 사용.

### `server/cors/`, `server/csrf/`, `server/headers/` — 보안 3종
- `cors.go`: 화이트리스트 기반 + `unsafeAllowAllOrigins`(개발 전용, 위험).
- `csrf/cookie.go`: CSRF 토큰 쿠키 발급/검증.
- `csrf/sec_fetch_site.go`: `Sec-Fetch-Site` 헤더 기반 판정 + 설정된 CORS 오리진 허용.
- `csrf/skipper.go`: `Authorization` 헤더가 있으면 CSRF 검사 스킵 (API 클라이언트 지원).
- `headers.go`: `forwardHeaders` 목록의 HTTP 헤더를 gRPC metadata로 전달.

### `server/rpc/` — gRPC 연결 & TLS
- `CreateGRPCConnection` / `Dial` + `errorInterceptor`.
- `tls.go`: mTLS(CA/cert/key를 **파일 또는 base64 데이터**로), 호스트 검증, SNI 서버명.

### `server/route/` — 라우팅
- `api.go`: `/api/v1` 그룹 + `DisableWriteMiddleware`(GET만 통과, 단 workflow query POST는 예외).
- `ui.go`: SPA 정적 파일 서빙, `publicPath` 런타임 치환(이때 CSP meta 제거 → 경고 로그),
  CSP `frame-src` 헤더 설정, 마크다운 렌더링 라우트(`/render`, nonce CSP + 임베드 CSS).
- `public_path.go`: 서브패스 프리픽스 제거 미들웨어.
- `health.go`: `/health`.

### `ui/` — 프론트엔드 통째 내장
```go
//go:embed all:assets
var assets embed.FS
```
- SvelteKit 빌드 산출물 **5.9MB**를 Go 바이너리에 임베드 → **단일 실행파일 배포**.
- 발견된 특징적 에셋: `agent-harness-demo.mp4`, `agent-harness-preview.jpg`,
  `integration-openai.svg`, `integration-tenuo.svg` → **AI 에이전트 관련 기능이 UI에 포함됨**.

### 빌드 / 배포 / CI
| 항목 | 내용 |
|---|---|
| `Dockerfile` | 3-스테이지(Go 빌드 → dockerize → alpine), 비루트 유저 `temporal`(uid 1000), `EXPOSE 8080` |
| `docker/start-ui-server.sh` | 레거시 `config-template.yaml` 감지 후 경고, `./ui-server --env docker start` 실행 |
| `.goreleaser.yml` | linux/windows/darwin × amd64/arm64 = **6개 플랫폼** 바이너리 릴리즈 |
| `.github/workflows/test.yml` | ubuntu/macos/windows 3중 빌드 + `go test -race` |
| `on-commit-dispatch.yml` | `temporalio/ui`에서 `sync-from-ui-commit` 신호 오면 동기화 커밋 |
| `on-release-dispatch.yml` | 릴리즈 동기화 |
| `on-commit.yaml` | Docker 이미지 빌드/푸시 |
| `manual-docker-push.yaml` | 수동 이미지 푸시(ui / cloud-ui / both) |
| `Makefile` | `build-server`(go mod tidy + go build), `test`(`-race`) |

---

## 4. 설정 옵션 전체 정리

| 분류 | 옵션 | 설명 |
|---|---|---|
| 연결 | `temporalGrpcAddress`, `host`, `port` | Temporal 주소(기본 `127.0.0.1:7233`), 리스닝 |
| UI | `enableUi`, `cloudUi`, `uiAssetPath`, `publicPath` | UI on/off, 번들 종류, 외부 에셋 경로, 서브패스 |
| 인증 | `auth.enabled`, `auth.redirectToProvider`, `auth.providers[]`, `auth.maxSessionDuration` | OIDC SSO |
| 프로바이더 | `providerUrl`, `issuerUrl`, `clientId`, `clientSecret`, `scopes`, `callbackUrl`, `options`, `useIdTokenAsBearer` | OIDC 상세 |
| CORS | `cors.allowOrigins`, `cors.unsafeAllowAllOrigins`, `cors.cookieInsecure` | ⚠️ `unsafeAllowAllOrigins`는 프로덕션 금지 |
| 백엔드 TLS | `tls.caFile/certFile/keyFile`, `caData/certData/keyData`, `enableHostVerification`, `serverName` | Temporal과의 mTLS |
| UI 자체 TLS | `uiServerTLS.certFile/keyFile` | UI 서버를 HTTPS로 |
| **읽기전용** | `disableWriteActions` | **GET만 허용 (전역 스위치)** |
| 액션 제어 | `workflowTerminateDisabled`, `workflowCancelDisabled`, `workflowSignalDisabled`, `workflowUpdateDisabled`, `workflowResetDisabled`, `workflowPauseDisabled`, `batchActionsDisabled`, `startWorkflowDisabled`, `activityCommandsDisabled` | 위험 버튼 개별 비활성 |
| 가시성 | `defaultNamespace`, `showTemporalSystemNamespace`, `navCollapsedByDefault`, `hideWorkflowQueryErrors`, `refreshWorkflowCountsDisabled` | UI 표시 제어 |
| 암호화 | `codec.endpoint`, `passAccessToken`, `includeCredentials`, `defaultErrorMessage`, `defaultErrorLink` | Codec(복호화) 서버 연동 |
| 확장 | `customUi.enabled`, `customUi.iframeExtensions[]` (`id`, `title`, `slot`, `src`, `allowedOrigin`, `routePatterns`, `sandbox`, `sizing`, `permissions`) | iframe 확장 |
| 운영 | `refreshInterval`, `forwardHeaders`, `hideLogs`, `feedbackUrl`, `disableNewsFetch` | 기타 |

### 자주 쓰는 환경변수

```bash
TEMPORAL_ADDRESS=host:7233
TEMPORAL_UI_PORT=8080
TEMPORAL_UI_PUBLIC_PATH=/temporal
TEMPORAL_DEFAULT_NAMESPACE=default
TEMPORAL_DISABLE_WRITE_ACTIONS=true      # 읽기 전용 모드
TEMPORAL_CORS_ORIGINS=https://a.com,https://b.com
TEMPORAL_AUTH_ENABLED=true
TEMPORAL_AUTH_PROVIDER_URL=https://accounts.google.com
TEMPORAL_AUTH_CLIENT_ID=...
TEMPORAL_AUTH_CLIENT_SECRET=...          # 반드시 Secret Manager 사용
TEMPORAL_AUTH_CALLBACK_URL=https://host:8080/auth/sso/callback
TEMPORAL_CODEC_ENDPOINT=http://codec:8081
TEMPORAL_HIDE_LOGS=true
```

---

## 5. 동작 흐름 (요청 라이프사이클)

```
👤 브라우저
   │ ① GET /              → 임베드된 index.html 반환 (route/ui.go + ui/embed.go)
   │ ② GET /api/v1/settings → feature flag JSON (api/handler.go)
   │ ③ 로그인 필요 시 /auth/sso → OIDC 프로바이더 → /auth/sso/callback → 암호화 쿠키
   │ ④ GET /api/v1/namespaces/default/workflows
   ▼
🧑‍💼 ui-server (Echo)
   │  CORS → CSRF → 인증 헤더 확인 → grpc-gateway 변환
   ▼ gRPC :7233
👨‍🍳 Temporal 서버  →  🧊 DB (이벤트 히스토리)
   ▲
   └ ⑤ 응답을 Temporal 전용 JSON 마샬러로 변환 → 브라우저 렌더링
```

**비유**: Temporal은 gRPC만 말하는 주방, 브라우저는 HTTP/JSON만 아는 손님,
ui-server는 통역하고 신분증 확인하고 메뉴판까지 들고 있는 웨이터.

---

## 6. 설치 및 사용법

### 방법 1: Temporal CLI (가장 쉬움, 추천)

```bash
brew install temporal                                # macOS
curl -sSf https://temporal.download/cli.sh | sh      # Linux

temporal server start-dev              # 엔진 + UI 동시 기동 → http://localhost:8233
temporal server start-dev --ui-port 8080
```

> `config/base.yaml`의 기본 포트가 `8233`인 이유가 바로 이 내장 모드 때문입니다.

### 방법 2: Docker

```bash
docker run -d --network host \
  -e TEMPORAL_ADDRESS=127.0.0.1:7233 \
  -e TEMPORAL_UI_PORT=8080 \
  -p 8080:8080 \
  temporalio/ui:latest
```

전체 스택:
```bash
git clone https://github.com/temporalio/docker-compose.git
cd docker-compose && docker compose up -d
```

프로덕션급(SSO + mTLS) — `docker/README.md` 예시 기반:
```bash
docker run \
  -e TEMPORAL_ADDRESS=127.0.0.1:7233 \
  -e TEMPORAL_UI_PORT=8080 \
  -e TEMPORAL_AUTH_ENABLED=true \
  -e TEMPORAL_AUTH_PROVIDER_URL=https://accounts.google.com \
  -e TEMPORAL_AUTH_CLIENT_ID=xxxxx.apps.googleusercontent.com \
  -e TEMPORAL_AUTH_CLIENT_SECRET=xxxxxxxx \
  -e TEMPORAL_AUTH_CALLBACK_URL=https://my.com:8080/auth/sso/callback \
  -e TEMPORAL_AUTH_SCOPES=openid,email,profile \
  -e TEMPORAL_TLS_CA=/certs/ca.cert \
  -e TEMPORAL_TLS_CERT=/certs/cluster.pem \
  -e TEMPORAL_TLS_KEY=/certs/cluster.key \
  -e TEMPORAL_TLS_ENABLE_HOST_VERIFICATION=true \
  temporalio/ui:latest
```

서브패스 배포:
```bash
docker run -d --network host -e TEMPORAL_UI_PUBLIC_PATH=/custom-path -t temporal-ui
# → http://localhost:8080/custom-path   (주의: CSP meta가 제거됨)
```

### 방법 3: 이 레포에서 소스 빌드

```bash
cd /path/to/ui-server

make build-server                       # go mod tidy + go build → ./ui-server
./ui-server --env development start     # config/development.yaml → :8081
./ui-server --env with-auth start       # OIDC 테스트용
make test                               # go test ./... -race
```

- 요구 사항: **Go 1.26.5+** (`go.mod` 기준), Temporal 서버가 `:7233`에 기동되어 있어야 데이터 조회 가능.
- `--env` 값은 `config/{값}.yaml` 파일명. 현재 제공: `development`, `docker`, `e2e`, `with-auth`.
- 개발 시 핫리로드는 [Air](https://github.com/air-verse/air) 사용(README 참고).

---

## 7. 기술 분류: 플러그인 / 스킬 / MCP 아님

| 개념 | 정의 | ui-server |
|---|---|---|
| Claude 플러그인 | Claude Code 기능 확장 묶음 | ❌ |
| Claude 스킬 | `SKILL.md`로 작업 절차를 알려주는 문서 | ❌ |
| MCP 서버 | LLM에 도구/리소스를 제공하는 JSON-RPC 표준 | ❌ |
| **ui-server** | **Go 독립 HTTP 웹 서버 (BFF + API 게이트웨이 + SPA 호스팅 + OIDC 프록시)** | ✅ |

### 혼동 포인트 2가지

1. **`plugins/` 폴더** — Temporal 내부 개념의 플러그인입니다.
   `ConfigProvider` 인터페이스 구현체(`fs_config_provider`)를 분리해둔 것으로,
   Consul/etcd 기반 구현으로 교체 가능하게 만든 확장 지점입니다. Claude 플러그인과 무관합니다.

2. **`customUi.iframeExtensions`** — ui-server **자체의 확장 시스템**입니다.
   내 웹 페이지를 Temporal UI의 지정된 슬롯(`app.top-nav.sub-nav` 등)에 iframe으로 삽입합니다.

> 참고: ui-server의 REST API(`/api/v1/*`)를 감싸는 **MCP 서버를 별도로 만드는 것은 가능**하며,
> 이는 아래 수익화 아이디어에서 활용됩니다.

---

## 8. API 토큰이 필요한가

| 상황 | 필요 여부 | 준비물 |
|---|---|---|
| 로컬 개발 (`start-dev`) | ❌ 불필요 | 없음 |
| self-hosted, 인증 끔 | ❌ 불필요 | `auth.enabled: false` (기본) |
| self-hosted + SSO | ✅ 필요 | OIDC `clientId` + `clientSecret` |
| self-hosted + mTLS | ✅ 인증서 | CA / cert / key (토큰 아님) |
| Temporal Cloud | ✅ 필요 | API Key 또는 클라이언트 인증서 |
| 암호화 payload | ⚠️ 선택 | codec endpoint + `passAccessToken` |

### 토큰이 등장하는 3곳

1. **OIDC 클라이언트 시크릿** — Auth0/Google/Okta 등에서 앱 등록 시 발급.
2. **사용자 액세스 토큰** — 로그인 후 ui-server가 암호화 쿠키에 저장하고
   Temporal 호출 시 `Authorization: Bearer`로 자동 전달 (`auth.SetUser`, `ValidateAuthHeaderExists`).
3. **Codec 서버 인증** — `passAccessToken`, `includeCredentials`로 토큰/쿠키 전달 여부 결정.

### LLM API 토큰은 불필요

ui-server는 AI 도구가 아니라 인프라 대시보드입니다. Anthropic/OpenAI 키는 필요 없습니다.
(`integration-openai.svg`는 워크플로가 OpenAI를 사용한 경우를 표시하는 아이콘일 뿐)

### 보안 주의사항

- `clientSecret`을 YAML에 하드코딩하지 말고 환경변수 + Secret Manager 사용.
- `config/development.yaml`의 `unsafeAllowAllOrigins: true`는 **개발 전용**. 프로덕션 반영 금지.
- `publicPath` 사용 시 CSP meta가 제거되므로 리버스 프록시에서 CSP 헤더를 보강할 것.

---

## 9. 왜 GitHub에서 유명한가

### 전제: 이 레포 자체보다 생태계가 유명함

| 레포 | 인기도 |
|---|---|
| `temporalio/temporal` | 매우 높음 (Go 인프라 분야 최상위권, 1만 스타대 규모) |
| `temporalio/ui` | 중간 |
| `temporalio/ui-server` | 상대적으로 적음 (자동 미러) |

> 정확한 스타 수는 본 세션에서 조회하지 않았으므로 규모 수준으로만 참고하세요.

### Temporal 생태계가 유명한 이유

1. **분산 시스템의 최난제를 우아하게 해결** — 상태 머신 + 큐 + 재시도 + 멱등성을 수천 줄 짜야 했던 것을
   평범한 코드처럼 작성 가능하게 만듦.
2. **혈통** — Uber Cadence 개발자들이 창업. Netflix, Snap, Coinbase, Stripe, Datadog 등 프로덕션 사용.
3. **오픈소스/상용 균형** — MIT로 전체 기능 self-host 무료 + Temporal Cloud 매니지드 판매.
4. **멀티 언어 SDK** — Go, Java, TypeScript, Python, .NET, PHP, Ruby, Rust.
5. **AI 에이전트 붐의 수혜** — 에이전트의 골칫거리(재시도, 장기 실행, 크래시 복구, human-in-the-loop,
   추적성)가 Temporal이 푸는 문제와 정확히 일치. 이 레포 에셋의 `agent-harness-demo.mp4`가 그 방향의 증거.

### UI 자체의 평가 요소
- SvelteKit 기반으로 가볍고 빠름
- 이벤트 히스토리 타임라인 시각화 품질이 높음 (디버깅 경험)
- 다크모드, i18n, 접근성
- 단일 바이너리 배포 (운영 편의성)

---

## 10. 로컬 AI 에이전트 구축에 도움이 되는가

### 결론: 매우 도움 됨

#### 문제: 순수 루프 기반 에이전트의 한계
```python
while not done:
    response = llm.call(messages)     # 타임아웃 → 처음부터
    result = execute_tool(response)   # 툴 실패 → 전부 소실
    # 재부팅 → 몇 시간 작업 소멸 / 승인 대기 → 프로세스 상시 점유 / 원인 추적 불가
```

#### Temporal 적용 후
```python
@workflow.defn
class AgentWorkflow:
    @workflow.run
    async def run(self, goal: str):
        while not self.done:
            plan = await workflow.execute_activity(
                call_llm, self.messages,
                start_to_close_timeout=timedelta(minutes=5),
                retry_policy=RetryPolicy(maximum_attempts=5),
            )
            result = await workflow.execute_activity(run_tool, plan)
            if plan.is_dangerous:
                await workflow.wait_condition(lambda: self.approved)  # 며칠 대기도 리소스 0
        return self.result
```

| 문제 | Temporal의 해법 |
|---|---|
| LLM 타임아웃 / 429 | Activity 자동 재시도 + 백오프 |
| 프로세스 크래시 | 이벤트 히스토리 기반 정확한 지점 재개 |
| 장기 실행 / 승인 대기 | 무제한 대기, 대기 중 리소스 미소비 |
| human-in-the-loop | Signal / Update |
| 멀티 에이전트 | Child Workflow |
| 원인 추적 | 전체 이벤트 히스토리 (= ui-server가 보여주는 것) |

#### ui-server가 에이전트 개발에 주는 가치
- 이벤트 히스토리 타임라인 (LLM 호출/툴 실행 순서, 입출력, 소요 시간)
- Payload 뷰어 (프롬프트/응답 원본)
- 실패 스택트레이스 + 재시도 이력
- 검색/필터 (실패한 실행만, 특정 유저만)
- **Signal 전송 / 종료 / Reset 후 재실행** ← 프롬프트 수정 후 실패 지점부터 재시도 가능 (킬러 기능)

#### 권장 로컬 스택
```bash
temporal server start-dev            # 엔진 + UI(:8233)
pip install temporalio anthropic     # SDK
```
```
내 에이전트 Worker (Python)
  ├─ Workflow: 에이전트 루프 (결정론적)
  └─ Activity: LLM 호출 / 툴 실행 / DB
        │ gRPC :7233
   Temporal 서버 (상태 기록)
        │
   ui-server :8233 (관제탑)
```

#### 이 코드에서 바로 활용할 아이디어
1. `customUi.iframeExtensions`로 "에이전트 토큰/비용 패널"을 UI 안에 삽입
2. `codec.endpoint`로 민감 프롬프트 암호화 저장 + UI에서만 복호화
3. `disableWriteActions`로 팀 공유용 읽기 전용 대시보드
4. `/api/v1/*` REST API를 내 관리도구나 **MCP 서버**가 호출 → "실패한 에이전트 원인 분석해줘"

#### 트레이드오프
| 장점 | 단점 |
|---|---|
| 내구성/추적성 압도적 | 학습 곡선 (워크플로 결정론 제약) |
| 프로덕션까지 그대로 확장 | 단순 스크립트엔 과함 |
| 언어 선택 자유 | 인프라 1개 추가 운영 |

판단 기준: **몇 분 이상 걸리고, 여러 툴을 호출하고, 실패 비용이 크고, 사람 승인이 필요한 에이전트**라면 적합.

---

## 11. React / PHP로 만들 수 있는가

ui-server는 **① 프론트엔드(화면)** 와 **② 백엔드 프록시(REST→gRPC + 인증)** 두 역할을 합니다.
각각 따로 판단해야 합니다.

### React로 UI 재작성: 가능하며 추천

이미 깔끔한 REST/JSON API가 노출되어 있습니다:
```
GET  /api/v1/settings
GET  /api/v1/namespaces
GET  /api/v1/namespaces/{ns}/workflows?query=...
GET  /api/v1/namespaces/{ns}/workflows/{wid}/executions/{rid}/history
POST /api/v1/namespaces/{ns}/workflows/{wid}/terminate
POST /api/v1/namespaces/{ns}/workflows/{wid}/signal/{name}
GET  /api/v1/cluster-info, /api/v1/system-info
GET  /api/v1/ui-extensions
```

**방법 A: ui-server를 API 전용으로 사용 (권장)**
```yaml
# config/my.yaml
enableUi: false                  # UI 끄고 API만
port: 8080
cors:
  allowOrigins:
    - http://localhost:3000      # React 개발 서버
```
```jsx
const { data } = useQuery({
  queryKey: ['workflows', ns],
  queryFn: () => fetch(`/api/v1/namespaces/${ns}/workflows`, {
    credentials: 'include',
    headers: { 'X-CSRF-Token': csrfToken },
  }).then(r => r.json()),
});
```
`uiAssetPath`를 쓰면 **React 빌드 결과물을 ui-server가 그대로 서빙**할 수도 있습니다(Go 코드 수정 불필요).

**방법 B: Next.js 풀스택** — 서버 사이드에서 `@temporalio/client`로 gRPC 직접 호출도 가능
(브라우저에서는 gRPC 불가).

**주의사항**

| 항목 | 대응 |
|---|---|
| CSRF | POST 요청에 `X-CSRF-Token` 헤더 필수 (쿠키에서 읽기) |
| CORS | `cors.allowOrigins`에 프론트 오리진 추가 |
| Payload | protobuf `Payload.data`가 base64 → 디코딩 로직 필요 |
| 히스토리 렌더링 | 이벤트 타입이 50종 이상 → 여기가 가장 공수 큼 |

추천 스택: `React + TypeScript + TanStack Query + Tailwind + shadcn/ui`

### PHP: 부분적으로 가능

**가능한 것**
1. **Workflow/Activity를 PHP로 작성** — 공식 [Temporal PHP SDK](https://github.com/temporalio/sdk-php) (RoadRunner 기반)
2. **Laravel/Symfony 관리 대시보드** — ui-server REST API를 Guzzle로 호출
   ```php
   $workflows = Http::withHeaders(['Authorization' => "Bearer {$token}"])
       ->get('http://ui-server:8080/api/v1/namespaces/default/workflows')
       ->json();
   ```
3. 리포팅/관리자 페이지

**어려운 것**

| 목표 | 난이도 | 이유 |
|---|---|---|
| ui-server 자체를 PHP로 재작성 | 매우 어려움 | grpc-gateway 등가물이 없어 protobuf/gRPC 변환을 직접 구현해야 함 |
| PHP에서 Temporal gRPC 직접 호출 | 번거로움 | `ext-grpc` + protobuf 컴파일, PHP-FPM 요청 모델과 부적합 |
| 실시간 업데이트 | 어려움 | 롱폴링/WebSocket이 PHP의 약점 |

### 권장 조합

```
React 프론트엔드 (직접 개발)
        │ REST/JSON + 쿠키
ui-server (그대로 사용, enableUi: false)
        │ gRPC
Temporal 서버
        ▲
        │ PHP SDK로 Workflow/Activity 작성 (선택)
Laravel Worker
```

핵심 조언: **프록시 + 인증 계층은 재작성하지 말 것.**
CSRF 엣지케이스, 쿠키 분할, OIDC 리프레시, protobuf 마샬링을 다시 만드는 비용이 매우 큽니다.
**프론트엔드만 교체하면 1~2주에 MVP가 나옵니다.**

---

## 12. 수익화 아이디어 7선

### 시장 지형

**유리한 점**

| 요인 | 의미 |
|---|---|
| MIT 라이선스 | 상업적 사용/수정/재배포/유료 판매 가능 |
| **ui-server의 구조적 공백** | 인증(Authentication)은 있으나 **인가(Authorization/RBAC)가 없음** |
| `customUi` 확장 슬롯 | 공식 확장 포인트 존재 → Fork 없이 제품 삽입 가능 |
| AI 에이전트 붐 | 관측/비용 관리 수요 증가 |
| 한국어 자료 공백 | 선점 기회 |

**리스크**

| 리스크 | 대응 |
|---|---|
| Temporal Cloud와 경쟁 | 정면 충돌 회피, **self-hosted 고객** 타겟 |
| Temporal이 직접 구현 | 니치 + 지역 밀착 + 실행 속도 |
| 상표 | 제품명에 "Temporal" 미사용, "for Temporal" 표현만 |
| self-hosted 시장 규모 | 고단가 엔터프라이즈 중심 |

---

### 아이디어 1: 엔터프라이즈 인증/권한(RBAC) 게이트웨이 ⭐ 최우선

**근거 (코드 전수조사에서 발견한 공백)**
- ui-server는 인증만 처리하고, 쓰기 차단은 `disableWriteActions` **전역 스위치**뿐입니다.
- 즉 "A팀은 prod 읽기만, B팀은 dev 전체 권한" 같은 **네임스페이스별 권한 분리가 불가능**합니다.
- 규제 산업은 이게 없으면 도입 자체가 막힙니다.

**제품 구조**
```
브라우저
   ↓
내 게이트웨이 (제품)
   ├─ SSO / SAML / SCIM
   ├─ 네임스페이스 × 액션 RBAC
   ├─ 승인 워크플로 (위험 액션 2인 승인)
   ├─ 감사 로그 → SIEM 연동
   └─ 세션/IP/시간대 정책
   ↓ 허용된 요청만
ui-server (그대로) → Temporal
```
구현: ui-server 앞단 리버스 프록시에서 `/api/v1/*` 경로 + HTTP 메서드를 검사.
`route/api.go`의 `DisableWriteMiddleware`를 정교화한 형태.

**가격**

| 플랜 | 월 가격 | 내용 |
|---|---|---|
| Team | $299 | 사용자 20명, 기본 RBAC, 감사 로그 30일 |
| Business | $999 | 무제한 사용자, SAML/SCIM, 승인 워크플로, 감사 1년 |
| Enterprise | $3,000+ | 온프레미스, SIEM, SLA, 전담 지원 |

- 타겟: 금융/의료/공공 등 규제 산업의 self-hosted 팀
- 난이도: 중 / MVP: 4~6주
- MVP 범위: OIDC 그룹 → 네임스페이스별 read/write 매핑 + 감사 로그 + Docker 배포
- 스택: Go(또는 Node) 프록시 + PostgreSQL + React 관리 콘솔
- 검증: Temporal 커뮤니티 Slack에서 RBAC 수요 확인 후 착수

---

### 아이디어 2: AI 에이전트 관제 SaaS (Agent Observability) ⭐

**근거**: ui-server는 "워크플로 관점"만 제공. 에이전트 팀이 원하는 지표는 다름.

| ui-server 제공 | 에이전트 팀이 원하는 것 |
|---|---|
| Activity 목록 | 실행당 비용($) |
| Payload JSON | 프롬프트 ↔ 응답 대화 뷰 |
| 성공/실패 | 답변 품질 점수, 환각 탐지 |
| 재시도 횟수 | 툴별 실패율/지연 |
| 단건 조회 | 모델 A/B 비교, 회귀 탐지 |

**제품 구성**: 토큰/비용 집계, 프롬프트 타임라인 뷰어, 툴 성공률 분석,
LLM-as-judge 품질 평가, 예산 알림/차단, 프롬프트 버전 A/B 비교.
→ `customUi` iframe 패널로 Temporal UI 안에 삽입도 가능.

**가격**: 사용량 기반(월 100만 이벤트 $99~), 시트 기반($29/개발자),
또는 **비용 절감 성과제(절감액의 10~20%)**.

- 난이도: 중~상 / MVP: 6~8주
- MVP 범위: **비용 추적 한 가지만 완성도 높게**
- 차별점: LangSmith/Langfuse는 프레임워크 중심 → 우리는 **Temporal 네이티브 + 장기 실행 에이전트 특화**

---

### 아이디어 3: iframe 확장 패널 / 마켓플레이스

**근거**: 공식 확장 포인트가 이미 코드에 존재 (Fork 불필요)
```yaml
customUi:
  enabled: true
  iframeExtensions:
    - id: cost-panel
      title: 비용 분석
      slot: app.top-nav.sub-nav
      src: https://my-product.com/panel
      allowedOrigin: https://my-product.com
      permissions: [...]
      sandbox: { allowForms: true }
      sizing: { defaultHeight: 112, minHeight: 72, maxHeight: 200 }
```

**판매 가능 패널**

| 패널 | 가치 |
|---|---|
| 비용 분석 | 네임스페이스/팀별 실행 비용 |
| SLA/SLO 리포트 | 워크플로 완료 시간 준수율 |
| 이슈 연동 | 실패 → Jira/Linear 티켓 1클릭 |
| 런북 패널 | 에러별 대응 문서 인라인 |
| 데이터 계보 | 워크플로 간 데이터 흐름 |
| **AI 진단 어시스턴트** | 실패 히스토리 → LLM이 원인/해결책 한국어 설명 |

- 가격: 패널당 $49~199/월 (번들 할인)
- 난이도: **낮음** / MVP: **2~3주** ← 가장 빠르게 시작 가능
- 첫 MVP 추천: **AI 진단 어시스턴트 패널** (데모 임팩트 최대)

---

### 아이디어 4: Codec Server as a Service

**근거**: 암호화 payload를 UI에서 보려면 codec server가 필요하지만,
키 관리/로테이션/권한별 복호화/감사 로그를 직접 구현·운영하기 번거롭습니다.

**제품**: 매니지드 또는 온프레미스 codec server + KMS/Vault 연동 키 관리 + 자동 로테이션
+ **역할별 선택적 복호화**(개발자 마스킹 / 보안팀 원본) + PII 자동 마스킹 + 복호화 감사 로그(GDPR/HIPAA).

- 가격: $499~2,000/월, 온프레미스 라이선스 $10,000+/년
- 타겟: 금융/의료/보험
- 난이도: 중 / MVP: 4~6주
- 주의: 보안 제품은 평판/인증(SOC2 등)이 판매 전제 조건

---

### 아이디어 5: 한국 시장 컨설팅 / 교육 / 매니지드 ⭐ 즉시 시작 가능

**근거**: 한글 자료 거의 없음 + 국내 금융/공공의 self-hosted 수요 + AI 에이전트 도입 증가.

| 상품 | 가격 |
|---|---|
| 도입 컨설팅(PoC 설계) | 프로젝트당 500만~3,000만원 |
| 기업 교육(2일 워크숍) | 회당 300만~800만원 |
| 온라인 강의 | 인당 5~15만원 × N |
| 유지보수/온콜 리테이너 | 월 200만~1,000만원 |
| 매니지드 운영 | 월 300만원+ |

- 난이도: **낮음** (제품 개발 없이 시작) / 시작: 즉시~2주
- 첫 스텝: 한글 Temporal 튜토리얼 연재 → "Temporal + AI 에이전트" 포지셔닝 → 웨비나 → 컨설팅 전환
- 부수효과: 아이디어 1~4 제품의 최고 마케팅 채널

---

### 아이디어 6: 멀티 클러스터 통합 관제 콘솔

**근거**: ui-server 인스턴스 하나는 클러스터 하나만 조회. 실무는 dev/staging/prod × 리전.

**제품**: 환경 셀렉터, 통합 검색("모든 환경의 실패 워크플로"), 환경 간 비교,
통합 헬스 대시보드/알림, 환경별 권한 분리(prod 읽기 전용).

- 가격: 클러스터당 $99/월 또는 통합 $499/월~
- 난이도: 중 / MVP: 4주
- 구현 팁: 각 클러스터의 ui-server REST API를 aggregate → React 프론트 중심 작업

---

### 아이디어 7: 온콜/알림 브릿지 + 자동 복구

**근거**: ui-server는 화면 표시까지만 담당. 야간 장애 감지/대응은 공백.

**제품**: 실패/지연/SLA 위반 감지 → Slack / PagerDuty / 카카오워크 알림,
워크플로 타입별 담당팀 라우팅, 알림 내 1클릭 액션(재시도/종료/에스컬레이션),
자동 복구 규칙, 주간 신뢰성 리포트.

- 가격: $199~999/월
- 난이도: 낮~중 / MVP: 3주
- 한국 특화: 카카오워크/네이버웍스/잔디 연동 (해외 경쟁자 미커버 영역)
- 주의: Temporal에 웹훅이 없어 폴링 또는 Advanced Visibility 쿼리 기반 감지 필요

---

### 종합 비교

| # | 아이디어 | 난이도 | MVP | 시장 | 단가 | 추천도 |
|---|---|---|---|---|---|---|
| 1 | 인증/RBAC 게이트웨이 | 중 | 4-6주 | 중 | 높음 | ★★★★★ |
| 2 | AI 에이전트 관제 | 중상 | 6-8주 | 큼 | 중 | ★★★★★ |
| 3 | iframe 확장 패널 | 낮음 | 2-3주 | 중 | 낮음 | ★★★★ |
| 4 | Codec SaaS | 중 | 4-6주 | 작음 | 높음 | ★★★ |
| 5 | 한국 컨설팅/교육 | 낮음 | 즉시 | 중 | 중 | ★★★★★ |
| 6 | 멀티 클러스터 관제 | 중 | 4주 | 중 | 중 | ★★★ |
| 7 | 알림/자동복구 | 낮~중 | 3주 | 중 | 낮음 | ★★★★ |

---

## 13. 추천 실행 전략 (3단계)

### Phase 1 (0~1개월): 콘텐츠 — 리스크 0
- 한글 Temporal 블로그/영상 시리즈 연재
  1. "Temporal이 뭔데 다들 쓴다는 거야?"
  2. "ui-server 코드 전수조사" ← 본 문서 활용
  3. "Temporal로 죽지 않는 AI 에이전트 만들기"
  4. "React로 Temporal UI 직접 만들어보기"
- 즉시 수익: 컨설팅/교육 문의 (아이디어 5)
- 부수효과: 전문가 포지셔닝 + 고객 인터뷰 기회

### Phase 2 (1~3개월): 최소 제품 — AI 진단 패널
- `customUi` iframe 패널로 "AI 워크플로 진단 어시스턴트"
- 실패 워크플로 히스토리 → LLM → 한국어 원인 분석/해결책
- 난이도 낮음(2~3주), 데모 임팩트 큼, Fork 불필요
- Phase 1에서 모은 리드에게 무료 베타 → $49~99/월로 첫 유료 전환

### Phase 3 (3~9개월): 엔터프라이즈 제품
- 규제 산업 문의가 많으면 → 아이디어 1 (RBAC 게이트웨이)
- AI 팀 문의가 많으면 → 아이디어 2 (에이전트 관제 SaaS)
- $299~3,000/월 계약 목표

---

## 14. 라이선스 및 법적 체크리스트

| 항목 | 지침 |
|---|---|
| MIT 라이선스 | 상업적 사용/수정/판매 가능. **단 LICENSE 및 저작권 고지 유지 필수** |
| 상표 | 제품명에 "Temporal" 포함 지양. "XXX for Temporal" 형태만 권장 |
| 로고 | Temporal 로고 무단 사용 금지 |
| 포크 배포 | 원본 출처 명시, 공식 제품과의 혼동 유발 금지 |
| 커뮤니티 | 홍보 전에 기여부터 (신뢰가 최고의 마케팅) |
| 본 레포 수정 | 자동 미러이므로 직접 커밋은 덮어써질 수 있음 → `temporalio/ui`에 PR |

---

## 15. 코드에서 배울 만한 패턴 Top 3

### 1위: 프론트엔드를 바이너리에 임베드
```go
//go:embed all:assets   // 5.9MB SvelteKit 빌드 결과물을 바이너리에 포함
var assets embed.FS
```
배포 산출물이 실행파일 1개. nginx 설정, `node_modules`, 복잡한 COPY 불필요.
→ 개인 프로젝트(React 등)에 그대로 적용 가능.

### 2위: 레이어드 설정 관리
```
base.yaml (공통) + {env}.yaml (환경별) + 환경변수 템플릿 + refreshInterval 자동 리로드
```
`{{ env "TEMPORAL_ADDRESS" | default "127.0.0.1:7233" }}` — YAML 내부에서 기본값 있는 환경변수 치환.
`# enable-template` 주석으로 템플릿 기능을 opt-in 하는 설계가 깔끔합니다.

### 3위: Feature flag로 위험 기능 차단
```yaml
disableWriteActions: true          # 전역 읽기 전용
workflowTerminateDisabled: true    # 개별 액션 차단
```
구현은 `DisableWriteMiddleware`에서 `GET`만 통과시키는 단순한 방식인데 효과가 큽니다.
멀티테넌트/권한 분리 제품에서 바로 재사용 가능한 패턴.

### 보너스: grpc-gateway로 REST ↔ gRPC 자동 변환
`runtime.NewServeMux`에 서비스 핸들러를 등록하기만 하면 REST 경로가 gRPC 호출로 번역됩니다.
gRPC 백엔드에 웹 프론트를 붙일 때 표준 해법.

---

## 부록: 빠른 참조

```bash
# 가장 빠른 체험
temporal server start-dev                      # → http://localhost:8233

# Docker
docker run -d --network host -p 8080:8080 temporalio/ui:latest

# 이 레포에서 빌드
make build-server && ./ui-server --env development start   # → :8081
make test

# 읽기 전용 모드로 팀 공유
TEMPORAL_DISABLE_WRITE_ACTIONS=true

# React 프론트를 붙일 때
enableUi: false + cors.allowOrigins: [http://localhost:3000]
```

| 주요 엔드포인트 | 용도 |
|---|---|
| `/health` | 헬스체크 |
| `/api/v1/settings` | UI 부트스트랩 설정 |
| `/api/v1/ui-extensions` | iframe 확장 목록 |
| `/api/v1/namespaces/...` | 워크플로 CRUD (gRPC로 번역) |
| `/auth/sso`, `/auth/sso/callback`, `/auth/logout` | OIDC 인증 |
| `/render` | 마크다운 → HTML 렌더링 |
