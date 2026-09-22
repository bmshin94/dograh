# Dograh 전수조사 & 활용 전략 정리 (한국어)

> 작성일: 2026-09-22
> 대상 저장소: **https://github.com/bmshin94/dograh** (원본: **https://github.com/dograh-hq/dograh**)
> 문서 목적: Dograh가 무엇인지, 어떻게 쓰는지, 어떻게 수익화할 수 있는지를 한 문서로 정리

---

## 📑 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [저장소 전수조사 결과](#2-저장소-전수조사-결과)
3. [핵심 아키텍처](#3-핵심-아키텍처)
4. [쉬운 설명 (비유 버전)](#4-쉬운-설명-비유-버전)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인? 스킬? MCP?](#6-플러그인-스킬-mcp)
7. [API 토큰이 필요한가?](#7-api-토큰이-필요한가)
8. [왜 GitHub에서 유명한가](#8-왜-github에서-유명한가)
9. [로컬 에이전트 구축에 주는 도움](#9-로컬-에이전트-구축에-주는-도움)
10. [React / PHP로 만들 수 있나?](#10-react--php로-만들-수-있나)
11. [수익화 아이디어](#11-수익화-아이디어)
12. [실행 로드맵](#12-실행-로드맵)
13. [리스크 체크리스트](#13-리스크-체크리스트)
14. [참고 링크](#14-참고-링크)

---

## 1. 한 줄 요약

> **전화로 사람과 대화하는 AI 상담원을, 코딩 없이 그래프(순서도)로 만들어 실제 전화망에 연결해주는 오픈소스 플랫폼.**

- 상용 서비스 **Vapi / Retell AI**의 셀프호스팅 오픈소스 대체재
- 라이선스: **BSD 2-Clause** (상업적 이용·수정·재배포 자유, 저작권 표시만 유지)
- 저작권자: Zansat Technologies Private Limited (YC 출신 창업자)
- Product Hunt **Day / Week / Month 1위**, Trendshift 등재

---

## 2. 저장소 전수조사 결과

### 2.1 규모

| 항목 | 수치 |
|---|---|
| 전체 파일 수 | 1,680개 (git 추적 기준) |
| Python | 809개 파일 / **약 175,000줄** |
| TypeScript + TSX | 395개 파일 / **약 83,000줄** |
| 문서 (mdx) | 129개 |
| 셸 스크립트 | 34개 |

### 2.2 폴더 구조

| 폴더 | 정체 | 주요 내용 |
|---|---|---|
| `api/` | FastAPI 백엔드 (핵심) | 라우트, 서비스, DB 클라이언트, MCP 서버, ARQ 태스크 |
| `ui/` | Next.js 15 + React 19 프론트엔드 | 워크플로우 빌더, 대시보드, 리포트 |
| `pipecat/` | 음성 파이프라인 (git submodule) | STT → LLM → TTS 실시간 처리 |
| `sdk/` | Python / TypeScript SDK | 코드로 워크플로우 생성 |
| `docs/` | Mintlify 문서 사이트 | 설치·개념·API 레퍼런스 |
| `deploy/` | 배포 자산 | **Helm 차트**, nginx, coturn, 옵저버빌리티 |
| `evals/` | 평가 도구 | **STT 벤치마크** (Deepgram / Speechmatics 등) |
| `examples/` | 예제 | Python·TS 각 4종 |
| `scripts/` | 운영 스크립트 30여개 | `start_docker.sh`, 마이그레이션, 릴리스 |
| `.agents/skills/` | 저장소 자체 AI 스킬 | `review-pr`, `review-agents-md`, `merge-pipecat-upstream` |
| `config/coturn/` | TURN 서버 설정 | WebRTC용 |

### 2.3 백엔드 주요 모듈 (`api/`)

- **`routes/`** (26개): `workflow`, `campaign`, `telephony`, `knowledge_base`, `reports`, `webrtc_signaling`, `public_embed`, `agent_stream`, `service_keys`, `superuser` 등
- **`services/workflow/`**: `pipecat_engine.py`(대화 엔진), `workflow_graph.py`(그래프), `disposition_extraction.py`(결과 분류), `agent_transfer.py`(사람에게 전환), `qa/`(품질평가)
- **`services/pipecat/`**: `pipeline_builder.py`, `answer_classification.py`(사람/자동응답기 구분), `audio_mixer.py`, `termination_funnel_processor.py`(발화 종료 판정), `recording_router_processor.py`
- **`services/campaign/`**: `campaign_orchestrator.py`, `circuit_breaker.py`, `campaign_retry.py`, `call_concurrency`
- **`services/telephony/providers/`**: `twilio`, `vonage`, `telnyx`, `plivo`, `exotel`, `vobiz`, `cloudonix`, `ari`(Asterisk)
- **`mcp_server/`**: MCP 도구 + AST 기반 TS 검증기(`ts_validator/`)
- **`tasks/`**: ARQ 백그라운드 (웹훅 전달, 지식베이스 처리, 통화 완료 후처리)
- **`native/rnnoise/`**: C 기반 노이즈 제거 바인딩

### 2.4 인프라 (docker-compose 기준)

`postgres(pgvector)` · `redis` · `minio(S3 호환)` · `nginx` · `coturn(TURN)` · `api` · `ui` · `dograh-init` · `cloudflared`

---

## 3. 핵심 아키텍처

### 3.1 대화 = 그래프 (노드 + 엣지)

```
[startCall] --"고객이 인사함"--> [의도파악] --"상담 원함"--> [지원담당]
                                     └----"구매 원함"--> [영업담당] --> [endCall]
```

**노드 타입 7종**

| 노드 | 역할 |
|---|---|
| `startCall` | 통화 연결 시 첫 멘트 (telephony 진입점) |
| `agentNode` | LLM 기반 대화 단계 (핵심 블록) |
| `globalNode` | 전체 공통 규칙 (말투, 언어, 폴백) |
| `endCall` | 통화 종료 |
| `trigger` | API 기반 실행 진입점 |
| `webhook` | HTTP 호출 (CRM 갱신 등) |
| `qa` | 통화 종료 후 자동 품질 평가 |

> **핵심 포인트:** 엣지의 전이 조건이 **자연어**다. `"고객이 관심을 보이면"` 이라고 쓰면 LLM이 판단한다.

### 3.2 실시간 음성 루프

```
전화망 → [STT 음성인식] → [LLM 응답생성] → [TTS 음성합성] → 전화망
                               ↑
                   노드 프롬프트 + 대화이력 + 컨텍스트
```

통화 종료 후: 변수 추출 → 웹훅 발사 → Run 레코드(전사/녹음/비용/결과) 저장

### 3.3 지원 프로바이더

| 구분 | 목록 |
|---|---|
| **전화망 (8종)** | Twilio, Vonage, Telnyx, Plivo, Exotel, Vobiz, Cloudonix, **Asterisk ARI** |
| **STT** | Deepgram, AssemblyAI, Speechmatics, Azure, Groq, Sarvam |
| **TTS** | ElevenLabs, Cartesia, Rime, MiniMax, Speechify, Smallest, Azure, Google |
| **LLM** | OpenAI, Google AI Studio/Vertex, Azure OpenAI, AWS Bedrock, Groq, OpenRouter, HuggingFace, MiniMax, Sarvam, **Ollama / vLLM(로컬)** |
| **스토리지** | MinIO(기본), AWS S3, S3 호환(rustfs, Ceph 등) |
| **옵저버빌리티** | Langfuse, OpenTelemetry, Prometheus, Sentry, PostHog |

### 3.4 배포 모드

| 모드 | 설명 |
|---|---|
| `DEPLOYMENT_MODE=oss` | 기본값. 셀프호스팅. 로컬 JWT 인증 + MinIO |
| `DEPLOYMENT_MODE=saas` | Dograh MPS(Managed Platform Services) 연동 |

인증은 `AUTH_PROVIDER=local`(내장 이메일/비번) 또는 `stack`(Stack Auth 소셜 로그인).

---

## 4. 쉬운 설명 (비유 버전)

**Dograh = "전화 받아주는 AI 직원을 만들어주는 공장"**

1. **대본을 그림으로 그린다** — 파워포인트 도형 그리듯 노드를 배치하고, 화살표 조건은 한국어로 그냥 쓴다.
   - 옛날: `if (intent == "order" && confidence > 0.8)`
   - Dograh: `"손님이 주문하려는 의사를 보이면"`
2. **AI 3명이 릴레이로 일한다** — 받아쓰기 담당(STT) → 생각 담당(LLM) → 성우 담당(TTS)
3. **진짜 전화번호에 연결한다** — 인바운드(받기) / 아웃바운드(걸기), 그리고 CSV 업로드로 수천 통 자동 발신(캠페인)
4. **통화 끝나면 자동 정리** — 전사, 녹음, 변수 추출, 웹훅, 품질 채점, 비용 계산

**헷갈림 정리**

| 질문 | 답 |
|---|---|
| 챗봇과 차이? | 챗봇은 글자, 이건 목소리 + 실제 전화 |
| ChatGPT 음성모드와 차이? | 그건 개인용, 이건 기업이 고객에게 전화하는 시스템 |
| 공짜? | 셀프호스팅 시 소프트웨어 0원. AI/통신 사용량만 과금 |
| 상업적 이용 가능? | BSD 2-Clause라 가능 |

---

## 5. 설치 및 사용법

### 5.1 빠른 체험 (2분, Docker만 필요)

```bash
curl -o docker-compose.yaml https://raw.githubusercontent.com/dograh-hq/dograh/main/docker-compose.yaml \
  && curl -o start_docker.sh https://raw.githubusercontent.com/dograh-hq/dograh/main/scripts/start_docker.sh \
  && chmod +x start_docker.sh && ./start_docker.sh
```

→ **http://localhost:3010**

- `start_docker.sh`가 JWT 시크릿 / MinIO 키를 자동 생성 (python3 → openssl → /dev/urandom 순)
- 텔레메트리 비활성화: `ENABLE_TELEMETRY=false`
- 첫 실행 2~3분 소요

**첫 봇 만들기**
1. Inbound / Outbound 선택
2. 봇 이름 + 용도를 5~10단어로 설명
3. **Test Agent** 클릭
4. **Test Audio**(브라우저 통화) 또는 **Test Chat**(텍스트 반복 튜닝)
   - Test Chat에서는 사용자 턴을 수정/리플레이하면 그 지점부터 응답과 전이가 재생성됨

### 5.2 개발 환경 (소스 수정용)

```bash
git clone https://github.com/bmshin94/dograh
cd dograh
# VS Code → "Dev Containers: Reopen in Container"
bash scripts/start_services_dev.sh            # 터미널 1: 백엔드
cd ui && npm run dev -- --hostname 0.0.0.0    # 터미널 2: 프론트
```
→ **http://localhost:3000**

| 작업 | 명령 |
|---|---|
| 백엔드 정지 | `bash scripts/stop_services.sh` |
| 로그 확인 | `tail -f logs/latest/*.log` |
| pipecat 동기화 | `git submodule update --init --recursive` |
| 디버깅 | `.vscode/launch.json` 구성 사용 (F5) |

> `api/` 편집은 자동 리로드. `ari_manager`, `campaign_orchestrator`, `arq`는 재시작 필요.

### 5.3 기타 배포

- **클라우드**: https://app.dograh.com (가입 후 바로 사용)
- **원격 서버**: Docker Compose + nginx + cloudflared (HTTPS)
- **Kubernetes**: `deploy/helm/dograh/` (AWS / k3s-prod / single-node / managed values 예제 포함)
- **Heroku**: `docs/deployment/heroku.mdx`

### 5.4 주요 환경변수

| 변수 | 비고 |
|---|---|
| `DATABASE_URL` | **필수**. `postgresql+asyncpg://...` |
| `REDIS_URL` | **필수** |
| `OSS_JWT_SECRET` | **필수(OSS)**. 반드시 강한 랜덤값으로 교체 |
| `PUBLIC_BASE_URL` | 공개 origin. `BACKEND_API_ENDPOINT`, `MINIO_PUBLIC_ENDPOINT` 파생 |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | **필수(OSS)** |
| `ENABLE_AWS_S3` | `true`면 S3 백엔드 사용 (private 버킷 presigned URL 필요 시 권장) |
| `ENABLE_SIGNUP` | `false`면 공개 가입 차단 |
| `ENABLE_TELEMETRY` | `false`로 익명 사용 데이터 수집 해제 |

---

## 6. 플러그인? 스킬? MCP?

**결론: Dograh 본체는 "독립 웹 애플리케이션(제품)"이다. 다만 MCP 서버를 내장하고 있고, 별도의 Claude Code 플러그인도 존재한다.**

| 구분 | Dograh 본체 | 내장 MCP 서버 | dograh-plugins |
|---|---|---|---|
| 정체 | 독립 웹 애플리케이션 | MCP 엔드포인트 | Claude Code / Codex 플러그인 |
| 위치 | 이 저장소 전체 | `api/mcp_server/` | `dograh-hq/dograh-plugins` |
| 용도 | 음성 AI 제작·운영 | AI가 Dograh를 조작 | AI가 Dograh를 설치·설정 |
| 실행 | Docker 9개 컨테이너 | `/api/v1/mcp/` | `/plugin install dograh@dograh` |

### 6.1 내장 MCP 도구 목록

| 도구 | 기능 |
|---|---|
| `list_workflows` | 에이전트 목록 |
| `get_workflow_code` | 워크플로우를 TypeScript 코드로 투영 |
| `create_workflow` / `save_workflow` | 생성(v1 게시) / 수정(draft 저장) |
| `list_node_types` / `get_node_type` | 노드 스펙 조회 |
| `search_docs` / `read_doc` / `list_docs` | 문서 검색·열람 |
| `get_voice_prompting_guide` | 단계별 프롬프트 작성 가이드 주입 |
| `create_tool` | 에이전트용 툴 생성 |
| `list_tools` / `list_credentials` / `list_documents` / `list_recordings` | 참조 카탈로그 |

### 6.2 연결 방법

```bash
# Claude Code
claude mcp add --transport http dograh https://app.dograh.com/api/v1/mcp/ \
  --header "X-API-Key: YOUR_API_KEY"
claude mcp list
```

```toml
# Codex (~/.codex/config.toml)
[mcp_servers.dograh]
url = "https://app.dograh.com/api/v1/mcp/"
http_headers = { "X-API-Key" = "YOUR_API_KEY" }
```

> 자체 서명 인증서로 원격 배포한 경우 Claude Code는 `NODE_TLS_REJECT_UNAUTHORIZED=0 claude` 필요. 커스텀 도메인 + Let's Encrypt면 불필요.

### 6.3 설계에서 배울 점 (중요)

- MCP instructions가 **Plan → Create → Review 3단계를 강제**하고, 계획 승인 전 코드 작성을 금지
- 워크플로우 소스 파서가 **AST 화이트리스트 방식** — 함수, 화살표 함수, 반복문, 조건문, 삼항, spread, 구조분해, 템플릿 보간, `export`, `.map/.forEach` 전부 거부
- `test_mcp_instructions_drift.py` — 지침서가 등록되지 않은 툴을 언급하거나, 툴 docstring에 에러코드가 없으면 **테스트 실패**
- 저장소 자체 스킬 `.agents/skills/review-pr` — 테넌트 격리 누락, 인증 없는 라우트, 서명 안 된 웹훅 신뢰, `api/db/*_client.py` 밖 SQL, 워커 동기화 없는 캐시 갱신 등 **레포 고유 실수 패턴**만 집중 점검

---

## 7. API 토큰이 필요한가?

**체험은 0개, 실전은 필요.**

### 7.1 키 4종류

| 키 | 용도 | 필요 시점 |
|---|---|---|
| **Dograh API Key** | 외부에서 Dograh 조작 (n8n, MCP, SDK) | 자동화 |
| **Service Key** | Dograh 관리형 LLM/TTS/STT 사용 | Dograh 크레딧 사용 시 |
| **BYOK (본인 키)** | OpenAI / ElevenLabs / Deepgram 등 | 본인 계정 사용 시 |
| **Telephony 키** | Twilio SID/Token 등 | 실제 전화 연결 시 |

> README 명시: *"No API keys needed. Dograh ships with auto-generated keys and its own LLM / TTS / STT stack."*
> 설치 → 봇 생성 → 브라우저 음성 테스트까지는 키 없이 가능.

### 7.2 상황별 필요 키

| 목표 | 필요 키 |
|---|---|
| 로컬 테스트만 | 없음 |
| 고품질 음성 | ElevenLabs / Cartesia |
| GPT-4o 사용 | OpenAI |
| 한국어 인식 개선 | Deepgram / AssemblyAI |
| 실제 전화번호 연결 | Twilio 등 (필수) |
| 코드 자동화 | Dograh API Key |
| MCP 연동 | Dograh API Key |

### 7.3 완전 무과금 / 폐쇄망 조합

```
LLM   : Ollama / vLLM (OpenAI 호환 엔드포인트)
STT   : 로컬 Whisper 계열
TTS   : 로컬 TTS
전화망 : Asterisk ARI (자체 PBX)
인증   : AUTH_PROVIDER=local
```
→ 외부 네트워크 없이 동작 가능. 의료·금융·공공 영업의 핵심 포인트.

> 주의: `OSS_JWT_SECRET`, `MINIO_*` 키는 반드시 강한 랜덤값으로 교체할 것. Service Key는 생성한 Dograh Cloud 계정에 종속되며, 과금도 그 계정에서 발생.

---

## 8. 왜 GitHub에서 유명한가

### 8.1 지표

- 2026-05-17 기준 **1,508 스타** (24시간 +236)
- Product Hunt **Day / Week / Month 1위**
- Trendshift 등재 (repo id 31007), Better Stack 유튜브 피처
- 창업자 자체 집계 오가닉 노출 100만+

### 8.2 이유 7가지

1. **가격 파괴** — Vapi/Retell은 분당 과금. 셀프호스팅 시 소프트웨어 비용 0원
2. **명확한 포지셔닝** — README 첫 줄이 "self-hostable alternative to Vapi & Retell"
3. **데이터 주권** — 음성 통화는 개인정보 덩어리. HIPAA/GDPR/개인정보보호법 대응이 계약 조건인 산업이 많음
4. **오픈소스인데 UI가 좋다** — React Flow 기반 드래그앤드롭 빌더 + 대시보드/리포트/녹음
5. **MCP 네이티브** — "AI 어시스턴트로 음성 에이전트를 만든다"는 2026년 최강 키워드 조합
6. **진입장벽 최소** — `curl | bash` 한 줄, 2분, API 키 불필요
7. **진짜 프로덕션 코드** — 멀티테넌시, 과금/쿼터, 서킷브레이커, Helm, OTEL/Langfuse, Alembic, 테스트

---

## 9. 로컬 에이전트 구축에 주는 도움

1. **그 자체로 로컬 에이전트 런타임** — Ollama + 로컬 STT/TTS + Asterisk 조합으로 폐쇄망 음성 에이전트 구성 가능
2. **"AI가 AI를 만드는" 구조의 교과서**

| 기법 | 위치 | 배울 점 |
|---|---|---|
| 단계 강제 | `api/mcp_server/instructions.py` | Plan→Create→Review, 승인 전 코드 금지 |
| AST 화이트리스트 | `api/mcp_server/ts_validator/` | 문법 자체를 좁혀 LLM 오작동 차단 |
| 스펙 기반 검증 | `sdk/python/src/dograh_sdk/_validation.py` | 호출 시점 즉시 ValidationError |
| 가이드 주입 | `get_voice_prompting_guide` | 단계별로 필요한 지침만 주입 |
| 드리프트 테스트 | `test_mcp_instructions_drift.py` | 지침-코드 불일치 시 CI 실패 |

3. **에이전트 = 상태머신(그래프) 패턴** — 하나의 거대 프롬프트 대신, 노드별 좁은 프롬프트 + 자연어 전이 조건
4. **실시간 스트리밍 처리 교본** — `termination_funnel_processor.py`(발화 종료), `answer_classification.py`(사람/자동응답기), `audio_mixer.py`, `in_memory_buffers.py`, `turn_context.py`, `native/rnnoise/`
5. **평가 인프라** — `evals/stt/benchmark.py`로 STT 프로바이더별 정확도/지연 실측

### 추천 학습 루트

| 주차 | 할 일 |
|---|---|
| 1주차 | Docker로 띄우고 직접 통화해보기 |
| 2주차 | MCP 연결 후 AI에게 워크플로우 생성 시키기 |
| 3주차 | `api/mcp_server/` + `instructions.py` 정독 → 자체 에이전트에 이식 |
| 4주차 | `pipecat_engine.py` 읽고 실시간 처리 패턴 흡수 |
| 5주차 | Ollama로 교체해 완전 로컬 구성 완성 |

---

## 10. React / PHP로 만들 수 있나?

### 10.1 React — 이미 React 기반

`ui/`가 Next.js 15 + React 19 + `@xyflow/react`(React Flow) + shadcn/Radix + Zustand.

| 목표 | 가능 여부 | 방법 |
|---|---|---|
| UI 커스터마이징 | 매우 쉬움 | `ui/` 직접 수정 |
| 내 React 앱에 음성 위젯 삽입 | 쉬움 | `public/embed`, `public_embed` 라우트 |
| 내 React 앱에서 Dograh 조작 | 쉬움 | REST API + `@dograh/sdk` |
| 빌더 화면만 새로 제작 | 가능 | React Flow 사용 |
| React만으로 전체 재구현 | 불가 | 실시간 음성 파이프라인은 서버 필요 |

### 10.2 PHP — 권장하지 않음 (단, 연동은 좋음)

| 필요 기능 | PHP 상황 |
|---|---|
| 실시간 양방향 WebSocket 다중 연결 | ReactPHP/Swoole 필요, 생태계 빈약 |
| 비동기 동시성(통화 수백 건) | 요청-응답 모델 한계 |
| 오디오 DSP(리샘플링, VAD, 노이즈 제거) | 라이브러리 거의 없음 |
| Pipecat류 프레임워크 | 존재하지 않음 |
| ML/AI SDK 생태계 | Python 우선 |

**권장 아키텍처**

```
[ PHP 서비스 (Laravel 등) ]  --REST API-->  [ Dograh (Docker) ]
  고객관리 / 결제 / 관리자화면  <--Webhook--   음성처리 / 전화망
```

```php
// 통화 트리거
$res = Http::withHeaders(['X-API-Key' => $key])
    ->post("$base/api/v1/workflow/{$id}/runs", [
        'phone_number'    => '+821012345678',
        'initial_context' => ['customer_name' => '김철수'],
    ]);

// 통화 종료 웹훅 수신
Route::post('/webhook/dograh', fn(Request $r) => CallLog::create($r->all()));
```

**상황별 추천**

| 스택 | 추천 |
|---|---|
| React/Next 개발자 | `ui/` 직접 수정 + TS SDK |
| PHP(Laravel) 개발자 | Dograh는 Docker 블랙박스로 두고 REST + 웹훅 연동 |
| 둘 다 | PHP=비즈니스 로직 / React=고객 화면 / Dograh=음성 엔진 |
| 밑바닥부터 제작 | Python + Pipecat 학습 (결국 같은 길) |

---

## 11. 수익화 아이디어

### 11.0 시장 계산 (한국 기준)

```
전화상담원 1명 인건비           : 월 약 250~300만원
24시간(야간·주말) 커버 시 4~5명   : 월 1,200만원+
AI 에이전트 24시간 운영          : 인프라 + API 월 50~150만원
→ 월 1,000만원 내외 절감 여지
```
고객은 "AI 기술"이 아니라 **"인건비 절감"**에 지불한다.

### 11.1 버티컬 SaaS (1순위 추천)

| 버티컬 | 해결 문제 | 예상 단가 |
|---|---|---|
| 병원/치과 | 예약 전화 폭주, 노쇼 | 월 30~80만원 |
| 학원 | 신규 상담 전화 누락 | 월 20~50만원 |
| 부동산 | 매물 문의 반복 응대 | 월 30~60만원 |
| 정비소 | 작업 중 전화 수신 불가 | 월 20~40만원 |
| 미용실 | 시술 중 전화 수신 불가 | 월 15~30만원 |
| 법무/세무 | 상담 문의 선별 | 월 50~100만원 |
| 동물병원 | 야간 응급 문의 | 월 30~60만원 |

- 모델: **기본료 월 29~99만원 + 초과 통화 분당 150~300원**
- 고객 100곳 × 월 50만원 = 월 5,000만원
- 강점: 업종 특화 템플릿 재사용 → 온보딩 10분, 범용 경쟁사 대비 차별화
- 시작법: 지인 업체 1곳 무료 3개월 → 실측 성과(놓친 전화 0건, 예약 +23% 등) → 동종 업계 영업

### 11.2 온프레미스 구축 SI (단가 최상)

| 타겟 | 이유 |
|---|---|
| 금융/보험/카드 | 금융보안 규제, 망분리 |
| 대형병원 | 의료법상 민감정보 |
| 공공기관/지자체 | 보안규정, 클라우드 제한 |
| 대기업 콜센터 | 내부 보안정책 |
| 채권추심/대부업 | 녹취 보관 의무 |

- 구축비 3,000만~2억원 / 유지보수 연 15~20% / 추가 워크플로우 건당 300~1,000만원
- 경쟁 우위: BSD 라이선스로 소스 납품 가능, Vapi/Retell은 온프레미스 자체 불가, Helm 차트 완비, **Asterisk ARI로 기존 사내 PBX 연결 가능**

### 11.3 AI 콜센터 BPO (대행)

| 서비스 | 과금 | 단가 예시 |
|---|---|---|
| 리드 검증 | 유효 리드당 | 5,000~30,000원 |
| 예약 확인/노쇼 방지 | 건당 | 200~500원 |
| 미수금 1차 안내 | 회수액 % | 3~10% |
| 설문/만족도 조사 | 완료 건당 | 1,000~3,000원 |
| 부재중 콜백 | 월 정액 | 30~100만원 |

- 무기: `campaign_orchestrator`의 동시성 제한 + 시간대 제한 + 재시도 + 서킷브레이커
- CSV의 추가 컬럼이 자동으로 `initial_context`가 되어 개인화 멘트에 주입됨

### 11.4 템플릿 / 에셋 마켓

| 상품 | 가격대 |
|---|---|
| 업종별 검증 워크플로우 팩(10종) | 30~100만원 |
| 한국어 특화 globalNode 프롬프트 | 10~30만원 |
| 이의제기 대응 스크립트 세트 | 20~50만원 |
| 온라인 강의 | 10~30만원/인 |
| 유료 커뮤니티 | 월 3~5만원 |

### 11.5 한국 특화 애드온

| 애드온 | 내용 |
|---|---|
| 국내 통신사 연동 | LG U+, KT 기업전화, 070 사업자 |
| 카카오 연동 | 통화 후 알림톡, 상담톡 전환 |
| 국내 CRM 연동 | 더존, 영림원, 자체 ERP 웹훅 |
| 한국어 STT 튜닝 | 사투리, 숫자/주소 인식 개선 |
| 국내 TTS | 네이버 클로바, 카카오 |
| 컴플라이언스 팩 | 동의 스크립트, 녹취 보관, AI 고지 |

전략: 일부는 오픈소스로 업스트림 기여(브랜딩·인지도) + 핵심은 상용 라이선스.

### 11.6 교육 / 컨설팅

| 상품 | 가격 |
|---|---|
| 기업 워크샵(1일) | 200~500만원 |
| PoC 대행(2~4주) | 500~2,000만원 |
| 기술 자문(월 정액) | 100~300만원 |
| 도입 타당성 리포트 | 300~800만원 |

### 11.7 최종 추천

> **1순위 버티컬 SaaS(병원/학원) + 2순위 온프레미스 SI.**
> 버티컬로 빠르게 현금흐름과 레퍼런스를 만들고, 그 레퍼런스로 억대 온프레미스를 수주하는 조합.
> 해자는 기술이 아니라 **업종 이해도**다.

---

## 12. 실행 로드맵

| 기간 | 목표 | 체크리스트 |
|---|---|---|
| 0~1개월 | 검증 | Docker 구동 / 한국어 STT·TTS 조합 탐색(`evals/` 활용) / **통화 1건 원가 계산** / 지인 업체 무료 설치 |
| 2~3개월 | 첫 매출 | 버티컬 1개 확정 / 템플릿 완성 / 유료 고객 3곳 / 성과 데이터 수집 |
| 4~6개월 | 확장 | 랜딩+데모영상 / 셀프서비스 가입 / 고객 20곳(월 1,000만원) / 온프레미스 제안서 1건 |
| 7~12개월 | 스케일 | 버티컬 2개 / 온프레미스 1건 수주 / 템플릿·강의 상품화 / 업스트림 기여 |

---

## 13. 리스크 체크리스트

| 리스크 | 대응 |
|---|---|
| 한국어 음성 품질 | 최우선 검증 항목. STT/TTS 조합 실측 후 착수 |
| 원가 관리 | LLM + STT + TTS + 통신 4중 과금. 통화당 마진 상시 계산 |
| 법규 | 정보통신망법(광고 전화 사전 동의), 개인정보보호법, 녹취 동의, AI 고지 |
| 경쟁 | 국내 대기업 진입 가능. 속도 + 버티컬 깊이로 대응 |
| 운영 부담 | 통화 실패 = 즉시 컴플레인. 모니터링/알림 필수 |
| 업스트림 의존 | 포크 관리 전략 수립 (`merge-pipecat-upstream` 스킬 참고) |

> 초기에는 광고성이 아닌 업무(예약 확인, 배송 안내, 설문, A/S 안내)부터 시작하는 것이 안전하다.

---

## 14. 참고 링크

### 저장소
- 본 포크: **https://github.com/bmshin94/dograh**
- 원본: **https://github.com/dograh-hq/dograh**
- 조직: **https://github.com/dograh-hq**
- 플러그인/스킬: **https://github.com/dograh-hq/dograh-plugins**
- Pipecat(음성 프레임워크): **https://github.com/pipecat-ai/pipecat**

### 서비스 / 문서
- 클라우드: **https://app.dograh.com**
- 공식 문서: **https://docs.dograh.com**
- MCP 가이드: **https://docs.dograh.com/integrations/mcp**
- Docker 배포: **https://docs.dograh.com/deployment/docker**
- 개발 환경 설정: **https://docs.dograh.com/contribution/setup**

### 패키지
- Python SDK: **https://pypi.org/project/dograh-sdk/**
- Node SDK: **https://www.npmjs.com/package/@dograh/sdk**

### 커뮤니티 / 레퍼런스
- Slack: **https://join.slack.com/t/dograh-community/shared_invite/zt-4787daqcn-3TDiQUh~3xrr3pwAqR9wpQ**
- GitHub Discussions: **https://github.com/orgs/dograh-hq/discussions**
- Product Hunt: **https://www.producthunt.com/products/dograh**
- Better Stack 소개 영상: **https://www.youtube.com/watch?v=xD9JEvfCH9k**
- Indie Hackers(창업자 회고): **https://www.indiehackers.com/post/i-built-a-voice-ai-platform-and-open-sourced-it-hit-1m-organic-impressions-6bfccb1df7**
- MCP 스펙: **https://modelcontextprotocol.io/**

---

*본 문서는 저장소 소스 코드와 공식 문서를 직접 조사하여 작성되었습니다. 가격·단가는 시장 추정치이며 실제 계약 조건과 다를 수 있습니다.*
