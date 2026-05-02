# Customer Re-engagement & Stalled Deal Recovery

Trigger: Weekly Monday afternoon, or manually when Chris asks ("who's gone cold", "what's stalled", "find dropped balls").

$ARGUMENTS

---

## Overview

Find current customers and stalled deals that have gone quiet, pull relevant meeting notes, identify what was promised and never followed up, draft re-engagement messages for Chris to review. Do not send anything without Chris's explicit approval on each individual message.

---

## Step 1 — Build the candidate list

Call `mcp__HubSpot__tool_guidance` first to confirm available object types and property names, then proceed.

### Bucket A: Current customers (Lifecycle Stage = Customer)

Call `mcp__HubSpot__search_crm_objects` with `objectType: "contacts"` filtering on:
- `lifecyclestage = "customer"`
- `hs_last_sales_activity_timestamp` older than 30 days from today, OR
- `notes_last_updated` older than 45 days, OR
- Any open tasks with `hs_task_status = "NOT_STARTED"` and `hs_timestamp` more than 14 days ago

For each matching contact also pull their associated company via `mcp__HubSpot__get_crm_objects` to get company name, ARR/deal value, and account owner.

### Bucket B: Stalled or dropped deals (not Closed Won / not Closed Lost)

Call `mcp__HubSpot__search_crm_objects` with `objectType: "deals"` filtering on:
- `dealstage` not in `["closedwon", "closedlost"]`
- AND at least one of:
  - `hs_date_entered_[current_stage]` older than 45 days (stage hasn't moved)
  - `closedate` is in the past with no recent update
  - `hs_next_step` date passed by 7+ days
  - `notes_last_updated` older than 45 days

For each deal, fetch the associated contact(s) and company via `mcp__HubSpot__get_crm_objects`.

### Ranking and cap

- Combine both buckets
- Sort: deal value (`amount`) descending first, then `notes_last_updated` ascending (longest since touch first)
- Cap at 15 total candidates
- **Hard filter**: exclude any contact or deal where the associated contact has `hs_email_optout = true` or is marked Do Not Contact

Before moving to Step 2, list the 15 candidates in a numbered table: Name | Company | Type | Deal Value | Days Since Last Touch. Ask Chris if the list looks right or if anyone should be removed before continuing. Wait for confirmation.

---

## Step 2 — Fetch meeting context

For each candidate (work through them in order):

1. **Fireflies search**: Call `mcp__Fireflies__fireflies_search` with the contact's name and company name as the query. Retrieve the last 1–3 meetings. For each matching meeting:
   - Call `mcp__Fireflies__fireflies_get_summary` to get the AI summary
   - Call `mcp__Fireflies__fireflies_get_transcript` only if the summary is insufficient to identify commitments and open questions (saves tokens; use judgment)

2. **HubSpot meeting notes**: Call `mcp__HubSpot__get_crm_objects` to pull `engagements` (type = MEETING and type = NOTE) associated with the contact, reading body text for commitments, next steps, and open questions.

3. **Gmail check (recent 14 days)**: Call `mcp__Gmail__search_threads` with `query: "from:[contact email] OR to:[contact email]"` and `maxResults: 5`. If any thread shows a reply in the last 14 days, **remove this candidate from the queue entirely** (do not re-engage someone who just replied).

Compile per-candidate context notes internally before moving to Step 3.

---

## Step 3 — Diagnose each candidate

For each candidate write a one-paragraph diagnosis covering:

- The last real moment of progress (what happened, when, what was agreed)
- What broke the rhythm: missed follow-up on Chris's side, customer went silent, blocker raised and never addressed, internal champion left, external factor, or genuinely unknown
- Confidence level: HIGH (clear meeting notes), MEDIUM (partial notes), LOW (thin or missing notes — say so explicitly; do not guess)
- Verdict: **Re-engage**, **Mark dead**, or **Escalate to Liam**

Criteria for "Mark dead": last activity 90+ days ago, no deal value recorded, or notes indicate budget/authority removed with no new champion.
Criteria for "Escalate to Liam": enterprise account, contract renewal at risk, or relationship requires someone with signing authority to weigh in.

---

## Step 4 — Draft re-engagement messages

Draft one message per candidate with verdict = Re-engage. Do not draft for "Mark dead" or "Escalate" entries.

**Rules — apply strictly:**
- Reference something specific: a person's name, a number, a decision made, a feature discussed, a deadline mentioned
- No phrase "just checking in", "circling back", "following up", "touching base", "wanted to reach out"
- No generic openers like "Hope you're well" or "I hope this finds you"
- Length: 60–150 words
- End with one concrete ask or proposed next step with a specific date (use a date 5–7 business days from today)
- Never recycle the same sentence structure across two drafts in the same run

**Format by type:**

*Customer re-engagement*: Acknowledge the gap without apologizing excessively. Reference the last project or use case by name. Ask one specific question about how it's going or what changed since you last spoke.

*Stalled deal recovery*: Open by naming the last commitment (Chris's or theirs). State the open question plainly. Propose a 30-minute call with a specific date and time option.

---

## Step 5 — Write the review queue to Notion

Call `mcp__Notion__notion-search` with query "Re-engagement Queue" to check if a page already exists. If it does, fetch it with `mcp__Notion__notion-fetch` to get the page ID.

If the page does not exist, call `mcp__Notion__notion-create-pages` to create it. Use a title like "Re-engagement Queue — [Today's Date]" and place it in the top-level workspace or the same parent as existing sales/CRM pages (search for a "Sales" or "CRM" parent page first).

Write the following structure to the page using `mcp__Notion__notion-update-page` or by creating with rich content:

```
# Re-engagement Queue — [DATE]
Run completed: [timestamp]
Candidates reviewed: [N] ([X] customers, [Y] stalled deals)

---

## Priority 1: Re-engage

### [Contact Name] — [Company]
**Type:** Customer / Stalled Deal
**Deal Value:** $[amount]
**Days since last touch:** [N]
**Diagnosis:** [one paragraph]
**Suggested action:** Re-engage
**Draft message:** [collapsible toggle block containing the draft]

[repeat for each Re-engage candidate, sorted by deal value desc]

---

## Priority 2: Mark Dead (Awaiting Chris's Approval)

### [Contact Name] — [Company]
**Type:** ...
**Days since last touch:** [N]
**Diagnosis:** [one paragraph]
**Suggested action:** Mark dead — [one-line reason]

[repeat]

---

## Priority 3: Escalate to Liam

### [Contact Name] — [Company]
**Type:** ...
**Days since last touch:** [N]
**Diagnosis:** [one paragraph]
**Suggested action:** Escalate to Liam — [reason]

[repeat]

---

## Flagged for Chris

[List any candidates where diagnosis confidence = LOW, meeting notes were missing, or something could not be determined. Describe what's missing.]
```

After writing the page, return the Notion URL to Chris.

---

## Step 6 — Post-approval actions

Present the Notion link and the candidate summary to Chris. Then wait.

When Chris approves specific entries, act on each approved item:

### For "Re-engage" approvals:

1. **Send email**: Call `mcp__Gmail__create_draft` with the approved message text, the contact's email address as `to`, and a subject line that references the company or project (not "Following up"). Then confirm with Chris before calling send, or present the draft ID so Chris can review in Gmail.

2. **Update HubSpot Last Outreach Date**: Call `mcp__HubSpot__manage_crm_objects` to update the contact record:
   - Set `hs_sales_email_last_replied` or `notes_last_updated` to today
   - Create a new NOTE engagement with body: "Re-engagement outreach sent via Claude routine — [date]. Draft: [first 100 chars of message]"

3. **Set next follow-up**: Call `mcp__HubSpot__manage_crm_objects` to create a new TASK on the contact:
   - Subject: "Follow up with [Name] — re-engagement"
   - Due date: 14 days from today
   - `hs_task_status: "NOT_STARTED"`

### For "Mark dead" approvals:

For contacts: Call `mcp__HubSpot__manage_crm_objects` to set `hs_lead_status = "UNQUALIFIED"` on the contact. Add a note with the one-line reason.

For deals: Call `mcp__HubSpot__manage_crm_objects` to update the deal stage to `"closedlost"` and set `closed_lost_reason` to the one-line reason from the diagnosis. Do not do this without Chris's explicit approval per deal.

### For "Escalate to Liam" approvals:

Call `mcp__HubSpot__manage_crm_objects` to reassign the contact/deal owner to Liam (search for Liam's user ID first via `mcp__HubSpot__search_owners`). Add a note explaining the handoff reason.

---

## Output summary (present this to Chris immediately after Step 5)

```
Re-engagement Queue — [DATE]

Candidates reviewed: [N]
  • Customers (Bucket A): [X]
  • Stalled deals (Bucket B): [Y]

Top 5 most at-risk:
  1. [Name] — [Company] ([N] days, $[value]) — [Re-engage / Mark dead / Escalate]
  2. ...
  3. ...
  4. ...
  5. ...

Notion queue: [URL]

Flagged for Chris:
  • [Any items with LOW confidence diagnosis or missing data]
  • [Any candidates removed because of recent email reply]
  • [Any HubSpot data gaps that prevented scoring]
```

---

## Guardrails (enforce throughout — these are hard stops)

- **Never message** a contact who replied to anything in the last 14 days
- **Never auto-send** — always create a draft or present for approval first
- **Never mark a deal Closed Lost** without Chris explicitly approving that specific deal
- **Never recycle** the same draft language or sentence structure across two messages in the same run
- **Never reach out** to anyone marked Do Not Contact (`hs_email_optout = true`)
- **If notes are thin or missing**, say so in the diagnosis; do not invent context
- **If a deal is genuinely dead**, recommend closing cleanly rather than padding the pipeline with zombie entries
- **Do not touch** contacts or deals with activity in the last 14 days (the Morning Outreach Queue handles active deals)
