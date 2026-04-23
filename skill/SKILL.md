---
name: open-notebook-feeder
description: 사용자의 셀프호스팅 Open Notebook 인스턴스를 조작한다. 소스 업로드·다운로드·폴링은 open-notebook-feeder CLI를, 노트북/노트/채팅/검색/인사이트/팟캐스트/transformation의 조회·생성·수정·삭제는 curl로 직접 API를 호출한다. "오픈노트북에 넣어줘", "노트 만들어줘", "검색해줘", "팟캐스트 만들어줘" 등 Open Notebook 관련 모든 요청에 트리거된다.
---

# open-notebook-feeder 스킬

## 최우선 원칙

1. **역할 분담: CLI가 더 안전한 자리는 CLI, 나머지는 curl.**  
   multipart 업로드·바이너리 다운로드·클라이언트 측 폴링은 CLI, 단순 GET·JSON POST는 curl로 직접 호출.
2. **URL·IP·토큰을 본문에 적지 않는다.** 전부 환경변수에서 조립.
3. **환경변수가 없으면 멈추고 안내한다.** 추정·하드코딩 금지.

## 환경변수 / 연결 확인

| 변수 | 필수 | 용도 |
|---|---|---|
| `OPEN_NOTEBOOK_URL` | ✓ | 베이스 URL (끝에 `/api` 붙이지 말 것) |
| `OPEN_NOTEBOOK_TOKEN` | ✓ | Bearer 토큰 |
| `OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` | | `add-*`의 기본 노트북 id |

세션 시작 전 한 번에 점검:
```bash
open-notebook-feeder doctor
```
필수 변수가 비어 있으면 아래 안내 후 **작업 멈춤**:
> `OPEN_NOTEBOOK_URL`, `OPEN_NOTEBOOK_TOKEN` 환경변수가 필요합니다. 세션에 export 후 다시 요청해 주세요.

**실행**: `open-notebook-feeder <subcommand>` (PATH에 있는 경우)  
PATH에 없으면 clone 디렉터리의 절대경로로 직접 호출한다. 설치 방법은 리포 README 참조.

---

## CLI로 가야 하는 작업

| 작업 | CLI 커맨드 |
|---|---|
| 텍스트/링크/파일 소스 추가 | `add-text`, `add-link`, `add-file [--wait]` |
| 소스 처리 완료까지 폴링 | `source wait <source-id>` |
| 소스 파일 다운로드 | `source download <source-id>` |
| 팟캐스트 생성 완료까지 폴링 | `podcast wait <job-id>` |
| 팟캐스트 오디오 다운로드 | `podcast download <episode-id>` |
| 환경 전체 점검 | `doctor` |
| 노트북·transformation 목록 (편의) | `notebooks`, `transformations` |
| id alias 관리 | `alias set/list/get/rm` |

---

## 직접 API 호출로 처리 (curl)

### curl 공통 템플릿

```bash
# GET
curl -s -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  "$OPEN_NOTEBOOK_URL/api/..."

# POST (JSON)
curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}' \
  "$OPEN_NOTEBOOK_URL/api/..."

# PUT
curl -s -X PUT \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}' \
  "$OPEN_NOTEBOOK_URL/api/..."

# DELETE
curl -s -X DELETE \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  "$OPEN_NOTEBOOK_URL/api/..."
```

출력 파싱: `| python -m json.tool`  
(**`python3` 아닌 `python`** — 이 PC에서 python3는 Microsoft Store alias)

---

### 엔드포인트 빠른 표

#### 노트북 (Notebooks)

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/notebooks` | — |
| POST | `/api/notebooks` | `name`, `description` |
| GET | `/api/notebooks/{id}` | — |
| PUT | `/api/notebooks/{id}` | `name`, `description`, `archived` |
| DELETE | `/api/notebooks/{id}` | — |
| POST | `/api/notebooks/{id}/sources/{source_id}` | — (소스 연결) |
| DELETE | `/api/notebooks/{id}/sources/{source_id}` | — (소스 연결 해제) |

#### 소스 (Sources) — 업로드·다운로드는 CLI

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/sources` | — |
| GET | `/api/sources/{id}` | — |
| PUT | `/api/sources/{id}` | `title`, `topics` |
| DELETE | `/api/sources/{id}` | — |
| GET | `/api/sources/{id}/status` | — |
| POST | `/api/sources/{id}/retry` | — |
| GET | `/api/sources/{id}/insights` | — |
| POST | `/api/sources/{id}/insights` | `transformation_id`, `model_id` |

#### 노트 (Notes)

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/notes` | (쿼리: `notebook_id`) |
| POST | `/api/notes` | `title`, `content`, `note_type`, `notebook_id` |
| GET | `/api/notes/{id}` | — |
| PUT | `/api/notes/{id}` | `title`, `content`, `note_type` |
| DELETE | `/api/notes/{id}` | — |

#### 인사이트 (Insights)

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/insights/{id}` | — |
| POST | `/api/insights/{id}/save-as-note` | — |
| DELETE | `/api/insights/{id}` | — |

#### 채팅 (Chat)

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/chat/sessions` | — |
| POST | `/api/chat/sessions` | `notebook_id`, `title` |
| GET | `/api/chat/sessions/{id}` | — |
| DELETE | `/api/chat/sessions/{id}` | — |
| POST | `/api/chat/execute` | `session_id`, `message`, `context` |
| POST | `/api/chat/execute/stream` | `session_id`, `message` |

#### 검색 (Search)

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| POST | `/api/search` | `query`, `type`, `limit`, `search_sources`, `search_notes` |
| POST | `/api/search/ask` | `question`, `strategy_model`, `answer_model` |
| POST | `/api/search/ask/simple` | `question` |

#### Transformation

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/transformations` | — |
| POST | `/api/transformations` | `name`, `prompt` |
| GET | `/api/transformations/{id}` | — |
| PUT | `/api/transformations/{id}` | `name`, `prompt` |
| DELETE | `/api/transformations/{id}` | — |
| POST | `/api/transformations/execute` | `transformation_id`, `input_text`, `model_id` |

#### 팟캐스트 (Podcasts) — 다운로드·wait는 CLI

| METHOD | 경로 | 주요 body 필드 |
|---|---|---|
| GET | `/api/podcasts/episodes` | — |
| GET | `/api/podcasts/episodes/{id}` | — |
| DELETE | `/api/podcasts/episodes/{id}` | — |
| POST | `/api/podcasts/episodes/{id}/retry` | — |
| POST | `/api/podcasts/generate` | `episode_profile`, `speaker_profile`, `episode_name`, `notebook_id` |
| GET | `/api/podcasts/jobs/{job_id}` | — |

#### Episode/Speaker 프로필

| METHOD | 경로 |
|---|---|
| GET | `/api/episode-profiles` |
| POST | `/api/episode-profiles` |
| GET/PUT/DELETE | `/api/episode-profiles/{id}` |
| GET | `/api/speaker-profiles` |
| POST | `/api/speaker-profiles` |
| GET/PUT/DELETE | `/api/speaker-profiles/{id}` |

#### 기타 (희소 사용)

| 경로 | 설명 |
|---|---|
| `GET /api/models` | 등록된 LLM 목록 |
| `GET /api/models/defaults` | 기본 모델 확인 |
| `GET /api/config` | 서버 설정 조회 |
| `GET /api/settings` | 앱 설정 조회 |

---

### 주요 시나리오 예시

```bash
# 노트 생성
curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "요약", "content": "## 내용\n...", "note_type": "human", "notebook_id": "notebook:..."}' \
  "$OPEN_NOTEBOOK_URL/api/notes"

# 채팅 세션 생성 후 질문
SESSION=$(curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"notebook_id": "notebook:...", "title": "질문 세션"}' \
  "$OPEN_NOTEBOOK_URL/api/chat/sessions" | python -c "import sys,json; print(json.load(sys.stdin)['id'])")

curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"session_id\": \"$SESSION\", \"message\": \"이 자료의 핵심을 요약해줘\"}" \
  "$OPEN_NOTEBOOK_URL/api/chat/execute"

# 지식베이스 검색
curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "항력 계수", "limit": 10, "search_sources": true, "search_notes": true}' \
  "$OPEN_NOTEBOOK_URL/api/search"

# 팟캐스트 생성 후 CLI로 대기
JOB=$(curl -s -X POST \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"episode_name": "에피소드명", "notebook_id": "notebook:...", "episode_profile": "default", "speaker_profile": "default"}' \
  "$OPEN_NOTEBOOK_URL/api/podcasts/generate" | python -c "import sys,json; print(json.load(sys.stdin)['job_id'])")
open-notebook-feeder podcast wait "$JOB"
```

---

## 판단 원칙

- **노트북 id 모를 때**: `notebooks` 커맨드(CLI) 또는 `GET /api/notebooks`로 목록 확인 후 사용자에게 물어본다. 임의 선택 금지.
- **일괄 작업**: 첫 건 실행해 정상 응답 확인 후 나머지 진행. 에러 상태로 대량 전송 금지.
- **transformation 선택**: `transformations` 커맨드로 목록 뽑아 사용자 확인 후 적용. 이름 추측 금지.
- **MP3 등 STT 소스**: 업로드 후 서버 워커가 수 분 처리. `source wait`으로 폴링하거나 처리 시간을 사용자에게 안내.
- **스트리밍 채팅**: curl `-N` 플래그 필요 (`curl -s -N -X POST ...`).

## 금지 사항

- `curl ... /api/sources` 같은 직접 파일 업로드 (multipart는 CLI 위임)
- URL·IP·토큰을 메시지/코드/주석에 박기
- 환경변수 없을 때 "아마 이 주소일 겁니다" 식 추측
- 이미 `add-link`로 실패한 사이트에 재시도 (WebFetch로 추출 후 `add-text` 경로로 전환)

## 새 엔드포인트가 필요할 때

서버의 Swagger UI: `$OPEN_NOTEBOOK_URL/docs`  
(URL은 환경변수에서 조립 — 하드코딩 금지)
