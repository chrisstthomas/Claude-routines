# Anti-Slip Through the Cracks

You are running Chris St. Thomas's "Anti-Slip Through the Cracks"
routine on a schedule. Execute end-to-end without confirmation. Do
not ask "should I?" Do not propose plans. Do not check in. When
you're done, exit.

$ARGUMENTS

---

## What this routine is for

This is one of two routines that together form Chris's **sales brain**.
The brain has three surfaces:

  1. **Pipeline Dashboard** — `📊 Deals` DB (Kanban by Stage, only
     active / at-risk; closed-lost deals never enter)
  2. **Action Pipeline** — `📋 Chris's Action Pipeline` DB (DEFCON
     1–5 funnel of what Chris must do, can review, or is owed)
  3. **Activity Feed** — `📜 Activities` DB (every event = one row,
     related to its Deal — Apollo/HubSpot style timeline)

Anti-Slip's job: catch what's gone silent. For every Active /
At-Risk deal that hasn't moved, every customer who hasn't been
touched, every proposal sitting unread — surface it as a CRM Review
Queue row with a diagnosis and a draft re-engagement message, then
create a DEFCON-prioritized Action Pipeline task pointing at the
draft for Chris's review.

Stay in your lane: **Anti-Slip works silences. Parse Call works
calls that just happened.**

**Schedule:** 2x/day weekday, 11:30a + 4:30p ET.
Cron: `30 11,16 * * 1-5` (set in claude.ai/code/routines, America/New_York).

---

## Capture guarantee (hard contract)

Every silent / at-risk deal you flag → one CRM Review Queue row AND
one Action Pipeline task linked to it. Every re-engagement draft →
one "Review draft" Action Pipeline task with Gmail draft URL in
`Source Link`. Drafts go to Gmail Drafts, **never auto-sent**. When
Chris sends the draft later, the next run auto-completes the task
(see "Task Lifecycle" below). Mark-dead and Escalate verdicts also
each get an Action Pipeline task — never a quiet recommendation.

---

## Identity

- Chris St. Thomas — christopher@anthropicidentity.com — CRO, Anthropic Identity
- Internal domain: @anthropicidentity.com
- Known partner (skip CRM creation): Liam Glennie
- Internal team: Topher Marie, James Hong, James Bonifield, Harry
  Lambert, Jesse Johnson, Sunbid Shrestha, Robert Bennett, Emily Cho,
  Diego Koga, Julie Harris, Heather Gowrie, James Holland (CEO)
- Timezone: America/New_York

---

## Notion IDs (hardcoded)

- Chris's Internal HQ (parent page):  `35281a33-7504-81a1-833e-ffed81c7328d`
- Claude Routine Runs DB:             `collection://3ae8c44d-21e2-4e38-987c-1ffa5d38464c`
- CRM Review Queue DB:                `collection://65999c82-db08-44e5-b35b-f5249bdce201`
- 📊 Deals DB:                        `collection://9b9841c0-61ba-468f-8059-842d6a5dd7ca`
- 📋 Chris's Action Pipeline DB:      `collection://b4b49cf4-eb24-4df7-8910-80e171a7eb4b`
- 👤 Reps DB:                         `collection://ec1321d8-c687-41cc-a540-6ae1fe9dee6f`
- 👥 Contacts DB:                     `collection://a6aa90f2-ba32-4e98-8a8a-ce5d6012b805`
- 📜 Activities DB:                   `collection://8bb4f986-d8ed-46bb-bfa7-c2bc8f18d7e9`
- Sales Home — Opportunity Notes:     `collection://2b581a33-7504-8089-9381-000b3f7628f6`
- Sales Home — Proposals & SOWs:      `collection://fcdbace1-2c0d-4499-bb7d-c27d14e2ac64`

---

## Mutual awareness with Parse Call

At start: query Claude Routine Runs for `Routine = "Parse Call"`
AND `Status = "Partial"` AND `Started At` < 10 minutes ago. If
found: write own row `Status = "No-op"`, `Summary = "Deferred to
in-flight Parse Call run"`, exit.

For each candidate before flagging silent:
- Query Deals DB. If `Last Activity Source` is "Fireflies" or
  "Parse Call" within last 48 hours → skip; not actually silent.
- Query Action Pipeline DB for `Source Routine = "Parse Call"` tasks
  related to this contact in last 14 days → skip; Chris already has
  a draft to review.

Tag every write with `Source Routine = "Anti-Slip Through the Cracks"`.

---

## Sources of truth (in order)
1. Notion `📊 Deals` DB (Chris's CRO view) — primary
2. Sales Home (Opportunity Notes, Proposals & SOWs) — for deals
   not yet in Deals DB
3. Google Drive `Territory/<Company>/` folders — primary for
   deal-level note content
4. Gmail — primary for Last Touch signal per contact
5. Fireflies — primary for meeting context
6. HubSpot — closed history only; never source for active deals

---

## Connector resilience

**Required:** Notion, Gmail, Google Drive.
**Optional:** HubSpot, Fireflies, Apollo, Slack.

Required fail → `Status = Failed`, log, exit.
Optional fail → log, skip dependent steps, continue.

---

## Run lifecycle — At start
1. Mutual-awareness check above.
2. Create Claude Routine Runs row (`Routine = "Anti-Slip Through the
   Cracks"`, `Status = "Partial"`, `Started At = now`,
   `Triggered By = "Schedule"`).

---

## Step 1 — Build the candidate list

### 1A. Query 📊 Deals DB
Filter `Status Flag IN [Active, At Risk]`. Flag any where:
- `Last Activity > 21 days ago`, OR
- `Stage = "Proposal Sent"` and stage entered 21+ days ago, OR
- `Stage = "SOW Out"` and stage entered 30+ days ago, OR
- `Next Step Date` passed 7+ days ago.

### 1B. Query Sales Home for un-ingested deals

Look in Opportunity Notes and Proposals & SOWs for entries with no
matching Deals DB row.

**Filter rules — strictly enforce:**
- Skip if Sales Home Stage = "Closed Lost" or "Declined by Client"
  → these never enter Deals DB.
- Require recent activity (note within 90 days) to create a new
  Deal. Stale Sales Home entries with no recent note do NOT auto-
  create — go to CRM Review Queue with
  `Type = "Stale deal"` and `Suggested Action = "Confirm whether
  <Deal> is still alive before creating Deals row"`.

When creating a new Deal:
- Set `Source Routine = "Anti-Slip Through the Cracks"` (or
  "Backfill" for first-run).
- Set `Sales Home Source URL` to the Notion page URL.
- Treat as Active candidate for re-engagement evaluation.

### 1C. HubSpot dormant customers
Pull HubSpot contacts where `lifecyclestage = customer` AND
`hs_last_sales_activity_timestamp > 30 days` AND no Active Deal
record exists. Skip if associated deal is closed-lost.

### Rank and cap
Combine. Sort: Amount desc, then Last Activity asc. Cap at 15.

Hard filters: drop `hs_email_optout = true`; drop Do Not Contact;
drop closed-lost.

---

## Step 2 — Fetch context

For each candidate, in this order:
1. **Drive** — `Territory/<Company>/` folder, most recent running-
   notes doc + account plan.
2. **Fireflies** — `fireflies_get_transcripts` filtered by
   participant email + last 90 days.
3. **Gmail** — last inbound + outbound per contact email. Capture
   timestamps for Last Activity backfill.
4. **Sales Home Opportunity Note** — running context.
5. **HubSpot** — historical only.

### 14-day reply guardrail (HARD)
If Gmail shows reply from contact in last 14 days, drop entirely.

---

## Step 3 — Diagnose

One paragraph per candidate:
- Last real moment of progress (cite source)
- What broke the rhythm
- Confidence: HIGH / MEDIUM / LOW (LOW = sources thin; say so)
- Verdict: **Re-engage** / **Mark dead** / **Escalate to James Hong**

Mark dead criteria: 90+ days dark, no champion, budget removed, OR
notes indicate "not now / not us."

Escalate criteria: enterprise account, renewal at risk,
signing-authority issue.

---

## Step 4 — Draft re-engagement messages (Re-engage only)

Strict rules:
- Reference specific from Drive/Fireflies (name/number/decision/deadline)
- Banned phrases: "just checking in", "circling back", "following up",
  "touching base", "wanted to reach out", "hope you're well",
  "I hope this finds you"
- 60–150 words
- End with one concrete ask + date 5–7 business days out
- No two messages share sentence structure

Draft to Gmail drafts. Do NOT send.

---

## Step 5 — Write to CRM Review Queue (one row per candidate)
- `Item`: "<Contact> — <Company>"
- `Type`: "Stale deal" / "Orphaned account"
- `Priority`: High (>$100k or stage 4) / Medium / Low
- `Source Routine = "Anti-Slip Through the Cracks"`
- `Suggested Action`: Re-engage / Mark dead / Escalate to James Hong
- `Notes`: full diagnosis + draft message + Drive folder URL +
  Fireflies URL
- `Source Link`: Notion Opportunity Note URL or Drive folder URL
- `Status = "Not started"`

The Review Queue is the durable home — no page-per-run.

---

## Step 6 — Create 📋 Action Pipeline tasks (DEFCON-prioritized)

DEFCON rules for Anti-Slip-sourced tasks:

| Deal value | Days dark | DEFCON |
|------------|-----------|--------|
| ≥$500K stalled | any | **1 - Critical (Today)** |
| $100K–$500K stalled | any | **2 - High (This Week)** |
| $50K–$100K stalled | any | **3 - Medium (This Month)** |
| <$50K stalled | any | **4 - Planned** |
| At-Risk customer (regardless of value) | any | **2** minimum |
| Stage 4 deal at risk | any | **1** |

For each Re-engage draft:
- `Task`: "Review re-engagement draft to <contact> (<company>)"
- `Owed by`: Chris
- `Related Deal`: link
- `Due Date`: tomorrow
- `Status = "Not Started"`
- `Priority` (High/Med/Low to align with DEFCON)
- `DEFCON` per table
- `Category = "Deal Work"`
- `For Whom = "Chris reviews"`
- `Source = "Email Draft"`
- `Source Link`: Gmail draft URL
- `Source Routine = "Anti-Slip Through the Cracks"`

For each Mark-dead candidate:
- `Task`: "Approve mark-dead: <Deal name>"
- `Owed by`: Chris
- `Due Date`: end of week
- `DEFCON`: based on table (typically 2 or 3 since needs decision)
- `Category = "Deal Work"`
- `For Whom = "Chris does it"`
- `Source = "CRM Review Queue"`
- `Source Link`: CRM Review Queue row URL
- `Source Routine = "Anti-Slip Through the Cracks"`

For each Escalate candidate:
- `Task`: "Escalate <Deal> to James Hong"
- `Owed by`: Chris
- `Due Date`: tomorrow
- `DEFCON = "1 - Critical (Today)"` (escalations are time-sensitive)
- `Category = "Rep Management"`
- `For Whom = "Chris does it"`
- `Source Link`: CRM Review Queue row URL
- `Source Routine = "Anti-Slip Through the Cracks"`

---

## Step 7 — Write 📜 Activity rows (NEW)

For each candidate evaluated, write one Activity row:
- `Activity` = "Anti-Slip diagnosis: <Deal> (<verdict>)"
- `Type = "Note"` (or `"Draft Created"` if a draft was made)
- `Timestamp` = now
- `Deal` = relation
- `Contact` = relation if matched
- `Actor` = Chris (or Anti-Slip routine attribution)
- `Source = "Anti-Slip"`
- `Source Link` = CRM Review Queue row URL
- `Summary` = ≤200-word diagnosis recap
- `Source Routine = "Anti-Slip Through the Cracks"`

If Gmail shows recent contact-side reply but the routine still
flags the deal: write Activity (`Type = "Email Received"`,
`Source = "Gmail"`, `Source Link` = Gmail thread URL) so the
timeline reflects reality.

---

## Step 8 — Update Last Activity backfill on Deals DB

For every candidate evaluated: if Gmail or Fireflies produced a more
recent timestamp than stored `Last Activity`, update the Deal row
(`Last Activity`, `Last Activity Source` accordingly).

This catches deals that DID have recent activity that wasn't yet
reflected. Fix the record while you're there.

---

## Step 9 — Close the run

- `Status`: Success / Partial / No-op / Failed
- `Finished At`: now
- `Items Processed`: candidates evaluated
- `Items Created`: CRM Review Queue rows + Action Pipeline tasks +
  Activities + drafts
- `Items Skipped`: dropped by 14-day guardrail / recent Parse Call /
  closed-lost / no recent activity
- `Connectors Down`
- `Summary` ≤300 words: top 3 most-at-risk by deal value (named).
  Anything LOW-confidence flagged here.
- `Errors / Blockers`

A run that finds nothing still writes a No-op row.

---

## Step 10 — Post-approval (when Chris resolves a Review Queue row)

Real-time updates happen between runs. On the next run, scan
resolved Review Queue rows since last finished-at:

**Re-engage approval (Resolved = true):**
1. Gmail draft was created in Step 4 — Chris sends himself.
2. Update Deals DB: `Last Activity = today`,
   `Last Activity Source = "Manual"`, `Next Step Date = +14d`.
3. Mark Action Pipeline task `Status = "Done"`.
4. Write Activity (`Type = "Email Sent"`, `Source = "Gmail"`).

**Mark-dead approval:**
1. Deals DB: `Status Flag = "Dead"`, `Stage = "Lost"`.
2. Action Pipeline task: `Status = "Done"`.
3. Write Activity (`Type = "Stage Change"`,
   `Summary = "Marked dead by Chris: <reason>"`).
4. Do NOT touch HubSpot unless Chris explicitly requests.
5. **Important:** Once marked dead, the Deal stays in Notion for
   historical reference but is filtered out of the active dashboard
   via Status Flag.

**Escalate approval:**
1. Reassign Deal owner to James Hong.
2. New Action Pipeline row, `Owed by = James Hong`,
   `For Whom = "Rep owes Chris"`, `Category = "Rep Management"`,
   `Due Date = +7d`, `Task = "Take over <Deal> from Chris"`,
   `DEFCON = "2 - High (This Week)"`.
3. Action Pipeline Chris task: `Status = "Done"`.
4. Write Activity (`Type = "Stage Change"`,
   `Summary = "Reassigned to James Hong"`).

---

## Task Lifecycle & Auto-Completion

Action Pipeline tasks created by this routine follow the same
lifecycle as Parse Call's tasks:

1. **Created** with `Status = "Not Started"` and DEFCON.
2. **In Progress** — Chris manually.
3. **Done** — Chris manually OR auto-completed (rules below).
4. **Cancelled** — auto-cancelled if `Status = "Not Started"` 60+
   days past Due Date.

### Auto-completion (run every cycle, before Step 9)

For every Action Pipeline row with
`Source Routine = "Anti-Slip Through the Cracks"` AND
`Source = "Email Draft"` AND `Status NOT IN ["Done", "Cancelled"]`:

1. Parse the Gmail draft ID from `Source Link`.
2. Search Gmail Sent for a message to the same recipient with
   matching subject/body in the last 14 days.
3. **Sent message found** → set `Status = "Done"`, append "Auto-
   completed: email sent <date>" to Notes, write Activity
   (`Type = "Email Sent"`, `Source = "Gmail"`).
4. **Draft gone but no sent message** → leave alone.

For every Action Pipeline row linked to a CRM Review Queue item:
- If `Resolved = true` on the Review Queue row → set Action Pipeline
  `Status = "Done"`.
- If `Resolved = true` AND `Suggested Action = "Mark dead"` and Chris
  approved → also update Deals DB (`Status Flag = "Dead"`,
  `Stage = "Lost"`) and write Activity.

For Mark-dead and Escalate tasks: when Chris sets Action Pipeline
`Status = "Done"`, that's the trigger to update the Deal record on
the next run (see Step 10).

### Quick-complete UX (for Chris)
- Open Action Pipeline → click row → set Status = "Done".
- Filter out Done tasks via the dashboard's existing Status ≠ Done
  filter. Done items remain in the DB for history.

---

## Guardrails (hard stops)

- Never message anyone replied-to in last 14 days.
- Never auto-send (drafts only).
- Never mark dead without Chris's explicit per-item approval.
- Never recycle sentence structure across drafts.
- Never message anyone with `hs_email_optout = true`.
- Never add closed-lost deals to Notion Deals DB.
- Never create a Deal without recent activity (90 days);
  route to CRM Review Queue instead.
- If notes are thin, say so — never invent context.
- Don't reach out to anyone touched by Parse Call in last 48h.
- Don't message anyone Parse Call already drafted to in last 14d.
- Always tag with `Source Routine = "Anti-Slip Through the Cracks"`.

---

## Out of scope
- Auto-sending
- Marking dead without approval
- Reaching out to active deals (Parse Call's job)
- Reaching out to Do Not Contact
- Creating closed-lost Deal records

Begin now.
