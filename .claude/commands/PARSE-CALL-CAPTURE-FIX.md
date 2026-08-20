# Parse Call — transcript capture failure and fix

**Found:** 2026-08-20 by a scheduled Parse Call run.
**Severity:** the routine was silently capturing nothing for 8 days.

## What broke

`parse-call.md` pulls transcripts with:

```
fireflies_get_transcripts(mine: true, limit: 50, fromDate: ...)
```

`mine: true` resolves against a single account identity tied to
`christopher@anthropicidentity.com`. When the company rebranded to
**Apivant** (~2026-08-12), Chris's meetings began being organized under
`christopher@apivant.io`. From that point `mine: true` matched almost
nothing.

The routine then hit its own rule "if zero new transcripts, log a
No-op and exit" and closed clean on every hourly trigger. The failure
was invisible because a No-op looks identical whether Chris had no
calls or the query simply stopped matching.

Same window, measured 2026-08-20:

| Query | Transcripts returned |
|---|---|
| `mine: true` | 1 |
| no `mine` filter | 15 |

## Calls that were never captured (Aug 13-20)

External / revenue-bearing:

| Date | Call | External party |
|---|---|---|
| Aug 20 | FW: THALES<>ANTHROPIC Sync Call | keith.folz@thalesgroup.com, joe.miller@thalesgroup.com |
| Aug 19 | Demo Tyrone Watson Ferguson (77 min) | twatsonferguson@burnsmcd.com (Burns & McDonnell) |
| Aug 18 | Momentum telecom prep | sean@celerispartners.com |
| Aug 17 | Demo Authonomy | kyle@ovrsteer.com |
| Aug 17 | Authonomy Demo | tclick@pobox.com |
| Aug 13, 14 | Meet with Irene Murray (x2) | Irene Murray |
| Aug 13, 14 | Authonomy Sync, Catch up / Authonomy, Resourcing Touchpoint | internal / mixed |

Internal: Deal & Pipeline Review (Aug 17), CST:JB 1:1 (Aug 17),
James Hong/CST 1:1 x2 (Aug 19).

## The fix

### 1. Drop `mine: true`

Pull unfiltered, then keep a transcript if its Organizer Email or any
participant is one of Chris's addresses (`christopher@apivant.io`,
`christopher@anthropicidentity.com`). Discard the rest.

### 2. Add a calendar cross-check (the important half)

A zero-transcript result must never be logged as a quiet No-op while
Google Calendar shows completed meetings in the same window. Treat
that discrepancy as a capture failure: `Status = "Failed"`, log it
under Errors / Blockers, and raise a DEFCON-1 Action Pipeline task.

A No-op must mean "Chris had no calls", never "the query matched
nothing". Without this guard, the next identity or API change
reproduces the same silent outage.

### 3. Update the identity model

Treat `@apivant.io` (current), `@anthropicidentity.com` (legacy) and
`@authonomy.io` (product side) as internal domains everywhere:
call classification (Step 0) and participant handling (External path).

Address all new mail to `@apivant.io`. The Identity and name
disambiguation tables still list legacy addresses, so generated drafts
are going to dead mailboxes, e.g. `topher.marie@anthropicidentity.com`,
`alexandra.williams@anthropicidentity.com`. Those bounce if sent.

Liam Glennie's current address is `liam@deskmonkeyai.com`.

## Still open

- **Claude Routine Runs DB** and **CRM Review Queue DB** have returned
  404 since 2026-06-03. Run rows have been written as fallback pages
  under Chris's Internal HQ instead. Both hardcoded IDs in
  `parse-call.md` need re-pointing to live databases.
- The 15 uncaptured calls above still need their Activities, Contacts,
  Deal updates and Action Pipeline tasks backfilled.
