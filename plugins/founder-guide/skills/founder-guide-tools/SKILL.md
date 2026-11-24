---
name: founder-guide-tools
description: How to effectively use the Founder Guide MCP tools to read and update a founder's workspace — business plan, demand cases and buyer evidence, outreach campaigns and replies, sales-call playbooks, content and social posts with the founder's voice profile, content themes, contacts, tasks and goals, documents, the customer journey, cash-flow forecasting and invoice drafts, and time tracking. Use when a Founder Guide MCP connector is configured and the assistant needs to discover workspaces, read plan sections, or create/update records via tools such as list_workspaces, get_section_state, read_entity, write_entities, upsert_task, create_content_item, ensure_demand_case, add_case_evidence, record_outreach_sent, get_cash_forecast, and log_time.
---

# Founder Guide Tools

This skill teaches an AI assistant to use the **Founder Guide MCP tools** correctly and
efficiently. It covers tool *mechanics* only — discovery, the read-before-write loop,
batch writes, and how to read the error/permission signals.

**What this is not:** it does **not** replicate Founder Guide's in-product AI specialists
(Problem Scout, Persona Builder, Market Analyst, etc.). Those are product features. This skill
just makes the raw tool surface legible so you call it reliably.

## Prerequisite: an active connection

This skill assumes the Founder Guide MCP server is already connected in your client
(`https://www.founder-guide.com/api/mcp`, OAuth sign-in). If the tools below are not
available, the connector is not set up — see the Help Center article "Connecting External AI
Agents (MCP)". This skill does not establish the connection.

## The golden path (do this every session)

Follow these six rules. Most failures come from skipping one of them.

1. **Discover first — call `list_workspaces`.** It needs no `knowledge_graph_id` and returns
   the workspaces the user can act on, each with a stable id and name. Pick the right one
   (ask the user if more than one).
2. **Pass `knowledge_graph_id` on EVERY other tool call.** The discovery tools `list_workspaces` and `list_vnext_workspaces` do not require it;
   use the live schema for all other calls. A call without it — or with an id the user can't access —
   fails authorization. Reuse the id from step 1 for the whole session.
3. **Read before you write, and read the right thing.** Never guess an id or a field name.
   For plan sections, `get_section_state` returns the schema **and** the current data (plus
   `get_section_completeness` for what's missing). Everything else has its own reader:
   `list_entities`, `list_demand_cases`, `list_outreach_campaigns`, `list_goals`,
   `list_time_entries`, and so on. If you don't have an id, you are looking for a `list_*`
   tool.
4. **Batch hierarchies with `write_entities`.** To create multiple related entities (e.g. a
   journey plus its stages and steps), prefer one `write_entities` call using `local_id` +
   `parent_refs` over many single calls. See `reference/write-entities.md`.
5. **Know which writes need saying yes.** `write_entities` defaults to `mode: "propose"`,
   which is **not a dry run** — it writes review cards the user must accept, and it does
   **not** soften `op: "update"` or `op: "delete"`, both of which apply immediately in either
   mode. Confirm every update, delete, publish, and invoice action with the user first, and
   report what actually happened (`data.mode`) rather than what you asked for.
6. **Respect the signals.** Honor empty states ("nothing here yet" is a valid answer — never
   fabricate). On `ACCESS_DENIED` or a tool missing from the list, stop — the user didn't
   grant that capability; do not retry. See `reference/errors-and-limits.md`.

## A note on permissions

Tools are gated three ways, all server-enforced: the user must be a workspace **owner or
editor** (`can_use_ai`); the connection's OAuth token must have been **granted** that tool
(capability bundles the user picked at connect time); and write/link tools also need the
`workspace:write` scope. So the tool list you see is already the user's effective surface —
trust it, and don't assume a tool exists just because it's documented here.

When something you need is missing, **name the bundle** rather than describing the tool: "I
need the *Manage outreach* permission to log that." The bundle is what the user ticks, and
`reference/tool-catalog.md` is organised by bundle so you can look up which one owns a tool.

## What Founder Guide will not let you do

Worth knowing up front, because these are the requests that otherwise end in a retry loop:

- **Founder Guide never sends anything.** No emails, no social posts, no replies to inbound
  comments. You plan and you record; the founder sends from their own inbox and accounts.
- **You cannot issue an invoice**, only draft one. Issuing, voiding, and recording payment
  against an issued invoice are human-only in the app.
- **You cannot move a demand case field's evidence state without a verbatim buyer quote**, and
  you cannot manufacture one from aggregates like a reply rate.
- **The sales pipeline and fundraising are not exposed.** `convert_outreach_target` opens a
  deal; everything after that happens in the app.

## Reference files

Load these as needed:

- **`reference/tool-catalog.md`** — every tool exposed over MCP, grouped by the **capability
  bundle** that grants it (which is what decides whether it is in your `tools/list` at all),
  with key parameters and the trap to avoid for each. Ends with what is deliberately withheld,
  so you don't reach for it.
- **`reference/workflows.md`** — ordered recipes for real requests: build and test a demand
  case, run an outreach campaign, prepare for a sales call, write a post in the founder's
  voice, plan content against themes, answer a cash-flow question, keep the forecast fed, log
  time and bill it, work the social inbox, track a goal, turn a brainstorm into backlog items,
  move a backlog over from Trello, upload and analyze a document, review
  the journey, capture assumptions, suggest changes without committing them.
- **`reference/write-entities.md`** — the universal batch writer in depth. **Read it before
  your first `write_entities` call:** `propose` vs `commit` is easy to get wrong, the batch is
  not atomic, and payload field names differ by kind (a stage has a `name`, a step has a
  `title`).
- **`reference/errors-and-limits.md`** — error codes, the permission model, rate limits, upload
  limits, and empty-state handling.
