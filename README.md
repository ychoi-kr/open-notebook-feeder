# open-notebook-feeder

셀프호스팅 [Open Notebook](https://github.com/lfnovo/open-notebook) 인스턴스에 소스(마크다운/링크/파일)를 밀어 넣는 개인용 CLI. 비공식 도구이며, Open Notebook 공식 프로젝트와 관계 없다.

- 파이썬 표준 라이브러리만 사용. 설치할 의존성 없음.
- URL·토큰·기본 노트북 ID는 **전부 환경변수**로 주입. 코드나 커밋에 하드코딩 금지.
- 주 용도: wikidocs에서 뽑은 마크다운, PDF, 외부 링크를 Open Notebook 노트북에 일괄 주입.

## 환경변수

| 변수 | 필수 | 설명 |
|---|---|---|
| `OPEN_NOTEBOOK_URL` | ✓ | 베이스 URL. 예: `http://192.0.2.10:5055` 또는 `https://open-notebook.example.com`. 끝에 `/api` 붙이지 말 것 |
| `OPEN_NOTEBOOK_TOKEN` | ✓ | Bearer 토큰 (서버의 `OPEN_NOTEBOOK_PASSWORD`) |
| `OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` | | `add-*` 서브커맨드의 기본 노트북 id. 예: `notebook:abcd1234...` |

`.zshrc`/`.bashrc`에 export해 두거나, 실행 직전에 인라인으로 넘긴다.

## 설치

```bash
git clone <repo-url> ~/open-notebook-feeder
chmod +x ~/open-notebook-feeder/open-notebook-feeder
# PATH 편의용 심볼릭 링크 (선택)
ln -s ~/open-notebook-feeder/open-notebook-feeder ~/bin/open-notebook-feeder
```

## 사용법

```bash
# 서버 상태
open-notebook-feeder health
open-notebook-feeder auth-status

# 리소스 조회
open-notebook-feeder notebooks
open-notebook-feeder transformations

# 마크다운 파일을 텍스트 소스로 추가
open-notebook-feeder add-text "Effective Python ch.3" ./ch3.md

# URL을 링크 소스로 추가 (서버가 직접 fetch — JS/로그인 사이트는 안 됨)
open-notebook-feeder add-link https://example.com/article --title "기사 제목"

# PDF/DOCX/MP3 업로드
open-notebook-feeder add-file ./document.pdf --title "문서 제목"

# 처리 상태 확인 (비동기 처리 시)
open-notebook-feeder status "source:abcdef..."
```

### 옵션

모든 `add-*` 서브커맨드에 공통:

- `--notebook NOTEBOOK_ID` — 기본값을 override. 없으면 `$OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` 사용
- `--no-embed` — 임베딩 비활성화
- `--sync` — `async_processing=false`로 호출 (서버가 끝날 때까지 대기)
- `--transformation TRANSFORMATION_ID` — 변환 적용. 반복 사용 가능

### 예: 다른 노트북에 여러 변환 적용

```bash
open-notebook-feeder add-text "논문 제목" ./paper.md \
  --notebook notebook:xxxx... \
  --transformation transformation:dense_summary \
  --transformation transformation:key_insights
```

## 함정

- **multipart/form-data 필수**: `POST /api/sources`는 JSON body를 받지 않는다. 이 CLI는 자동으로 처리하지만, curl로 직접 칠 때는 `-F` 사용.
- **bool은 문자열로**: 서버는 `embed`/`async_processing`을 `"true"`/`"false"` 문자열로 받는다. CLI가 이미 변환해 준다.
- **링크 소스의 한계**: Open Notebook 의 URL fetcher는 로그인·JS 렌더링·anti-bot 사이트(wikidocs.net 포함)에서 실패한다. 외부에서 마크다운으로 먼저 추출한 뒤 `add-text`로 주입할 것.
- **토큰 URL-인코딩 금지**: 특수문자가 있어도 Authorization 헤더 값은 그대로 넘긴다.

## 라이선스

개인용. 보증 없음.
