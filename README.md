# Daily QA TC Scanner — ClickUp + DeepSeek (n8n)

An n8n workflow that runs daily, finds ClickUp tickets **without test-case coverage**, generates comprehensive test cases with **DeepSeek**, and writes them back as `[TC]`-prefixed subtasks — fully unattended.

```
CRON (daily 00:00 IST)
  → expand configured ClickUp list IDs
  → fetch PARENT tickets per list (one page; no subtask pagination)
  → keep tickets in target statuses
  → per ticket: GET full detail (nested subtasks + requirement)
       has a [TC] subtask?  yes → skip
       thin description (< 100 chars)? → skip (monitor)
       else → DeepSeek generates 12+ structured test cases
  → create each TC as a [TC] subtask (structured body, assignee, priority)
  → paced writes + retry-on-429 (stays under ClickUp ~100 req/min)
  → VERIFY each ticket got its [TC] subtasks
       missing → recreate (bounded retries)
       still missing → STOP and throw an error
```

## Files

| File | Purpose |
|------|---------|
| `qa-tc-scanner-workflow.json` | The importable n8n workflow (the deliverable). |
| `B.L.A.S.T.md` | Build protocol / project constitution. |
| `CLAUDE.md` | Architecture & conventions for future edits. |
| `n8nTCScannerWorkflowPrompt.md` | Original design doc (drifted — JSON wins). |

## Prerequisites

- A running **n8n** instance (cloud or self-hosted).
- A **ClickUp API token** (`pk_...`): ClickUp → Settings → Apps → Generate.
- A **DeepSeek API key** (`sk-...`): https://platform.deepseek.com → API Keys.

## Setup

### 1. Create the two credentials in n8n

Both are **HTTP Header Auth** credentials (Credentials → New → *Header Auth*):

| Credential name | Header Name | Header Value |
|-----------------|-------------|--------------|
| `ClickUp API` | `Authorization` | `pk_your_clickup_token` *(no `Bearer`)* |
| `DeepSeek API` | `Authorization` | `Bearer sk_your_deepseek_key` |

### 2. Import the workflow

Settings → **Import from File** → select `qa-tc-scanner-workflow.json`.
When prompted, map the two credentials above to the **DeepSeek Generate TCs** node and the two **ClickUp** HTTP nodes (Fetch + Create).

### 3. Configure (the **Set Config** node)

All behavior is driven from this one node — no need to edit code:

| Key | Default | Meaning |
|-----|---------|---------|
| `CLICKUP_LIST_IDS` | `["901608730150"]` | **Array** of list IDs. Add more to scan multiple lists, one by one. |
| `TARGET_STATUSES` | ready for qa, ready for development, in development, story defined | **Array** of statuses to scan. Matched **case-insensitively in the detection Code node** (not as a query param — repeated `statuses[]` params transmit unreliably through n8n). |
| `ASSIGNEE_USER_ID` | `94964380` | ClickUp user assigned to created TCs. |
| `TC_PREFIX` | `[TC]` | Marks a subtask as a test case (used for detection **and** creation). |
| `MIN_DESC_LENGTH` | `100` | Tickets with shorter descriptions are skipped (`monitor`). |
| `MAX_TICKETS_PER_RUN` | `10` | Cap per run, **only used when `AUTO_ADJUST_BATCH` is false**. |
| `AUTO_ADJUST_BATCH` | `true` | `true` = process **every** in-scope ticket each run (no overflow; covered ones skip fast). Set `false` to cap at `MAX_TICKETS_PER_RUN` and rely on idempotent daily re-runs. |
| `RATE_LIMIT_SECONDS` | `2` | Seconds to wait between ClickUp writes (paces under the ~100 req/min limit). Raise it if you still hit rate limits. |
| `DEEPSEEK_MODEL` | `deepseek-v4-pro` | DeepSeek model id. |
| `DEEPSEEK_MAX_TOKENS` | `16000` | Max output tokens for generation. |
| `MIN_TCS` | `12` | Minimum test cases; the model increases this until all scenario types are covered. |
| `VERIFY_MAX_RETRIES` | `2` | How many times the verify step re-creates missing `[TC]` subtasks before throwing. |
| `TEAM_ID` | `9016989623` | ClickUp workspace/team id (reference). |

### 4. Test, then activate

1. **Execute Node** on *Fetch Candidate Tickets* → expect the list's parent tickets (e.g. ~79), not 1 and not empty.
2. **Execute Node** on *Filter Target-Status Candidates* → expect multiple tickets (all in your target statuses).
3. **Execute Workflow** manually (keep `MAX_TICKETS_PER_RUN` small at first) → confirm `[TC]` subtasks appear **nested under multiple tickets** with the structured body, correct assignee, and priority.
4. Toggle the workflow **Active** — the Schedule Trigger then fires daily at 00:00 IST (18:30 UTC).

## How coverage is decided (per ticket)

| Condition | Action |
|-----------|--------|
| Has a `[TC]` subtask | **skip** (idempotent — no duplicates) |
| No `[TC]`, description ≥ `MIN_DESC_LENGTH` | **generate** test cases |
| No `[TC]`, thin description | **monitor** (flagged, not generated) |

## Test case types generated

Smoke · Functional – Positive · Functional – Negative · Security · Performance · Non-Functional · Edge Case · Regression.

## Notes

- **DeepSeek is OpenAI-compatible** — `POST https://api.deepseek.com/chat/completions`, `Authorization: Bearer`, answer at `choices[0].message.content`.
- `deepseek-v4-pro` is a reasoning model, so the workflow does not use JSON-mode/`response_format`; the parser strips fences and falls back to regex extraction.
- **Rate limiting (ClickUp 429):** every ClickUp call retries with backoff (`retryOnFail`, 5 tries, 5s) so a "rate limit reached" blip doesn't fail the run, and the `RATE_LIMIT_SECONDS` Wait between writes keeps the sustained rate under ~100 req/min. Raise `RATE_LIMIT_SECONDS` (or set `AUTO_ADJUST_BATCH=false`) if a large catch-up run still hits limits. Do not remove the Wait nodes.
- **Fetch is parents-only and paginated; coverage is checked per ticket.** Fetching the list with `subtasks=true` was unreliable (subtasks flooded page 0). Instead the Fetch node pulls **parent tickets only** (`subtasks=false`), **paginated** (`page` until `last_page`, up to 200 pages) so lists with >100 parents aren't truncated — and the detection node **throws** if it detects a truncated page. Each ticket's `[TC]` coverage and requirement then come from `GET /task/{id}?include_subtasks=true&include_markdown_description=true`, which returns subtasks **nested** and reliable.
- **Every in-scope ticket is processed each run** (`AUTO_ADJUST_BATCH=true`), so none are left in overflow; already-covered tickets skip fast.
- **Each ticket is verified after creation.** The workflow re-reads the ticket, confirms every generated `[TC]` exists, recreates any missing (up to `VERIFY_MAX_RETRIES`), and **throws/stops** if it still can't reach full coverage.
- **Not yet built:** a run-summary / Slack / email / webhook notification. The pipeline currently ends after writing subtasks. See the "Stylize" phase in `B.L.A.S.T.md`.
