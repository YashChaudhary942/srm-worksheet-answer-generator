# How to Use — SRM Worksheet Automation

A local, Windows-first tool that turns your **manually downloaded** SRM worksheet files
(PDF/DOCX) into reviewed answer PDFs, uploads them to your own Google Drive, and verifies
link sharing — then stops. It **never** signs in to SRM for you, never fills the answer-link
field, and never clicks **UPDATE/SUBMIT**. Final submission stays a manual, human step.

> **Safety first:** this tool reads worksheet files you already have, calls a text model to
> draft answers, and uploads answer PDFs to a Drive folder you control. It performs no
> portal writes. Keep your secrets private (see [Secrets & safety](#secrets--safety)).

---

## Table of contents
1. [What it does](#what-it-does)
2. [Requirements](#requirements)
3. [One-time setup](#one-time-setup)
4. [Configure your `.env`](#configure-your-env)
5. [Put your worksheet files in place](#put-your-worksheet-files-in-place)
6. [The normal workflow](#the-normal-workflow)
7. [Command reference](#command-reference)
8. [Recovering failed worksheets](#recovering-failed-worksheets)
9. [Rate limits & token pacing](#rate-limits--token-pacing)
10. [Where your files end up](#where-your-files-end-up)
11. [Troubleshooting](#troubleshooting)
12. [Secrets & safety](#secrets--safety)
13. [Running the tests](#running-the-tests)

---

## What it does

For each worksheet it runs a safe, resumable pipeline and **stops at
`READY_FOR_MANUAL_SUBMISSION`**:

```
discover worksheets  ->  extract questions (PDF)  ->  generate answers (model API)
  ->  render + validate an A4 answer PDF  ->  upload to your Google Drive folder
  ->  set "Anyone with the link - Viewer" and verify it  ->  write a review report
```

- **Resumable:** anything already done is reused. Validated PDFs are not re-rendered,
  existing Drive links are not re-uploaded.
- **Isolated:** one worksheet failing never aborts the batch.
- **No SRM automation:** you paste the Drive links into SRM yourself.

---

## Requirements

- **Windows 10/11**
- **Python 3.11+** (`Add Python to PATH` enabled during install)
- A **free text-model API key** (Google Gemini free tier recommended; any OpenAI-compatible
  endpoint such as Groq also works)
- A **Google account** with the Drive API enabled (for the upload step)

---

## One-time setup

1. Extract the project to a stable path, e.g. `C:\SRMWorksheetAutomation`.
2. From the project folder, run the Windows setup script (creates the virtual environment
   and installs dependencies):

   ```bat
   setup_windows.bat
   ```

3. Confirm the environment is healthy (reports presence of config and dependencies — never
   prints secrets):

   ```bat
   .venv\Scripts\python.exe -m srm_worksheet_automation doctor
   ```

---

## Configure your `.env`

Copy the template and fill in your **private** values. The real `.env` is gitignored and must
never be shared or committed.

```bat
copy .env.example .env
```

### Model API (answer generation)

**Option A — Google Gemini free tier (recommended, OpenAI-compatible):**

```ini
LLM_PROVIDER=openai
LLM_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/
LLM_MODEL=gemini-2.5-flash
LLM_API_KEY=        # paste your key from https://aistudio.google.com/apikey
```

**Option B — Groq free tier (OpenAI-compatible):**

```ini
LLM_PROVIDER=openai
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_MODEL=openai/gpt-oss-120b
LLM_API_KEY=
```

Verify connectivity before generating (prints `SUCCESS` or a redacted error — your key is
never shown, and it never silently switches to a paid model):

```bat
.venv\Scripts\python.exe -m srm_worksheet_automation check-llm
```

### Google Drive upload

1. In the Google Cloud Console, create/select a project, enable the **Google Drive API**,
   and create an **OAuth client → Desktop app**.
2. Download its JSON into the project (e.g. `credentials.json`, which is gitignored) and set:

   ```ini
   GOOGLE_OAUTH_CREDENTIALS_PATH=credentials.json
   GOOGLE_TOKEN_FILE=data/google_token.json
   GOOGLE_DRIVE_FOLDER_ID=
   GOOGLE_DRIVE_SHARING_MODE=     # set to anyone_with_link_viewer only if you accept public view links
   ```

3. If you don't have a destination folder ID yet, create a dedicated app-owned root folder
   and paste the printed ID into `.env` (this is the only command that creates a Drive
   folder, and only when you run it):

   ```bat
   .venv\Scripts\python.exe -m srm_worksheet_automation setup-drive-root
   ```

### Student metadata (used in the answer PDFs)

```ini
STUDENT_NAME=Your Name
STUDENT_REG_NO=RAxxxxxxxxxxxx
STUDENT_DEPARTMENT=Department of Computing Technologies
STUDENT_INSTITUTION=SRM Institute of Science and Technology
STUDENT_COURSE_CODE=21CSE456T
STUDENT_COURSE_NAME=Software Defined Networks
```

Safety flags stay at their defaults — leave them as-is:

```ini
REVIEW_MODE=true
ENABLE_PORTAL_SUBMISSION=false
```

---

## Put your worksheet files in place

The folder-based workflow reads a tree like:

```
input\<Subject Name>\Unit <N>\<file>.pdf
input\<Subject Name>\Unit <N>\<file>.docx
```

Example:

```
input\Software Defined Networks\Unit 1\1011.pdf
input\Software Defined Networks\Unit 1\1011.docx
```

Each subject must be listed in `config\subjects.json`. The PDF and DOCX of the same
worksheet are paired automatically. **PDF is the default source** for question extraction.

---

## The normal workflow

**1. Preview with a dry run (no model calls, no Drive changes, no file writes).** This shows
what was discovered and how many questions each worksheet yields:

```bat
.venv\Scripts\python.exe -m srm_worksheet_automation generate-all --input-root input --source-format pdf --dry-run
```

**2. Run it for real** (generates answers, renders/validates PDFs, uploads to Drive, verifies
sharing, writes the Excel report). Nothing is submitted to SRM:

```bat
.venv\Scripts\python.exe -m srm_worksheet_automation generate-all --input-root input --source-format pdf
```

**3. Review and submit manually.** Open the generated Excel report, check each answer PDF and
its Drive link, then paste each link into the matching SRM worksheet field yourself and click
UPDATE. The tool never does this for you.

---

## Command reference

Run all commands from the project folder with `.venv\Scripts\python.exe -m srm_worksheet_automation <command>`.

| Command | What it does |
|---|---|
| `doctor` | Report dependency/config readiness (no secrets printed). |
| `check-llm` | Send a harmless test prompt to the configured model; prints SUCCESS or a redacted error. |
| `setup-drive-root` | Create one dedicated Drive root folder and print its ID to paste into `.env`. |
| `generate-all --input-root input --source-format pdf --dry-run` | Preview discovery + extraction; zero side effects. |
| `generate-all --input-root input --source-format pdf` | Full local pipeline → `READY_FOR_MANUAL_SUBMISSION`. No SRM submit. |
| `recover-failed --subject <slug> --source-format pdf --dry-run` | Preview which failed worksheets would be re-processed. |
| `recover-failed --subject <slug> --source-format pdf` | Re-process only the failed worksheets for one subject. |
| `migrate-manifest` / `migrate-manifest --apply` | Preview/apply the legacy→per-subject manifest migration. |

Common flags: `--dry-run` (preview only), `--source-format pdf|docx|auto` (default `auto`;
use `pdf` to force the PDF source), `--subjects-file <path>` (custom `subjects.json`).

---

## Recovering failed worksheets

If some worksheets failed (e.g. a transient rate limit or a model response that couldn't be
parsed), recover **only** the failed ones for a subject without touching the successful ones.
It backs up the manifest first, reuses any valid answers/PDFs/Drive links, and continues past
a single failure.

Preview first:

```bat
.venv\Scripts\python.exe -m srm_worksheet_automation recover-failed --subject software-defined-networks --source-format pdf --dry-run
```

Then run it:

```bat
.venv\Scripts\python.exe -m srm_worksheet_automation recover-failed --subject software-defined-networks --source-format pdf
```

Only records whose status is `ANSWER_FAILED` are selected; successful worksheets are never
re-generated or re-uploaded.

---

## Rate limits & token pacing

Free model tiers cap tokens-per-minute (TPM) and requests-per-minute (RPM). The tool paces
itself under those caps and retries transient HTTP 429 errors with the provider's suggested
delay (or exponential backoff with jitter). It does **not** retry auth errors or exhausted
daily quotas forever. Tune these in `.env` if needed (defaults shown; safe for Groq free tier):

```ini
LLM_MAX_TOKENS_PER_MINUTE=7000
LLM_MAX_REQUESTS_PER_MINUTE=25
LLM_RATE_LIMIT_MAX_RETRIES=6
LLM_RATE_LIMIT_SAFETY_MARGIN=0.125
```

---

## Where your files end up

| Location | Contents |
|---|---|
| `answers\` | Generated answer PDFs (gitignored). |
| `data\manifest\<subject>.json` | Per-subject state: questions, answers, PDF path, Drive IDs/URLs, status. |
| `data\manifest\<subject>.backup_*.json` | Timestamped manifest backups created before recovery runs. |
| Excel report | Combined worksheet → Drive-link report for manual submission. |

---

## Troubleshooting

- **`check-llm` says NOT configured** — set `LLM_MODEL` and `LLM_API_KEY` in `.env`.
- **Rate limit (HTTP 429) persists** — lower `LLM_MAX_TOKENS_PER_MINUTE` /
  `LLM_MAX_REQUESTS_PER_MINUTE`, or wait for the free-tier window to reset, then re-run
  `recover-failed`.
- **Drive folder not accessible (404)** — the app uses the minimal `drive.file` scope, which
  only sees files/folders it created. Run `setup-drive-root` and use the printed folder ID,
  and make sure you authorized the same Google account that owns the folder.
- **A worksheet shows `needs_review`** — the extractor found text but no recognizable
  question; open the source file and check its format. Nothing is fabricated.
- **A worksheet stays `ANSWER_FAILED` after HTTP 200** — re-run `recover-failed`; the tool
  preserves any valid model response rather than discarding it.
- **Manifest write fails with a Windows lock error** — close anything holding the file
  (Excel, a cloud-sync client), then re-run; the existing manifest is never corrupted.

---

## Secrets & safety

**Never commit or share these** (they grant access to your accounts):

- `.env` and any `.env_*` backup
- `credentials.json` (and any `*.json` OAuth client file)
- `data\google_token.json`
- `data\browser_profile\` (may contain authenticated session data)
- generated answer PDFs and private screenshots

The project `.gitignore` excludes the common secret files. Before pushing anywhere public,
double-check `git status` shows none of the files above. The tool itself never prints API
keys or tokens, and never performs an SRM submission.

---

## Running the tests

All tests are offline (no live API/Drive/SRM calls):

```bat
.venv\Scripts\python.exe -m unittest discover -s tests
```
