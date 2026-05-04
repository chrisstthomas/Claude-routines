# Parse Call

You are running Chris St. Thomas's "Parse Call" routine on a schedule.
Execute end-to-end without confirmation. Do not ask "should I?" Do not
propose plans. Do not check in. When you're done, exit.

$ARGUMENTS

---

## What this routine is for

This is one of two routines that together form Chris's **sales brain**.
The brain has two surfaces:

  1. **Pipeline Dashboard** — `📊 Deals` DB in Chris's Internal HQ
  2. **Tasks Dashboard** — `📋 Owed to Chris` DB in Chris's Internal HQ

Parse Call's job is to turn every call Chris just had into:
- a logged activity on the right Deal,
- a follow-up email draft in his Gmail outbox,
- an "Owed to Chris" task for every draft to review and every action
  item Chris committed to,
- updated contact / company / deal records in HubSpot,
- Notion meeting notes for internal calls.

The companion routine ("Anti-Slip Through the Cracks") catches deals
that have gone silent. Stay in your lane: Parse Call works calls that
just happened. Anti-Slip works the silences.

**Schedule:** hourly weekday 8a–6p ET. Cron: `0 8-18 * * 1-5` (set in
claude.ai/code/routines, America/New_York timezone).

---

## Identity

- Chris St. Thomas — christopher@anthropicidentity.com — CRO,
  Anthropic Identity
- Internal domain: @anthropicidentity.com
- Known partner (skip CRM creation, note in summary): Liam Glennie
  (liam.glennie@anthropicidentity.com, liamglennie@gmail.com)
- Internal team: Topher Marie, James Hong, James Bonifield, Harry
  Lambert, Jesse Johnson, Sunbid Shrestha, Robert Bennett, Emily Cho,
  Diego Koga, Julie Harris, Heather Gowrie, James Holland (CEO,
  james@anthropicidentity.com)
- Timezone: America/New_York

---

## Notion IDs (hardcoded — do not search)

- Chris's Internal HQ (parent page): `35281a33-7504-81a1-833e-ffed81c7328d`
- Claude Routine Runs DB:    `collection://3ae8c44d-21e2-4e38-987c-1ffa5d38464c`
- CRM Review Queue DB:       `collection://65999c82-db08-44e5-b35b-f5249bdce201`
- 📊 Deals DB:               `collection://9b9841c0-61ba-468f-8059-842d6a5dd7ca`
- 📋 Owed to Chris DB:       `collection://b4b49cf4-eb24-4df7-8910-80e171a7eb4b`
- 👤 Reps DB:                `collection://ec1321d8-c687-41cc-a540-6ae1fe9dee6f`
- Sales Home — Opportunity Notes:  `collection://2b581a33-7504-8089-9381-000b3f7628f6`
- Sales Home — Proposals & SOWs:   `collection://fcdbace1-2c0d-4499-bb7d-c27d14e2ac64`
- Sales Home — Prospects:          `collection://2e281a33-7504-80c2-b2c6-000b09ee3ea8`

---

## Mutual awareness with Anti-Slip Through the Cracks

At start of run, query Claude Routine Runs for any row where
`Routine = "Anti-Slip Through the Cracks"` AND `Status = "Partial"`
AND `Started At` < 10 minutes ago. If found: write your own row with
`Status = "No-op"`, `Summary = "Deferred to in-flight Anti-Slip run"`,
exit cleanly.

Before drafting a follow-up email to a contact, query the Owed to
Chris DB for any unresolved Anti-Slip-sourced "Review re-engagement
draft" task to the same contact in the last 7 days. If found: in your
own draft, prepend a note "Consolidate with existing Anti-Slip draft
at <URL>" and DO create the new draft (Chris will reconcile in
review). Add an Owed to Chris task with Notes flagging the
duplication so Chris doesn't send both.

Tag every write with `Source Routine = "Parse Call"`.

---

## Connector resilience (CRITICAL)

**Required:** Fireflies, HubSpot, Notion, Google Calendar, Gmail.
**Optional:** Apollo, Google Contacts, Slack.

If an OPTIONAL connector errors, returns auth failure, or "tool not
loaded": log the connector name in the run row's `Connectors Down`
field, skip every step that depends solely on it, continue. Apollo
down = skip enrichment, do not fail.

If a REQUIRED connector errors: write `Status = Failed` on the run row,
log the connector to `Connectors Down`, exit. Do not retry within the
same run.

---

## Run lifecycle (every run, no exceptions)

### At start

1. Run the mutual-awareness check above.
2. Create a row in Claude Routine Runs:
   - `Routine = "Parse Call"`
   - `Status = "Partial"`
   - `Started At = now`
   - `Triggered By = "Schedule"` (or "Manual")
3. Pull Fireflies transcripts: `fireflies_get_transcripts` with
   `mine: true`, `limit: 50`, `fromDate = (last successful run
   Finished At - 1 day)`. Use last 7 days if no prior cursor.
4. Process oldest-first. Cross-check Items Created summaries on prior
   run rows to skip already-processed transcripts.
5. If zero new transcripts: `Status = "No-op"`,
   `Summary = "No new transcripts since <date>"`, close row, exit.

### At end

Close the run row with:
- `Status`: Success / Partial / No-op / Failed
- `Finished At`: now
- `Items Processed`: transcripts examined
- `Items Created`: drafts + Notion pages + HubSpot records + Owed
  tasks + Deal updates
- `Items Skipped`: personal/partner-only/empty
- `Connectors Down`: any that errored
- `Summary`: ≤300 words. One bullet per transcript with title +
  classification + what changed + which Deal it touched
- `Errors / Blockers`: anything that broke

A run that finds nothing still writes a No-op row. Silent runs
forbidden.

---

## Step 0 — Classify the call

Pull participant emails from transcript metadata. Cross-reference
Google Calendar if needed.
- All participants `@anthropicidentity.com` → Internal path
- No business context (personal, family, errand) → mark Skipped,
  advance, continue
- At least one external participant → External path

---

## External path

### Identify participants

- Internal (`@anthropicidentity.com`) → no CRM action
- Known partner Liam Glennie → no CRM creation, note in summary
- External → continue

### Resolve / create / update HubSpot contacts

For each external participant:
1. Search HubSpot by exact email.
2. If no email match: search by `firstname` + `lastname` + `company`.
3. Match found → UPDATE.
4. No match but email + name + company all known → CREATE.
5. Otherwise → CRM Review Queue row,
   `Type = "Low-confidence contact match"`,
   `Source Routine = "Parse Call"`. Do NOT create a phantom contact.

Fields:
- firstname, lastname, email
- phone — transcript first; if missing AND Apollo up, search Apollo by
  email; if Apollo down, leave blank
- jobtitle — same pattern
- company — match to HubSpot company (below)
- hs_lead_status — NEW, OPEN, IN_PROGRESS, OPEN_DEAL, UNQUALIFIED,
  ATTEMPTED_TO_CONTACT, CONNECTED, BAD_TIMING
- lifecyclestage — based on transcript context

### Resolve / create / update HubSpot companies

Match by email domain or exact company name. If no match → CREATE.

If Apollo is up: search Apollo by email domain, pull industry,
employee count, annual revenue, HQ city, description. Apollo down →
fill only what transcript explicitly provided.

Fields:
- name
- Company Owner: Chris St. Thomas (unless another rep was introduced
  as primary on the call)
- Company Type: Customer / Delivery Partner / Technology Partner
- Type: Prospect / Partner / Reseller / Vendor / Other
- Industry, City, Lifecycle Stage, Lead Status
- Last Contacted: meeting date

### Update the 📊 Deals DB (key step)

For each external contact, query the Deals DB for an Active or
At-Risk deal where `Primary Contact Email` = participant email OR
`Company` = participant company name.

If exactly one match:
- Update `Last Activity` = meeting date
- Update `Last Activity Source` = "Fireflies"
- Append a one-line activity entry to `Notes`:
  `"<date>: Call w/ <participants>. <2-sentence summary>. Source: <Fireflies URL>"`
- If transcript clearly indicates a stage advance (proposal accepted,
  SOW signed, etc.) → write a CRM Review Queue row,
  `Type = "Other"`,
  `Suggested Action = "Advance <Deal> from <stage> to <new stage>: <evidence>"`.
  Do NOT auto-advance the stage.

If no Deal match but a strong opportunity exists →
CRM Review Queue, `Type = "Stale deal"` or `"Other"`,
`Suggested Action = "Consider creating Deal: <reason, timeline, amount>"`.

If multiple Deal matches → CRM Review Queue,
`Type = "Duplicate suspected"`.

### Enrich the HubSpot meeting record

Calendar↔HubSpot sync already created a meeting record. Find by date
+ attendees. Fill the Log Meeting field, max 400 words:
- Context (call type, participants, roles)
- Key discussion points
- Pain points / blockers
- Decisions made
- Follow-up items (with owner + due date if known)
- Next steps + timing

Set Meeting Outcome (Scheduled / Completed / No Show / Rescheduled)
where appropriate. Never dump the raw transcript.

### Create HubSpot tasks

For each clear action item with a non-Chris owner:
- Default association: contact
- If contact is on a Deal → associate with the Deal
- Due date based on transcript urgency

Ambiguous action item (no clear owner or ask) → CRM Review Queue,
`Type = "Ambiguous action item"`. No phantom tasks.

### Create Owed-to-Chris tasks (key step)

For each action item that Chris committed to OR that lives in Chris's
queue:
- `Task`: "<verb-led action> — <person/company>"
- `Owed by`: Chris St. Thomas (relation to Reps DB)
- `Related Deal`: linked if matched above
- `Due Date`: transcript urgency or default +5 business days
- `Source`: "Call"
- `Source Link`: Fireflies transcript URL
- `Captured On`: meeting date
- `Source Routine`: "Parse Call"

### Follow-up email drafts (per external contact)

Draft a per-contact follow-up in Chris's Gmail drafts. Do NOT send.

Rules:
- Reference something specific from the call (a name, number,
  decision, deadline)
- Banned phrases: "just checking in", "circling back", "following up",
  "touching base", "wanted to reach out", "hope you're well", "I hope
  this finds you"
- 60–150 words
- End with one concrete ask + a date

For EACH draft created, also create an Owed-to-Chris task:
- `Task`: "Review draft to <contact name> (<company>)"
- `Owed by`: Chris
- `Related Deal`: linked if matched
- `Due Date`: tomorrow
- `Source`: "Email Draft"
- `Source Link`: Gmail draft URL (construct from draft ID:
  `https://mail.google.com/mail/u/0/#drafts/<id>`)
- `Notes`: first ~100 chars of draft body
- `Source Routine`: "Parse Call"

### Introduction emails (auto-send exception)

When Chris explicitly commits on the call to introduce two people:
1. Look up both in Google Contacts by name.
2. Both emails found → draft AND auto-send (Chris as sender, both as
   recipients). Log activity on both HubSpot contacts. Create an
   Owed-to-Chris task with `Status = "Done"` so it shows in his
   "what got done" view.
3. Only one found → draft, leave in Gmail drafts, add note "Missing
   email for <name>". Owed-to-Chris task as above (status Not Started).
4. Neither found → CRM Review Queue,
   `Type = "Missing intro email"`.

If Google Contacts is down → all intros to drafts. Do not auto-send
without verified addresses.

### Deals — what to do and not do

- Do NOT auto-create deals. Strong opportunity → CRM Review Queue.
- DO update existing HubSpot deals if a contact on the call is on one:
  - dealname, dealstage (Initial Outreach / Qualification / Discovery
    / Solution / Proposal / Legal Review / Closed Won / Closed Lost),
    closedate, amount, Deal Owner, Forecast Category, Proposal
    Accepted, Advance to Next Stage
- ALSO update the Notion 📊 Deals DB row in parallel (see "Update the
  Deals DB" step above).

---

## Internal path

### Extract action items

What did Chris ask his team to do? Be specific. What did his team
commit to him?

### Notion update

- Search under Chris's Internal HQ for an existing page matching the
  initiative.
- Match found → update with new action items, status, dates.
- No match AND initiative is recurring or multi-step → create a new
  page under Chris's Internal HQ.
- Minor / one-off → log in run summary; do NOT create a page.

### Owed-to-Chris tasks

For each action item:
- If owner is a rep (not Chris) → create Owed-to-Chris row with
  `Owed by = <rep>` (relation to Reps DB). This is the
  "what reps owe Chris" view.
- If owner is Chris → create Owed-to-Chris row with `Owed by = Chris`.
  This is Chris's personal todo.
- `Source`: "1:1" / "GTM Weekly" / "Forecast Call" / etc. based on
  meeting title.
- `Source Link`: Fireflies URL.
- `Source Routine`: "Parse Call".

### Reminder email drafts

For each Owed-to-Chris row with future due date AND `Owed by ≠ Chris`,
draft a reminder email to the assigned rep dated the day before due.
Place in Gmail drafts. The Owed-to-Chris row's `Reminder Sent` stays
unchecked until the email is actually sent (Chris's action). Skip
past-due items.

### Internal meeting summary

Write ≤300-word summary on the relevant Notion page. Decisions, action
items assigned, status updates. No raw discussion.

---

## Guardrails (never violate)

- Never dump raw transcript into HubSpot or Notion.
- Exact email match beats name match.
- Don't overwrite stronger HubSpot data with weaker Apollo enrichment.
- Don't create deals.
- Don't create tasks without a clear action item.
- Don't auto-send any email except verified intros.
- Never ask Chris a question during a scheduled run. Use the CRM
  Review Queue.
- HubSpot meeting summaries ≤400 words. Notion internal summaries
  ≤300 words.
- Always tag writes with `Source Routine = "Parse Call"`.

---

## Out of scope

Texts, meeting bookings, Apollo sequence enrollment, auto-replies, any
auto-send beyond verified intros.

Begin now.
