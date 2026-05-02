# Claude-routines

A collection of Claude Code custom commands (slash commands) that automate recurring sales, customer success, and operations workflows for Chris. Each routine orchestrates multiple MCP tool integrations — HubSpot, Fireflies, Gmail, Google Calendar, Notion, Slack — and is designed to require human approval before taking any irreversible action.

## Structure

```
.claude/
  commands/           # Claude Code custom slash commands
    customer-reengagement.md
```

Each file in `.claude/commands/` becomes an invocable slash command in Claude Code (e.g., `/customer-reengagement`).

## Available commands

### `/customer-reengagement`

**Purpose:** Find current customers and stalled deals that have gone quiet, diagnose what broke the rhythm, draft re-engagement messages, and build a Notion review queue for Chris to approve.

**Trigger:** Weekly Monday afternoon, or manually with prompts like "who's gone cold", "what's stalled", "find dropped balls".

**Integrations:** HubSpot, Fireflies, Gmail, Notion

**What it does NOT do:** Auto-send messages, mark deals Closed Lost, or touch contacts with activity in the last 14 days.

## Design principles

- **Approval-gated**: No routine sends a message, closes a deal, or modifies shared CRM data without Chris's explicit per-item approval.
- **Specific over generic**: Every drafted message must reference something concrete from meeting notes. Generic check-ins are explicitly prohibited.
- **Transparent about gaps**: If source data is thin or missing, routines say so rather than guessing.
- **Non-destructive defaults**: Routines create drafts, stage changes for review, and flag uncertainty rather than acting unilaterally.

## MCP tools used

| Service    | Tools                                                                                  |
|------------|----------------------------------------------------------------------------------------|
| HubSpot    | `search_crm_objects`, `get_crm_objects`, `manage_crm_objects`, `search_owners`, `tool_guidance` |
| Fireflies  | `fireflies_search`, `fireflies_get_summary`, `fireflies_get_transcript`                |
| Gmail      | `search_threads`, `create_draft`                                                       |
| Notion     | `notion-search`, `notion-fetch`, `notion-create-pages`, `notion-update-page`           |
