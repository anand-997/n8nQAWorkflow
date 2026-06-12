






# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single n8n workflow that automates daily QA test-case (TC) generation for ClickUp. There is **no application code, build, or test runner** — the deliverable is the importable n8n workflow JSON. The files:

- `qa-tc-scanner-workflow.json` — the **actual artifact** and source of truth for runtime behavior. Import into n8n via Settings → Import from File.
- `B.L.A.S.T.md` — the project constitution / build protocol (Blueprint, Link, Architect, Stylize, Trigger) specialized to this workflow.
- `README.md` — setup, credentials, config, and how to import/run.
- `n8nTCScannerWorkflowPrompt.md` — the **original** design doc. It has drifted (shows the old Anthropic/Claude LLM call and a summary node that does not exist). When it disagrees with the JSON, the JSON wins.

## Working on the workflow

Edit `qa-tc-scanner-workflow.json` directly, or import it into a running n8n, edit visually, then re-export. There is no CLI to lint it — validation is: (1) it parses as JSON, and (2) it imports into n8n and a manual "Execute Workflow" run succeeds. There are no tests.

> If you regenerate the JSON programmatically (the Code-node JS and HTTP body expressions need heavy escaping), build it with a small Python script that constructs the dict and `json.dump`s it — hand-escaping is error-prone. Delete the helper script afterward; it is not part of the deliverable.

To run n8n locally: `npm install -g n8n && n8n start` (or Docker).

## LLM provider: DeepSeek (not Anthropic)

The generator uses **DeepSeek**, which is **OpenAI-compatible** — not the Anthropic Messages API.

- Endpoint: `POST https://api.deepseek.com/chat/completions`
- Auth: `Authorization: Bearer <key>` (via a stored n8n HTTP Header Auth credential)
- Body: OpenAI shape — `{ model, max_tokens, messages: [{role:"system",...},{role:"user",...}] }`
- Response: the answer is at **`choices[0].message.content`** (the parser reads this). Reasoning text, if any, is separate in `reasoning_content` and is ignored.
- Model: `deepseek-v4-pro` (DeepSeek V4, launched 2026-04-24; reasoning-heavy, 1M context). It is a **reasoning model**, so the workflow does **not** send `response_format` or `temperature` — it relies on the prompt demanding raw JSON plus the parser's fence-strip + regex fallback. The DeepSeek node uses a 300s timeout.
- Legacy aliases `deepseek-chat`/`deepseek-reasoner` route to V4-Flash and retire 2026-07-24.

## Rate limiting (ClickUp 429)

ClickUp allows ~100 requests/min per token. With `AUTO_ADJUST_BATCH=true` the run makes many calls (per ticket: 1 detail GET + N create POSTs + 1 verify GET, + recreates). To survive bursts:
- **Every ClickUp HTTP node** (`Fetch Candidate Tickets`, `Get Ticket Detail`, `Create TC Subtask`, `Verify Coverage`, `Recreate Missing TCs`) has `retryOnFail: true, maxTries: 5, waitBetweenTries: 5000` — a 429 backs off and retries instead of failing the run.
- Both Wait nodes (`Wait Rate Limit`, `Wait (Recreate)`) pace writes by `RATE_LIMIT_SECONDS` (Set Config, default `2`s ≈ 30 writes/min), leaving headroom under the limit.
- If 429s still occur on a huge catch-up run, set `AUTO_ADJUST_BATCH=false` and rely on idempotent daily re-runs.

## Credentials (stored in n8n, not env vars)

Both HTTP integrations use **stored n8n credentials of type HTTP Header Auth** (`authentication: genericCredentialType`, `genericAuthType: httpHeaderAuth`). No secrets live in the JSON. On import, n8n flags the two credentials as unmatched and you select your stored ones:

- **ClickUp API** — header `Authorization` = raw `pk_...` token (**no `Bearer` prefix**). Reused by the new comment / dashboard / alert nodes.
- **DeepSeek API** — header `Authorization` = `Bearer <deepseek key>`. Reused by the remediation `DeepSeek Classify` fallback.
- **n8n API (Header Auth)** — header `X-N8N-API-KEY` = your n8n API key. **New**, used only by Branch 3's `Fetch Executions` node to read `/api/v1/executions`.

The **Microsoft Teams incoming-webhook URL** is **not** a credential — it is stored as `TEAMS_WEBHOOK_URL` in the config Set nodes.

### One-time manual step (required for the remediation agent)

After import, open the workflow's **Settings → Error Workflow** and select **THIS workflow itself**, so the `Error Trigger` (Branch 2) actually fires on failures. The workflow's internal id isn't known until import, so this cannot be pre-set in the JSON.

## Pipeline architecture (data flow, ~47 nodes across 3 trigger-rooted branches)

The workflow now hosts **three independent trigger-rooted branches on one canvas** (see "Three-branch architecture" below). This section documents **Branch 1 — the daily scanner pipeline** (the original core). Branches 2 (remediation) and 3 (monitoring) plus the shared notifier are documented further down.

Key design point (learned the hard way): **do not fetch the list with `subtasks=true` and rely on pagination** — that path silently fetched only page 0, so only one candidate ticket was ever detected. Instead, fetch **parent tickets only** (one page) and check `[TC]` coverage + the requirement **per ticket** via the task-detail endpoint, which returns subtasks **nested** and reliably.

1. **Daily Midnight IST** (scheduleTrigger) — fires 18:30 UTC = 00:00 IST. Timezone workflow-wide (`settings.timezone: Asia/Kolkata`).
2. **Set Config** — raw-JSON Set node, threaded through the Code nodes via `$('Set Config')`. Keys: `CLICKUP_LIST_IDS` (array), `TARGET_STATUSES` (array — `ready for qa`, `ready for development`, `in development`, `story defined`), `ASSIGNEE_USER_ID`, `TC_PREFIX`, `MIN_DESC_LENGTH`, `MAX_TICKETS_PER_RUN`, `AUTO_ADJUST_BATCH`, `RATE_LIMIT_SECONDS`, `DEEPSEEK_MODEL`, `DEEPSEEK_MAX_TOKENS`, `MIN_TCS`, `VERIFY_MAX_RETRIES`, `TEAM_ID`. **Reliability/monitoring keys (used by Branches 1–3):** `MAIN_WORKFLOW_ID`, `N8N_BASE_URL`, `MAX_AUTO_RERUNS_PER_DAY` (2), `STUCK_MINUTES` (20), `EXPECTED_RUN_HOUR` (0), `TEAMS_WEBHOOK_URL`, `ALERTS_CLICKUP_LIST_ID`, `ALERTS_TASK_ID`, `DASHBOARD_TASK_ID`, `DASHBOARD_HISTORY_LIMIT` (30). **The remediation and monitor branches run as separate executions and cannot read `Set Config`** — each has its own config Set node (`Remediation Config` / `Monitor Config`) that must be kept in sync with these keys.
3. **Expand List IDs** (Code) — one item per `list_id`; Fetch runs once per list.
4. **Fetch Candidate Tickets** (HTTP GET) — `GET /list/{{ $json.list_id }}/task?subtasks=false&include_closed=false`. **Parent tickets only**, **paginated** (`page = $pageCount` until `last_page === true`, up to 200 pages) so lists with >100 parents are fully fetched and no ticket is missed. No `statuses[]` query param (transmits unreliably through n8n).
5. **Filter Target-Status Candidates** (Code) — flattens `tasks[]` across **all** page items via `$input.all()`; **throws if pagination looks truncated** (last page returned 100 tasks but `last_page !== true`); emits each top-level ticket (`parent == null`) whose `status` ∈ `TARGET_STATUSES` (case-insensitive). No `[TC]`/thin check here.
6. **Batch Limiter Dynamic** (Code) — **`AUTO_ADJUST_BATCH` defaults `true` → processes EVERY candidate per run** (no overflow; already-covered ones skip cheaply downstream). When `false`, caps at `MAX_TICKETS_PER_RUN` (rely on idempotent daily re-runs). Drops the no-candidates sentinel.
7. **Loop Each Ticket** (splitInBatches v3, size 1) — loop = index 1; done (index 0) → **Build Run Summary** (Code; records run KPIs to `staticData.runHistory` and posts a summary via the shared notifier), then the run ends.
8. **Get Ticket Detail** (HTTP GET) — `GET /task/{{ $json.task_id }}?include_subtasks=true&include_markdown_description=true`. Returns **nested** `subtasks[]` (reliable `[TC]` detection) + the full `markdown_description` (the requirement to generate from).
9. **Evaluate Coverage** (Code) — from the detail: `has_tcs` (any `subtasks[].name` starts with `TC_PREFIX`), `is_thin` (requirement < `MIN_DESC_LENGTH`); sets `action` + `need_generate`; carries the `requirement` text.
10. **Filter Needs Generation** (IF `need_generate`) — true → DeepSeek; false (already covered / thin) → back to **Loop Each Ticket** (advance).
11. **DeepSeek Generate TCs** (HTTP POST) — requirement from `$json.requirement`. System prompt: raw JSON array, min `MIN_TCS` (12) but increase until all scenario categories covered; types = Smoke, Functional ±, Security, Performance, Non-Functional, Edge Case, Regression.
12. **Parse TCs from DeepSeek** (Code) — `choices[0].message.content` → fence-strip → `JSON.parse` → regex fallback → per-field defaults. Sets `tc_name = "[TC] <id> - <title>"` (used by verification). Emits `error: true` on failure.
13. **Filter Parse Errors** (IF) — true → Loop Each TC; false → **Build Failure Comment** + **Comment Generation Failure** (post a "generation failed" comment on the ticket) → back to **Loop Each Ticket** (advance).
14. **Loop Each TC** (splitInBatches v3, size 1) — loop (index 1) → Create subtask; **done (index 0) → Verify Coverage**.
15. **Create TC Subtask in ClickUp** (HTTP POST) — `POST /list/{{ $json.list_id }}/task` with `parent` = ticket id, name = `tc_name`, assignee + structured `markdown_description`. Priority int map: `urgent→1, high→2, normal→3, else→4`. **`retryOnFail` (5×, 5s backoff) for 429**.
16. **Wait Rate Limit** (`RATE_LIMIT_SECONDS`) → Loop Each TC. Keeps ClickUp under ~100 req/min — do not remove.
17. **Verify Coverage** (HTTP GET) — re-fetch `GET /task/{ticket}?include_subtasks=true` (ticket id via `$('Loop Each Ticket')`).
18. **Assert Coverage** (Code) — compares generated `tc_name`s vs the ticket's current `[TC]` children. All present → `coverage_ok:true`. Missing & `verify_attempt < VERIFY_MAX_RETRIES` → re-emits the missing TC payloads (attempt+1). Missing & retries exhausted → **`throw`** (stops the whole workflow).
19. **Filter Coverage OK** (IF) — true → **Build TC Comment** + **Comment TCs Added** (post an "added N test cases, by type/priority" comment on the ticket) → **Loop Each Ticket** (advance); false → Recreate Missing TCs.

> **Alternate entry — `Execute Workflow Trigger`:** Branch 1 also has an `Execute Workflow Trigger` entry node so the remediation agent (Branch 2) can re-run the scanner internally without an API key.
20. **Recreate Missing TCs** (HTTP POST, `retryOnFail` 5×) → **Wait (Recreate)** → back to **Verify Coverage** (re-verify; Assert throws if still short).

### Nested-loop + verify wiring (easy to break)

Doubly-nested SplitInBatches. The **outer** loop (per ticket) advances only when control returns to **Loop Each Ticket** — fed by: Filter Needs Generation *false*, Filter Parse Errors *false*, and **Filter Coverage OK *true*** (the normal post-verify path). The **inner** loop (per TC): Loop Each TC *loop* → Create → Wait → Loop Each TC; *done* → Verify Coverage. The verify/recreate forms a bounded self-heal loop (Verify → Assert → Filter OK *false* → Recreate → Wait → Verify) that ends by advancing the outer loop or throwing.

## Three-branch architecture (scanner + remediation agent + monitor)

The single workflow JSON hosts **three independent trigger-rooted branches on one canvas**, plus a **shared notifier** that all three feed. Each branch is a separate execution; they coordinate only through workflow `staticData` and through `Execute Workflow` calls.

### Branch 1 — Daily scanner (the pipeline above)

Rooted at **Daily Midnight IST**. In addition to the core pipeline, it now: records run KPIs via **Build Run Summary** on the `Loop Each Ticket` done branch; posts **per-ticket ClickUp comments** (success: `Build TC Comment` → `Comment TCs Added` on `Filter Coverage OK` *true*; failure: `Build Failure Comment` → `Comment Generation Failure` on `Filter Parse Errors` *false*); and exposes an **`Execute Workflow Trigger`** alternate entry so Branch 2 can re-run it.

### Branch 2 — Remediation / self-heal agent

Rooted at an **`Error Trigger`** node (fires when *this* workflow fails — requires the manual one-time Settings step below). Chain:

`Error Trigger → Remediation Config (Set) → Classify Failure (Code) → IF Known → [unknown → DeepSeek Classify (LLM fallback) → Parse Classify] → Re-run Guard (Code) → IF Auto-fix → Re-trigger Scanner (Execute Workflow) → Build Remediation Notification → shared notifier`.

- **Classify Failure** applies deterministic rules mapping the error to a category: `transient_rate_limit` / `transient_upstream` / `generation_parse` / `coverage_incomplete` / `auth` / `unknown`. It also records the failed run to `staticData.runHistory`. Only `unknown` falls through to the DeepSeek LLM-classify fallback.
- **Re-run Guard** bounds auto-reruns by `MAX_AUTO_RERUNS_PER_DAY` using a per-day counter in `staticData.reruns`.
- **Re-trigger Scanner** calls this same workflow via Branch 1's `Execute Workflow Trigger` (no API key needed).
- Policy: it **always alerts**; it **auto-re-runs only transient / idempotent-safe failures**, capped per day. **Auth errors are escalate-only** (alert, never auto-rerun).

### Branch 3 — Monitoring / health agent

Rooted at **`Monitor Schedule`** (every 30 min). Chain:

`Monitor Schedule → Monitor Config (Set) → Fetch Executions (HTTP GET) → Health Evaluator (Code) → IF Report → Build Monitor Notification → shared notifier`, with a parallel `Health Evaluator → Build Dashboard (Code) → Update Dashboard Task (HTTP PUT)`.

- **Fetch Executions** GETs the n8n REST API `/api/v1/executions` for the main workflow (`MAIN_WORKFLOW_ID` on `N8N_BASE_URL`), authenticated with the new n8n API key credential.
- **Health Evaluator** computes four signals: failed executions, **missed-run heartbeat** (dead-man's-switch for the 00:00 IST run, vs `EXPECTED_RUN_HOUR`), **stuck / long-running** executions (`STUCK_MINUTES`), and run-summary KPIs (from `staticData.runHistory`). `IF Report` gates whether a notification is sent.
- **Build Dashboard** renders a rolling "tickets updated / failed in last N runs" markdown view (`DASHBOARD_HISTORY_LIMIT`) from `staticData.runHistory`; **Update Dashboard Task** overwrites a dedicated ClickUp "QA Bot Dashboard" task (`DASHBOARD_TASK_ID`) in place via HTTP PUT.

### Shared notifier (all three branches feed it)

`Notify Teams → Notify ClickUp`. **Notify Teams** POSTs a MessageCard to a Microsoft Teams incoming-webhook URL (`TEAMS_WEBHOOK_URL`); **Notify ClickUp** POSTs a comment (`ALERTS_TASK_ID` / `ALERTS_CLICKUP_LIST_ID`). Alerts thus land in both Microsoft Teams and ClickUp.

### staticData (cross-execution state)

Because the three branches run as separate executions, they coordinate via workflow `staticData`:
- **`runHistory`** — rolling per-run KPIs (written by Build Run Summary and Classify Failure; read by Health Evaluator / Build Dashboard).
- **`reruns`** — per-day auto-rerun counter (Re-run Guard) enforcing `MAX_AUTO_RERUNS_PER_DAY`.
- **`currentRun`** — in-flight run bookkeeping.

### Three nested retry ceilings

Failures are bounded at three escalating levels: **per-node `retryOnFail`** (429 backoff) → **in-run `VERIFY_MAX_RETRIES`** (Assert Coverage self-heal) → **cross-run `MAX_AUTO_RERUNS_PER_DAY`** (remediation agent, via `staticData.reruns`).

## Conventions that matter when editing

- **TC identity is the `[TC]` name prefix.** A subtask counts as a test case iff its name starts with the configured `TC_PREFIX`. Both the detection node and the creation node depend on it.
- **Code nodes are `n8n-nodes-base.code` v2**; logic lives JSON-escaped in the `jsCode` string. Easier to edit in the n8n UI and re-export than in raw JSON.
- **Idempotency:** a ticket with any `[TC]` subtask is skipped — no duplicates on re-runs.

## Hardcoded / default environment-specific IDs

Defaults in Set Config tie the workflow to one ClickUp workspace; change them for another tenant: list `901608730150` (array), team `9016989623`, assignee `94964380`.

## Verify / retry / throw (self-heal per ticket)

After a ticket's TCs are created, **Verify Coverage** re-reads the ticket and **Assert Coverage** confirms every generated `[TC]` exists. Missing ones are re-created (bounded by `VERIFY_MAX_RETRIES`, plus `retryOnFail` on the create nodes). If coverage is still incomplete after the retries, the workflow **throws and stops** — nothing is silently left half-covered.

## Phase S — built (formerly "known gaps")

These gaps are now **closed** by the three-branch build:

- **Run-summary notification — BUILT.** `Build Run Summary` (Loop Each Ticket done branch) records KPIs to `staticData.runHistory` and posts a summary through the shared **Notify Teams → Notify ClickUp** notifier (Microsoft Teams + ClickUp).
- **Parse-failure reporting — BUILT.** A ticket whose DeepSeek response fails to parse now gets a **`Comment Generation Failure`** comment on the ticket (no longer silent), and the failure surfaces in the run summary / monitor.
- **Monitoring / health — BUILT.** Branch 3 (`Monitor Schedule`, every 30 min) tracks failed executions, a **missed-run heartbeat** (dead-man's-switch), **stuck** executions, and KPIs, and maintains a rolling ClickUp **"QA Bot Dashboard"** task.
- **Self-heal across runs — BUILT.** Branch 2 (`Error Trigger` remediation agent) classifies failures and auto-re-runs transient/idempotent-safe ones, capped by `MAX_AUTO_RERUNS_PER_DAY`; auth errors are escalate-only.

### Still open

- Notifications target Microsoft Teams + ClickUp only (no Slack/email channel).
- The DeepSeek-classify fallback only triggers for `unknown` categories; deterministic mis-classification is possible for novel errors.
