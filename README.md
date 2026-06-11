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
- An **n8n API key** (for the monitoring branch): n8n → Settings → n8n API → Create an API key.
- A **Microsoft Teams incoming-webhook URL** (for alerts): Teams channel → Connectors → Incoming Webhook.

## Setup

### 1. Create the credentials in n8n

All three are **HTTP Header Auth** credentials (Credentials → New → *Header Auth*):

| Credential name | Header Name | Header Value |
|-----------------|-------------|--------------|
| `ClickUp API` | `Authorization` | `pk_your_clickup_token` *(no `Bearer`)* |
| `DeepSeek API` | `Authorization` | `Bearer sk_your_deepseek_key` |
| `n8n API (Header Auth)` | `X-N8N-API-KEY` | `your_n8n_api_key` |

The first two were always required (ClickUp read/write, DeepSeek generation). `n8n API (Header Auth)` is **new** — the monitoring branch uses it to read `/api/v1/executions`. The **Microsoft Teams webhook is not a credential**; you paste it into the config as `TEAMS_WEBHOOK_URL` (see step 3).

### 2. Import the workflow

Settings → **Import from File** → select `qa-tc-scanner-workflow.json`.
When prompted, map the credentials above: `ClickUp API` and `DeepSeek API` to their HTTP nodes (ClickUp Fetch/Create/comment/dashboard/alert nodes; DeepSeek generate + classify), and `n8n API (Header Auth)` to the monitoring **Fetch Executions** node.

> **One-time manual step (required for self-heal):** after import, open the workflow's **Settings → Error Workflow** and select **this same workflow**. That wires the `Error Trigger` (remediation branch) to fire when the workflow fails. The workflow's internal id isn't known until import, so it cannot be pre-set in the JSON. Without this step the remediation agent never runs.

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
| `MAIN_WORKFLOW_ID` | *(set after import)* | This workflow's id — the monitor reads its executions; the remediation agent re-triggers it. |
| `N8N_BASE_URL` | *(your n8n URL)* | Base URL of your n8n instance, for the `/api/v1/executions` call. |
| `MAX_AUTO_RERUNS_PER_DAY` | `2` | Cap on remediation auto-reruns per day (`staticData.reruns`). |
| `STUCK_MINUTES` | `20` | An execution running longer than this is flagged as stuck. |
| `EXPECTED_RUN_HOUR` | `0` | Hour (IST) the daily run is expected — drives the missed-run heartbeat. |
| `TEAMS_WEBHOOK_URL` | *(your webhook)* | Microsoft Teams incoming-webhook URL for alerts. |
| `ALERTS_CLICKUP_LIST_ID` | *(your list)* | ClickUp list for alert comments. |
| `ALERTS_TASK_ID` | *(your task)* | ClickUp task that receives alert comments. |
| `DASHBOARD_TASK_ID` | *(your task)* | The dedicated "QA Bot Dashboard" ClickUp task overwritten each monitor run. |
| `DASHBOARD_HISTORY_LIMIT` | `30` | How many recent runs the dashboard view renders. |

> **Keep the per-branch config in sync.** The remediation and monitoring branches run as **separate executions** and cannot read the main `Set Config` node, so each carries its own config Set node (`Remediation Config` / `Monitor Config`). If you change any of the reliability keys above, update them in all the relevant config Set nodes.

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

## Reliability & monitoring

The workflow JSON now contains **three trigger-rooted branches on one canvas** that share a notifier, so the daily scanner runs unattended *and* watches/heals itself:

- **Per-ticket comments + run summary.** On success a ticket gets an "added N test cases (by type/priority)" ClickUp comment; on a parse failure it gets a "generation failed" comment. At the end of each run a **run summary** of KPIs is recorded and posted.
- **Remediation / self-heal agent** (rooted at an **Error Trigger**). When the workflow fails, it classifies the error (transient rate-limit / upstream / generation-parse / coverage-incomplete / auth / unknown), **always alerts**, and **auto-re-runs only transient, idempotent-safe failures** — capped at `MAX_AUTO_RERUNS_PER_DAY` per day. **Auth errors are escalate-only.** (Requires the *Settings → Error Workflow → this workflow* step above.)
- **Monitoring / health agent** (a **30-minute schedule**). It reads the n8n `/api/v1/executions` API and computes four signals: failed executions, a **missed-run heartbeat** (dead-man's-switch for the 00:00 IST run), **stuck/long-running** executions (`STUCK_MINUTES`), and run KPIs — alerting when something looks wrong.
- **Live ClickUp dashboard.** The monitor also overwrites a dedicated **"QA Bot Dashboard"** ClickUp task (`DASHBOARD_TASK_ID`) with a rolling "tickets updated / failed in the last N runs" view.
- **Shared alerting** goes to **Microsoft Teams** (a MessageCard to your webhook) **and ClickUp** (a comment).
- **Three nested retry ceilings:** per-node `retryOnFail` (429 backoff) → in-run `VERIFY_MAX_RETRIES` (coverage self-heal) → cross-run `MAX_AUTO_RERUNS_PER_DAY` (remediation agent).

Cross-execution state (run history, the per-day rerun counter, in-flight run bookkeeping) lives in n8n workflow `staticData` (`runHistory`, `reruns`, `currentRun`).

## Test case types generated

Smoke · Functional – Positive · Functional – Negative · Security · Performance · Non-Functional · Edge Case · Regression.

## Notes

- **DeepSeek is OpenAI-compatible** — `POST https://api.deepseek.com/chat/completions`, `Authorization: Bearer`, answer at `choices[0].message.content`.
- `deepseek-v4-pro` is a reasoning model, so the workflow does not use JSON-mode/`response_format`; the parser strips fences and falls back to regex extraction.
- **Rate limiting (ClickUp 429):** every ClickUp call retries with backoff (`retryOnFail`, 5 tries, 5s) so a "rate limit reached" blip doesn't fail the run, and the `RATE_LIMIT_SECONDS` Wait between writes keeps the sustained rate under ~100 req/min. Raise `RATE_LIMIT_SECONDS` (or set `AUTO_ADJUST_BATCH=false`) if a large catch-up run still hits limits. Do not remove the Wait nodes.
- **Fetch is parents-only and paginated; coverage is checked per ticket.** Fetching the list with `subtasks=true` was unreliable (subtasks flooded page 0). Instead the Fetch node pulls **parent tickets only** (`subtasks=false`), **paginated** (`page` until `last_page`, up to 200 pages) so lists with >100 parents aren't truncated — and the detection node **throws** if it detects a truncated page. Each ticket's `[TC]` coverage and requirement then come from `GET /task/{id}?include_subtasks=true&include_markdown_description=true`, which returns subtasks **nested** and reliable.
- **Every in-scope ticket is processed each run** (`AUTO_ADJUST_BATCH=true`), so none are left in overflow; already-covered tickets skip fast.
- **Each ticket is verified after creation.** The workflow re-reads the ticket, confirms every generated `[TC]` exists, recreates any missing (up to `VERIFY_MAX_RETRIES`), and **throws/stops** if it still can't reach full coverage.
- **Now built:** run-summary notifications, per-ticket comments, an Error-Trigger self-heal agent, and a scheduled monitor + ClickUp dashboard — alerting to Microsoft Teams + ClickUp. See "Reliability & monitoring" above and the "Stylize" phase in `B.L.A.S.T.md`. (Slack/email channels are still not wired.)
