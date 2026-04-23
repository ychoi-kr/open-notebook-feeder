# open-notebook-feeder 확장 기능 명세

> 상태: 초안 v2 (옵션 C 채택 반영).
> 대상 Open Notebook 리비전: lfnovo/open-notebook `main` (2026-04 기준).

## 0. 배경과 방향성

지금의 `open-notebook-feeder`는 "웹 UI에서 mp3 업로드가 막힌다"는 개인적 불편에서 출발한 단방향 feeder. Open Notebook 서버는 21개 라우터 / 약 85개 엔드포인트를 노출하지만, 지금 CLI는 6개만 쓴다.

### 채택된 방향: 하이브리드 (옵션 C)

**CLI는 작게 두고, 스킬이 API 폭을 담당**한다.

- CLI가 맡을 것: multipart 업로드, 폴링(`--wait`), 바이너리 다운로드, alias, doctor — **curl로 할 때 매번 틀리기 쉬운 부분**.
- 스킬이 맡을 것: 단순 GET/JSON POST 전부. Claude가 그때그때 엔드포인트 표를 보고 `curl -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" ...`로 조립.

### 목표

- NotebookLM 대응 기능(Chat, Search/Ask, Podcast, Note, Insight, Transformation)을 **Claude Code 세션 안에서** 매끄럽게 쓸 수 있게 한다.
- 기존 feeder 워크플로는 손대지 않는다 (backward compatible).
- CLI는 원칙 유지: **Python stdlib only, 환경변수 설정, 단일 파일**. 추가 작업도 이 제약 안에서.
- 스킬은 현재 SKILL.md를 "CLI 전용"에서 **"CLI + 직접 API 호출의 역할 분담"** 으로 개정.

---

## 1. 역할 분담 결정 기준

어떤 동작이 CLI에 가야 할지 판단하는 기준:

| 기준 | CLI로 | 스킬로 (직접 curl) |
|---|---|---|
| multipart/form-data 필요? | ✓ | |
| 응답 바이너리(오디오/파일)? | ✓ | |
| 클라이언트 측 폴링 루프 필요? | ✓ | |
| 쉘 escape 복잡한 인수(긴 질문 문장 등)? | | ✓ (LLM이 JSON 페이로드 조립이 더 깔끔) |
| 단순 GET? | | ✓ |
| JSON body POST (복잡한 중첩 없음)? | | ✓ |
| 파일 저장 경로 결정 로직? | ✓ | |
| 환경 검증·alias 해석? | ✓ | |

---

## 2. 현재 CLI 커버리지 vs 서버 API

### 현재 CLI (6개)
`health` / `auth-status` / `notebooks` / `transformations` / `add-text|link|file` / `status`

### 서버 API 라우터 전수 (참고용)

| 라우터 | 경로 접두사 | 엔드포인트 수 | 본 명세에서 |
|---|---|---|---|
| notebooks | `/api/notebooks` | 8 | 스킬 |
| sources | `/api/sources` | 12 | 업로드·다운로드·wait는 CLI, 나머지 스킬 |
| notes | `/api/notes` | 5 | 스킬 |
| insights | `/api/insights` | 3 | 스킬 |
| chat | `/api/chat` | 7 | 스킬 |
| search | `/api/search` | 3 | 스킬 |
| context | `/api/notebooks/{id}/context` | 1 | 스킬 |
| transformations | `/api/transformations` | 8 | 스킬 |
| podcasts | `/api/podcasts` | 7 | 다운로드·wait는 CLI, 나머지 스킬 |
| episode_profiles | `/api/episode-profiles` | 6 | 스킬 |
| speaker_profiles | `/api/speaker-profiles` | 6 | 스킬 |
| models | `/api/models` | 13 | 스킬 (희소 사용) |
| commands | `/api/commands/jobs` | 5 | job list/show는 스킬, wait 로직은 CLI가 내부 사용 |
| embedding | `/api/embed` | 1 | 스킬 |
| embedding_rebuild | `/api/embeddings/*` | n | 스킬 |
| auth/config/settings/credentials/languages | — | — | 일부만 CLI (`doctor`) |

---

## 3. 설계 원칙

### CLI

1. **Stdlib only.** `urllib`, `argparse`, `json`, `mimetypes`, `pathlib`, `uuid`, `time`만.
2. **Env-first.** 기존 3개 변수 유지. alias 저장은 `~/.config/open-notebook-feeder/aliases.json` (XDG 규약).
3. **단일 파일 유지.** 패키지 분할은 하지 않는다. 새 기능은 함수 추가로 끝. 대략 20KB 이내 목표.
4. **Noun-first 신규 명령 + 기존 명령 별칭 유지.**
5. **출력 두 축.** 사람용(text) + `--json` (기계용). `--raw`는 `--json`의 deprecated alias로 유지.

### 스킬

1. **역할 역전.** 이제 "오직 CLI만" 아님. "CLI가 더 안전한 자리는 CLI, 나머지는 curl"로 분명히 서술.
2. **API 엔드포인트 요약 표 포함.** Claude가 매번 Swagger를 읽지 않게.
3. **curl 호출 템플릿 준비.** 인증·Content-Type·예시 body 포함. Claude가 복사 후 수정하는 패턴.
4. **금지 사항은 유지.** URL/IP/토큰 하드코딩, 환경변수 없을 때 추측, 이미 실패한 사이트 재시도.

---

## 4. CLI 명세 (최소)

신규 기능은 5개 영역. 기존 8개 서브커맨드는 전부 별칭으로 유지.

### 4.1 신규: `doctor`

`health` + `auth-status` + 환경변수 3종 + 기본 노트북 존재 여부를 한 번에 체크.

```
$ open-notebook-feeder doctor
OPEN_NOTEBOOK_URL           ok (https://...)
OPEN_NOTEBOOK_TOKEN         ok (set, length 64)
OPEN_NOTEBOOK_DEFAULT_NB    ok (notebook:abcd...)
server health               ok
auth                        ok (enabled, token valid)
default notebook reachable  ok (name: "연구자료")
```

실패 시 해당 줄만 빨강 + 종료 코드 1. `--json`은 구조화 출력.

### 4.2 신규: `source wait`

현재 `status`는 1회 호출. `wait`는 완료/실패까지 폴링.

```
open-notebook-feeder source wait <source-id> [--timeout 600] [--interval 3]
```

내부적으로 `GET /api/sources/{id}/status`를 `interval`초마다 치고, `status in {completed, failed}`이면 즉시 반환. timeout 초과 시 종료 코드 2.

### 4.3 신규: `source download`

```
open-notebook-feeder source download <source-id> [--output PATH]
```

`GET /api/sources/{id}/download`. `Content-Disposition`에서 파일명을 파싱해 기본 저장명으로 쓰고, `--output` 주면 덮어씀.

### 4.4 신규: `podcast download` + `podcast wait`

```
open-notebook-feeder podcast download <episode-id> [--output PATH]
open-notebook-feeder podcast wait <job-id> [--timeout 1800] [--interval 10]
```

전자는 오디오 바이너리, 후자는 `/api/podcasts/jobs/{id}` 폴링. 팟캐스트 생성은 시간이 걸리므로 기본 timeout 30분.

### 4.5 신규: `alias`

노트북·소스·팟캐스트 id 타이핑 회피.

```
open-notebook-feeder alias set research notebook:abcd1234...
open-notebook-feeder alias list
open-notebook-feeder alias get research          # → notebook:abcd1234...
open-notebook-feeder alias rm research
```

저장: `~/.config/open-notebook-feeder/aliases.json`. `--notebook`, 바이너리 다운로드 대상 id에 alias 해석이 자동 적용.

### 4.6 기존 명령: 그대로 유지 + `--wait` 추가

`add-text/add-link/add-file`에 `--wait` 플래그 추가. `--sync`(서버 블로킹)와 구분되는 **클라이언트 측 대기**.

```
open-notebook-feeder add-file ./doc.pdf --wait
# → 업로드 응답 받자마자 source_id를 자동 추출해 wait로 넘김
```

`status` 커맨드는 호환용으로 남기고, 도움말에는 `source status`를 1차 표기.

### 4.7 전역 플래그 추가

- `--json` — 모든 명령에서. 기존 `--raw`는 별칭.
- `--quiet` — ID만 출력(파이프용).

---

## 5. 스킬 명세 (업데이트 방향)

`~/.claude/skills/open-notebook-feeder/SKILL.md` 개정.

### 5.1 구조 변경

```
# open-notebook-feeder 스킬

## 최우선 원칙
  - URL/토큰 하드코딩 금지
  - 환경변수 없으면 멈춤
  - 역할 분담: "CLI가 더 안전한 자리는 CLI, 나머지는 curl"  ← 신규

## 환경변수 / 연결 확인
  - `open-notebook-feeder doctor` 한 줄로 점검  ← 신규

## CLI로 가야 하는 작업 (표)
  - add-text/add-link/add-file (+ --wait)
  - source wait / source download
  - podcast wait / podcast download
  - alias / doctor
  - notebooks / transformations (편의)

## 직접 API 호출로 처리 (표 + 예시)
  - Notebook CRUD
  - Note CRUD
  - Insight show/delete/save-as-note
  - Chat session + execute + context
  - Search + ask
  - Podcast generate / list / show / retry / delete
  - Episode/Speaker profile CRUD
  - Transformation CRUD / execute
  - Models (희소)

## curl 공통 템플릿
  - GET:  curl -s -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" "$OPEN_NOTEBOOK_URL/api/..."
  - POST: curl -s -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
               -H "Content-Type: application/json" \
               -d '{"...": "..."}' "$OPEN_NOTEBOOK_URL/api/..."
  - 파일 업로드 / 다운로드는 CLI를 쓴다 (금지는 아니지만 권장)

## 엔드포인트 빠른 표
  - (카테고리별로 method/path/주요 필드만. 본 SPEC의 §2 내용을 요약)

## 판단 원칙
  - 노트북 id 모를 때: 목록 먼저
  - 일괄 작업: 첫 건 검증 후 진행
  - transformation 선택: 목록 뽑고 확인

## 금지 사항 (기존 유지)
```

### 5.2 스킬에 들어갈 curl 예시 (일부)

```bash
# Notebook 생성
curl -s -X POST "$OPEN_NOTEBOOK_URL/api/notebooks" \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "연구 자료", "description": "..."}'

# 노트북에 질문
curl -s -X POST "$OPEN_NOTEBOOK_URL/api/chat/execute" \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"notebook_id": "notebook:abcd...", "message": "이 자료에서 ..."}'

# 검색
curl -s -X POST "$OPEN_NOTEBOOK_URL/api/search" \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type": "vector", "query": "...", "limit": 10}'

# 팟캐스트 생성 (후속 --wait는 CLI로)
JOB=$(curl -s -X POST "$OPEN_NOTEBOOK_URL/api/podcasts/generate" \
  -H "Authorization: Bearer $OPEN_NOTEBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"episode_name": "...", "content": "...", "episode_profile_name": "default", "speaker_profile_name": "default"}' | jq -r .job_id)
open-notebook-feeder podcast wait "$JOB"
```

---

## 6. 하위 호환

| 기존 명령 | 유지 방식 |
|---|---|
| `health`, `auth-status` | 그대로. `doctor`에 포함되지만 개별 호출도 살림 |
| `notebooks`, `transformations` | 그대로. 향후 `notebook list` / `transformation list` 별칭 추가 여지 남김 |
| `add-text`, `add-link`, `add-file` | 그대로 + `--wait` 추가 |
| `status` | 그대로. 도움말만 `source status` 권장으로 바꿈 |
| `--raw` | `--json`의 별칭으로 남김. 경고 없음 |

리포/실행 파일 이름은 그대로 `open-notebook-feeder`.

---

## 7. 구현 순서

1. **CLI 개정**
   - `doctor` 명령 구현 (env·health·auth-status·notebook lookup)
   - `alias` 커맨드 + JSON 저장/읽기, `--notebook`에서 해석
   - 전역 `--json`/`--quiet` 처리
   - `source wait` 구현 (공통 폴링 헬퍼)
   - `source download` 구현 (Content-Disposition 파싱)
   - `add-*`에 `--wait` 플래그 연결 (업로드 응답 → wait)
   - `podcast wait`, `podcast download` 구현
   - README 갱신
2. **스킬 개정**
   - SKILL.md 재작성 (§5 구조)
   - 엔드포인트 표 + curl 템플릿 추가
   - "직접 curl 금지" 문구 제거, 대신 **"CLI가 더 안전한 자리"** 기준 제시
3. **수동 회귀 테스트**
   - 기존 플로 (add-text/add-link/add-file/status) 전부 작동
   - 신규 플로: doctor, alias, source wait, source download, podcast wait/download
   - 스킬 호출 시 Claude가 올바른 판단(CLI vs curl)을 하는지 몇 가지 시나리오로 확인

---

## 8. 열린 결정

1. **`onf` 짧은 alias symlink 제공할지** — 제공 권장. README에서 `ln -s` 1줄 안내.
2. **`source status` / `notebook list` 같은 neo-style 별칭 추가할지** — 미래 v0.3에 미룸. 지금은 기존 플랫 구조 유지해도 충분.
3. **alias 파일 스키마** — 단순 `{name: id}` 맵으로 시작. id 종류(notebook/source/episode) 태깅은 필요해질 때 추가.
4. **스킬에 Swagger URL을 hint로 둘지** — 환경변수 기반으로 조립 가능. 넣어도 좋음.
5. **MCP 서버** — 이번 범위에서 완전히 제외. C를 택한 이유가 거기에 있음.
