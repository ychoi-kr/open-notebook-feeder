---
name: open-notebook-feeder
description: 사용자의 셀프호스팅 Open Notebook 인스턴스에 소스(마크다운 텍스트·외부 URL·파일)를 주입하거나, 노트북/transformation 목록·처리 상태를 조회할 때 사용한다. "오픈노트북에 넣어줘", "Open Notebook에 소스 추가", "ON 노트북 목록", PDF를 Open Notebook에 업로드, 처리 상태 확인 등의 요청에 트리거된다. open-notebook-feeder CLI를 통해서만 호출하며, 절대 직접 API를 curl로 두드리거나 URL/토큰을 추정·하드코딩하지 않는다.
---

# open-notebook-feeder 스킬

사용자의 셀프호스팅 Open Notebook에 소스를 넣는 작업은 **전부 `open-notebook-feeder` CLI**로 처리한다. 모든 접속 정보는 환경변수로 주입된다.

## 최우선 원칙

1. **CLI만 사용한다.** `curl /api/...`, `requests.post(...)` 같은 직접 호출 금지. Swagger 탐색이 필요한 수준의 새 엔드포인트가 나오면, 먼저 사용자에게 "CLI에 서브커맨드 추가할까요?"라고 물어본다.
2. **URL·IP·토큰을 본문에 적지 않는다.** 스킬·스크립트·커밋 어디에도. 전부 환경변수.
3. **환경변수가 없으면 멈추고 안내한다.** 추정·하드코딩 금지.

## 환경변수 (사용자가 세션에 export해 둠)

| 변수 | 필수 | 용도 |
|---|---|---|
| `OPEN_NOTEBOOK_URL` | ✓ | 베이스 URL. 끝에 `/api` 붙이지 말 것 |
| `OPEN_NOTEBOOK_TOKEN` | ✓ | Bearer 토큰 |
| `OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` | | `add-*`의 기본 노트북 id |

작업 시작 전 체크:
```bash
env | grep '^OPEN_NOTEBOOK_' | sed 's/=.*/=<set>/'
```
필수 변수가 비어 있으면 아래처럼 안내하고 **작업을 멈춘다**:

> Open Notebook 접근에는 `OPEN_NOTEBOOK_URL`, `OPEN_NOTEBOOK_TOKEN` 환경변수가 필요합니다. 세션에 export 후 다시 요청해 주세요.

## CLI 호출 경로

`open-notebook-feeder`가 PATH에 있다고 가정하고 예제를 쓴다. 설치 방법은 리포 README 참조. PATH에 없으면 clone 디렉터리의 절대경로로 호출하거나 심링크를 만들어 둘 것.

```
open-notebook-feeder <subcommand> [args...]
```

## 서브커맨드 한눈에

| 커맨드 | 용도 | 인증 |
|---|---|---|
| `health` | 서버가 살아 있는지 | 불필요 |
| `auth-status` | 인증 활성화 여부 | 불필요 |
| `notebooks` | 노트북 목록 (`id\tname` 탭 구분) | 필요 |
| `transformations` | transformation 목록 | 필요 |
| `add-text TITLE FILE` | 마크다운/텍스트 파일 주입 | 필요 |
| `add-link URL [--title T]` | 외부 URL 주입(서버가 fetch) | 필요 |
| `add-file PATH [--title T]` | PDF/DOCX/MP3 업로드 | 필요 |
| `status SOURCE_ID` | 처리 상태 조회 | 필요 |

공통 옵션 (`add-*`에만):
- `--notebook NOTEBOOK_ID` — 대상 노트북. 생략 시 `$OPEN_NOTEBOOK_DEFAULT_NOTEBOOK`
- `--no-embed` — 임베딩 끔 (기본 켜짐)
- `--sync` — 서버 처리 완료까지 대기 (기본 비동기)
- `--transformation TRANSFORMATION_ID` — 변환 적용 (반복 가능)

`--raw`는 `notebooks`/`transformations`에만 있고, JSON 원본 출력.

## 사용 시나리오별 지침

### A. "노트북 목록 좀"
```bash
open-notebook-feeder notebooks
```
탭 구분이라 `awk -F'\t'`로 파싱 쉬움. 사용자가 "전체 정보" 요구하면 `--raw`.

### B. 다른 도구로 뽑은 마크다운을 Open Notebook에 주입
1. 외부 도구(다른 스킬, WebFetch, 직접 작성 등)로 마크다운/텍스트 파일을 먼저 만든다.
2. 해당 파일을 `add-text`로 주입:
   ```bash
   open-notebook-feeder add-text "문서 제목 — ch.3" ./ch3.md
   ```
3. 응답 JSON의 `id`(예: `source:abcd...`)를 기억. 사용자가 처리 완료 여부를 물으면 `status` 커맨드로 확인.

### C. 웹 기사 링크 넣기
- 먼저 해당 사이트가 **로그인/JS 렌더링/anti-bot 없는 정적 페이지**인지 판단한다.
- 정적이면:
  ```bash
  open-notebook-feeder add-link "https://example.com/article" --title "기사 제목"
  ```
- **로그인벽·JS 렌더링·anti-bot 방어가 있는 사이트는 `add-link`로 실패한다** (대표적으로 로그인 필요한 뉴스 사이트, 동적 렌더링 블로그, 일부 SNS 등). 이런 경우:
  1. 외부 도구(WebFetch, 수동 복사 등)로 먼저 마크다운/텍스트를 추출한다.
  2. 파일로 저장한 뒤 `add-text`로 주입.
  3. 사용자에게 "이 사이트는 서버 fetcher로 안 되어서 텍스트 추출 후 주입했습니다"라고 명시.

### D. PDF/DOCX/MP3
```bash
open-notebook-feeder add-file ./document.pdf --title "문서 제목"
```
MP3 같은 STT 필요한 파일은 업로드 즉시 200이지만 서버 워커가 transcribe → embed를 백그라운드로 돌린다. 시간이 걸린다는 걸 사용자에게 알리고, `status`로 확인한다.

### E. 처리 상태 확인
```bash
open-notebook-feeder status "source:abcdef..."
```
`status` 필드: `pending → running → completed/failed`. `failed`면 사용자에게 즉시 보고하고, 원인이 추정되면 같이 전달.

## 판단 원칙

- **노트북 id를 모를 때**: 먼저 `notebooks`로 목록을 뽑고 사용자에게 어떤 노트북에 넣을지 물어본다. 임의 선택 금지.
- **여러 장/여러 파일 일괄 주입**: 루프 돌기 전에 첫 한 건을 실행해 정상 응답 확인 후 나머지 진행. 에러 터진 상태로 100건 보내지 말 것.
- **`--sync`를 쓸 때**: 파일 하나당 수 분 걸릴 수 있음. 일괄 주입 시엔 기본값(비동기) 유지하고 마지막에 `status`로 폴링.
- **transformation 선택**: 사용자가 "요약도 해 줘" 하면 먼저 `transformations` 목록을 뽑아 보여주고 어느 것을 적용할지 확인한다. 이름만 보고 추측 금지.

## 금지 사항

- `curl ... /api/sources` 같은 직접 호출 (CLI로 대체)
- URL·IP·토큰을 메시지/코드/주석에 박기
- `OPEN_NOTEBOOK_URL`이 없을 때 "아마 192.0.2.x일 겁니다" 같은 추측
- 사용자가 이미 `add-link` 실패한 사이트에 재시도 (텍스트 추출 경로로 전환)

## 새 엔드포인트가 필요해질 때

Open Notebook API의 다른 엔드포인트가 필요하면:
1. 사용자에게 **CLI에 서브커맨드 추가 제안**. 임시 curl로 때우지 않는다.
2. CLI는 stdlib만 쓰는 단일 파일이라 서브커맨드 추가가 쉽다.
3. 전체 엔드포인트 명세는 서버의 Swagger UI(`$OPEN_NOTEBOOK_URL/docs`)에서 확인. URL은 Claude가 하드코딩하지 말고 환경변수에서 조립.

## 참고 파일

- CLI 저장소: 이 리포 (`open-notebook-feeder`)
- CLI README: 리포 루트의 `README.md`
- 기능 확장 명세: 리포 루트의 `SPEC.md`
