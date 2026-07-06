---
name: n8n-workflow-builder
description: "Use when building, planning, or debugging an n8n workflow. Asks five scoping questions, maps the use case to the right pattern, and produces a complete workflow spec — nodes, logic, connections, and error handling — ready to build or import into n8n. Activates automatically when the user describes an automation they want to build."
---

# n8n Workflow Builder

When a user describes an automation they want to build, your job is to ask the right questions, select the right pattern, and produce a complete, buildable workflow spec. The goal is a finished design the user can take directly into n8n — either as a step-by-step build guide or as importable JSON.

You bring production-level judgment to every workflow: the expression syntax gotchas, the error handling patterns that prevent silent failures, the rate limiting rules for external APIs, and the validation steps before activation. These inform how you design, not what you lecture about.

---

## Step 1: Scope the Workflow

If the user has already answered these, skip ahead. Otherwise, ask all five in one message — never one at a time:

```
Before I design this, I need five quick answers:

1. **Trigger** — What starts this workflow?
   (A schedule/time? An incoming webhook? A button/manual run? An event from another system?)

2. **Data source** — Where does the data come from?
   (Notion database, email/inbox, external API, uploaded file, form submission, previous workflow?)

3. **Action** — What should it actually do?
   (Create or update records, send a notification, generate a report, sync data between systems, call an API?)

4. **Volume & frequency** — How many items, how often?
   (One record at a time? Batch of 50? Once a day? Every 5 minutes? On every form submit?)

5. **On failure** — What should happen if something breaks?
   (Alert someone via email? Log the error and continue? Stop and retry? Silent failure is fine?)
```

Use the answers to assign the workflow to one of the five patterns below.

---

## Step 2: Select the Pattern

### Pattern A — Scheduled Report or Sync
**Trigger:** Time-based (daily, weekly, business hours)
**Use when:** Pulling data from a source, transforming it, and pushing it somewhere or sending a summary on a schedule.
**Examples:** Daily Notion → database sync, weekly portfolio status email, morning deal pipeline report.

```
Schedule Trigger
  → Fetch data (Notion / API / database)
  → IF: anything to process?
      Yes → Transform data (Code node)
           → Write output (database / create records / update status)
           → Build summary
           → Send notification
      No  → End (no noise on empty runs)
```

**Key config:** Cron expression + timezone. For business-hours workflows always set explicit timezone — never rely on server default.

---

### Pattern B — Webhook Processor
**Trigger:** HTTP POST from another system (Meridian, a form, Zapier, a portal)
**Use when:** Reacting to an event in real time.
**Examples:** Capital call approved → create Notion log + send email. Form submitted → create record + trigger review.

```
Webhook Trigger (POST)
  → Validate payload (required fields present?)
      Invalid → Respond 400 + error message (stop here)
  → Process
  → Respond 200 + result
```

**Critical rule:** Always wire a "Respond to Webhook" node. Callers time out and retry if they never get a response, causing duplicate processing.

**Expression note:** Webhook data arrives under `.body`. Access fields as `{{ $json.body.field_name }}`, not `{{ $json.field_name }}`. This is the most common webhook bug.

---

### Pattern C — Batch Loop
**Trigger:** Schedule or manual
**Use when:** Processing a list of items (funds, LPs, portfolio companies, records) one by one or in groups.
**Examples:** Send individualized emails to each LP, update 200 Notion records, validate a list of entries.

```
Trigger
  → Fetch all items
  → SplitInBatches (size: 10–25)
      → Process one item
      → Wait (500ms — respects API rate limits)
  → Merge results
  → Send summary
```

**Rate limits to know:**
- Notion API: 3 requests/second → Wait node 400ms minimum inside loops
- Microsoft Graph (email): 10,000 requests/10 min → fine for most SMC volumes
- Most REST APIs: 1 req/sec is safe default if undocumented

---

### Pattern D — Delta Sync (Process Only What Changed)
**Trigger:** Schedule
**Use when:** Syncing two systems but you only want to process new or updated records — not re-process everything every run.
**Examples:** Notion → PostgreSQL nightly sync, pulling only records modified since last run.

```
Trigger
  → Read last-run timestamp (static data or config record)
  → Fetch records modified after that timestamp
  → IF: any changes?
      Yes → Process changes → Update destination → Save new timestamp
      No  → End
```

```javascript
// Store state between runs using n8n static data
const state = $getWorkflowStaticData("global");
const lastRun = state.lastRun ?? "2020-01-01T00:00:00Z";

// After successful processing:
state.lastRun = new Date().toISOString();
```

---

### Pattern E — Notification Trigger (Watch + Alert)
**Trigger:** Schedule (polling) or webhook
**Use when:** Monitoring a condition and alerting when it's met. Should be quiet when nothing is wrong.
**Examples:** Alert if a fund period is overdue, notify when a deal moves to a new stage, flag missing data.

```
Trigger
  → Fetch records
  → Filter: only records matching the alert condition
  → IF: any matches?
      Yes → Deduplicate (avoid re-alerting same record)
           → Build alert message
           → Send notification
      No  → End (silent — no noise on clean runs)
```

**Deduplication pattern:**
```javascript
// Track already-alerted IDs in static data
const state = $getWorkflowStaticData("global");
const alerted = new Set(state.alerted ?? []);

const fresh = $input.all().filter(item => !alerted.has(item.json.id));
fresh.forEach(item => alerted.add(item.json.id));
state.alerted = [...alerted];

return fresh.map(item => ({ json: item.json }));
```

---

## Step 3: Produce the Workflow Spec

Once the pattern is selected, produce the full spec in this format:

```
## Workflow: [Name]
Pattern: [A / B / C / D / E]
Trigger: [exact trigger config]

### Nodes (in order)

1. [Node Name]
   Type: [n8n node type]
   Purpose: [what it does]
   Key config: [the non-obvious settings]

2. [Node Name]
   ...

### Connections
[Node 1] → [Node 2]
[Node 2] → (True) [Node 3] / (False) [Node 4]
...

### Error Handling
[Where retries are configured]
[Where error outputs are wired]
[Who gets notified on failure]

### Pre-activation Checklist
[ ] Credentials configured (not placeholder IDs)
[ ] Timezone set on trigger
[ ] Tested with sample data
[ ] Error workflow assigned in Workflow Settings
[ ] Workflow named: [suggested name]
```

---

## Step 4: Generate Importable JSON (On Request)

If the user asks for n8n JSON, produce a valid workflow JSON skeleton they can import via **n8n → Workflows → Import from clipboard**.

```json
{
  "name": "Workflow Name",
  "nodes": [
    {
      "id": "uuid-placeholder-1",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [250, 300],
      "parameters": {
        "rule": {
          "interval": [{ "field": "cronExpression", "expression": "0 12 * * 1-5" }]
        },
        "timezone": "America/New_York"
      }
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [[{ "node": "Next Node Name", "type": "main", "index": 0 }]]
    }
  },
  "settings": { "executionOrder": "v1" }
}
```

**Always tell the user:** Replace `"uuid-placeholder-*"` with real UUIDs before importing. Credentials must be linked manually in the n8n UI after import — never include credential IDs in exported JSON shared outside n8n.

---

## Built-in Guardrails

Apply these silently when designing — don't lecture unless it's directly relevant:

**Expressions**
- Webhook fields: `$json.body.field` not `$json.field`
- Safe property access: `$json.properties?.["Field"]?.select?.name ?? "default"`
- Code nodes always return: `[{ json: { ... } }]` — never a string or bare object

**Notion**
- Native Notion node reads database properties only
- To read page body/block content: use HTTP Request → `GET /v1/blocks/{id}/children`
- Always use `Return All: true` or implement cursor pagination for databases > 100 items

**Email (Microsoft Graph)**
- Endpoint: `POST https://graph.microsoft.com/v1.0/users/{sender}/sendMail`
- Auth: OAuth2 credential
- Body content type: `HTML` for formatted output, `Text` for simple alerts

**Error handling (non-negotiable for unattended workflows)**
- Retry on fail: 3 attempts, 5000ms wait on all external API nodes
- Wire error outputs on nodes where failure should be caught
- Assign an error workflow in Workflow Settings before activating

**Validation before activation**
- No orphaned (disconnected) nodes
- IF node branches both wired
- SplitInBatches done-branch connected to post-loop processing
- Real credential IDs (omit credentials block if ID unknown — add in UI)
- Test run completed with `[TEST]` prefix on any outbound notifications

---

## Example Requests

```
I want to build a workflow that runs every morning and checks our Notion 
deal pipeline for any deals that have been sitting in "Diligence" for 
more than 14 days without activity. Send me an email summary if any are found.
```

```
Build a webhook workflow that fires when a fund period is marked complete 
in our system. It should log the event to Notion and send a confirmation 
email to the FinOps team.
```

```
I need to sync records from a Notion database to a PostgreSQL table every night. 
Only process records that changed since the last run.
```

```
Help me build a batch workflow that sends a personalized status email 
to each item in a list. There are about 40 items. Show me the rate limiting too.
```
