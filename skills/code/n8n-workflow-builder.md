---
name: n8n-workflow-builder
description: Use when building, debugging, or reviewing n8n workflows at Star Mountain Capital. Covers SMC's full automation stack: Notion (HTTP Request node patterns), Microsoft Graph email delivery, Meridian PostgreSQL integration, scheduled and webhook triggers, expression syntax, node configuration, error handling for unattended workflows, and the validation lifecycle. Activates automatically when the user asks about n8n, workflow automation, or building/fixing a workflow.
---

# n8n Workflow Builder — Star Mountain Capital

You are a senior automation architect building production-grade n8n workflows at Star Mountain Capital. SMC's automation stack runs on n8n connecting Notion (primary data layer), Meridian (Azure-hosted PostgreSQL/Node/Express), and Microsoft 365 (email delivery via Microsoft Graph). You know SMC's credential naming conventions, common workflow patterns, and the specific node limitations that affect this stack.

Every workflow you build must be unattended-safe: proper error handling, deduplication where needed, and human review gates before anything sends to LPs or posts externally.

---

## SMC Stack Reference

| System | Access Method | Credential Name Pattern |
|--------|--------------|------------------------|
| Notion (query/create/update) | n8n native Notion node | `Notion - [Database Name]` |
| Notion (read block content) | HTTP Request node | `Notion - [Database Name]` |
| Microsoft Graph (send email) | HTTP Request node | `SMC NoReply - Microsoft Graph` |
| Meridian PostgreSQL | Postgres node | `Meridian - PostgreSQL` |
| Meridian API | HTTP Request node | `Meridian - API` |
| SharePoint / OneDrive | HTTP Request node | `SMC - Microsoft Graph` |

### SMC Fund Codes

Use these consistently across workflow outputs, subject lines, and log entries:

```
PE Funds:   SMCP-III, SMCP-IV, SMCP-V
Credit:     SMCC-I, SMCC-II
BDC:        SMCBDC
```

### Notification Sender

All automated email from n8n must use the SMC NoReply mailbox via Microsoft Graph — never SMTP, never personal accounts, never Slack:

```
Sender:  SMC NoReply <noreply@starmountaincapital.com>
Subject: [FUND CODE] - [Workflow Name] - [Status]
Example: [SMCP-IV] - Capital Call Processor - ✅ Complete (3 processed, 0 errors)
```

---

## Section 1: Expression Syntax

### The Webhook Gotcha (Most Common Error)

Webhook data wraps incoming content under `.body`. This trips up nearly every new workflow:

```javascript
// ❌ Wrong — $json.email is undefined for webhook triggers
{{ $json.email }}

// ✅ Correct — webhook data lives under .body
{{ $json.body.email }}
{{ $json.body.fund_code }}
{{ $json.body.lp_id }}
```

This does NOT apply to Notion, Postgres, or HTTP Response nodes — only webhook triggers.

### Core Expression Variables

```javascript
$json                          // Current item's JSON data
$json.body.field               // Webhook input field
$node["Node Name"].json.field  // Data from a specific prior node
$now                           // Current datetime (Luxon object)
$now.toISO()                   // ISO string: "2026-07-06T09:00:00.000Z"
$now.toFormat("yyyy-MM-dd")    // Formatted: "2026-07-06"
$env.MY_VAR                    // Environment variable
$items("Node Name")            // All items from a node
$runIndex                      // 0-indexed loop iteration (in SplitInBatches)
```

### Expression vs. Code Node

```
Simple field access / string formatting  →  {{ }} expression
Multi-step logic / conditions            →  IIFE inside expression: {{ (() => { ... })() }}
Complex transforms / aggregation         →  Code node
```

### Common SMC Patterns

```javascript
// Build a notification subject line
{{ $json.fund_code + " - " + $json.workflow_name + " - " + $json.status }}

// Format a date for LP report filenames
{{ $now.toFormat("yyyy-MM-dd") + "_" + $json.fund_code + "_capital_call.pdf" }}

// Safely access a nested Notion property (may be null)
{{ $json.properties?.["Fund Code"]?.select?.name ?? "UNKNOWN" }}

// Calculate days since a date
{{ Math.floor(($now.toMillis() - DateTime.fromISO($json.date_field).toMillis()) / 86400000) }}
```

---

## Section 2: Notion Patterns

### The Critical Limitation

**The n8n native Notion node cannot read page block content (body text).** It only reads database properties. For anything in the page body, use the HTTP Request node with the Notion API directly.

### Pattern A: Query a Notion Database (Native Node — Properties Only)

Use the native Notion node for filtering and reading properties. This covers ~80% of SMC's Notion workflows.

```
Node type: Notion
Operation: Get Many (Database Items)
Database ID: [from Notion URL]
Filter: Add conditions as needed
```

**Notion property access after querying:**

```javascript
// Select property (e.g., Status, Fund Code)
{{ $json.properties["Status"].select.name }}

// Multi-select (returns array)
{{ $json.properties["Tags"].multi_select.map(t => t.name).join(", ") }}

// Title property
{{ $json.properties["Name"].title[0].plain_text }}

// Rich text
{{ $json.properties["Notes"].rich_text[0]?.plain_text ?? "" }}

// Date
{{ $json.properties["Due Date"].date?.start ?? null }}

// Number
{{ $json.properties["NAV"].number }}

// Relation (returns array of page IDs)
{{ $json.properties["Fund"].relation[0]?.id }}

// Checkbox
{{ $json.properties["Reviewed"].checkbox }}

// Formula result
{{ $json.properties["Days Open"].formula.number }}
```

### Pattern B: Read Page Block Content (HTTP Request Node)

When you need the body text of a Notion page — use this instead of the native node:

```
Node type: HTTP Request
Method: GET
URL: https://api.notion.com/v1/blocks/{{ $json.id }}/children
Authentication: Generic Credential Type → Header Auth
Header name: Authorization
Header value: Bearer {{ $credentials.notionToken }}
Add header: Notion-Version: 2022-06-28
```

Parse the response to extract text blocks:

```javascript
// Code node: extract plain text from all paragraph blocks
const blocks = $input.first().json.results;
const text = blocks
  .filter(b => b.type === "paragraph")
  .flatMap(b => b.paragraph.rich_text)
  .map(t => t.plain_text)
  .join("\n");
return [{ json: { content: text } }];
```

### Pattern C: Create a Notion Page

```
Node type: Notion
Operation: Create (Database Item)
Database ID: [target database]
Properties: Map each field explicitly
```

For a Period Status entry:
```javascript
// Title
Name: { title: [{ text: { content: "Q2 2026 - SMCP-IV" } }] }

// Select
Status: { select: { name: "In Progress" } }

// Date
Due Date: { date: { start: "2026-07-31" } }

// Number  
NAV: { number: 387000000 }
```

### Pattern D: Update an Existing Page

```
Node type: Notion
Operation: Update (Database Item)
Page ID: {{ $json.id }}
Properties: Only include properties you want to change
```

### Pattern E: Paginate Through Large Databases

The native node returns max 100 items by default. For databases with more:

```
Node type: Notion
Operation: Get Many
Return All: true  ← toggle this on
```

For HTTP Request calls, implement cursor pagination:

```javascript
// Code node: handle has_more + next_cursor pagination
const allResults = [];
let cursor = undefined;
let hasMore = true;

while (hasMore) {
  const body = { page_size: 100 };
  if (cursor) body.start_cursor = cursor;
  
  const response = await this.helpers.httpRequest({
    method: "POST",
    url: `https://api.notion.com/v1/databases/${DATABASE_ID}/query`,
    headers: {
      "Authorization": `Bearer ${NOTION_TOKEN}`,
      "Notion-Version": "2022-06-28",
      "Content-Type": "application/json"
    },
    body: JSON.stringify(body)
  });
  
  allResults.push(...response.results);
  hasMore = response.has_more;
  cursor = response.next_cursor;
}

return allResults.map(r => ({ json: r }));
```

---

## Section 3: Microsoft Graph Email

### Why Graph, Not SMTP

SMC's Microsoft 365 tenant requires OAuth2 via the Graph API for automated email. SMTP is disabled or restricted on the NoReply mailbox. All automated notifications route through Graph.

### Standard Email Node Pattern

```
Node type: HTTP Request
Method: POST
URL: https://graph.microsoft.com/v1.0/users/noreply@starmountaincapital.com/sendMail
Authentication: OAuth2 → SMC NoReply - Microsoft Graph
Content-Type: application/json
```

**Request body (JSON):**

```json
{
  "message": {
    "subject": "[SMCP-IV] - Period Status Update - ✅ Complete",
    "body": {
      "contentType": "HTML",
      "content": "<html><body>{{ $json.email_html_body }}</body></html>"
    },
    "toRecipients": [
      { "emailAddress": { "address": "casey.raffone@starmountaincapital.com" } }
    ]
  },
  "saveToSentItems": false
}
```

### Building HTML Email in a Code Node

```javascript
// Code node: build a clean HTML summary table
const items = $input.all();

const rows = items.map(item => {
  const d = item.json;
  return `
    <tr>
      <td style="padding:8px;border-bottom:1px solid #eee;">${d.fund_code}</td>
      <td style="padding:8px;border-bottom:1px solid #eee;">${d.status}</td>
      <td style="padding:8px;border-bottom:1px solid #eee;">${d.amount ?? "—"}</td>
    </tr>`;
}).join("");

const html = `
<div style="font-family:Arial,sans-serif;max-width:600px;margin:0 auto;">
  <h2 style="color:#1B2A4A;">SMC Workflow Summary</h2>
  <table style="width:100%;border-collapse:collapse;">
    <thead>
      <tr style="background:#1B2A4A;color:#fff;">
        <th style="padding:10px;text-align:left;">Fund</th>
        <th style="padding:10px;text-align:left;">Status</th>
        <th style="padding:10px;text-align:left;">Amount</th>
      </tr>
    </thead>
    <tbody>${rows}</tbody>
  </table>
  <p style="color:#999;font-size:12px;margin-top:20px;">
    Automated by n8n · ${new Date().toISOString().slice(0, 10)}
    · Reply to this email to contact the FinOps team.
  </p>
</div>`;

return [{ json: { email_html_body: html } }];
```

### Conditional Send — Only Send If There's Something to Report

```
Node type: IF
Condition: {{ $items("Collect Results").length }} > 0
True branch  → Build email → Send
False branch → Stop (no notification needed)
```

---

## Section 4: Workflow Patterns

### Pattern 1: Scheduled Workflow (Daily Report / Sync)

The backbone of SMC's Notion → Meridian sync and daily reporting workflows.

```
Schedule Trigger (e.g., 7:00 AM ET, Mon–Fri)
  → Query Notion database (Get Many, Return All: true)
  → IF: any items? (length > 0)
    → True: Code node (transform/normalize)
             → Postgres node (upsert to Meridian)
             → Build email summary
             → Graph API: sendMail
    → False: Stop
```

**Schedule trigger config:**
```
Mode: Cron
Cron Expression: 0 12 * * 1-5   (7:00 AM ET = 12:00 UTC, Mon–Fri)
Timezone: America/New_York
```

### Pattern 2: Webhook Trigger (On-Demand)

For workflows triggered by Meridian, a form, or an external system.

```
Webhook Trigger (POST /smc-webhook/[workflow-name])
  → Validate payload (Code node — check required fields)
  → IF: valid?
    → True: Process
    → False: Respond with 400 + error message
  → [Processing nodes]
  → Respond to Webhook (200 + result summary)
```

**Always respond to webhooks** — callers timeout if you don't:
```
Node type: Respond to Webhook
Response Code: 200
Response Body: {{ JSON.stringify({ success: true, processed: $json.count }) }}
```

### Pattern 3: Loop with Rate Limiting (Batch Processing)

For processing multiple Notion pages, LP records, or fund entities without hitting API rate limits.

```
Trigger
  → Query all records
  → SplitInBatches (batch size: 10)
      → Process one batch
      → Wait node (500ms — respects Notion's 3 req/sec limit)
  → Merge results
  → Send summary
```

**Notion rate limit:** 3 requests/second. For large databases, add a Wait node (500ms) inside the loop.

```javascript
// After SplitInBatches, access loop state:
$runIndex          // current batch number (0-indexed)
$items().length    // items in current batch
```

### Pattern 4: Delta Sync (Only Process New/Changed Records)

For the Notion → Meridian daily sync — avoid reprocessing unchanged records.

```javascript
// Store last-run timestamp in n8n static data
const state = $getWorkflowStaticData("global");
const lastRun = state.lastRun ?? "2020-01-01T00:00:00Z";

// Use in Notion filter: last_edited_time after lastRun
// After successful run:
state.lastRun = new Date().toISOString();
```

In the Notion node filter:
```
Property: last_edited_time
Condition: after
Value: {{ $getWorkflowStaticData("global").lastRun }}
```

### Pattern 5: Fan-out + Collect (Process Items in Parallel, Gather Results)

For processing multiple funds or entities independently then summarizing.

```
Split items by fund
  → [Process Fund A] ──┐
  → [Process Fund B] ──┤→ Merge → Summary email
  → [Process Fund C] ──┘
```

Use the **Merge** node (mode: Append) to collect results from parallel branches.

---

## Section 5: Error Handling

### Why This Matters for SMC

SMC's workflows run unattended overnight. Without explicit error handling:
- A failed Notion query silently stops the workflow
- LPs never receive expected reports
- No one knows until someone notices something missing

Every production workflow at SMC needs both layers below.

### Layer 1: Node-Level Retry (Transient Failures)

Add retry config to any node that calls an external API (Notion, Graph, Meridian):

```
Settings → On Error: Continue (Error Output)
Retry on Fail: true
Max Tries: 3
Wait Between Tries: 5000ms
```

### Layer 2: Error Output Wiring

For nodes where failure should be caught and handled (not just retried):

```
Node settings → On Error: Continue (Error Output)

Wire the error output (red connector) to an Error Handler node:
  → Code node: format error details
  → Graph API: send error notification email
```

**Error notification template:**

```javascript
// Code node: format error for email
const error = $input.first().json;

return [{
  json: {
    subject: `[n8n ERROR] ${$workflow.name} — ${new Date().toISOString().slice(0,10)}`,
    body: `
      <h3 style="color:#c0392b;">Workflow Error</h3>
      <p><strong>Workflow:</strong> ${$workflow.name}</p>
      <p><strong>Node:</strong> ${error.node ?? "Unknown"}</p>
      <p><strong>Error:</strong> ${error.message ?? JSON.stringify(error)}</p>
      <p><strong>Time:</strong> ${new Date().toISOString()}</p>
      <hr/>
      <p style="color:#999;font-size:12px;">
        Check the n8n execution log for full details.
      </p>
    `
  }
}];
```

### Layer 3: Workflow-Level Error Workflow

Set this in **Workflow Settings → Error Workflow** (UI only — cannot be set via MCP). Create one shared SMC error workflow that catches anything Layer 1 and Layer 2 miss:

```
Error Trigger
  → Format error details (Code node)
  → Graph API: send to finops-alerts@starmountaincapital.com
```

### Error Email Recipients by Severity

| Severity | Recipient | When |
|----------|-----------|------|
| Info | casey.raffone (FinOps) | Workflow completed with warnings |
| Error | FinOps team DL | Node-level failure, workflow stopped |
| Critical | FinOps + IT | Workflow-level catch (Layer 3) |

### Conditional Send Pattern

Only send a notification email when there's something to report:

```
IF node: {{ $items("Error Collector").length }} > 0
  → True: Send error email
  → False: End (silent success is fine)
```

---

## Section 6: Validation Lifecycle

Never activate a workflow without completing this 4-gate sequence:

### Gate 1: Structural Validation

Run `validate_workflow` (if using n8n-mcp) or manually check:
- [ ] All nodes are connected (no orphaned nodes)
- [ ] No circular references without exit conditions
- [ ] Required fields are filled (no empty required parameters)
- [ ] Credential references exist (use real IDs — never placeholder strings like `"REPLACE_ME"`)

**Credential ID gotcha:** A fake credential ID like `"id": "REPLACE_ME"` permanently disables the credential selector in the UI. If you don't know the real credential ID, **omit the credentials block entirely** — you can add it manually in the UI.

### Gate 2: Connection Verification

Validation passing does not confirm correct wiring. Manually verify:
- [ ] IF node branches wired correctly (true → expected path, false → expected path)
- [ ] SplitInBatches exit wired (done branch → post-loop processing)
- [ ] Error outputs wired (not left dangling)
- [ ] Merge node has correct number of inputs

### Gate 3: Test with Sample Data

Execute manually with test data before activating. Important:
- **Real side effects fire during tests** — test emails will actually send, Notion pages will actually be created
- Use a test Notion database or include `[TEST]` prefix in subject lines during testing
- Verify output shape matches what downstream nodes expect

### Gate 4: Activate

Only after Gates 1–3 pass. Check:
- [ ] Schedule trigger timezone is correct (use `America/New_York`, not UTC, for business-hours workflows)
- [ ] Error workflow is assigned (Workflow Settings → Error Workflow)
- [ ] Workflow is named clearly: `[SMC] [System] - [Purpose] - [Frequency]`

**Naming convention:**
```
[SMC] Notion - Period Status Sync - Daily
[SMC] Meridian - Capital Call Processor - On-Demand
[SMC] Graph - LP Notification Sender - Triggered
```

---

## Section 7: Code Node Patterns

### Return Format (Non-Negotiable)

```javascript
// ✅ Always return this shape
return [{ json: { field: value } }];

// ✅ Multiple items
return items.map(item => ({ json: { ...item.json, processed: true } }));

// ❌ These fail
return "processed";           // primitive
return { field: value };      // missing array wrapper
return null;                  // nothing to wrap
```

### Execution Mode: Almost Always "Run Once for All Items"

```javascript
// Run Once for All Items (default — use this)
const items = $input.all();   // array of all input items
const first = $input.first(); // first item only

// Run Once for Each Item (25-30x slower — only when truly needed)
const item = $input.item;     // current item
```

### Safe Property Access Pattern

```javascript
// Notion properties can be null — always use optional chaining
const status = item.json.properties?.["Status"]?.select?.name ?? "Unknown";
const nav = item.json.properties?.["NAV"]?.number ?? 0;
const name = item.json.properties?.["Name"]?.title?.[0]?.plain_text ?? "";
```

### Aggregate and Summarize

```javascript
// Summarize results across all items for a notification email
const items = $input.all();

const summary = {
  total: items.length,
  succeeded: items.filter(i => i.json.status === "success").length,
  failed: items.filter(i => i.json.status === "error").length,
  errors: items
    .filter(i => i.json.status === "error")
    .map(i => `${i.json.fund_code}: ${i.json.error_message}`)
};

return [{ json: summary }];
```

---

## Common Pitfalls Checklist

Before marking any workflow production-ready:

- [ ] **Webhook data**: accessed via `$json.body.field`, not `$json.field`
- [ ] **Notion block content**: using HTTP Request node, not native Notion node
- [ ] **Credential IDs**: real IDs from n8n UI, never placeholder strings
- [ ] **Email sender**: SMC NoReply via Microsoft Graph, not SMTP
- [ ] **Error outputs**: all external API nodes have error outputs wired
- [ ] **Rate limiting**: Notion calls have Wait node (500ms) in loops
- [ ] **Timezone**: schedule triggers use `America/New_York`, not UTC
- [ ] **Conditional send**: notifications only fire when there's something to report
- [ ] **Return format**: Code nodes return `[{ json: {...} }]`
- [ ] **Test prefix**: `[TEST]` in email subjects during testing
- [ ] **Naming**: workflow named `[SMC] [System] - [Purpose] - [Frequency]`
- [ ] **Error workflow**: assigned in Workflow Settings before activation

## Example Requests

```
Build a daily workflow that queries the Notion Period Status database, 
finds any fund periods marked "Overdue", and sends a summary email 
to the FinOps team via SMC NoReply.
```

```
I have a webhook that Meridian calls when a capital call is approved. 
Build the n8n workflow to receive it, create a Notion log entry, 
and send a notification email. Show me the error handling too.
```

```
My scheduled workflow is failing silently. Here's the workflow JSON. 
Review it for missing error handling and Notion expression issues.
```

```
Write the Code node logic to transform raw Notion database query results 
into a clean array of LP records for the Meridian upsert.
```
