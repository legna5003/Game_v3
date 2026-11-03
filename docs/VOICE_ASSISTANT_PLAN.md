# 자비스(JARVIS)형 음성 비서 서비스 구축 플랜 — Gemini 단일 스택

작성일: 2026-11-03 (v2: Gemini 전면 채택으로 변경)
대상: 사내 테스트 → 외부 사용자 서비스로 확장
AI 엔진: **Google Gemini 전체 사용** (음성·판단·도구 호출·문서 이해 모두 Gemini)

---

## 0. 한 줄 결론

**가능하며, Gemini 단일 스택이 구조적으로 가장 단순합니다.**

- **음성 + 두뇌**: `gemini-3.8-live` 한 모델이 듣기·판단·도구 호출·말하기를 모두 처리 (speech-to-speech, 별도 STT/TTS 불필요)
- **무거운 작업(견적서 판독, 긴 문서, 코드)**: Live 모델이 `gemini-3.8-flash` / `gemini-3.8-pro` 서브에이전트를 도구처럼 호출
- **프레임워크**: Google **ADK**(Agent Development Kit) `run_live()` + **Vertex AI Agent Engine**(세션·메모리 뱅크 내장)
- **클라이언트**: 브라우저/앱이 **Ephemeral Token**으로 Gemini Live WebSocket에 직접 연결 → 오디오가 우리 서버를 거치지 않아 지연·인프라 최소
- **Google Workspace 연동**: 같은 Google 생태계라 OAuth·API·MCP가 가장 매끄러움

v1(Claude 두뇌 + 외부 음성 스택) 대비 장점: 구성 요소 3개 → 1개, 지연 최저, 끼어들기·감정 톤·선제 발화(proactive audio) 기본 제공.
단점: Google 종속, Live 모델 컨텍스트가 작아(약 128K) 긴 세션은 압축 필요, 복잡 추론은 Pro 모델 위임 필요.

---

## 1. 요구사항 → 구현 매핑

| # | 요구사항 | Gemini 스택 구현 |
|---|---|---|
| R1 | 음성으로 메일 발송 | Live 모델이 `gmail_send` 함수 호출 → 발송 전 음성 확인 게이트 |
| R2 | 견적서 업로드 | 사진/파일 → `gemini-3.8-flash` 서브에이전트가 거래처·금액·날짜 추출 → Drive 업로드 + Sheets 기록 |
| R3 | 스케줄 등록 | `calendar_create` 함수, 자연어 날짜 해석은 Live 모델이 담당 |
| R4 | 직접 만든 앱 조작 (GitHub) | 레포의 OpenAPI 스펙/`jarvis.manifest.json` → ADK `OpenAPIToolset`로 자동 도구화, 코드 작업은 Jules/Gemini CLI 비동기 |
| R5 | 비서 업무 전반 | Google Search 그라운딩, Memory Bank(장기 기억), 리마인더 |
| R6 | 조작 단순 | 푸시투톡 1버튼, 선제 발화로 "자비스"라고 부르면 반응 (proactive audio) |
| R7 | 외부 사용자 연동 용이 | "Google로 로그인" 1회로 Gmail/Drive/Calendar 동시 동의, 앱은 GitHub App 설치 1회 |

---

## 2. 모델 역할 분담

| 역할 | 모델 | 이유 |
|---|---|---|
| 실시간 대화·도구 호출 (메인) | `gemini-3.8-live` | 네이티브 오디오, 비동기 함수 호출 기본(NON_BLOCKING), 함수 스케줄링(SILENT / WHEN_IDLE / INTERRUPTED), 선제 발화 |
| 문서·이미지 판독, 긴 요약, 대량 메일 분류 | `gemini-3.8-flash` | 1M 컨텍스트, 비전·문서 이해, 저렴 |
| 복잡 판단(계약 검토, 다단계 계획) | `gemini-3.8-pro` | 고추론. Live가 `ask_expert(question)` 함수로 위임 |
| 코드 수정·배포·PR | Jules / Gemini CLI (비동기) | 음성 세션과 분리해 완료 시 결과만 보고 |

> 핵심 패턴: **Live 모델은 "접수·판단·보고"만 하고, 오래 걸리는 일은 서브에이전트에 넘긴 뒤 비동기 함수 결과가 오면 WHEN_IDLE로 자연스럽게 끼어들어 알려줌.** 사용자는 기다리지 않습니다.

---

## 3. 시스템 아키텍처

```
┌────────────────────┐  WebSocket(Ephemeral Token)  ┌──────────────────────────┐
│ 클라이언트            │ ◀──────────────────────────▶ │ Gemini Live API           │
│ 웹 PWA / 모바일      │   오디오 양방향 스트림          │ gemini-3.8-live           │
│ 푸시투톡 · 화면공유   │                              │ (VAD·끼어들기·음성 생성)   │
└─────────┬──────────┘                              └─────────────┬────────────┘
          │ 토큰 발급 / 로그인                                      │ function call (async)
          ▼                                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ Jarvis Agent (Google ADK, Python)  — Vertex AI Agent Engine 배포                │
│  LlmAgent(live) ── tools ──┬─ Workspace tools (Gmail/Drive/Calendar/Sheets)    │
│                            ├─ GitHub remote MCP                                │
│                            ├─ App Registry (OpenAPIToolset, 사용자 앱 자동 등록) │
│                            ├─ ask_expert → gemini-3.8-pro 서브에이전트          │
│                            ├─ read_document → gemini-3.8-flash 서브에이전트     │
│                            └─ google_search (그라운딩)                          │
│  정책 엔진: 발송/삭제/배포는 음성 확인 강제 (ADK Tool Confirmation + 자체 게이트)  │
│  Sessions(대화 상태) · Memory Bank(장기 기억) · Cloud Logging(감사)              │
└──────────────────────────────────────────────────────────────────────────────┘
          │                                   │
          ▼                                   ▼
   Secret Manager (사용자별 OAuth 토큰)   Firestore / Cloud SQL (앱 레지스트리, 감사로그)
```

### 3.1 연결 방식 두 가지 (둘 다 지원, 단계별 선택)

| 방식 | 흐름 | 장점 | 단점 | 사용 시점 |
|---|---|---|---|---|
| **A. 클라이언트 직결** | 서버가 Ephemeral Token 발급(모델·시스템 프롬프트·도구를 토큰에 고정) → 브라우저가 `@google/genai` `live.connect()`로 직접 연결 → 함수 호출은 클라이언트가 받아 우리 백엔드 REST로 실행 | 오디오 중계 서버 불필요, 지연 최저 | 함수 실행 왕복이 클라이언트 경유 | P1 MVP |
| **B. 서버 중계 (ADK run_live)** | 브라우저 ↔ 우리 서버(WebSocket) ↔ Gemini Live. ADK `LiveRequestQueue`로 오디오 전달, 도구는 서버에서 실행 | 도구·세션·메모리·정책을 서버가 완전 통제, 멀티유저 관리 쉬움 | 오디오 중계 비용, 지연 +100~200ms | P2 이후 정식 |

### 3.2 세션 운영 규칙 (Live API 제약 대응)

- 오디오 전용 세션은 **압축 없이 15분 제한** → `context_window_compression` 항상 켜기
- **session resumption** 켜기 (네트워크 끊김·토큰 만료 시 재접속), Ephemeral Token은 `uses: 1`, 30분 메시지 창
- 커뮤니티 보고: 압축을 켜도 약 58분에 세션이 닫힌 사례 있음 → 55분마다 선제적으로 재접속 로직 포함, 실측 필요
- Live 컨텍스트 ~128K → 긴 문서는 절대 Live에 넣지 않고 Flash 서브에이전트가 요약본만 반환

### 3.3 안전 정책 (필수)

| 행위 | 정책 |
|---|---|
| 읽기(메일 검색, 일정 조회) | 즉시 실행 (NON_BLOCKING, 결과는 WHEN_IDLE로 보고) |
| 생성(일정 등록, 업로드) | 요약 읽어주고 "등록할까요?" 1회 확인 |
| 발송·삭제·결제·배포 | 수신자/제목 읽어주고 명시적 "응/보내" 확인, 10초 무응답 시 취소, 발송 후 30초 "취소" 윈도우 |
| 사용자 앱 조작 | 매니페스트 `risk` 등급(`auto` / `confirm` / `strict`)에 따라 위 규칙 적용 |

확인은 모델 판단에 맡기지 않고 **서버 정책 엔진이 도구 실행 직전에 강제**합니다 (ADK Tool Confirmation 사용, 직결 모드에서는 백엔드 REST가 확인 토큰 요구).

---

## 4. Google Workspace 연동

| 서비스 | 1차(사내) | 2차(외부 사용자) |
|---|---|---|
| Gmail | Gmail REST API를 ADK 함수 도구로 래핑 (send, search, draft) | 동일 + Google 공식 Gmail 원격 MCP가 GA되면 교체 |
| Drive | Drive API (upload, search, 폴더 규칙) | 공식 Drive MCP(현재 Developer Preview, `drive.file` 스코프) |
| Calendar | Calendar API (create, list, freebusy) | 공식 Calendar MCP(현재 프리뷰, 읽기 위주 스코프라 생성은 API 유지) |
| Sheets | 견적서 장부 기록 | 동일 |

- OAuth: Google Identity "Google로 로그인" 1회에 필요한 스코프 일괄 동의 → refresh token을 Secret Manager에 사용자별 저장
- 사내는 Workspace 도메인 내부 OAuth 앱으로 심사 생략, 외부 서비스 전환 시 **Google OAuth 앱 검증(민감 스코프 심사)** 필요 — 4~6주 소요, P3 초반에 착수

---

## 5. "직접 만든 앱" 연동 (GitHub)

**레벨 1 — OpenAPI 스펙 또는 매니페스트 (권장, 가장 쉬움)**
레포 루트에 `openapi.yaml` 또는 `jarvis.manifest.json`을 두면 GitHub App 웹훅으로 읽어 ADK `OpenAPIToolset`이 함수 도구를 자동 생성. 각 엔드포인트에 `x-jarvis-risk: confirm` 같은 확장 필드로 확인 등급 지정.

```json
{
  "name": "재고관리앱",
  "openapi": "./openapi.yaml",
  "auth": { "type": "oauth2", "token_url": "https://inv.example.com/oauth/token" },
  "risk_overrides": { "POST /orders": "confirm", "DELETE /orders/{id}": "strict" }
}
```

**레벨 2 — 앱이 MCP 서버 노출**
Streamable HTTP MCP URL만 등록. ADK `MCPToolset`으로 연결.

**레벨 3 — 코드 레벨 조작**
"배포해", "버그 고쳐서 PR 올려" → Jules(비동기 코딩 에이전트) 또는 Gemini CLI를 Cloud Run Job으로 실행, 완료 시 Live 세션에 WHEN_IDLE 함수 결과로 "PR 올렸어요" 보고.

---

## 6. 핵심 업무 흐름 예시

**"자비스, 김부장님한테 어제 견적서 보내줘"**
1. Live 모델이 `contacts_search("김부장")`, `drive_search(name~견적, modified=yesterday)` 두 함수를 **동시에 비동기 호출**, "찾아볼게요" 하고 즉시 응답
2. 결과 도착 → 후보 여러 개면 "A사 견적서와 B사 견적서 중 어떤 거요?"
3. `gmail_send` 호출 → 정책 엔진이 가로채 확인 요구 → "김OO 부장님께 A사 견적서.pdf 첨부해서 보낼게요. 보낼까요?"
4. "응" → 발송 → "보냈습니다." (목표 왕복 3~5초)

**"이 견적서 올려줘"** (사진 첨부 또는 화면 공유)
1. Live가 `read_document(image)` → Flash 서브에이전트가 거래처·금액·날짜 JSON 반환
2. `/견적서/{연도}/{거래처}` 업로드 + Sheets 행 추가
3. "A사 320만원 견적서를 2026/A사 폴더에 올리고 장부에 기록했어요."

**선제 알림**: 서버 스케줄러가 회의 10분 전 세션에 텍스트 이벤트 주입 → 모델이 "10분 뒤 회의예요" 발화 (proactive audio)

---

## 7. 단계별 로드맵

| 단계 | 기간 | 산출물 | 완료 기준 |
|---|---|---|---|
| **P0 텍스트 PoC** | 1주 | ADK 에이전트 + Gmail/Calendar/Drive 도구 3개, `adk web` 텍스트 UI | 메일 발송·일정 등록·업로드 텍스트 성공 |
| **P1 음성 MVP** | 2주 | 방식 A(직결) 웹 PWA, Ephemeral Token 서버, 확인 게이트 | 한국어 음성으로 P0 시나리오 성공, 왕복 지연 p50 < 2초 |
| **P2 사내 파일럿** | 3~4주 | 방식 B(ADK run_live) 전환, Agent Engine 배포, 멀티유저 OAuth, Memory Bank, 견적서 Flash 판독, 앱 레벨1, 감사 로그 | 사내 5~10명 2주 사용, 성공률 ≥ 90%, 오발송 0건, 55분 재접속 안정 |
| **P3 제품화** | 4~6주 | Google OAuth 앱 검증, 셀프 온보딩 UI, 앱 등록 UI, 모바일, 과금 | 외부 사용자 10분 내 온보딩, 보안 점검 통과 |
| **P4 고도화** | 지속 | 화면/카메라 공유(멀티모달), 전화 연결, 아바타, Jules 코드 작업 | — |

---

## 8. 기술 스택

| 영역 | 선택 | 비고 |
|---|---|---|
| 에이전트 | Python 3.12, Google ADK (`google-adk`), `google-genai` SDK | ADK JS는 Live 스트리밍이 아직 PR 단계라 Python 고정 |
| 모델 | gemini-3.8-live / gemini-3.8-flash / gemini-3.8-pro | Vertex AI 경유(기업 데이터 처리 조건), 개발 초기엔 Gemini API 키 |
| 배포 | Vertex AI Agent Engine (Sessions, Memory Bank) + Cloud Run (토큰 발급·웹훅·REST) | |
| 클라이언트 | React PWA (TypeScript) + `@google/genai` live.connect | 이후 Flutter/React Native |
| 데이터 | Firestore(앱 레지스트리, 사용자), Cloud SQL(감사로그), Secret Manager(OAuth 토큰) | |
| 도구 | Workspace REST API 래핑, GitHub remote MCP, OpenAPIToolset, MCPToolset | |

---

## 9. 비용 개략 (대화 1분, 2차 출처 단가 기준 추정)

| 항목 | 추정 | 비고 |
|---|---|---|
| Live 오디오 입력 (약 1,900 토큰/분 × $3/M) | ~$0.006 | |
| Live 오디오 출력 (발화 비율 50% 가정 × $12/M) | ~$0.012 | |
| 시스템 프롬프트·도구 호출 텍스트 | ~$0.005 | |
| Flash 서브에이전트(문서 판독 건당) | ~$0.002 | |
| **합계** | **약 $0.02~0.04/분** | v1(Claude+외부 음성) 추정 $0.06~0.12/분 대비 1/3 수준 |

단가는 2차 출처 기준이므로 Google 공식 가격표로 재확인 필요. 방식 B는 Cloud Run 오디오 중계 비용 소폭 추가.

---

## 10. 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 세션 한도(15분/58분 보고) | 압축+재개 항상 켜기, 55분 선제 재접속, P2에서 장시간 부하 테스트 |
| 오디오 입출력 시 함수 호출 품질 저하(구 문서 경고) | 3.8 Live에서 실측, 함수 이름·설명 한국어 예시 포함해 명확화, 실패 시 "다시 말씀해 주세요" 복구 |
| 한국어 고유명사 오인식(거래처명) | 시스템 프롬프트에 사용자 연락처·거래처 사전 주입(캐시), 후보 재확인 |
| 선제 발화가 항상 켜져 있어 원치 않을 때 말함 | 푸시투톡 모드에서는 버튼 해제 시 마이크 스트림 중단 |
| 오발송·오삭제 | 서버 정책 엔진 강제 + 30초 취소 윈도우 + 감사 로그 |
| Google OAuth 민감 스코프 심사 지연 | P2 중 심사 서류 준비, 사내는 내부 앱으로 우회 |
| 매니페스트 프롬프트 인젝션 | 설명 길이 제한·검증, risk 등급 강제, 사용자 앱 도구는 별도 네임스페이스 |
| Google 종속 | 도구 레이어는 MCP/OpenAPI 표준으로 작성해 모델 교체 시 재사용 가능하게 유지 |

---

## 11. 바로 시작할 일 (P0 첫 주)

1. GCP 프로젝트 생성, Vertex AI·Gmail·Calendar·Drive·Sheets API 활성화, 내부용 OAuth 동의화면
2. `pip install google-adk` → `LlmAgent(model="gemini-3.8-live")` 골격, 도구 3개(gmail_send, calendar_create, drive_upload), 확인 게이트
3. `adk web`으로 텍스트 모드 검증 → `run_live()`로 음성 전환
4. Ephemeral Token 발급 엔드포인트 + 브라우저 `live.connect()` 샘플 (방식 A)
5. 사내 테스트 시나리오 10개(평가셋) + 지연·성공률 측정 스크립트

---

## 참고 출처

- Gemini 3.8 Live 모델 페이지: https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live
- Gemini 3.8 Live 개발자 가이드: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/gemini-3-8-live
- Live API 세션 관리(압축·재개): https://ai.google.dev/gemini-api/docs/live-api/session-management
- Live API 모범 사례: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/live-api/best-practices
- Ephemeral Tokens: https://ai.google.dev/gemini-api/docs/ephemeral-tokens
- ADK Live/음성 에이전트: https://adk.dev/live/index
- ADK 스트리밍 개발 가이드: https://google.github.io/adk-docs/streaming/dev-guide/part3
- Agent Engine Memory Bank: https://docs.cloud.google.com/agent-builder/agent-engine/memory-bank/set-up
- Google Workspace 공식 MCP(Drive 프리뷰): https://www.developersdigest.tech/mcp-servers/google-workspace-drive
- 58분 세션 종료 보고(커뮤니티): https://discuss.ai.google.dev/t/gemini-3-8-live-live-sessions-are-closed-at-exactly-58-00-with-1011-deadline-expired-resumption-afterwards-fails-with-1007/186392
- Gemini 3.8 Live 가격(2차 출처): https://aireiter.com/blog/gemini-3-8-live-api-pricing
- Gemini 3.8 Flash 가격(2차 출처): https://llmgateway.io/models/gemini-3.8-flash
