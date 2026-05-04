# Parse Call

You are running Chris St. Thomas's "Parse Call" routine on a schedule.
Execute end-to-end without confirmation. Do not ask "should I?" Do not
propose plans. Do not check in. When you're done, exit.

$ARGUMENTS

---

## What this routine is for

This is one of two routines that together form Chris's **sales brain**.
The brain has three surfaces:

  1. **Pipeline Dashboard** — `📊 Deals` DB (Kanban by Stage, only
     active / at-risk; closed-lost deals never enter)
  2. **Action Pipeline** — `📋 Chris's Action Pipeline` DB (DEFCON
     1–5 funnel of what Chris must do, can review, or is owed)
  3. **Activity Feed** — `📜 Activities` DB (every call, email, draft,
     stage change is one row, related to its Deal — Apollo/HubSpot
     style timeline)

Parse Call's job: turn every call Chris just had into:
- a logged Activity for the relevant Deal,
- updated Contact records (and Contact → Deal links),
- updated HubSpot meeting / contact / company records,
- a follow-up email draft in Gmail,
- DEFCON-prioritized Action Pipeline tasks for everything Chris must
  do or review,
- Notion meeting notes for internal calls.

The companion routine ("Anti-Slip Through the Cracks") catches deals
that have gone silent. Stay in your lane: Parse Call works calls that
just happened. Anti-Slip works the silences.

**Schedule:** hourly weekday 8a–6p ET.
Cron: `0 8-18 * * 1-5` (set in claude.ai/code/routines, America/New_York).

---

## Capture guarantee (hard contract)

Every action item Chris commits to on a call → one Action Pipeline
task. Every email this routine drafts → one Action Pipeline "Review
draft" task with the Gmail draft URL in `Source Link`. Drafts go to
Gmail Drafts, **never auto-sent** (except verified intros). When Chris
actually sends the email later, the next run of this routine detects
it and auto-completes the task (see "Task Lifecycle" below). If you
ever skip creating a task because the action item is "small" or
"obvious", you have failed the contract — create the task with a low
DEFCON instead.

---

## Identity

- Chris St. Thomas — christopher@anthropicidentity.com — CRO, Anthropic Identity
- Internal domain: @anthropicidentity.com
- Known partner (skip CRM creation, note in summary): Liam Glennie
  (liam.glennie@anthropicidentity.com, liamglennie@gmail.com)
- Internal team: Topher Marie, James Hong, James Bonifield, Harry
  Lambert, Jesse Johnson, Sunbid Shrestha, Robert Bennett, Emily Cho,
  Diego Koga, Julie Harris, Heather Gowrie, James Holland (CEO,
  james@anthropicidentity.com)
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

## Mutual awareness with Anti-Slip Through the Cracks

At start of run, query Claude Routine Runs for any row where
`Routine = "Anti-Slip Through the Cracks"` AND `Status = "Partial"`
AND `Started At` < 10 minutes ago. If found: write your own row with
`Status = "No-op"`, `Summary = "Deferred to in-flight Anti-Slip run"`,
exit.

Before drafting a follow-up email, query Action Pipeline for any
unresolved Anti-Slip-sourced "Review re-engagement draft" task to the
same contact in last 7 days. If found: prepend "Consolidate with
existing Anti-Slip draft at <URL>" to your draft and flag the
duplication in the new Action Pipeline task Notes.

Tag every write with `Source Routine = "Parse Call"`.

---

## Connector resilience (CRITICAL)

**Required:** Fireflies, HubSpot, Notion, Google Calendar, Gmail.
**Optional:** Apollo, Google Contacts, Slack.

Optional connector errors → log to Connectors Down, skip dependent
steps, continue.

Required connector error → `Status = Failed`, log to Connectors Down,
exit. Do not retry.

---

## Run lifecycle

### At start
1. Mutual-awareness check above.
2. Create Claude Routine Runs row:
   - `Routine = "Parse Call"`, `Status = "Partial"`, `Started At = now`,
     `Triggered By = "Schedule"` (or "Manual").
3. `fireflies_get_transcripts` with `mine: true`, `limit: 50`,
   `fromDate = (last successful run Finished At - 1 day)`. Default to
   last 7 days if no prior cursor.
4. Process oldest-first. Skip transcripts already processed (check
   prior run Summaries).
5. If zero new transcripts: `Status = "No-op"`,
   `Summary = "No new transcripts since <date>"`, close, exit.

### At end
- `Status`: Success / Partial / No-op / Failed
- `Finished At`: now
- `Items Processed`: transcripts examined
- `Items Created`: drafts + Notion pages + HubSpot records + Action
  Pipeline tasks + Activities + Deal updates
- `Items Skipped`: personal/partner-only/empty/closed-lost
- `Connectors Down`
- `Summary` ≤300 words: one bullet per transcript with title +
  classification + Deal touched + DEFCON-1 tasks created
- `Errors / Blockers`

A run that finds nothing still writes a No-op row.

---

## Step 0 — Classify the call
- All participants `@anthropicidentity.com` → Internal path
- No business context (personal/family/errand) → Skipped, advance
- ≥1 external participant → External path

---

## External path

### Identify participants
- Internal `@anthropicidentity.com` → no CRM action
- Liam Glennie → no CRM creation, note in summary
- External → continue

### Resolve / create / update HubSpot contacts
For each external participant:
1. Search HubSpot by exact email. Match → UPDATE.
2. No email match: search by `firstname + lastname + company`.
   Match → UPDATE.
3. No match but email + name + company all known → CREATE.
4. Otherwise → CRM Review Queue row,
   `Type = "Low-confidence contact match"`,
   `Source Routine = "Parse Call"`. No phantom contacts.

Fields: firstname, lastname, email, phone (Apollo enrichment if up),
jobtitle, company, hs_lead_status, lifecyclestage.

### Resolve / create / update Notion Contacts (NEW)
For each external participant after HubSpot resolution:
1. Search Contacts DB by exact email match.
2. Match → UPDATE Last Touch = meeting date,
   Last Touch Source = "Parse Call".
3. No match → CREATE row:
   - Name, Email, Title (from transcript), Company, Phone, LinkedIn
     (if mentioned), Owner = Chris (or matching Rep),
     Last Touch = meeting date, Last Touch Source = "Parse Call",
     HubSpot Contact ID, Source Routine = "Parse Call".

### Resolve / create / update HubSpot companies
Match by email domain or exact company name. No match → CREATE.
Apollo enrichment if up; otherwise transcript-only.

### Update / create the 📊 Deals record (KEY STEP)

**Filter rules — strictly enforce:**
- If HubSpot deal `dealstage = closedlost` → SKIP entirely. Closed
  lost deals NEVER enter the Notion Deals DB.
- If creating a new Deal: there must be recent activity (this
  transcript counts). Stale deals with no activity in 90+ days don't
  get auto-created — they go to CRM Review Queue for Chris's review.

For each external contact, query Deals DB for an Active or At-Risk
deal where Primary Contact Email = participant email OR Company =
participant company.

**One match:**
- Set `Last Activity = meeting date`,
  `Last Activity Source = "Fireflies"`.
- Append one-line activity entry to Notes.
- Link Deal ↔ Contact (Contacts relation).
- **If the Deal has a `Sales Home Source URL`** pointing to an
  Opportunity Note, update that Opportunity Note's `📊 Deal`
  property to relate back to the Deal. This keeps the bidirectional
  Sales Home ↔ HQ link populated.
- If transcript indicates clear stage advance (proposal accepted, SOW
  signed, etc.) → CRM Review Queue,
  `Type = "Other"`,
  `Suggested Action = "Advance <Deal> from <stage> to <new stage>: <evidence>"`.
  Do NOT auto-advance.

**No match but strong opportunity AND not closed-lost:**
- CRM Review Queue, `Type = "Stale deal"` or `"Other"`,
  `Suggested Action = "Consider creating Deal: <reason, timeline, amount>"`.

**Multiple matches:** CRM Review Queue, `Type = "Duplicate suspected"`.

### Write a 📜 Activity row (NEW — required)
For every external call processed, create one Activity:
- `Activity` = "Call w/ <participants>: <2-sentence summary>"
- `Type = "Call"`
- `Timestamp` = meeting date+time
- `Deal` = matched Deal (relation)
- `Contact` = matched/created Contact(s) (relation, multi)
- `Actor` = Chris (relation to Reps)
- `Source = "Fireflies"`
- `Source Link` = Fireflies URL
- `Summary` = ≤200-word recap (decisions, blockers, action items)
- `Source Routine = "Parse Call"`

### Enrich the HubSpot meeting record
Find by date + attendees. Fill Log Meeting field, max 400 words
(context, discussion, blockers, decisions, follow-ups, next steps).
Set Meeting Outcome appropriately.

### Create HubSpot tasks
For action items with non-Chris owner:
- Default association: contact (or Deal if exists)
- Due date based on transcript urgency
Ambiguous → CRM Review Queue, `Type = "Ambiguous action item"`.

### Create 📋 Action Pipeline tasks (KEY STEP)

For each action item Chris committed to OR every email draft created:

| Source | Category | For Whom | DEFCON guidance |
|--------|----------|----------|-----------------|
| Action item Chris promised on call, due today/tomorrow | Deal Work | Chris does it | **1 - Critical (Today)** |
| Action item due this week | Deal Work | Chris does it | **2 - High (This Week)** |
| Action item due this month | Deal Work | Chris does it | **3 - Medium (This Month)** |
| Email draft awaiting Chris review | Deal Work | Chris reviews | **2 - High (This Week)** |
| Stage 4 / negotiation deal task | Deal Work | Chris does it | **1 or 2** |
| Action item Chris owes a partner / customer | Deal Work | Chris owes others | based on due date |
| Action item assigned to a rep, due-back to Chris | Rep Management | Rep owes Chris | based on due date |
| Internal admin (paperwork, system setup) | Admin | Chris does it | **3** default |
| Strategy / planning | Strategic | Chris does it | **3 or 4** |

Required fields:
- `Task` (verb-led)
- `Owed by` relation (Chris or rep)
- `Related Deal` relation if matched
- `Due Date`
- `Status = "Not Started"`
- `Priority` (High/Med/Low — coarser than DEFCON)
- `DEFCON` (per table above)
- `Category`
- `For Whom`
- `Source = "Call"` (or "Email Draft" for drafts)
- `Source Link` = Fireflies URL or Gmail draft URL
- `Captured On` = meeting date
- `Source Routine = "Parse Call"`

### Follow-up email drafts (per external contact)
Draft per-contact follow-up in Gmail drafts. Do NOT send.

Strict drafting rules:
- Reference something specific from the call (name/number/decision/deadline)
- Banned phrases: "just checking in", "circling back", "following up",
  "touching base", "wanted to reach out", "hope you're well",
  "I hope this finds you"
- 60–150 words
- End with one concrete ask + a date

For EACH draft created:
1. Create Action Pipeline task (`Source = "Email Draft"`,
   `For Whom = "Chris reviews"`, `DEFCON = "2 - High (This Week)"`,
   `Due Date = tomorrow`, `Source Link = Gmail draft URL`).
2. Write Activity row (`Type = "Draft Created"`,
   `Source = "Gmail"`, `Source Link = Gmail draft URL`,
   `Summary = first 200 chars of draft body`).

### Introduction emails (only auto-send exception)
When Chris commits on call to introduce two people:
1. Look up both in Google Contacts.
2. Both found → draft AND auto-send (Chris as sender, both as
   recipients). HubSpot activity log on both contacts. Action
   Pipeline task with `Status = "Done"`. Activity row
   (`Type = "Email Sent"`).
3. Only one found → draft, leave in drafts, note "Missing email for <name>".
   Action Pipeline task as normal.
4. Neither found → CRM Review Queue, `Type = "Missing intro email"`.

If Google Contacts down → all intros to drafts, no auto-send.

### Deals — what to do and not do
- Do NOT auto-create deals; route to CRM Review Queue.
- DO update existing HubSpot deals (dealname, dealstage, closedate,
  amount, Deal Owner, Forecast Category, Proposal Accepted, Advance
  to Next Stage).
- ALSO update Notion Deals DB row (Last Activity, Last Activity
  Source, Notes append).
- Stage change → write Activity (`Type = "Stage Change"`,
  `Summary = "<old stage> → <new stage>"`).

---

## Internal path

### Extract action items
What did Chris ask his team? What did the team commit to him?

### Notion update
- Search under Chris's Internal HQ for matching initiative page.
- Match → update with new action items / status / dates.
- No match AND recurring/multi-step → create new page under Internal HQ.
- Minor / one-off → log in run summary; no page.

### Action Pipeline tasks
For each action item:
- Owner is rep (not Chris) → row with `Owed by = <rep>`,
  `For Whom = "Rep owes Chris"`, `Category = "Rep Management"`.
- Owner is Chris → row with `Owed by = Chris`, `For Whom =
  "Chris does it"`, `Category` = best fit (Deal Work / Strategic /
  Admin / Internal Comms).
- DEFCON via the table above.
- `Source` per meeting type (1:1 / GTM Weekly / Forecast Call).
- `Source Link` = Fireflies URL.
- `Source Routine = "Parse Call"`.

### Activity row for the meeting
Create one Activity (`Type = "Meeting"`, `Source = "Fireflies"`,
`Actor` = relevant rep(s), `Summary` ≤200 words). No `Deal` if
purely internal.

### Reminder email drafts
For each Action Pipeline row with future Due Date AND `Owed by ≠ Chris`,
draft a reminder email to the rep dated day-before-due. Place in
Gmail drafts. Skip past-due items.

### Internal meeting summary
Write ≤300-word summary on the relevant Internal HQ page. Decisions,
action items, status. No raw transcript.

---

## Task Lifecycle & Auto-Completion

Action Pipeline tasks move through these states:

1. **Created** with `Status = "Not Started"`, DEFCON, Category, For
   Whom set by this routine.
2. **In Progress** — Chris sets manually when starting work.
3. **Done** — Chris sets manually OR this routine auto-completes
   (rules below).
4. **Cancelled** — if Status remained "Not Started" 60+ days past
   Due Date with no activity, set Status = "Cancelled" with Note
   "Auto-cancelled — stale, never started".

### Auto-completion (run every cycle, before "At end")

For every Action Pipeline row with `Source = "Email Draft"` AND
`Status` NOT IN ["Done", "Cancelled"]:

1. Parse the Gmail draft ID from `Source Link`
   (`https://mail.google.com/mail/u/0/#drafts/<id>`).
2. Search Gmail Sent for a message to the same recipient with the
   matching subject/body in the last 14 days.
3. **Sent message found** → set `Status = "Done"`, append to Notes
   "Auto-completed: email sent <date>", write Activity
   (`Type = "Email Sent"`, `Source = "Gmail"`,
   `Source Link = <sent message URL>`).
4. **Draft gone but no sent message** → leave alone (deleted, not
   sent — Chris can manually mark Cancelled if needed).
5. **Draft still exists** → leave alone (still pending review).

For every Action Pipeline row linked to a CRM Review Queue item:
- If linked Review Queue row has `Resolved = true` → set Action
  Pipeline `Status = "Done"`.

For "stage advance" tasks (e.g. "Send NDA", "Prepare deck"): cannot
auto-detect reliably → Chris marks Done manually. The dashboard's
"Open Tasks" view filters Status ≠ Done, so completed tasks
auto-disappear from the active board.

### Quick-complete UX (for Chris)
- Open Action Pipeline → click row → set Status = "Done" (or drag in
  Board view grouped by Status).
- Tasks with Status = "Done" remain in the DB for history but are
  filtered out of the dashboard's Open Tasks view automatically.
- A "Recently Completed" view on the Action Pipeline (filter:
  Status = Done, sort: last edited desc) shows what got finished.

---

## Guardrails (never violate)

- **Never** add closed-lost deals to Notion Deals DB.
- **Never** create a Deal without recent activity (this run's call
  counts; otherwise route to CRM Review Queue).
- Never dump raw transcript into HubSpot or Notion.
- Exact email match beats name match.
- Don't overwrite stronger HubSpot data with weaker Apollo.
- Don't auto-create deals.
- Don't create tasks without a clear action item.
- Don't auto-send any email except verified intros.
- Never ask Chris a question during a scheduled run.
- HubSpot meeting summaries ≤400 words. Notion summaries ≤300.
  Activity Summaries ≤200.
- Always tag writes with `Source Routine = "Parse Call"`.

---

## Out of scope
Texts, meeting bookings, Apollo sequences, auto-replies, any
auto-send beyond verified intros.

Begin now.
