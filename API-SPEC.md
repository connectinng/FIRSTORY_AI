# FIRSTORY AI 서비스 API 스펙 v0.3

Spring 백엔드가 LangGraph 파이프라인을 HTTP로 호출하기 위한 FastAPI 서버 스펙이에요.
AI 저장소 코드를 직접 확인한 결과(2026-10)를 반영했어요.

- 표시: **[확정]** 코드로 확인했거나 팀이 정한 것, **[요청]** AI 쪽 수정이 필요한 것, **[결정 필요]** 팀이 정해야 하는 것

## v0.2 → v0.3 바뀐 점

| 항목 | v0.2 | v0.3 |
|---|---|---|
| 인터뷰 답변 위기 신호 | 그래프에 보낸 뒤 응답에 safety 포함 | **Spring이 그래프에 보내기 전에 `/safety/check`로 먼저 검사.** 통과한 답변만 그래프로 감 |
| 인터뷰 응답의 `safety` 필드 | 있음 | **없음** |
| AI가 만들 판단 기능 | 인터뷰 노드 + 엔드포인트 | **`/safety/check` 엔드포인트와 완성 본문 판단만** |
| 분위기(tone) 전달 | interview/start만 | interview/start와 **generate에서도** (인터뷰 중 민감 주제가 나올 수 있어서) |

이유: 답변을 그래프에 먼저 넣으면 해로운 내용이 이미 이야기 재료가 된 뒤에 판단이 나와서, "다시 써 주세요"로 되돌릴 수 없다.

## v0.1 → v0.2 바뀐 점

| 항목 | v0.1 | v0.2 |
|---|---|---|
| 동화 쪽수 | 10쪽 | **8쪽** |
| 요청 입력 형식 | 백엔드가 만든 형식 | **AI의 `StoryInputContext`를 그대로** 보냄 |
| 위기 신호 | flagged true/false | **level 4단계** (OK, SENSITIVE, REVISE, CRISIS) |
| 위기 신호 검사 위치 | 폼, 인터뷰 | 폼, 인터뷰, **다시 만들기 메모, 완성 본문** |
| 그림만 다시 | feedback 문자열 | **`pages: [3, 5]`** (FastAPI가 feedback으로 변환) |
| 버전 fork | 확인 요청 | **[확정] 가능** (아래 7장) |
| mock 모드 | 없음 | **환경 변수로 켜기** |

---

## 1. 구조와 역할

```
[브라우저] ⇄ [Spring 백엔드] ⇄ (내부 HTTP) ⇄ [FastAPI + LangGraph] ⇄ [OpenAI]
                  │                                  │
             PostgreSQL                         체크포인트 DB, 결과물 저장소
```

| 역할 | 담당 |
|---|---|
| 회원, 동화 상태, 버전 목록, 횟수 제한, 위기 신호 횟수 세기 | Spring |
| thread_id, run_id 생성 | Spring |
| 인터뷰, 동화·가이드·그림 생성 | AI |
| 위기 신호 **판단** (level, category, reason) | AI |
| 결과물 저장 (실행별 경로) | AI |

## 2. 결정 사항

- **[확정] thread_id는 Spring이 UUID로 만들어 넘긴다.**
- **[확정] 진행 상황은 Spring이 폴링한다.** 생성 요청은 바로 202, 그래프는 백그라운드에서 실행.
- **[확정] 서버 간 인증**: 모든 요청에 `X-Internal-Token` 헤더. 다르면 401. FastAPI는 외부에 공개하지 않는다.
- **[확정] 아이 이름(호칭)은 AI로 보내지 않는다.**
- **[확정] 결과물은 실행마다 다른 경로에 저장한다.** 경로 앞부분(`output_prefix`)은 Spring이 정해서 넘긴다. 처음에는 로컬 폴더, 나중에 S3.
- **[결정 필요] FastAPI를 누가 만들고 어디에 배포할지**

## 3. 파이프라인 변경

| # | 변경 | 상태 |
|---|---|---|
| G0 | 다시 만들기 메모 반영 버그 2개 (character feedback 무시, "3페이지" 인식) | **PR 올라감** (`fix/피드백-반영-누락`) |
| G1 | `interview_finish` 다음에 **생성 전 확인 멈춤 지점** 추가 | [요청] 작업 예정 |
| G2 | **위기 신호 판단** 추가 (4장) | [요청] **현재 코드에 전혀 없음.** 프롬프트 설계가 핵심 |
| G3 | **실행별 결과물 경로** (`runs/{run_no}/`) | [요청] 작업 예정. 지금은 다시 만들면 덮어써짐 |
| G4 | 그림만 다시 그릴 때 안 바뀐 쪽은 이전 파일 경로 유지 | [요청] G3와 함께 |
| G5 | `parent_review` 멈춤 시점의 checkpoint_id를 결과에 포함 | [요청] FastAPI에서 `get_state`로 가능 |
| G6 | mock 모드를 환경 변수로 켜기 (5장) | [요청] |
| G7 | 운영 전 `SqliteSaver` → `PostgresSaver` | [요청] 출시 전 |

## 4. 위기 신호 판단 (G2)

### 단계

| level | 의미 | 예시 | Spring 처리 |
|---|---|---|---|
| `OK` | 문제없음 | | 그대로 진행 |
| `SENSITIVE` | 무겁지만 건강하게 다룰 수 있는 주제 | 반려동물·조부모의 죽음, 이별, 이혼, 질병 | ST-04 분위기 선택 후 진행 |
| `REVISE` | 동화 방향이 아이에게 해로움 | 폭력을 해결책으로 그리기, 체벌·위협·공포·수치심 훈육, 차별, 불법 행위 미화, 죽음을 아이 탓으로 돌리기 | 1회째 다시 쓰기 안내, **2회째 차단(ST-05)** |
| `CRISIS` | 아이의 실제 안전이 걱정됨 | 학대·방임 정황, 아이의 자해·자살 언급, 아이를 해치려는 의도 | **1회만에 차단(ST-05)** |

- 판단은 **키워드가 아니라 맥락**으로. "할머니가 돌아가셨어요"는 SENSITIVE, "할머니가 돌아가신 건 네가 말을 안 들어서라고 알려주고 싶어요"는 REVISE.
- 횟수 세기와 차단 결정은 Spring이 한다. AI는 **판단 결과만** 돌려준다.

### 응답 형식

```json
{ "safety": { "level": "REVISE", "category": "harmful_discipline", "reason": "공포로 행동을 바꾸게 하는 훈육이 담겨 있어요" } }
```

- `reason`은 **부모에게 그대로 보여줄 수 있는 문장**으로. 판단·진단 표현은 쓰지 않는다.
- **[결정 필요]** `category` 값 목록. 제안: `violence`, `harmful_discipline`, `discrimination`, `illegal`, `self_harm`, `abuse_neglect`, `harm_intent`, `other`

### 검사 위치 (네 곳)

| 위치 | 방법 | 비고 |
|---|---|---|
| 폼 제출 | `POST /v1/safety/check` | 그래프 밖 |
| 인터뷰 답변 | `POST /v1/safety/check` | **Spring이 답변을 그래프에 보내기 전에** 호출. skip은 글이 없어 검사하지 않음 |
| 다시 만들기 메모 | `POST /v1/safety/check` | Spring이 review 전에 호출 |
| 완성 본문 | `result` 응답에 `safety` 포함 | 부모에게 보여주기 전 확인 |

- 구현: 판단 전용 프롬프트와 스키마를 새로 두고 `/safety/check`와 완성 본문 판단에서 같이 쓰면 된다. 인터뷰 노드는 고치지 않아도 된다.

## 5. mock 모드 (G6)

`FIRSTORY_MOCK=true`면 FastAPI가 시작할 때 `llm.set_structured_override`와 `illustrate.set_image_override`를 걸어서 **OpenAI를 호출하지 않는다.** `evals/mock_run.py`의 가짜 함수를 재사용하면 된다.

- Spring 개발과 테스트에서 진짜 FastAPI를 비용 없이 붙여 쓰기 위한 것
- mock의 `/safety/check`는 기본 `OK`. 글에 `[[SENSITIVE]]`, `[[REVISE]]`, `[[CRISIS]]`가 들어 있으면 그 level을 돌려준다 (Spring 테스트용)

## 6. 공통 규칙

- 기본 경로 `/v1`, JSON, `snake_case`, 시간은 ISO-8601 UTC
- 한 thread에 동시에 하나의 실행만. 실행 중이면 `409 THREAD_BUSY`
- 오류: `{ "error": { "code": "...", "message": "..." } }`

| HTTP | code | 언제 |
|---|---|---|
| 401 | UNAUTHORIZED | 토큰 불일치 |
| 404 | THREAD_NOT_FOUND | 없는 thread_id |
| 409 | THREAD_BUSY | 실행 중인 thread |
| 409 | INVALID_STATE | 멈춤 지점과 맞지 않는 요청 |
| 422 | VALIDATION_ERROR | `StoryInputContext` 검증 실패 포함 |
| 500 | INTERNAL_ERROR | |

- 응답 시간: 인터뷰·확인 API는 30초 이내 동기, 생성 API는 즉시 202

## 7. 엔드포인트

| # | 메서드 | 경로 | 화면 | 방식 |
|---|---|---|---|---|
| 1 | POST | `/v1/safety/check` | CRT-02, CRT-04, REV-02 | 동기 |
| 2 | POST | `/v1/threads/{thread_id}/interview/start` | CRT-04 → CRT-05 | 동기 |
| 3 | POST | `/v1/threads/{thread_id}/interview/answer` | CRT-05 | 동기 |
| 4 | POST | `/v1/threads/{thread_id}/generate` | CRT-06 확인 | 202 |
| 5 | GET | `/v1/threads/{thread_id}/status` | GEN-01 | 동기 |
| 6 | GET | `/v1/threads/{thread_id}/result` | REV-01 | 동기 |
| 7 | POST | `/v1/threads/{thread_id}/review` | REV-02, REV-04 | 확정 동기, 다시 만들기 202 |
| 8 | POST | `/v1/threads/{thread_id}/retry` | ST-03 | 202 |
| 9 | GET | `/health` | | 동기 |

### 7-1. POST /v1/safety/check

```json
{ "source": "form", "texts": ["어린이집에서 친구가 밀쳐서 넘어졌어요."] }
```

- `source`: `form` | `interview_answer` | `regen_feedback`
- `context`(선택): `interview_answer`일 때 직전 질문을 넣어 맥락 판단을 돕는다. 예: `{ "question": "그때 아이는 어떤 표정이었나요?" }`
- 응답 `200 { "safety": { ... } }`

### 7-2. POST /v1/threads/{thread_id}/interview/start

요청: **AI의 `StoryInputContext` 그대로** + 실행 정보

```json
{
  "run_id": "6c1f0f7e-2b8a-4f53-9a6e-0f5b8f0c2d11",
  "context": {
    "situation": { "category": "friend_conflict", "custom_category": null, "timing": "past", "description": "어린이집에서 친구가 밀쳐서 넘어졌어요." },
    "child": {
      "age_months": 66, "gender": "female",
      "traits": ["shy", "cautious"], "custom_trait": null,
      "interests": ["dinosaur", "other"], "custom_interest": "파란색, 기차"
    },
    "moment": { "reactions": ["cried"], "custom_reaction": null, "emotions": ["sad", "scared"] },
    "parent": { "response": null, "concern": null, "message": null, "message_direction": null }
  },
  "tone": null
}
```

- 값 목록은 AI `schemas.py` 그대로
- `custom_interest`: 부모가 직접 입력한 여러 개를 Spring이 `", "`로 합쳐서 한 칸으로 보냄
- `tone`: `plain` | `warm` | `undecided` | null (SENSITIVE 판단 후 부모가 고른 값). **[요청]** 이 값을 design_story에 전달하는 입력 추가 필요

응답 `200`: 인터뷰 응답 형식 (7-3)

### 7-3. POST /v1/threads/{thread_id}/interview/answer

```json
{ "answer": "같이 놀자고 했는데 대답을 안 해줬대요." }
```

또는 `{ "skip": true }` (내부적으로 `Command(resume="skip")`)

응답, 다음 질문이 있을 때:

```json
{
  "type": "question",
  "turn": 2,
  "question": "그때 아이는 어떤 표정이었나요?",
  "context_update": { "...": "..." }
}
```

응답, 인터뷰가 끝나 생성 전 확인(G1)에서 멈췄을 때:

```json
{
  "type": "confirm",
  "summary": "아이가 친구에게 다가갔다가 거절당한 경험…",
  "end_reason": "sufficient",
  "context": { "...": "인터뷰가 채운 parent 값까지 포함한 최종 StoryInputContext" }
}
```

- `end_reason`: `sufficient` | `max_turns` | `user_skip` (코드의 `interview_done_by`)
- 위기 신호는 이 API가 판단하지 않는다. Spring이 보내기 전에 `/safety/check`로 검사한다.
- Spring은 `context`를 `story_inputs.final_context`에 저장한다

### 7-4. POST /v1/threads/{thread_id}/generate

```json
{ "run_id": "…", "run_no": 1, "output_prefix": "stories/123/runs/1/", "tone": "warm" }
```

→ `202 { "accepted": true }`. G1 멈춤 지점을 `{"action": "generate", "tone": ...}`로 재개.
- `tone`: 인터뷰 중 민감 주제가 나와 부모가 분위기를 고른 경우. design_story에 전달 (D5)

### 7-5. GET /v1/threads/{thread_id}/status

```json
{ "state": "RUNNING", "run_no": 1, "current_node": "illustrate", "stage": 3, "error": null, "updated_at": "…" }
```

| state | 의미 | 근거 |
|---|---|---|
| `INTERVIEWING` | 인터뷰 질문에서 멈춤 | `next == ("interview_answer",)` |
| `CONFIRMING` | 생성 전 확인에서 멈춤 | G1 노드 |
| `RUNNING` | 생성 중 | 백그라운드 실행 중 |
| `REVIEWING` | 부모 검토에서 멈춤 | `parent_review` interrupt |
| `COMPLETED` | 확정 후 종료 | |
| `FAILED` | 예외 | `error`에 code, node, message |

| stage | 화면 문구 | 노드 |
|---|---|---|
| 1 | 이야기 짜기 | `design_story` |
| 2 | 글 쓰기 | `write_story` |
| 3 | 그림 그리기 | `reading_guide`, `illustrate`, `illustrate_only` |
| 4 | 책으로 묶기 | `render` |

- 참고: 그래프 state의 `status`(requested → generating_text → generating_images → reviewing → completed)도 함께 쓸 수 있음
- 실행 상태(특히 FAILED)는 서버 재시작 후에도 남아야 함. 체크포인트 DB에 작은 테이블 추천

### 7-6. GET /v1/threads/{thread_id}/result

`state`가 `REVIEWING`일 때만 200.

```json
{
  "run_no": 1,
  "checkpoint_id": "1ef8a…",
  "title": "공룡 아로의 새 친구",
  "design": { "...": "02-design.json" },
  "character_sheets": [ { "role": "main", "name": "아로", "path": "stories/123/runs/1/images/sheet-main-아로.png" } ],
  "pages": [
    { "page_no": 1, "text": "아로는 **쿵쿵** 걸어서…", "image_prompt": "…", "image_path": "stories/123/runs/1/images/page-01.png", "image_status": "done" }
  ],
  "guide": { "...": "04-guide.json 그대로 (before_reading, prompts[], after_reading)" },
  "cost": { "input_tokens": 41230, "output_tokens": 8120, "image_count": 9 },
  "safety": { "level": "OK", "category": null, "reason": null }
}
```

- `pages`는 **8개**
- review interrupt payload에는 이미지 정보가 없으므로 `graph.get_state(cfg).values["illustrations"]`에서 읽어 합친다
- `checkpoint_id`: `parent_review`에서 멈춘 시점 (`get_state(cfg).config["configurable"]["checkpoint_id"]`)
- `safety`: 완성 본문 판단 (4장)
- 그림만 다시 그린 경우, 안 바뀐 쪽의 `image_path`는 이전 실행 경로 그대로 (G4)

### 7-7. POST /v1/threads/{thread_id}/review

확정: `{ "action": "approve" }` → `200 { "state": "COMPLETED" }`

다시 만들기:

```json
{
  "action": "regenerate",
  "scope": "images",
  "pages": [3, 5],
  "feedback": null,
  "base_checkpoint_id": "1ef8a…",
  "run_id": "…",
  "run_no": 2,
  "output_prefix": "stories/123/runs/2/"
}
```

→ `202 { "accepted": true }`

| scope | 다음 노드 | 다시 만드는 범위 | 메모 사용 |
|---|---|---|---|
| `story` (화면: 이야기·인물) | `design_story` | 설계부터 전부 | `feedback` (G0 PR로 반영됨) |
| `text` (화면: 글) | `write_story` | 글부터, **그림도 전부 다시** | `feedback` |
| `images` (화면: 그림만) | `illustrate_only` | `pages`의 쪽만. 비어 있으면 전부 | `pages`를 `"3,5"`로 바꿔 feedback에 넣음 |

- 코드에는 `character`도 있지만 `story`와 같은 경로라 Spring은 `story`만 쓴다
- `base_checkpoint_id`: 어느 버전에서 다시 만들지 (8장)

### 7-8. POST /v1/threads/{thread_id}/retry

`FAILED`에서 마지막 성공 체크포인트부터 재실행 (`graph.invoke(None, config)`). 요청은 generate와 같은 형식. `FAILED`가 아니면 409.

### 7-9. GET /health

`200 { "status": "ok", "mock": false }`

## 8. 버전 되돌리기와 fork [확정]

실험으로 확인한 동작:

- `parent_review`에서 멈춘 v1 → 다시 만들기로 v2 → **v1의 checkpoint_id를 지정해 `Command(resume=regenerate)`** → v1에서 갈라진 v3가 생김. **v2 체크포인트도 그대로 남음**
- checkpoint_id 없이 `get_state(thread)`를 부르면 **마지막에 실행한 가지**가 나온다
- 가지는 step 번호가 아니라 `parent_checkpoint_id`로 구분된다 (두 가지가 같은 step 번호를 씀)

```python
config = {"configurable": {"thread_id": tid, "checkpoint_id": base_checkpoint_id}}
graph.invoke(Command(resume={...}), config)
```

- 버전 사이를 오가는 것(되돌리기, 앞으로 가기)은 **Spring만** 처리하고 AI는 호출하지 않는다
- 결과물 파일은 G3 전까지 덮어써지므로, **G3가 이 기능의 전제**

## 9. 결과물 저장 경로 (G3, G4)

```
{output_prefix}
├── images/
│   ├── sheet-{role}-{name}.png
│   └── page-01.png … page-08.png
├── story.json      (03-story.json)
├── guide.json      (04-guide.json)
├── design.json     (02-design.json)
└── cost.json       (99-cost.json)
```

- 파일 쓰는 곳을 함수 하나로 모아 두면 나중에 S3로 바꾸기 쉽다
- `book.html`은 운영에서 필요 없다 (프론트가 JSON으로 그림)

## 10. LangSmith

```python
config = {
    "configurable": {"thread_id": thread_id},
    "run_id": run_id,
    "run_name": f"firstory-run-{run_no}",
    "metadata": {"thread_id": thread_id, "run_no": run_no},
}
```

Spring은 `run_id`를 `generation_runs.langsmith_trace_id`에 저장한다.

## 11. 흐름

```
Spring                                        AI
  │ POST /safety/check (폼)  ───────────────▶  level
  │ POST /interview/start (context)  ───────▶  질문 1
  │ [답변마다] POST /safety/check  ─────────▶  level  (통과한 답변만 아래로)
  │ POST /interview/answer ×N  ─────────────▶  다음 질문
  │                          ◀── type: confirm (요약, 최종 context)
  │   [CRT-06 부모 확인]
  │ POST /generate  ────────────────────────▶  202
  │ GET /status (3초마다)  ─────────────────▶  stage 1→4
  │ GET /result  ───────────────────────────▶  8쪽, 가이드, 경로, checkpoint_id, safety
  │   [REV-01 부모 검토]
  │ POST /review approve   또는
  │ POST /safety/check (메모) → POST /review regenerate → 다시 status 폴링
```

## 12. 작업 순서

1. G0 PR 리뷰·병합
2. G1 생성 전 확인 멈춤 지점
3. G3, G4 실행별 결과물 경로
4. FastAPI 뼈대 (인증, 오류 형식, /health, **mock 모드 G6**)
5. 인터뷰 API (7-2, 7-3) — `viewer/app.py`의 `run_until_pause`를 옮기면 됨
6. 생성, 상태, 결과 API (7-4~7-6)
7. review, retry (7-7, 7-8)
8. G2 위기 신호 판단과 /safety/check
9. G7 PostgresSaver (출시 전)

## 13. 남은 결정

| # | 내용 | 누가 |
|---|---|---|
| D1 | FastAPI를 누가 만들지 | 팀 |
| D2 | FastAPI 배포 위치 | 팀 |
| D3 | 위기 신호 판단(G2)을 누가 맡을지. 프롬프트 설계라 AI 담당 추천 | 팀 |
| D4 | safety `category` 값 목록 | AI 담당, PM |
| D5 | `tone`을 design_story에 전달하는 방식 | AI 담당 |
| D6 | 이미지 설정의 실제 운영값 (.env) | AI 담당 |
