# 자비스(JARVIS)형 음성 비서 서비스 구축 플랜

작성일: 2026-11-03
대상: 사내 테스트 → 외부 사용자 서비스로 확장
AI 엔진: Claude (Anthropic) 고정

---

## 0. 한 줄 결론

**가능합니다.** 단, "음성"과 "두뇌"를 분리해야 합니다.

- **두뇌(판단·도구 호출)**: Claude Opus 5.5 (`claude-opus-5-5`) + Tool Use + MCP
- **음성 입출력**: Gemini 3.8 Live는 *Gemini가 두뇌인* speech-to-speech 모델이라 Claude를 끼워 넣을 수 없습니다. 따라서 **STT → Claude → TTS 캐스케이드** 구조(Pipecat 또는 LiveKit Agents)를 권장합니다.
- **가장 빠른 MVP**: ElevenLabs Agents(Claude 모델 내장 지원 + 커스텀 LLM 엔드포인트)로 2주 내 데모 가능.
- Anthropic은 2026-11 현재 **공개 speech-to-speech API를 제공하지 않습니다.** (Claude 앱 음성모드, Claude Code `/voice`는 있으나 개발자 API 아님) → Claude는 두뇌 역할, 음성은 외부 스택.

---

## 1. 요구사항 정리

| # | 요구사항 | 구현 방식 요약 |
|---|---|---|
| R1 | 음성으로 메일 발송 | STT → Claude가 `gmail.send` 도구 호출 → 발송 전 음성 확인 |
| R2 | 견적서 업로드 | 파일/사진 → Claude Vision으로 내용 파악 → Drive 폴더 규칙에 따라 업로드 + 시트 기록 |
| R3 | 스케줄 등록 | Google Calendar 도구, 자연어 날짜 해석은 Claude가 담당 |
| R4 | 직접 만든 앱 조작 (GitHub 연동) | 각 앱 레포에 `jarvis.manifest.json` 또는 MCP 서버 등록 → 자동 도구화 |
| R5 | 비서 업무 전반 (검색, 요약, 리마인드, 메모 등) | Claude 기본 능력 + 메모리 저장소 |
| R6 | 조작이 단순해야 함 | 푸시투톡 1버튼 + 선택적 웨이크워드 "자비스" |
| R7 | 사내 테스트 → 외부 사용자 연동이 쉬워야 함 | OAuth "Google로 연결" 원클릭, 앱 등록은 GitHub App 설치 1회 |

---

## 2. 음성 엔진 선택

### 2.1 후보 비교

| 후보 | 구조 | Claude를 두뇌로? | 한국어 | 장점 | 단점 |
|---|---|---|---|---|---|
| **Gemini 3.8 Live** (사용자 언급) | Speech-to-speech 단일 모델 | **불가** (Gemini가 두뇌). 우회하려면 "ask_jarvis" 함수 하나로 Claude에 위임 → LLM 2중 호출 | 지원 | 지연 최저, 끼어들기·감정 톤 자연스러움, 비동기 함수 호출 | Claude 교체 불가, 오디오 입출력 시 함수 호출 품질 저하 경고가 문서에 있음, 비용 이중 |
| **OpenAI gpt-realtime-2.x** | Speech-to-speech | 불가 (동일 이유) | 지원 | 성숙한 생태계 | Claude 불가, 오디오 토큰 단가 높음 |
| **ElevenLabs Agents** | 관리형 캐스케이드 (STT+LLM+TTS) | **가능** (Anthropic 모델 선택 또는 Custom LLM URL) | 지원 (32+ 언어, 한국어 포함 — 실측 필요) | 가장 빠른 MVP, 웹/모바일 SDK·전화 연결·MCP 서버 연결 기본 제공, 음성 품질 최상 | 벤더 종속, 분당 과금이 자체 구축보다 비쌈, 도구 로직을 ElevenLabs 측 또는 Custom LLM 서버에 둬야 함 |
| **Pipecat / LiveKit Agents (자체 구축 캐스케이드)** ★권장 | 오픈소스 프레임워크 + STT/TTS 벤더 조합 | **가능** (Anthropic 공식 통합) | STT/TTS 벤더에 따라 다름 | Claude·도구·메모리 완전 통제, 벤더 교체 자유, 장기적으로 단가 최저 | 초기 개발 2~3주 추가, WebRTC 인프라 운영 필요 |

### 2.2 권장안

**1단계(사내 PoC): ElevenLabs Agents + Custom LLM(=우리 Claude 서버)**
빠르게 "말하면 메일이 간다"를 2주 안에 증명. 도구 실행은 전부 우리 서버(Claude Tool Runner)에서 처리하므로, 나중에 음성 레이어를 바꿔도 두뇌 코드는 그대로 유지됩니다.

**2단계(정식): Pipecat 기반 자체 캐스케이드**
- STT: Deepgram Nova-3 (ko 스트리밍) 또는 Google Chirp 3 (한국어 강점) — 둘 다 테스트 후 선택
- LLM: Claude Opus 5.5 (복잡 업무) / Claude Haiku 5.5 (의도분류·간단 응답, 지연 단축)
- TTS: ElevenLabs Flash v2.5 (한국어, ~75ms) 또는 Cartesia Sonic-3.5
- 전송: WebRTC (Daily 또는 LiveKit)

> Gemini 3.8 Live를 꼭 쓰고 싶다면 "음성 프론트" 전용으로 두고 모든 판단을 `ask_jarvis(text)` 함수로 Claude에 넘기는 하이브리드가 가능하지만, 응답이 Gemini를 한 번 더 거쳐 재구성되므로 지연·비용·일관성 모두 불리합니다. 비추천.

---

## 3. 시스템 아키텍처

```
┌──────────────┐   WebRTC/WS    ┌───────────────────────────┐
│ 클라이언트     │ ─────────────▶ │ Voice Gateway (Pipecat)    │
│ 웹 PWA / 모바일│ ◀───────────── │  VAD → STT → [두뇌] → TTS   │
│ 푸시투톡/웨이크 │                └─────────────┬─────────────┘
└──────────────┘                               │ text + session
                                               ▼
                              ┌────────────────────────────────┐
                              │ Jarvis Brain Service            │
                              │  Claude Opus 5.5 Tool Runner    │
                              │  - 시스템 프롬프트(캐시)          │
                              │  - 사용자 메모리/프로필            │
                              │  - 확인(confirm) 정책 엔진        │
                              └──────┬───────────┬─────────────┘
                                     │ MCP       │ MCP
                     ┌───────────────▼──┐   ┌────▼──────────────────┐
                     │ Workspace MCP    │   │ App Registry MCP       │
                     │ Gmail/Drive/Cal  │   │ GitHub App → 각 앱의    │
                     │ (OAuth per user) │   │ manifest/MCP 자동 등록   │
                     └──────────────────┘   └────────────────────────┘
                                     │
                              ┌──────▼──────┐
                              │ Token Vault │  사용자별 OAuth 토큰(암호화)
                              │ Audit Log   │  모든 도구 호출 기록
                              └─────────────┘
```

### 3.1 구성 요소

1. **클라이언트** — 웹 PWA(React) 우선. 버튼 하나(누르고 말하기). 선택: Picovoice Porcupine로 "자비스" 웨이크워드(온디바이스). 모바일은 PWA → 필요 시 React Native.
2. **Voice Gateway** — Pipecat(Python). VAD·끼어들기 처리, STT/TTS 벤더 플러그인 교체 가능.
3. **Jarvis Brain** — Anthropic SDK `tool_runner`. Claude Opus 5.5, adaptive thinking, `effort: medium`(대화) / `high`(복합 업무), 프롬프트 캐싱, `fallbacks: "default"` 활성화. 도구 결과를 음성 친화적 짧은 문장으로 요약하도록 시스템 프롬프트 설계.
4. **MCP 도구 서버**
   - Google Workspace: Google 공식 원격 MCP(Gmail/Calendar/Drive, 현재 Developer Preview·일부 읽기 전용 스코프) 또는 커뮤니티 `google_workspace_mcp`(OAuth 2.1 멀티유저, 발송/생성 가능). **발송·생성이 필요하므로 초기엔 커뮤니티 서버 또는 자체 구현.**
   - GitHub: GitHub 공식 원격 MCP 서버.
   - App Registry: 사용자가 만든 앱을 도구화 (4절).
5. **Token Vault / Audit** — 사용자별 OAuth refresh token 암호화 저장(KMS), 모든 도구 호출·확인 응답 로깅.

### 3.2 안전 정책 (필수)

| 행위 유형 | 정책 |
|---|---|
| 읽기(메일 검색, 일정 조회) | 즉시 실행 |
| 생성(일정 등록, 드라이브 업로드) | 요약 읽어주고 "등록할까요?" 1회 확인 |
| 발송·삭제·결제·배포 | 수신자/제목 읽어주고 명시적 "응/보내" 확인, 10초 내 미응답 시 취소 |
| 앱 조작 | 매니페스트의 `risk` 등급에 따라 위와 동일 규칙 적용 |

확인은 Claude에게 맡기지 않고 **서버 정책 엔진**이 강제합니다(도구 `run()` 안에서 게이트).

---

## 4. "직접 만든 앱" 연동 설계 (GitHub)

목표: 개발자가 자기 앱을 자비스에 붙이는 데 **파일 1개 + GitHub App 설치 1회**.

### 4.1 레벨 1 — 매니페스트 (가장 쉬움)
레포 루트에 `jarvis.manifest.json`:

```json
{
  "name": "재고관리앱",
  "base_url": "https://inv.example.com/api",
  "auth": { "type": "oauth2", "token_url": "..." },
  "actions": [
    {
      "name": "create_order",
      "description": "거래처에 발주서를 생성한다",
      "method": "POST", "path": "/orders",
      "input_schema": { "type": "object", "properties": { "vendor": {"type":"string"}, "items": {"type":"array"} }, "required": ["vendor","items"] },
      "risk": "confirm"
    }
  ]
}
```
자비스 서버가 GitHub App 설치 이벤트/웹훅으로 매니페스트를 읽어 Claude 도구로 자동 변환합니다. 기존 OpenAPI 스펙이 있으면 그대로 지정해도 됩니다.

### 4.2 레벨 2 — 앱이 MCP 서버를 노출
앱이 Streamable HTTP MCP 엔드포인트를 제공하면 URL만 등록. Claude API의 MCP 커넥터(`mcp_servers` + `mcp_toolset`)로 바로 연결.

### 4.3 레벨 3 — 코드 레벨 조작 (개발자용)
"배포해", "이 버그 고쳐서 PR 올려" 같은 요청은 **Claude Agent SDK** 또는 **Claude Managed Agents**(GitHub 레포 마운트, 샌드박스 실행)로 비동기 작업을 돌리고, 완료 시 음성으로 결과 보고. 음성 대화 지연과 분리하기 위해 반드시 비동기 잡으로 처리.

---

## 5. 핵심 업무 흐름 예시

**"자비스, 김부장님한테 어제 견적서 보내줘"**
1. STT → "김부장님한테 어제 견적서 보내줘"
2. Claude: `contacts.search("김부장")` → `drive.search(type=pdf, name~견적, modified=yesterday)`
3. 후보 1개면 바로, 여러 개면 "A사 견적서와 B사 견적서 중 어떤 거요?"
4. Claude: `gmail.send` 호출 시도 → 정책 엔진이 가로채 "김OO 부장님께 'A사 견적서.pdf' 첨부해서 보낼게요. 보낼까요?"
5. "응" → 발송 → "보냈습니다." (전 과정 3~6초 목표)

**"이 견적서 올려줘"** (사진/파일 첨부)
1. Claude Vision이 거래처·금액·날짜 추출
2. 폴더 규칙(`/견적서/{연도}/{거래처}`)에 업로드, 시트에 행 추가
3. "A사 320만원 견적서를 2026/A사 폴더에 올리고 장부에 기록했어요."

---

## 6. 단계별 로드맵

| 단계 | 기간 | 산출물 | 완료 기준 |
|---|---|---|---|
| **P0 텍스트 PoC** | 1~2주 | Claude Tool Runner + Gmail/Calendar/Drive 도구, 텍스트 채팅 UI | 메일 발송·일정 등록·파일 업로드 3개 시나리오 텍스트로 성공 |
| **P1 음성 MVP** | 2~3주 | ElevenLabs Agents(Custom LLM=P0 서버) 또는 Pipecat 최소 구성, 웹 푸시투톡 | 한국어 음성으로 P0 시나리오 성공, 왕복 지연 p50 < 2.5초 |
| **P2 사내 파일럿** | 3~4주 | 멀티유저 OAuth, 확인 정책 엔진, 감사 로그, 견적서 Vision 업로드, 앱 매니페스트 레벨1 | 사내 5~10명 2주 사용, 작업 성공률 ≥ 90%, 오발송 0건 |
| **P3 제품화** | 4~6주 | Pipecat 자체 캐스케이드 전환, "Google로 연결" 온보딩, 앱 등록 UI, 웨이크워드, 모바일, 과금 | 외부 사용자 셀프 온보딩 10분 이내, 보안 점검 통과 |
| **P4 고도화** | 지속 | 장기 메모리, 선제 알림("10분 뒤 회의예요"), 전화(SIP) 연결, Managed Agents 비동기 작업 | — |

---

## 7. 기술 스택 (가정)

| 영역 | 선택 | 비고 |
|---|---|---|
| Brain / API | Python 3.12, FastAPI, `anthropic` SDK (tool runner) | Pipecat이 Python이라 통일 |
| Voice | Pipecat (P3) / ElevenLabs Agents (P1) | WebRTC: Daily 또는 LiveKit |
| 클라이언트 | React PWA (TypeScript) | 이후 React Native |
| MCP | Google Workspace MCP, GitHub remote MCP, 자체 App Registry MCP | Streamable HTTP |
| 저장소 | Postgres (사용자/앱 레지스트리/감사로그), Redis (세션) | 토큰은 KMS 암호화 |
| 배포 | Docker + 클라우드 (Cloud Run/ECS) | 음성 게이트웨이는 상시 프로세스 |

> 사내 기존 스택이 TypeScript 중심이면 LiveKit AgentsJS + `@anthropic-ai/sdk`로 전부 TS로 갈 수도 있습니다. 결정 필요.

---

## 8. 비용 개략 (대화 1분 기준, 2026-11 공개 단가 기준 추정)

| 항목 | 추정 | 비고 |
|---|---|---|
| STT (Deepgram 스트리밍) | ~$0.008 | |
| TTS (ElevenLabs Flash, 한국어 ~300자/분) | ~$0.03~0.06 | 요금제에 따라 변동 |
| Claude Opus 5.5 (턴당 캐시된 시스템 프롬프트 + 짧은 출력) | ~$0.02~0.05 | 캐싱 적용 시, Haiku 5.5 라우팅 시 1/10 |
| **합계** | **약 $0.06~0.12/분** | ElevenLabs Agents 관리형은 이보다 높음(분당 과금) |
| 비교: Gemini 3.8 Live 오디오 | 입력 $3/M, 출력 $12/M 토큰 (2차 출처) | Claude 비용이 별도로 추가됨 |

---

## 9. 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 한국어 STT 오인식(고유명사·거래처명) | 사용자별 연락처/거래처 사전을 STT 키워드 부스트 + Claude 후보 재확인 |
| Google 공식 MCP가 Preview·읽기 전용 스코프 | 발송/생성은 자체 OAuth 통합으로 구현, 공식 MCP GA 시 교체 |
| 오발송/오삭제 | 서버 측 확인 정책 강제, 발송 후 30초 "취소" 윈도우 |
| 지연(캐스케이드 3단) | 스트리밍 TTS, Haiku 5.5로 의도 분류 선처리, 프롬프트 캐싱, 도구 병렬 호출 |
| Claude 안전 분류기 거절(`stop_reason: refusal`) | `fallbacks: "default"` 활성화 + 사용자에게 "다시 말씀해 주세요" 응답 |
| 외부 사용자 OAuth 토큰 유출 | KMS 암호화, 최소 스코프, 토큰 회전, 감사 로그 |
| 앱 매니페스트의 악의적 설명(프롬프트 인젝션) | 매니페스트 검증·샌드박스, 도구 설명 길이 제한, `risk` 등급 강제 |

---

## 10. 바로 시작할 일 (P0 첫 주)

1. Google Cloud 프로젝트 생성, Gmail/Calendar/Drive API 활성화, OAuth 동의화면(내부용) 설정
2. `anthropic` SDK로 Brain 서버 골격 작성: 시스템 프롬프트, 3개 도구(gmail.send, calendar.create, drive.upload), 확인 게이트
3. ElevenLabs Agents 계정 생성 → Custom LLM URL로 Brain 서버 연결, 한국어 음성 선택
4. 사내 테스트 시나리오 10개 작성(평가셋) → 성공률·지연 측정 스크립트
5. 기술 스택(Python vs TypeScript) 확정

---

## 참고 출처

- Gemini 3.8 Live 개발자 가이드: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/gemini-3-8-live
- Gemini Live API 개요: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/live-api
- Claude 음성모드 업데이트(TechCrunch, 2026-07): https://techcrunch.com/2026/07/23/anthropic-updates-claude-voice-mode-with-more-capable-models/
- Claude Code 음성모드(TechCrunch, 2026-03): https://techcrunch.com/2026/03/03/claude-code-rolls-out-a-voice-mode-capability/
- ElevenLabs Agents 문서(한국어): https://elevenlabs.io/docs/ko/eleven-agents
- ElevenLabs 변경로그(Claude 모델 지원): https://elevenlabs.io/docs/changelog/2026/3/9
- Pipecat / LiveKit 비교: https://futureagi.com/blog/livekit-alternatives-2026/
- Google Workspace MCP 서버(공식 Drive/Calendar 프리뷰): https://www.developersdigest.tech/mcp-servers/google-workspace-drive
- Deepgram 한국어 스트리밍 모델: https://developers.deepgram.com/changelog/2026/8/4
- Gemini 3.8 Live 가격(2차 출처): https://aireiter.com/blog/gemini-3-8-live-api-pricing
- OpenAI Realtime 가격(2차 출처): https://aireiter.com/blog/openai-realtime-api-pricing
