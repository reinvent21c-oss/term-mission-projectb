[Project B] 뉴스 요약 자동화 워크플로우 만들기_24팀

# [Project B] FIFA 2026 뉴스 요약 자동화 워크플로우

> **Make + Gemini API + Notion + Discord**를 연동한 뉴스 수집·요약·저장 자동화 파이프라인

---

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 미션명 | [Project B] 뉴스 요약 자동화 워크플로우 만들기 |
| 목적 | FIFA 2026 관련 한국어 뉴스를 자동으로 수집하고, AI로 요약한 뒤 Notion DB에 저장 |
| 자동화 도구 | Make (구 Integromat) |
| AI 모델 | Google Gemini API (`gemini-2.5-flash`) |
| 저장소 | Notion Database |
| 알림 | Discord Webhook |
| 실행 주기 | 매일 오후 15:30 (Daily schedule) |

---

## 👥 팀 구성 및 역할 분담

| 역할 | 이름 | 담당 업무 |
|------|------|-----------|
| 팀장 | 정민규 | 프로젝트 기획 및 일정 관리, Make 자동화 플로우 구현, GitHub 저장소 관리, README 작성 및 최종 제출 |
| 팀원 | 강민주 | Make 자동화 플로우 구현, 온오프라인 회의 참석, GitHub제출 전 오류 확인 |
| 팀원 | 김성태 | 프로젝트 아이디어 방향성 논의, 자동화 구현 방향 검토 및 기능 조사, GitHub제출 전 오류 확인 |
| 팀원 | 이서령 | 온라인 회의 참여, Make 자동화 플로우 구현 |

## 🏗️ 워크플로우 구성

```
RSS (Google News) → [키워드 필터] → HTTP (Gemini API) → Notion DB 저장 → Discord 알림
```

### 모듈 상세

| # | 모듈 | 역할 |
|---|------|------|
| 2 | RSS – Retrieve RSS feed items | Google News에서 "FIFA 2026 한국" 키워드로 최신 뉴스 최대 2건 수집 |
| Filter | FIFA 토너먼트 키워드 필터 | 한국 관련 기사만 추려내는 OR 조건 필터 |
| 3 | HTTP – POST (Gemini API) | Gemini에게 제목 기반 3줄 요약 요청 |
| 7 | Notion – Create a Data Source Item | 요약 결과를 Notion DB에 저장 |
| 8 | HTTP – POST (Discord Webhook) | Discord 채널로 알림 발송 |

---

## 🔍 주요 설정 상세

### 1. RSS 피드 (모듈 #2)

```
URL: https://news.google.com/rss/search?q=FIFA+2026+%ED%95%9C%EA%B5%AD&hl=ko&gl=KR&ceid=KR:ko
최대 수집 건수: 2
```

- Google News RSS를 통해 한국어 FIFA 2026 뉴스를 실시간 수집
- 제목(`title`), 설명(`description`), 링크(`link`), 발행일(`pubdate`) 필드 활용

### 2. 키워드 필터 (Filter)

- 필터명: `FIFA 토너먼트 키워드 필터`
- 조건 방식: **OR** (아래 중 하나라도 포함되면 통과)

| 필드 | 키워드 |
|------|--------|
| title / description | `한국` |
| title / description | `32강` |
| title / description | `16강` |
| title / description | `진출` |
| title / description | `탈락` |

> 모두 대소문자 구분 없음(case insensitive) 처리

**키워드 선택 이유:**  
FIFA 2026 뉴스 중 한국 독자에게 실질적으로 의미 있는 기사를 추리기 위해 토너먼트 진행과 한국 국가대표 관련 핵심 용어를 선정했습니다.

| 키워드 | 선택 이유 |
|--------|-----------|
| `한국` | 한국 국가대표팀 직접 관련 기사 포착 |
| `32강` | 조별리그 단계 — 본선 진출국 확정·경기 결과 |
| `16강` | 토너먼트 첫 번째 관문 — 한국 생존 여부의 핵심 분기 |
| `진출` | 다음 라운드 통과 소식 (16강·8강 등 전 단계 포괄) |
| `탈락` | 탈락 소식도 뉴스 가치 높음 — 필터 미포함 시 누락 위험 |

OR 조건으로 설정하여 하나라도 해당하면 수집 — 한국 관련 기사 누락을 최소화하는 방향으로 설계

### 3. Gemini API 호출 (모듈 #3)

- **엔드포인트:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- **메서드:** POST
- **인증:** API Key (URL 쿼리 파라미터 방식)
- **헤더:** `Content-Type: application/json`
- **Timeout:** 120초

**요청 바디:**
```json
{
  "contents": [{
    "parts": [{
      "text": "FIFA 2026 뉴스를 한국어 3줄로 요약해줘. 제목: {{2.Title}}"
    }]
  }]
}
```

> **주의:** `gemini-1.5-flash` 및 `gemini-3.5-flash`는 2026년 6월 기준 rate limit 불안정. 안정적으로 작동이 확인된 `gemini-2.5-flash` 사용

### 4. Notion DB 저장 (모듈 #7)

- **연결:** Notion FIFA (Internal Integration Token)
- **Input method:** Generic Fields
- **Data Source ID:** Notion Database ID (URL에서 추출한 32자리 UUID)

**필드 매핑:**

| Notion 컬럼 | Value Type | 값 |
|------------|------------|-----|
| 이름 | Title | `{{2.Title}}` (RSS 기사 제목) |
| 요약 | Rich text | `{{3.data.candidates[].content.parts[].text}}` (Gemini 응답) |
| 링크 | URL | `{{2.url}}` (기사 원문 URL) |
| 날짜 | Date | `{{2.dateCreated}}` (기사 발행일) |

### 5. Discord 알림 (모듈 #8)

- **방식:** Discord Incoming Webhook (HTTP POST)
- **엔드포인트:** `/api/webhooks/{webhook_id}/{token}`

**에러 처리 정책 선택 이유:**  
자동화 파이프라인에서 에러 발생 시 세 가지 대응 방식(①무시 ②워크플로우 중단 ③알림 발송)을 검토한 결과, Discord 웹훅 알림 방식을 채택했습니다.

| 방식 | 장점 | 단점 | 채택 여부 |
|------|------|------|-----------|
| 에러 무시 | 구현 간단 | 누락 기사 파악 불가 | ❌ |
| 워크플로우 중단 | 확실한 오류 표시 | 이후 기사 전체 처리 중단 | ❌ |
| Discord 알림 발송 | 실시간 인지 + 나머지 기사는 계속 처리 | 알림 채널 관리 필요 | ✅ **채택** |

Make의 에러 핸들러 라우트를 활용해 503 등 일시적 오류 발생 시 해당 건만 `HANDLED ERROR`로 처리하고, 나머지 아이템은 계속 진행하도록 설계했습니다. 운영자는 Discord 알림을 통해 에러 발생 여부를 즉시 확인할 수 있습니다.

---

## 🐛 트러블슈팅 기록

### 에러 1: `503 Service Unavailable` (Gemini API)

```json
{
  "error": {
    "code": 503,
    "message": "This model is currently experiencing high demand. Spikes in demand are usually temporary. Please try again later.",
    "status": "UNAVAILABLE"
  }
}
```

**원인:** Gemini API 무료 티어 트래픽 초과 (rate limit)  
**해결:** 재실행으로 정상 처리 확인 — 일시적 과부하이며 구조적 문제 없음  
**에러 핸들링 구조:** Make 시나리오 내 에러 핸들러 라우트 구성 → `HANDLED ERROR` 처리 후 `Commit` → `Finalization` 단계로 정상 종료 (에러 발생 시 Discord 웹훅으로 알림 발송)

### 에러 2: Notion `Data Source ID` 오류

**원인:** Notion 연동 시 "Database ID"와 "Data Source ID"를 혼동  
**해결:** Make의 Notion 모듈에서 요구하는 ID는 **Data Source ID** (Notion DB URL에서 추출한 UUID). Notion 페이지 ID와 다름에 주의

### 에러 3: Gemini API 503 반복 오류

**원인:** Gemini API 무료 티어는 트래픽이 집중되는 시간대에 rate limit 초과로 503이 반복 발생할 수 있음. FIFA 월드컵 기간 특성상 유사한 요청이 전 세계적으로 집중되어 빈도가 높아짐. 모델명(`gemini-2.5-flash`, `gemini-3.5-flash`)과 무관한 무료 티어 구조적 한계  
**해결:** 트래픽이 적은 시간대에 재실행하여 정상 처리 확인. 에러 핸들러가 503 발생 건을 `HANDLED ERROR`로 처리하고 Discord로 알림 발송 — 운영자가 즉시 인지 후 재실행 가능한 구조로 대응

---

## ✅ 최종 실행 결과

- **Notion DB 저장 확인:** 기사 제목, AI 요약 3줄, 링크, 날짜 모두 정상 저장 (2026년 6월 29일 기준)
- **에러 핸들러 동작 확인:** 503 에러 발생 시 `HANDLED ERROR` 처리 후 `Commit` → `Finalization` 정상 완료
- **전체 워크플로우:** ✅ RSS 2건 수집 → 필터 통과 → Gemini 요약 → Notion 저장 end-to-end 검증 완료

---

## ⚠️ 한계점 및 향후 개선 사항

| 항목 | 현재 상태 | 향후 개선 방향 |
|------|-----------|----------------|
| 중복 기사 방지 | 미적용 | RSS `id` 필드(GUID)를 Notion에 저장 후 저장 전 중복 체크 분기 추가 예정 |
| Gemini 재시도 횟수 제한 | 미적용 (에러 시 Discord 알림 후 스킵) | Make HTTP 모듈 재시도 설정으로 최대 2회 재시도 적용 예정 |

---

## 📁 파일 구성

```
.
├── README.md                         # 이 문서
├── blueprint.json                    # Make 시나리오 설계도 (가져오기용)
└── screenshots/
    ├── 01_workflow_full.png          # 전체 워크플로우 구성
    ├── 02_workflow_filter.png        # 키워드 필터 설정
    ├── 03_gemini_api_setup.png       # Gemini API HTTP 모듈 설정 (API Key 마스킹)
    ├── 04_gemini_prompt.png          # 프롬프트 및 Body 설정
    ├── 06_notion_mapping.png         # Notion 필드 매핑 설정
    └── 07_notion_database.png        # Notion DB 최종 결과
```

---

## 🔒 보안 사항

- API Key 및 Webhook URL은 스크린샷에서 마스킹 처리됨
- 실제 운영 시 Make의 **Connection** 기능 또는 환경변수 방식으로 키 관리 권장
- blueprint.json 내 API Key는 제출 전 반드시 교체 또는 삭제할 것

---

## 🛠️ 사용 기술 스택

| 도구 | 용도 | 비고 |
|------|------|------|
| Make (Integromat) | 워크플로우 자동화 | Free tier |
| Google News RSS | 뉴스 수집 소스 | 인증 불필요 |
| Google Gemini API | AI 요약 | `gemini-2.5-flash`, API Key 인증 |
| Notion API | 데이터 저장 | Internal Integration Token |
| Discord Webhook | 알림 발송 | Incoming Webhook |
