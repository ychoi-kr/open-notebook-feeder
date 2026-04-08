# open-notebook-feeder

A small, personal CLI for pushing sources (markdown, links, files) into a
self-hosted [Open Notebook](https://github.com/lfnovo/open-notebook) instance.
Unofficial — no affiliation with the Open Notebook project.

> 한국어 문서: [README.ko.md](README.ko.md)

- Python standard library only. No dependencies to install.
- URL, token, and default notebook id all come from **environment variables**.
  Nothing is hardcoded in the source or committed to the repo.
- Bulk-injects markdown text, external links, and files (PDF/DOCX/MP3/…)
  into a notebook.

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `OPEN_NOTEBOOK_URL` | ✓ | Base URL, e.g. `http://192.0.2.10:5055` or `https://open-notebook.example.com`. Do **not** append `/api`. |
| `OPEN_NOTEBOOK_TOKEN` | ✓ | Bearer token (the server's `OPEN_NOTEBOOK_PASSWORD`). |
| `OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` | | Default notebook id for `add-*` subcommands, e.g. `notebook:abcd1234...`. |

Export them in `.zshrc`/`.bashrc`, or pass them inline when invoking the CLI.

## Install

```bash
git clone https://github.com/ychoi-kr/open-notebook-feeder.git ~/open-notebook-feeder
chmod +x ~/open-notebook-feeder/open-notebook-feeder
# Optional: symlink for PATH convenience
ln -s ~/open-notebook-feeder/open-notebook-feeder ~/bin/open-notebook-feeder
```

## Usage

```bash
# Server status
open-notebook-feeder health
open-notebook-feeder auth-status

# List resources
open-notebook-feeder notebooks
open-notebook-feeder transformations

# Add a markdown/text file as a text source
open-notebook-feeder add-text "Effective Python ch.3" ./ch3.md

# Add a URL as a link source (the server fetches it)
open-notebook-feeder add-link https://example.com/article --title "Article title"

# Upload a PDF/DOCX/MP3/… as a file source
open-notebook-feeder add-file ./document.pdf --title "Document title"

# Check processing status (for async sources)
open-notebook-feeder status "source:abcdef..."
```

### Options

Common to all `add-*` subcommands:

- `--notebook NOTEBOOK_ID` — override the default notebook. Falls back to
  `$OPEN_NOTEBOOK_DEFAULT_NOTEBOOK` when omitted.
- `--no-embed` — disable embedding (enabled by default).
- `--sync` — send `async_processing=false` so the server finishes before
  responding (async by default).
- `--transformation TRANSFORMATION_ID` — apply a transformation. Repeatable.

### Example: multiple transformations on a specific notebook

```bash
open-notebook-feeder add-text "Paper title" ./paper.md \
  --notebook notebook:xxxx... \
  --transformation transformation:dense_summary \
  --transformation transformation:key_insights
```

## Notes and gotchas

- **multipart/form-data is required**: `POST /api/sources` does not accept a
  JSON body. This CLI handles that for you; if you hit the endpoint with curl,
  use `-F`.
- **Booleans are strings**: the server expects `embed` and `async_processing`
  as the literal strings `"true"`/`"false"`. The CLI converts them for you.
- **`add-link` only forwards the URL**: the actual fetching is done by the
  Open Notebook server. If the server's fetcher can reach the site, `add-link`
  is all you need. Sites that require login, heavy JS rendering, or have
  anti-bot protection may fail on the server side — working around that
  (external extraction, etc.) is outside this tool's scope.
- **Do not URL-encode the token**: even when it contains special characters,
  the Authorization header value is passed through verbatim.

## License

Personal use. No warranty.
