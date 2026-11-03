# Tradie Voice Assistant — 호주 트레이디·빌더 사장님용 음성 비서 플랜 v4

작성일: 2026-11-03
v4 변경점: 타깃을 **호주 트레이디·빌더 사업주**로 확정, 핵심 개체를 **프로젝트(현장 주소)**로 재설계, 도면·견적·SMS 업무 추가
AI 스택: Google Gemini 전체 (gemini-3.8-live / 3.8-flash / 3.8-pro, ADK, Vertex AI Agent Engine, 리전 australia-southeast1)

---

## 0. 전략 요약

**타깃**: 호주의 1~10인 규모 트레이디·빌더 사업주 (빌더, 전기, 배관, 목수, 타일, 조경 등). 하루 대부분을 유트(ute)와 현장에서 보내며, 손이 더럽거나 바쁘고, 사무 업무는 저녁에 몰아서 한다. **음성이 손보다 편한 상황이 하루 종일 이어지는 집단**이라 음성 비서의 수요 형태와 정확히 맞는다.

**시장 공백**: ServiceM8·Tradify·Fergus·simPRO 등 기존 잡 관리 앱은 2026년 현재 사장님용 음성 비서가 없다. 제3자 "AI 전화 접수"(월 $199 수준)는 *걸려오는 전화*를 받는 제품이지, 사장님이 *시키는 일*을 하는 비서가 아니다.

**승부처 3가지 (v3 계승)**

| 승부처 | 이 시장에서의 의미 | 성공 지표 |
|---|---|---|
| **① 신뢰: 고객에게 잘못 나가는 것 0건** | 견적·메일·SMS는 고객에게 직접 가는 문서. 오발송 한 번이면 끝 | 파일럿 오발송·오삭제 0건 |
| **② 좁고 깊은 업무** | "주소 하나로 그 현장의 모든 것"을 음성으로 | 5개 핵심 업무 음성 완료율 ≥ 90% |
| **③ 연동 용이** | Google 계정 1회 연결 + 쓰던 잡 관리 앱(ServiceM8/Tradify/Xero) 그대로 | 온보딩 10분, 기존 앱 교체 요구 없음 |

**하지 않을 것**: 잡 관리 앱을 새로 만들지 않는다(빅3와 경쟁 안 함). 인바운드 전화 접수는 하지 않는다(이미 있는 시장). 범용 비서도 하지 않는다.

---

## 1. 핵심 개체: 프로젝트 = 현장 주소

모든 업무는 **주소**로 시작한다. "12 Smith Street 견적 올려줘", "그 현장 도면 보여줘".

### 1.1 데이터 모델

```
Project
  address        "12 Smith St, Parramatta NSW 2150"   (정규화: Google Address Validation API)
  aliases        ["Smith St job", "the Parramatta reno", "Johnson's place"]
  client         { name, phone, email }
  drive_folder   /Jobs/12 Smith St Parramatta/
                   ├ Quotes/        Q-2026-041.pdf …
                   ├ Plans/         A-101 Site Plan.pdf, A-201 Floor Plan.pdf …
                   ├ Photos/
                   ├ Invoices/
                   └ Correspondence/
  status         lead | quoted | won | in_progress | invoiced | closed
  external_ids   { servicem8_job: "...", xero_contact: "..." }
  plan_index     [ {sheet:"A-201", title:"Ground Floor Plan", scale:"1:100", rev:"C", page:3, summary:"..."} … ]
```

### 1.2 주소 해석 규칙
- 음성으로 들어온 주소·별칭·고객 이름을 **프로젝트 목록과 퍼지 매칭**. 후보 1개면 확정, 2개 이상이면 선택지 제시 ("Smith Street가 Parramatta랑 Penrith 두 곳인데 어디요?").
- 사용자가 "그 현장", "아까 그 집"이라고 하면 세션 내 마지막 프로젝트를 참조.
- 새 주소가 나오면 "새 프로젝트로 만들까요?" (`confirm`).
- 프로젝트 목록·별칭·고객명·교외(suburb)명을 세션 시스템 프롬프트에 주입해 음성 인식 정확도를 올린다.

### 1.3 드라이브 폴더 규약
- 자비스가 폴더 구조를 만들고 유지. 사용자가 다른 구조를 쓰고 있으면 온보딩 때 **매핑 마법사**로 기존 폴더를 프로젝트에 연결.
- 파일명 규약: `Q-{연도}-{번호}` 견적, `INV-` 청구서, 도면은 원본 시트 번호 유지.

---

## 2. 승부처 ② 핵심 업무 5가지

### 업무 A. 견적서(Quote)
| 요청 예시 | 동작 | 등급 |
|---|---|---|
| "12 Smith St 견적 읽어줘" | Drive에서 최신 견적 찾기 → Flash가 요약(총액·GST·항목 수·유효기간) → 읽어줌 | auto |
| "그 견적 고객한테 보내줘" | 수신자(고객) 확정 → 메일 초안(템플릿) → **고객명·주소·총액 복창** → 발송 | strict |
| "견적에 데크 항목 $3,200 추가해서 새 버전 만들어" | 견적 템플릿(Sheets/Docs)에 항목 추가 → 새 PDF 생성 → 저장 → 요약 읽어줌 | confirm |
| "Smith St 견적 승인됐어, 잡으로 바꿔" | status → won, ServiceM8/Tradify에 잡 생성(연동 시) | confirm |
| "이번 달 나간 견적 총액은?" | Sheets 집계 | auto |

호주 요건: GST 10% 표시, ABN, 유효기간, 지급 조건. 견적 PDF는 Docs 템플릿 → PDF 변환.

### 업무 B. 도면(Plans/Drawings)
| 요청 예시 | 동작 | 등급 |
|---|---|---|
| "Smith St 도면 올려줘" (파일/사진) | 업로드 → **Flash가 시트별 인덱싱**(시트번호·제목·축척·리비전·요약) → plan_index 저장 → "시트 12장, 리비전 C로 저장했어요" | confirm |
| "그 현장 1층 평면도 보여줘" | plan_index에서 시트 선택 → 휴대폰 화면에 렌더 + 요약 읽기 | auto |
| "북쪽 입면의 창문 치수 확인해줘" | 시트 선택 → **Pro가 해당 영역 판독** → "창문 스케줄 기준 W3는 1200×900, 북쪽 입면에 3개" + 화면에 해당 영역 확대 표시 | auto |
| "주방 벽 길이 얼마야?" | 치수선에 **표기된 숫자**를 읽음. 표기가 없으면 "치수가 적혀 있지 않아요, 축척 1:100 기준 약 4.2m인데 현장 실측 권장" | auto |
| "리비전 B랑 C 뭐가 달라?" | 두 시트 비교 요약 | auto |

**원칙**: 도면 수치는 항상 **도면에 적힌 텍스트를 우선**하고, 축척으로 환산한 값은 "약"과 "실측 권장"을 붙여 말한다. 구조·전기 규정 판단은 하지 않는다(책임 경계).

### 업무 C. 메일·메시지(Email / SMS)
| 요청 예시 | 동작 | 등급 |
|---|---|---|
| "오늘 온 메일 중 급한 거 읽어줘" | Gmail 조회 → Flash 분류(고객/공급업체/청구/스팸) → 요약 | auto |
| "Johnson한테 내일 8시 도착한다고 문자 보내줘" | 고객 전화번호 확정 → SMS 초안 → **수신자·내용 복창** → 발송 | strict |
| "Reece에 PO 보내면서 도면 첨부해줘" | 공급업체 메일 + Drive 파일 첨부 → 복창 → 발송 | strict |
| "답장 초안 써놔" | Gmail 초안 저장 (발송 안 함) | confirm |

SMS는 호주 전용 번호(long code)로 발송해 ACMA Sender ID 등록 의무(알파 태그만 해당)를 피하고, 고객 회신도 같은 번호로 받아 세션에 보고한다. 제공사: Sinch(MessageMedia) 또는 Twilio AU.

### 업무 D. 캘린더·현장 일정
| 요청 예시 | 동작 | 등급 |
|---|---|---|
| "내일 오전 Smith St 2시간 잡아줘" | 겹침 확인 → 이벤트 생성(주소 포함, 내비 링크) | confirm |
| "목요일 전기 검사 일정 잡고 전기공한테 문자해줘" | 캘린더 + SMS 연쇄 | confirm + strict |
| "오늘 일정 읽어줘 / 다음 현장 어디야?" | 조회 + 이동시간 | auto |
| 회의 20분 전 | 선제 발화 "20분 뒤 Parramatta 현장이에요, 지금 출발하면 돼요" | — |

### 업무 E. 현장 기록
| 요청 예시 | 동작 | 등급 |
|---|---|---|
| "사진 찍어서 Smith St에 저장해, 배관 완료" | 사진 → Photos/ 업로드 + 캡션 + 날짜 | auto |
| "오늘 Smith St 작업 메모: 데크 프레임 완료, 내일 데킹" | Correspondence/에 일지 추가, 필요 시 고객 진행 보고 초안 | auto / confirm |

### 업무 확장 기준 (유지)
주 3회 이상 반복 · 손으로 3분 이상 · 음성이 편한 상황 · 되돌리기 가능 — 4개 모두 만족 시에만 추가.

---

## 3. 승부처 ① 신뢰 설계 — 고객에게 잘못 나가는 것 0건

| 등급 | 대상 | 규칙 |
|---|---|---|
| `auto` | 조회·읽기·내부 저장(사진, 메모) | 즉시 실행 |
| `confirm` | 일정 생성, 견적 새 버전, 초안 저장, 새 프로젝트 | 요약 후 "할까요?" 1회 |
| `strict` | 고객·공급업체에게 **나가는 모든 것**(메일·SMS·견적 발송), 삭제, 외부 앱 쓰기(잡 생성·청구) | **서버 고정 템플릿**으로 수신자·주소·금액 복창, 명시적 긍정만 허용, 10초 무응답 취소 |

- 등급은 **서버 정책 엔진이 강제**. 모델이 호출해도 확인 토큰 없이는 실행되지 않는다.
- **고유명사 사전**: 프로젝트 주소·별칭, 고객·공급업체 이름, 교외명을 세션에 주입. 유사 이름(Johnson/Johnston)은 선택지 강제.
- **금액·치수 복창**: "three thousand two hundred dollars including GST".
- **발송 후 30초 취소 창** (메일·SMS 모두 지연 큐).
- **영구 삭제 없음**, 휴지통만.
- **감사 로그 + 저녁 요약**: "오늘 자비스가 한 일" 메일/푸시를 매일 17:30에 발송. 트레이디가 퇴근길에 확인하는 습관을 만든다.
- **책임 경계 문구**: 도면 치수·규정 관련 답변엔 "실측/인증 전문가 확인 권장"을 서버가 붙인다.

---

## 4. 승부처 ③ 연동 — "Google 1회 + 쓰던 앱 그대로"

### 4.1 온보딩 (목표 10분)
1. "Google로 로그인" 1회 → Gmail·Drive·Calendar·Contacts 동의
2. Drive 폴더 매핑 마법사 (기존 폴더 → 프로젝트 자동 인식, 주소 추출은 Flash)
3. 호주 SMS 번호 발급 (자동)
4. 선택: ServiceM8 / Tradify / Xero 연결 (OAuth)
5. 음성 테스트 1회 ("내 프로젝트 몇 개야?")

### 4.2 외부 앱 연동 우선순위
| 앱 | 용도 | 시점 |
|---|---|---|
| Xero | 견적→청구서 전환, 입금 확인 읽기 | P3 |
| ServiceM8 | 잡 생성·조회·일정 동기화 (iPhone 배관·전기 다수) | P3 |
| Tradify | 동일 (안드로이드·범용) | P3 |
| Outlook/Microsoft 365 | Gmail 안 쓰는 빌더 | P4 |
| 자체 앱 (GitHub 매니페스트) | 개발 역량 있는 고객·파트너용, v3 규격 유지 | P4 |

모든 외부 앱 쓰기 동작은 `strict`. 연동은 OpenAPI/MCP 표준 도구 레이어로 작성해 모델·앱 교체 가능.

---

## 5. 도면 판독 파이프라인 (기술 상세)

```
업로드(PDF/사진) ─▶ 페이지 분할 ─▶ Flash: 시트별 인덱싱 ─▶ plan_index(Firestore) + 페이지 PNG(Drive)
                                    (시트번호·제목·축척·리비전·범례·요약)

질문 ─▶ Live: 프로젝트·시트 선택(plan_index 검색) ─▶ Pro: 해당 시트(+필요 시 인접 시트) 판독
       ─▶ 답변 텍스트 + bbox ─▶ 서버: 해당 영역 크롭 이미지 생성 ─▶ 휴대폰 화면 표시 + 음성 답변
```

- 인덱싱은 업로드 시 1회(비동기, WHEN_IDLE로 "저장 끝났어요" 보고).
- 질문 시 Live에는 도면 원본을 절대 넣지 않는다(컨텍스트 128K). Pro 서브에이전트가 해당 시트만 본다.
- 답변 형식: `{answer, sheet, region_bbox, source_text, confidence, caveat}` → 서버가 caveat를 발화에 강제 삽입.
- 평가셋: 실제 호주 주택 도면 10세트 × 질문 10개 = 100문항 (치수 읽기, 스케줄 조회, 시트 찾기, 리비전 비교). 목표 정확도: 표기 치수 읽기 ≥ 95%, 시트 찾기 ≥ 95%.

---

## 6. 시스템 아키텍처 (Gemini 단일 스택)

```
┌──────────────────┐ WebSocket(Ephemeral Token) ┌─────────────────────┐
│ 모바일 앱 (iOS/안드) │ ◀────────────────────────▶ │ Gemini Live API      │
│ 푸시투톡·CarPlay/   │        오디오 양방향         │ gemini-3.8-live      │
│ Android Auto·화면  │                            └──────────┬──────────┘
└────────┬─────────┘                                       │ function call (async)
         │                                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Jarvis Agent (Google ADK, Python) — Vertex AI Agent Engine (au-southeast1) │
│  정책 엔진(auto/confirm/strict, 30초 취소 큐, 감사 로그)                   │
│  프로젝트 리졸버(주소·별칭 퍼지 매칭, 사전 주입)                             │
│  도구: Gmail·Drive·Calendar·Contacts·Sheets/Docs(견적) · SMS(Sinch/Twilio) │
│        Xero·ServiceM8·Tradify(OpenAPI/MCP) · 자체 앱 매니페스트             │
│  서브에이전트: read_plan/index_plan → 3.8-pro/flash · classify_mail → flash │
│  Sessions · Memory Bank · 선제 알림 스케줄러                               │
└────────────────────────────────────────────────────────────────────────┘
   Secret Manager(OAuth)  Firestore(프로젝트·plan_index)  Cloud SQL(감사로그)
```

- **클라이언트는 모바일 앱이 1순위**(유트·현장). CarPlay/Android Auto 음성 버튼, 잠금 화면 위젯. 웹은 사무실용 보조.
- 세션: 압축·재개 항상 켜기, 55분 선제 재접속, 운전 중 네트워크 끊김 대비 오프라인 큐(명령 저장 후 재연결 시 실행, `strict`는 제외).
- 언어: 호주 영어 1차. Live 모델은 악센트·트레이디 용어("sparky", "chippy", "reno", "slab")를 시스템 프롬프트 용어집으로 보강.
- 데이터 거주: Vertex AI australia-southeast1, Drive는 사용자 소유(우리는 복사본 최소화).

---

## 7. 로드맵

| 단계 | 기간 | 검증 | 산출물 | 완료 기준 |
|---|---|---|---|---|
| **P0 텍스트 PoC** | 1주 | 업무 정의 | ADK 에이전트, 프로젝트 리졸버, 업무 A·D 도구, 정책 엔진 골격 | 텍스트로 "주소→견적 읽기→발송" 완료 |
| **P1 음성 MVP** | 3주 | ① 신뢰 | 모바일 앱(직결), strict 템플릿, 30초 취소, 사전 주입, SMS 번호, 업무 A·C·D | 음성 완료율 ≥ 80%, 오발송 0, 지연 p50 < 2초 |
| **P2 도면·파일럿** | 5주 | ② 깊이 | 도면 인덱싱·판독, 화면 표시, 업무 B·E, Agent Engine 서버 중계, 저녁 요약, 감사 대시보드 | 사내+지인 트레이디 5명 3주 사용, 완료율 ≥ 90%, 오발송 0, 도면 치수 읽기 ≥ 95% |
| **P3 연동·베타** | 4주 | ③ 연동 | Xero·ServiceM8·Tradify 연동, 온보딩 마법사, Google OAuth 심사, 과금 | 외부 트레이디 10명 베타, 온보딩 10분 |
| **P4 출시** | 6주 | 전체 | Outlook, 자체 앱 매니페스트, CarPlay/Android Auto, 호주 개인정보법 점검, 마케팅 | 유료 전환 ≥ 30%, 주당 15회 이상 사용 |
| **P5 확장** | 지속 | — | 뉴질랜드, 인바운드 전화 접수 파트너 연동, 팀원 계정, 업무 확장 | — |

**단계 전환 규칙**: 오발송·오삭제 1건이라도 발생하면 다음 단계로 넘어가지 않는다.

---

## 8. 기술 스택

| 영역 | 선택 |
|---|---|
| 에이전트 | Python 3.12, `google-adk`, `google-genai` |
| 모델 | gemini-3.8-live(대화) / 3.8-flash(인덱싱·분류·요약) / 3.8-pro(도면 판독·복잡 판단) |
| 배포 | Vertex AI Agent Engine + Cloud Run (토큰 발급·웹훅·취소 큐·SMS 수신) — australia-southeast1 |
| 모바일 | Flutter 또는 React Native, `@google/genai` live.connect (직결) → P2부터 서버 중계 |
| 문서 | Google Docs/Sheets 템플릿 → PDF (견적), PDF→PNG 렌더(도면) |
| SMS | Sinch(MessageMedia) 또는 Twilio AU 전용 번호 |
| 주소 | Google Address Validation API (AU) |
| 데이터 | Firestore(프로젝트·plan_index·사전), Cloud SQL(감사로그), Secret Manager(OAuth) |
| 연동 | Xero API, ServiceM8 API, Tradify API (OpenAPI 도구), GitHub App 매니페스트 |

---

## 9. 비용·가격

**원가 (사용자 1인/월, 2차 출처 단가 기준 추정)**

| 항목 | 추정 |
|---|---|
| Live 대화 300분 | ~$6~12 |
| 도면 인덱싱·판독(월 5세트, 질문 60회) | ~$2~4 |
| SMS 200건 | ~$10~14 (AUD) |
| 인프라·번호 | ~$3 |
| **합계** | **약 AUD 25~35** |

**가격 가설**: AUD 79~99/월 (사업주 1인). 빅3 잡 관리 앱이 월 $29~349, AI 전화 접수가 $199인 시장에서 "사장님 전용 비서"로 포지셔닝. 트레이디 1인의 사무 시간 주 5시간 × 시급 $100 = 월 $2,000 가치 대비.

---

## 10. 호주 규제·컴플라이언스

| 항목 | 대응 |
|---|---|
| ACMA SMS Sender ID Register (2026-07 시행) | 알파 태그 대신 **전용 번호(long code)** 사용 → 등록 대상 아님. 브랜드 태그 원하면 ABN 기반 등록 지원 |
| Spam Act 2003 (SMS/메일 동의) | 기존 거래 관계 고객에게만, 수신 거부 처리 자동 |
| Privacy Act 1988 / APPs | 데이터 AU 리전, 개인정보 처리방침, 사용자 데이터 삭제 요청 처리 |
| Google OAuth 민감 스코프 심사 | P2 중 착수(4~6주) |
| 도면·규정 책임 | 치수·규정 답변에 면책 문구 서버 강제, 구조·전기 규정 판단 거부 |

---

## 11. 리스크

| 리스크 | 대응 |
|---|---|
| 현장 소음·악센트로 오인식 | 푸시투톡 + 사전 주입 + 복창 확인, 소음 환경 평가셋 포함 |
| 도면 판독 오류로 현장 실수 | 표기 텍스트 우선, 축척 환산은 "약"+실측 권장, 신뢰도 낮으면 화면 확인 유도 |
| 운전 중 네트워크 끊김 | 세션 재개, 오프라인 명령 큐(`auto`/`confirm`만) |
| 트레이디가 앱을 안 깐다 | 온보딩 10분, 첫 주 가치 체감 설계(저녁 요약·선제 알림) |
| 빅3가 음성 기능 추가 | 그들 앱과 **연동**해 보완재로 포지셔닝, 도면·멀티앱 횡단이 차별점 |
| 세션 한도(15분/58분) | 압축·재개·선제 재접속 |
| Google 종속 | 도구 레이어 OpenAPI/MCP 표준 |

---

## 12. 첫 주 할 일

1. GCP 프로젝트(au-southeast1), Vertex AI·Gmail·Drive·Calendar·Contacts·Sheets·Address Validation API 활성화, 내부용 OAuth
2. 프로젝트 데이터 모델 + 리졸버(주소·별칭 퍼지 매칭) 구현
3. 정책 엔진 골격: 등급 테이블, strict 템플릿, 확인 토큰, 30초 큐, 감사 로그
4. ADK `LlmAgent(model="gemini-3.8-live")` + 도구 6개: `find_project`, `quote_read`, `quote_send`, `calendar_create`, `calendar_list`, `gmail_search`
5. 평가셋 초안: 업무 A·C·D 각 15개 호주 영어 시나리오, 샘플 견적 5개·도면 2세트 확보
6. SMS 제공사(Sinch vs Twilio AU) 번호 발급 테스트

---

## 참고 출처

- 호주 트레이디 잡 관리 앱 현황: https://chippieapp.com/blog/best-job-management-software-australia , https://tradiescaler.com/job-management/
- 트레이디용 AI 도구·AI 전화 접수: https://www.melbourneaiagents.com.au/blog/top-ai-tools-for-australian-tradies-2026
- ServiceM8 가격: https://tradieautomate.com/blog/servicem8-pricing-plans-australia-2026/
- ACMA SMS Sender ID Register: https://www.acma.gov.au/1-introduction-businesses-and-organisations , https://business.gov.au/news/new-rules-for-businesses-sending-text-messages
- Gemini 3.8 Live: https://ai.google.dev/gemini-api/docs/models/gemini-3.8-live , 세션 관리 https://ai.google.dev/gemini-api/docs/live-api/session-management
- Ephemeral Tokens: https://ai.google.dev/gemini-api/docs/ephemeral-tokens
- ADK Live: https://adk.dev/live/index , Memory Bank https://docs.cloud.google.com/agent-builder/agent-engine/memory-bank/set-up
- Gemini 도면 판독 초기 실험(참고): https://gwyllim.substack.com/p/reasoning-about-plans-with-language
- 가격(2차 출처): https://aireiter.com/blog/gemini-3-8-live-api-pricing
