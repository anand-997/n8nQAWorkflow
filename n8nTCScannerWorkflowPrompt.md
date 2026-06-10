# Claude Code Prompt — Build n8n QA TC Scanner Workflow

## 🎯 Project Goal

Build a **production-ready n8n workflow** that automates the daily QA Test Case scanning and generation for ClickUp. This workflow replaces manual Claude.ai prompting with a fully automated scheduled pipeline.

---

## 📋 What This Workflow Must Do (End-to-End)

### Flow Summary

```
CRON (daily midnight IST)
  → Fetch ClickUp tickets in target statuses from a specific list
  → For each ticket: check if [TC] prefix subtasks exist
  → Filter: tickets with missing TCs AND rich descriptions
  → For each missing-TC ticket: call DeepSeek API to generate test cases
  → Parse DeepSeek's response into individual TCs
  → Push each TC as a subtask to ClickUp (with [TC] prefix, structured body, priority, assignee)
  → Send summary report (Slack / Email / Webhook — configurable)
```

---

## 🏗️ Architecture — n8n Nodes (Build in This Order)

### Node 1: CRON Trigger

```
Type: Schedule Trigger
Schedule: Every day at 00:00 IST (18:30 UTC previous day)
Timezone: Asia/Kolkata
```

### Node 2: Set Configuration

```
Type: Set Node
Purpose: Centralized config — change once, affects entire workflow

Fields to set:
  - CLICKUP_LIST_ID: "901608730150"
  - TARGET_STATUSES: ["ready for qa", "ready for deployment", "story defined"]
  - CLICKUP_API_KEY: "{{ $env.CLICKUP_API_KEY }}"
  - CLAUDE_API_KEY: "{{ $env.CLAUDE_API_KEY }}"
  - ASSIGNEE_USER_ID: "94964380"
  - TC_PREFIX: "[TC]"
  - BATCH_SIZE: 10
  - MIN_DESCRIPTION_LENGTH: 100
```

### Node 3: Fetch Tickets from ClickUp

```
Type: HTTP Request (one per target status — use a SplitInBatches or loop)
Method: GET
URL: https://api.clickup.com/api/v2/list/{{ $json.CLICKUP_LIST_ID }}/task
Headers:
  Authorization: {{ $json.CLICKUP_API_KEY }}
Query Parameters:
  statuses[]: {{ current_status }}
  subtasks: true
  include_closed: false
  page: 0

For each status in TARGET_STATUSES, collect all tasks into one merged array.
```

**Alternative (simpler):** Use ClickUp's filtered search endpoint:

```
Method: POST
URL: https://api.clickup.com/api/v2/team/9016989623/task
Headers:
  Authorization: {{ $json.CLICKUP_API_KEY }}
  Content-Type: application/json
Body:
{
  "list_ids": ["901608730150"],
  "statuses": ["ready for qa", "ready for deployment", "story defined"],
  "subtasks": true,
  "include_closed": false
}
```

### Node 4: For Each Ticket — Check TC Subtasks

```
Type: SplitInBatches (process tickets one at a time)

For each ticket:
  1. HTTP Request → GET https://api.clickup.com/api/v2/task/{{ ticket.id }}
     Headers: Authorization: {{ $env.CLICKUP_API_KEY }}
     Query: include_subtasks=true

  2. Code Node (JavaScript) — TC Detection Logic:
```

```javascript
// TC Detection Logic — paste this into an n8n Code node
const task = $input.first().json;
const subtasks = task.subtasks || [];

// Check for [TC] prefix in subtask names
const tcSubtasks = subtasks.filter(st => st.name && st.name.startsWith('[TC]'));
const hasTCs = tcSubtasks.length > 0;

// Check description richness
const description = task.description || task.text_content || '';
const isThinDescription = description.length < 100;

// Determine action
let action = 'skip'; // default
if (!hasTCs && !isThinDescription) {
  action = 'generate';
} else if (!hasTCs && isThinDescription) {
  action = 'monitor'; // thin description — flag but don't generate
}

return [{
  json: {
    task_id: task.id,
    task_name: task.name,
    task_url: task.url,
    status: task.status?.status || 'unknown',
    description: description,
    has_tcs: hasTCs,
    tc_count: tcSubtasks.length,
    is_thin: isThinDescription,
    action: action,
    list_id: task.list?.id || '901608730150',
    priority: task.priority?.priority || 'normal'
  }
}];
```

### Node 5: Filter — Only "generate" Action Tickets

```
Type: IF Node
Condition: {{ $json.action }} equals "generate"
True branch → continue to Claude API
False branch → collect for summary report
```

### Node 6: Batch Limiter

```
Type: Code Node
Purpose: Limit to BATCH_SIZE (10) tickets per run to avoid API rate limits

Code:
```

```javascript
const items = $input.all();
const BATCH_SIZE = 10;
const batch = items.slice(0, BATCH_SIZE);

// Store overflow count for report
const overflow = items.length - BATCH_SIZE;

return batch.map(item => ({
  json: {
    ...item.json,
    total_missing: items.length,
    overflow_count: overflow > 0 ? overflow : 0
  }
}));
```

### Node 7: Generate TCs via Claude API

```
Type: HTTP Request (for each ticket in batch)
Method: POST
URL: https://api.anthropic.com/v1/messages
Headers:
  x-api-key: {{ $env.CLAUDE_API_KEY }}
  anthropic-version: 2023-06-01
  Content-Type: application/json

Body (JSON — use expression mode):
```

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 8000,
  "system": "You are a Senior QA Architect with 20+ years of experience. Generate comprehensive test cases for the given ClickUp ticket. Return ONLY valid JSON array — no markdown, no backticks, no explanation.\n\nEach test case object must have these exact keys:\n- tc_id: string (e.g. \"TC_001\")\n- title: string (short descriptive title)\n- type: string (one of: \"Functional – Positive\", \"Functional – Negative\", \"Security\", \"Non-Functional\", \"Edge Case\", \"Regression\")\n- priority: string (one of: \"urgent\", \"high\", \"normal\", \"low\")\n- module: string (feature/module name)\n- objective: string (what this TC verifies)\n- preconditions: string (setup needed)\n- steps: string (numbered steps)\n- test_data: string (inputs/data needed)\n- expected_result: string (what should happen)\n\nCoverage requirements:\n- Minimum 10 TCs per ticket, more for complex tickets\n- Must include: Functional Positive (happy path), Functional Negative (invalid inputs, missing fields), Security (token tampering, PII exposure, auth bypass), Non-Functional (performance, concurrency, edge cases), Regression (existing flows unaffected)\n- Priority distribution: 30% urgent/high, 40% high, 20% normal, 10% low\n- Every TC must be specific and testable — no vague descriptions\n\nReturn ONLY the JSON array. Example format:\n[{\"tc_id\":\"TC_001\",\"title\":\"...\",\"type\":\"...\",\"priority\":\"...\",\"module\":\"...\",\"objective\":\"...\",\"preconditions\":\"...\",\"steps\":\"...\",\"test_data\":\"...\",\"expected_result\":\"...\"}]",
  "messages": [
    {
      "role": "user",
      "content": "Generate comprehensive test cases for this ClickUp ticket:\n\nTicket ID: {{ $json.task_id }}\nTicket Name: {{ $json.task_name }}\nStatus: {{ $json.status }}\n\nDescription:\n{{ $json.description }}\n\nGenerate test cases covering ALL scenarios: functional positive, functional negative, security, non-functional, edge cases, and regression. Return ONLY valid JSON array."
    }
  ]
}
```

### Node 8: Parse Claude Response

```
Type: Code Node (JavaScript)
Purpose: Extract JSON array of TCs from Claude's response
```

```javascript
const response = $input.first().json;
const taskInfo = $('Node 6 — or reference the ticket data node').first().json;

// Extract text content from Claude response
let textContent = '';
if (response.content && Array.isArray(response.content)) {
  textContent = response.content
    .filter(block => block.type === 'text')
    .map(block => block.text)
    .join('\n');
}

// Clean and parse JSON
let cleanJson = textContent
  .replace(/```json\s*/g, '')
  .replace(/```\s*/g, '')
  .trim();

let testCases = [];
try {
  testCases = JSON.parse(cleanJson);
} catch (e) {
  // Try to extract JSON array from mixed content
  const jsonMatch = cleanJson.match(/\[[\s\S]*\]/);
  if (jsonMatch) {
    try {
      testCases = JSON.parse(jsonMatch[0]);
    } catch (e2) {
      // Return error for this ticket
      return [{
        json: {
          error: true,
          task_id: taskInfo.task_id,
          task_name: taskInfo.task_name,
          parse_error: e2.message,
          raw_response: textContent.substring(0, 500)
        }
      }];
    }
  }
}

// Map each TC into a separate item for batch creation
return testCases.map((tc, index) => ({
  json: {
    ...tc,
    task_id: taskInfo.task_id,
    task_name: taskInfo.task_name,
    task_url: taskInfo.task_url,
    list_id: taskInfo.list_id,
    parent_task_id: taskInfo.task_id,
    assignee_id: '94964380',
    tc_index: index + 1,
    total_tcs: testCases.length
  }
}));
```

### Node 9: Create TC Subtask in ClickUp

```
Type: HTTP Request (for each TC)
Method: POST
URL: https://api.clickup.com/api/v2/list/{{ $json.list_id }}/task
Headers:
  Authorization: {{ $env.CLICKUP_API_KEY }}
  Content-Type: application/json

Body:
```

```json
{
  "name": "[TC] {{ $json.tc_id }} - {{ $json.title }}",
  "parent": "{{ $json.parent_task_id }}",
  "assignees": [{{ $json.assignee_id }}],
  "priority": {{ $json.priority === 'urgent' ? 1 : $json.priority === 'high' ? 2 : $json.priority === 'normal' ? 3 : 4 }},
  "markdown_description": "## 📋 Test Case Details\n**TC ID:** {{ $json.tc_id }}\n**Module:** {{ $json.module }}\n**Type:** {{ $json.type }}\n**Objective:** {{ $json.objective }}\n\n## 🔧 Preconditions\n{{ $json.preconditions }}\n\n## 🪜 Steps\n{{ $json.steps }}\n\n## 🧪 Test Data\n{{ $json.test_data }}\n\n## ✅ Expected Result\n{{ $json.expected_result }}"
}
```

**IMPORTANT:** Add a `Wait` node (1 second delay) between each ClickUp API call to avoid rate limiting. ClickUp's rate limit is 100 requests per minute.

### Node 10: Rate Limiter

```
Type: Wait Node
Wait: 1 second
Purpose: Prevent ClickUp API rate limit (100 req/min)
Place between: Node 9 output → loop back for next TC
```

### Node 11: Summary Report — Collect Results

```
Type: Code Node
Purpose: Aggregate all results into a summary report
```

```javascript
const items = $input.all();

const created = items.filter(i => i.json.success !== false && !i.json.error);
const failed = items.filter(i => i.json.error);
const skipped = []; // collected from the filter node's false branch

const report = {
  run_date: new Date().toISOString(),
  run_date_ist: new Date().toLocaleString('en-IN', { timeZone: 'Asia/Kolkata' }),
  total_tickets_scanned: 0, // set from earlier node
  tickets_with_tcs: 0,
  tickets_missing_tcs: 0,
  tickets_thin_description: 0,
  tcs_generated: created.length,
  tcs_failed: failed.length,
  details: created.map(tc => ({
    tc_id: tc.json.tc_id,
    title: tc.json.title,
    parent_ticket: tc.json.task_name,
    clickup_url: tc.json.task_url
  })),
  errors: failed.map(f => ({
    task: f.json.task_name,
    error: f.json.parse_error || 'Unknown error'
  }))
};

return [{ json: report }];
```

### Node 12: Send Summary Notification (Choose One)

**Option A — Slack Notification:**

```
Type: Slack Node
Channel: #qa-automation (or your channel)
Message:
```

```
🤖 *Daily QA TC Scanner Report*
📅 {{ $json.run_date_ist }}

📊 *Scan Results:*
• Tickets scanned: {{ $json.total_tickets_scanned }}
• Already covered: {{ $json.tickets_with_tcs }}
• Missing TCs: {{ $json.tickets_missing_tcs }}
• Thin description (skipped): {{ $json.tickets_thin_description }}

✅ *TCs Generated:* {{ $json.tcs_generated }}
❌ *Failures:* {{ $json.tcs_failed }}

{{ $json.tcs_failed > 0 ? '⚠️ Check errors in the run log.' : '🎉 All tickets covered!' }}
```

**Option B — Email Notification:**

```
Type: Send Email Node
To: anand@yourcompany.com
Subject: Daily QA TC Scanner — {{ $json.run_date_ist }}
Body: (same content as Slack)
```

**Option C — Webhook (for custom dashboard):**

```
Type: HTTP Request
Method: POST
URL: {{ $env.WEBHOOK_URL }}
Body: {{ JSON.stringify($json) }}
```

---

## 🔐 Environment Variables Required

Create these in n8n Settings → Variables (or .env file for self-hosted):

```env
# ClickUp API
CLICKUP_API_KEY=pk_your_clickup_api_key_here

# Claude API (Anthropic)
CLAUDE_API_KEY=sk-ant-your_anthropic_api_key_here

# Optional — Slack webhook
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/your/webhook/url

# Optional — Custom webhook for dashboard
WEBHOOK_URL=https://your-server.com/api/qa-scanner-report
```

### How to Get These Keys

**ClickUp API Key:**
1. Go to ClickUp → Settings → Apps
2. Click "Generate" under API Token
3. Copy the token (starts with `pk_`)

**Claude API Key:**
1. Go to https://console.anthropic.com
2. Settings → API Keys → Create Key
3. Copy the key (starts with `sk-ant-`)

---

## 📁 Complete n8n Workflow JSON

Create a file named `qa-tc-scanner-workflow.json` with this structure. Import it directly into n8n via Settings → Import Workflow.

```json
{
  "name": "Daily QA TC Scanner — ClickUp + Claude AI",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "triggerAtHour": 18,
              "triggerAtMinute": 30
            }
          ]
        }
      },
      "id": "cron-trigger",
      "name": "Daily Midnight IST",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [240, 300]
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "CLICKUP_LIST_ID", "value": "901608730150" },
            { "name": "ASSIGNEE_USER_ID", "value": "94964380" },
            { "name": "TC_PREFIX", "value": "[TC]" }
          ],
          "number": [
            { "name": "BATCH_SIZE", "value": 10 },
            { "name": "MIN_DESC_LENGTH", "value": 100 }
          ]
        }
      },
      "id": "set-config",
      "name": "Set Config",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [460, 300]
    },
    {
      "parameters": {
        "method": "GET",
        "url": "=https://api.clickup.com/api/v2/list/{{ $json.CLICKUP_LIST_ID }}/task",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            { "name": "subtasks", "value": "true" },
            { "name": "include_closed", "value": "false" },
            { "name": "statuses[]", "value": "ready for qa" },
            { "name": "statuses[]", "value": "ready for deployment" },
            { "name": "statuses[]", "value": "story defined" }
          ]
        },
        "options": {}
      },
      "id": "fetch-tickets",
      "name": "Fetch ClickUp Tickets",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [680, 300],
      "credentials": {
        "httpHeaderAuth": {
          "id": "clickup-auth",
          "name": "ClickUp API Key"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// Split tickets into individual items and check TC status\nconst tasks = $input.first().json.tasks || [];\nconst results = [];\n\nfor (const task of tasks) {\n  const subtasks = task.subtasks || [];\n  const tcSubtasks = subtasks.filter(st => st.name && st.name.startsWith('[TC]'));\n  const hasTCs = tcSubtasks.length > 0;\n  const desc = task.description || task.text_content || '';\n  const isThin = desc.length < 100;\n  \n  let action = 'skip';\n  if (!hasTCs && !isThin) action = 'generate';\n  else if (!hasTCs && isThin) action = 'monitor';\n  \n  results.push({\n    json: {\n      task_id: task.id,\n      task_name: task.name,\n      task_url: task.url,\n      status: task.status?.status || 'unknown',\n      description: desc,\n      has_tcs: hasTCs,\n      tc_count: tcSubtasks.length,\n      is_thin: isThin,\n      action: action,\n      list_id: task.list?.id || '901608730150'\n    }\n  });\n}\n\nreturn results;"
      },
      "id": "check-tc-status",
      "name": "Check TC Status per Ticket",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": true },
          "conditions": [
            {
              "leftValue": "={{ $json.action }}",
              "rightValue": "generate",
              "operator": { "type": "string", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      },
      "id": "filter-missing-tcs",
      "name": "Filter: Missing TCs Only",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1120, 300]
    },
    {
      "parameters": {
        "jsCode": "// Limit batch size to prevent rate limiting\nconst items = $input.all();\nconst BATCH_SIZE = 10;\nreturn items.slice(0, BATCH_SIZE);"
      },
      "id": "batch-limiter",
      "name": "Batch Limiter (max 10)",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1340, 200]
    },
    {
      "parameters": {
        "batchSize": 1
      },
      "id": "loop-tickets",
      "name": "Loop Each Ticket",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [1560, 200]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://api.anthropic.com/v1/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "x-api-key", "value": "={{ $env.CLAUDE_API_KEY }}" },
            { "name": "anthropic-version", "value": "2023-06-01" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ model: 'claude-sonnet-4-20250514', max_tokens: 8000, system: 'You are a Senior QA Architect. Generate comprehensive test cases for the given ClickUp ticket. Return ONLY a valid JSON array — no markdown, no backticks, no preamble, no explanation. Each TC object must have keys: tc_id, title, type, priority, module, objective, preconditions, steps, test_data, expected_result. Types: Functional – Positive, Functional – Negative, Security, Non-Functional, Edge Case, Regression. Priorities: urgent, high, normal, low. Minimum 10 TCs. Cover all scenarios.', messages: [{ role: 'user', content: 'Generate test cases for:\\n\\nTicket: ' + $json.task_name + '\\nStatus: ' + $json.status + '\\nDescription:\\n' + $json.description.substring(0, 6000) + '\\n\\nReturn ONLY valid JSON array.' }] }) }}",
        "options": {}
      },
      "id": "claude-generate-tcs",
      "name": "Claude AI — Generate TCs",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1780, 200]
    },
    {
      "parameters": {
        "jsCode": "// Parse Claude response into individual TC items\nconst response = $input.first().json;\nconst ticketData = $('Loop Each Ticket').first().json;\n\nlet textContent = '';\nif (response.content && Array.isArray(response.content)) {\n  textContent = response.content\n    .filter(b => b.type === 'text')\n    .map(b => b.text)\n    .join('\\n');\n}\n\nlet clean = textContent.replace(/```json\\s*/g, '').replace(/```\\s*/g, '').trim();\nlet tcs = [];\n\ntry {\n  tcs = JSON.parse(clean);\n} catch (e) {\n  const m = clean.match(/\\[[\\s\\S]*\\]/);\n  if (m) {\n    try { tcs = JSON.parse(m[0]); } catch(e2) {\n      return [{ json: { error: true, task_id: ticketData.task_id, task_name: ticketData.task_name, msg: e2.message } }];\n    }\n  }\n}\n\nreturn tcs.map((tc, i) => ({\n  json: {\n    ...tc,\n    parent_task_id: ticketData.task_id,\n    task_name: ticketData.task_name,\n    list_id: ticketData.list_id,\n    assignee_id: '94964380'\n  }\n}));"
      },
      "id": "parse-tcs",
      "name": "Parse TCs from Claude",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2000, 200]
    },
    {
      "parameters": {
        "batchSize": 1
      },
      "id": "loop-tcs",
      "name": "Loop Each TC",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [2220, 200]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.clickup.com/api/v2/list/{{ $json.list_id }}/task",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Authorization", "value": "={{ $env.CLICKUP_API_KEY }}" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify({ name: '[TC] ' + ($json.tc_id || 'TC') + ' - ' + ($json.title || 'Test Case'), parent: $json.parent_task_id, assignees: [parseInt($json.assignee_id)], priority: $json.priority === 'urgent' ? 1 : $json.priority === 'high' ? 2 : $json.priority === 'normal' ? 3 : 4, markdown_description: '## 📋 Test Case Details\\n**TC ID:** ' + ($json.tc_id || '') + '\\n**Module:** ' + ($json.module || '') + '\\n**Type:** ' + ($json.type || '') + '\\n**Objective:** ' + ($json.objective || '') + '\\n\\n## 🔧 Preconditions\\n' + ($json.preconditions || 'None') + '\\n\\n## 🪜 Steps\\n' + ($json.steps || '') + '\\n\\n## 🧪 Test Data\\n' + ($json.test_data || 'N/A') + '\\n\\n## ✅ Expected Result\\n' + ($json.expected_result || '') }) }}",
        "options": {}
      },
      "id": "create-tc-subtask",
      "name": "Create TC Subtask in ClickUp",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [2440, 200]
    },
    {
      "parameters": {
        "amount": 1.2
      },
      "id": "rate-limiter",
      "name": "Wait 1.2s (Rate Limit)",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [2660, 200]
    }
  ],
  "connections": {
    "Daily Midnight IST": { "main": [[{ "node": "Set Config", "type": "main", "index": 0 }]] },
    "Set Config": { "main": [[{ "node": "Fetch ClickUp Tickets", "type": "main", "index": 0 }]] },
    "Fetch ClickUp Tickets": { "main": [[{ "node": "Check TC Status per Ticket", "type": "main", "index": 0 }]] },
    "Check TC Status per Ticket": { "main": [[{ "node": "Filter: Missing TCs Only", "type": "main", "index": 0 }]] },
    "Filter: Missing TCs Only": { "main": [[{ "node": "Batch Limiter (max 10)", "type": "main", "index": 0 }], []] },
    "Batch Limiter (max 10)": { "main": [[{ "node": "Loop Each Ticket", "type": "main", "index": 0 }]] },
    "Loop Each Ticket": { "main": [[], [{ "node": "Claude AI — Generate TCs", "type": "main", "index": 0 }]] },
    "Claude AI — Generate TCs": { "main": [[{ "node": "Parse TCs from Claude", "type": "main", "index": 0 }]] },
    "Parse TCs from Claude": { "main": [[{ "node": "Loop Each TC", "type": "main", "index": 0 }]] },
    "Loop Each TC": { "main": [[], [{ "node": "Create TC Subtask in ClickUp", "type": "main", "index": 0 }]] },
    "Create TC Subtask in ClickUp": { "main": [[{ "node": "Wait 1.2s (Rate Limit)", "type": "main", "index": 0 }]] },
    "Wait 1.2s (Rate Limit)": { "main": [[{ "node": "Loop Each TC", "type": "main", "index": 0 }]] }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "timezone": "Asia/Kolkata"
  },
  "tags": [
    { "name": "qa-automation" },
    { "name": "clickup" },
    { "name": "claude-ai" }
  ]
}
```

---

## 🔧 Setup Instructions

### Step 1 — Install n8n

```bash
# Option A: Docker (recommended)
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=your_password \
  -e CLICKUP_API_KEY=pk_your_key \
  -e CLAUDE_API_KEY=sk-ant-your_key \
  n8nio/n8n

# Option B: npm (local)
npm install -g n8n
export CLICKUP_API_KEY=pk_your_key
export CLAUDE_API_KEY=sk-ant-your_key
n8n start
```

### Step 2 — Import Workflow

1. Open n8n at http://localhost:5678
2. Go to Settings → Import from File
3. Select `qa-tc-scanner-workflow.json`
4. Workflow appears in your dashboard

### Step 3 — Configure Credentials

1. In n8n → Settings → Credentials
2. Add "Header Auth" credential named "ClickUp API Key":
   - Header Name: `Authorization`
   - Header Value: `pk_your_clickup_api_key`
3. Environment variables should be set (CLICKUP_API_KEY, CLAUDE_API_KEY)

### Step 4 — Test Manually

1. Open the workflow
2. Click "Execute Workflow" (manual trigger)
3. Watch each node execute
4. Verify TCs appear in ClickUp

### Step 5 — Activate for Daily Schedule

1. Toggle the workflow to "Active" in n8n
2. CRON trigger fires daily at 00:00 IST (18:30 UTC)
3. Monitor execution history in n8n dashboard

---

## ⚙️ Configuration Reference

All configurable values are in the "Set Config" node. Change once, affects entire workflow:

| Config | Default | Description |
|--------|---------|-------------|
| CLICKUP_LIST_ID | 901608730150 | ClickUp list to scan |
| TARGET_STATUSES | ready for qa, ready for deployment, story defined | Statuses to scan |
| ASSIGNEE_USER_ID | 94964380 | Your ClickUp user ID |
| TC_PREFIX | [TC] | Prefix to identify test case subtasks |
| BATCH_SIZE | 10 | Max tickets to process per run |
| MIN_DESC_LENGTH | 100 | Minimum chars to consider description "rich" |

---

## 🛡️ Error Handling Checklist

The workflow must handle these scenarios:

1. **ClickUp API rate limit (429)** → Wait node (1.2s) between calls + retry on 429
2. **Claude API error (5xx)** → Retry 1x with 5s delay, then skip ticket and log
3. **Claude returns non-JSON** → Regex extraction fallback, then skip if still fails
4. **Empty ticket list** → Skip to summary with "0 tickets found"
5. **ClickUp subtask creation fails** → Log failure, continue with remaining TCs
6. **Ticket description too long** → Truncate to 6000 chars before sending to Claude
7. **No tickets in target statuses** → Send "All clear" summary notification

---

## 📊 Monitoring & Observability

1. **n8n execution history** — every run logged with node-by-node data
2. **Slack/email summary** — sent after every run with counts
3. **Error alerts** — failed runs trigger n8n's built-in error workflow
4. **Manual re-run** — click "Execute" in n8n to run anytime outside schedule

---

## 🧪 Acceptance Criteria for This Workflow

Before going to production, verify:

- [ ] CRON fires at midnight IST daily
- [ ] Scans correct ClickUp list (901608730150)
- [ ] Correctly identifies tickets WITHOUT [TC] subtasks
- [ ] Skips tickets WITH [TC] subtasks (no duplicates)
- [ ] Skips tickets with thin/empty descriptions
- [ ] Claude API generates structured JSON TCs
- [ ] TCs pushed as subtasks with correct [TC] prefix format
- [ ] TCs assigned to correct user (94964380)
- [ ] TC body follows structured template (TC ID, Module, Steps, Expected Result)
- [ ] Priority mapping correct (urgent/high/normal/low)
- [ ] Rate limiting prevents ClickUp API 429 errors
- [ ] Summary notification sent after each run
- [ ] Errors logged but don't crash entire workflow
- [ ] Batch limiter caps at 10 tickets per run
- [ ] Workflow can be manually triggered anytime
