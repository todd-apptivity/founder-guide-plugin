# Workflows

Copy-paste recipes for common founder requests. Every recipe assumes you have already called
`list_workspaces` once and are passing that `knowledge_graph_id` on each call.

## Read the current elevator pitch

1. `get_section_state` with the messaging/elevator-pitch `entity_type` → read the current
   pitch from the returned data. If empty, say so plainly — don't invent one.
2. To list multiple pitch variants stored as entities, `read_entity` on each id (or read them
   from the section data).

## List recent social / content posts

1. `get_content_items` with filters, e.g. `status: 'published'` or a `scheduled_before/after`
   range, ordered by date. Use `platform` / `format` / `tags` to narrow.
2. For full detail on one, `get_content_item` with its `content_id`.

## Capture assumptions, questions, or risks

1. `get_section_state` for the target section to confirm it exists and see what's already there.
2. To add: `add_assumption` / `add_question` / `add_risk` with `entity_type` (or `section_id`),
   `description`, and importance/severity/status.
3. To revise an existing one: the matching `upsert_*` with its id. After resolving a question,
   `upsert_question` with the updated status to record the finding.

## Draft and save a content item

1. (Optional) `get_tone_style_guidelines` with `resolve_effective: true` for the target
   `platform`/`author_type` so the draft matches the user's voice.
2. `create_content_item` with `platform`, `format`, `status: 'draft'`, the descriptions, and
   any `tags` / `scheduled_for`.
3. Later edits: `update_content_item` by `content_id`; to record publication or scheduling,
   `update_content_status` (note: publishing requires a post URL on the item).

## Upsert a task

1. `upsert_task` with `title`, `status` (required: `to_do`, `doing`, `done`, or `cancelled`), `description`, optional `effort_estimation`, `due_by`, `section_ids`, and
   `parent_type`/`parent_id` as supported by the live tool schema. Omit `task_id` to create, include it to update.
2. Read a task back with `get_task`.

## Review or edit the customer journey

1. `read_journey` with `journey_id` to load the full assembled canvas (stages, steps, arrows,
   persona, decisions, frictions…).
2. Small targeted edits: `update_journey_canvas` with `canvas_updates`.
3. Building a journey or many nodes at once: one `write_entities` call (see
   `write-entities.md`) — far fewer round-trips than node-by-node.

## Attach a document as evidence

1. `list_document_links` first to see what's already attached and avoid duplicates.
2. `link_artifact` (or `link_document_target`) with the `document_id`/`artifact_id`, the
   `target_type`, and `target_id` of the section/assumption/question/risk it supports.

## Upload a document and analyze it

1. `upload_document` with `file_name`, `mime_type`, and the content — `content` (plain text)
   for md/txt/csv, `content_base64` for pdf/docx. Max ~3MB decoded; point the user at the
   in-app uploader for bigger files. Pick `file_kind` deliberately — it decides what happens
   next:

   | `file_kind` | analysis_mode | What happens |
   |-------------|---------------|--------------|
   | `evidence`, `customer_signal` | `auto` | Analysis queues automatically after ingestion |
   | `reference` | `manual` | You must call `trigger_document_analysis` once ready |
   | `operating_doc` | `never` | Not analyzable |

2. Ingestion is asynchronous. Poll `get_document` until `version.status` is `ready` — the
   response's `next_step` field tells you exactly what to do at each stage.
3. For `auto` documents, keep polling until `analysis.status` is `complete`. For `manual`
   documents, call `trigger_document_analysis` once ready, then poll. Analysis runs an LLM
   pipeline and uses the user's token allowance — don't trigger it without a reason.
4. Read the results from `get_document`: per-agent progress under `analysis.agents`, plus
   spawned `tasks`, `patterns`, and `recommendations`.
5. Uploading evidence for a specific section/assumption? Follow up with `link_artifact`
   (see "Attach a document as evidence").

## Organize documents with folders and tags

1. `list_documents` with `include_facets: true` to see the folders and tags already in use —
   reuse them instead of inventing near-duplicates.
2. On upload, pass `folder_path` (slash-separated, e.g. `"research/interviews"`) and `tags`.
3. To reorganize later: `update_document_metadata` with `folder_path` (or `null` to unfile)
   and/or `tags` (replaces the full list — read current tags from `get_document` first when
   adding).
4. To browse: `list_documents` with `folder_path` (exact), `folder_prefix` (subtree), or `tag`
   filters.

## Write a post in the user's voice

1. `resolve_voice_profile` with the draft's context (`author`, `length`, `channel`, `account`,
   `funnel_stage`). **Not `get_voice_profile`** — that returns every layer and leaves you to
   resolve them yourself. `resolve_voice_profile` returns the rules that actually apply, plus
   the negative-space `avoid` list and real exemplars from the user's own writing.
2. Follow what it returns. The `avoid` list is the half people skip; it is what stops a draft
   reading like every other post on the platform.
3. Tagging anyone on LinkedIn? `resolve_linkedin_mention` first, then paste the returned
   `mention_markup` inline into the body. **Plain `@Name` tags nobody**, and neither does the
   `creative.draft.mentions` array — it is inert.
4. `create_content_item` with the body in `creative.draft.body`.
5. Image, video, or PDF for the post? Attach it after the item exists:
   - File you generated or have locally → `request_asset_upload` (`content_id`, `mime_type`),
     run the returned `curl` (PUT raw bytes, not base64), then `attach_content_asset` with
     `content_id` + `upload_path`.
   - Already at a public https URL (e.g. an image API's output) → `attach_content_asset`
     with `source_url`. No upload step.
   - Always pass `alt_text` for images. Never inline a real image as `content_base64` —
     tool arguments are capped near 50KB and the call fails.

## Plan content against themes

1. `get_theme_coverage` — the whole point of this tool is spotting what is *not* being said.
   A theme with a `silent` bucket and an empty pipeline is the gap; an over-covered one is
   where to stop.
2. `get_theme_detail` on the theme you picked, for its angles and their `content_usage`. Prefer
   an angle with published work behind it over inventing a new one.
3. `get_weekly_focus` to check the week's `reinforce` / `explore` / `avoid` buckets before
   committing.
4. `create_content_item` with `theme_id` and `angle_id` set. An unthemed post is invisible to
   every tool in step 1.
5. Genuinely off-theme (an announcement, a personal note)? Set `theme_excluded: true` rather
   than leaving it unthemed — that is the difference between "deliberately not a theme post"
   and "somebody forgot", and only the second one should nag.

## Build and test a demand case

The full loop, and the ordering matters — evidence cannot land on a case that does not exist,
and draft writes stop working the moment a case goes under test.

1. `list_demand_cases` first. **If one exists, use it.** A second case is not a second opinion,
   it is a Demand section that disagrees with itself about what the business is doing.
2. `ensure_demand_case` with a `label` — returns the case, creating one only if there is
   nothing to use. If `adopted` comes back non-null you are holding something a website scan or
   another session wrote: say so, rather than reading it back as the user's own words.
3. Fill it in with `update_case_draft` as the user talks — a field or two per call. Everything
   lands `assumed`, which is honest: nobody outside the building has said any of it yet.
   `cost_of_problem` and `named_people` merge, so send only what is new.
4. The gate to start testing is **five real names** at real companies. `update_case_draft`
   returns `names_to_gate`; tell the user how far off they are instead of waiting until the end.
5. Testing begins — now the case is `testing` and draft writes are refused. Move to evidence:
   - `propose_case_update` when the user should decide. **This is the default.**
   - `add_case_evidence` when they have told you to file directly.
6. Either way you need the buyer's **verbatim quote**, plus `person_id` (from `list_people`) and
   `speaker`. The provenance ladder counts **distinct people** — one supporter makes a field
   `supported`, a second *different* person makes it `confirmed`. Two unattributed quotes from
   the same thread count once, so a missing `person_id` quietly costs the user a confirmation.
7. `get_demand_case` to read the state back and say what moved.

**Never** derive evidence from aggregates. A 40% reply rate is not something a buyer said, and
filing it as a quote corrupts the one record that is supposed to be checkable.

## Run an outreach campaign

**Founder Guide does not send anything.** The user sends from their own inbox; these tools plan
the list and record what happened. Never imply a message went out.

1. `create_outreach_campaign` with a `demand_case_id` — it is required, because a list that is
   not testing a case cannot produce evidence about one. Get the id from `list_demand_cases`.
2. `import_outreach_targets` with `rows`. It creates missing contacts and companies and links
   them. Re-importing is safe: nobody doubles, message plans the user edited are never
   overwritten, and an existing customer is never relabelled a lead.
3. Each day: `get_outreach_queue` for who is due, oldest first, with the signal that put them
   on the list and the next planned message.
4. After the user sends: `record_outreach_sent` with the **exact body** that went out. It
   advances the sequence, schedules the next touch, and returns how much of the weekly target
   is left. Pass `external_ref` so a retry cannot double-log.
5. When something comes back: `log_outreach_reply` with `replied` / `no_reply` / `bounced` /
   `not_interested`, keeping their verbatim words. `no_reply` changes nothing — they stay in the
   sequence. A reply stops it.
6. Adjusting the plan is `update_outreach_target` — disqualify with a reason, hold someone out
   of Today's focus until a date, or rewrite upcoming messages. **Editing the plan never changes
   what was already sent.**
7. A reply worth pursuing: `convert_outreach_target` opens a deal that inherits the demand case,
   the signal, and the contact as champion.
8. `list_outreach_campaigns` for reply rates and why people were disqualified.

Those replies are also the best evidence the user will ever get — feed the good ones back
through `propose_case_update` (see the demand-case loop above).

## Prepare for a sales call

1. `get_meeting_playbook` with `opportunity_id` (one deal) or `demand_case_id` (the reusable
   sheet). It reads a cache by default.
2. **`generate: true` costs the user's tokens.** Only pass it when the cached sheet is genuinely
   out of date — the case moved, or the deal did. Optional `the_ask`, `lead_offer`,
   `call_length_minutes` and `who_leads` shape what comes back.
3. After the call, the answers are evidence: `add_case_evidence` or `propose_case_update` with
   the quotes.

## Answer a cash-flow question

1. `get_cash_forecast` — weekly inflows, outflows, closing balances, and the **cash trough**,
   which is the number that matters. It also flags a safety-floor breach.
2. Money owed: `get_ar_aging` for what is invoiced-but-unpaid, with days overdue and each
   item's impact on that trough.
3. Project-level: `get_project_margin`. Read the bottom-up vs pace-implied hours divergence —
   that gap is optimism risk, and it is usually the real answer.
4. Testing a worry ("what if they pay 30 days late"): `upsert_scenario` with an overlay, then
   `get_cash_forecast` with that `scenario_id`. Do not hand-edit real records to model a maybe.

## Keep the forecast fed

1. First time only: `set_opening_balance` with the balance, its date, and a `safety_floor_amount`.
   Without it every forecast is relative to zero.
2. `upsert_project` for the engagement (contract value, dates, payment terms), `upsert_resource`
   for whoever does the work (cost rate, bill rate, payroll).
3. `upsert_billing_item` for each **planned** invoice — recurring, milestone-date, or
   milestone-completion.
4. `upsert_cost_item` for contractor bills, payroll, and pass-through expenses.
5. As reality lands: `draft_invoice` + `generate_invoice_from_milestones` (or
   `generate_invoice_from_time`), which the founder then issues — issuing is what creates the
   committed inflow, due on net terms. The founder records payment against an issued invoice in the app.
   `record_payment` takes a planned `billing_schedule_id`; do not use it to record payment
   against an issued invoice. The gap between issuing and payment teaches the forecast how
   this client pays. Issuing and sending are human-only: agents draft, founders send.
6. Keep `update_estimate_to_complete` current — it drives the margin pace check, and a stale
   estimate makes a project look healthier than it is.

## Log time and bill it

1. `log_time` with `resource_id`, `entry_date`, `hours`, and a project or task. You can log on
   **another resource's behalf** — that is how a contractor's hours get in. Rates snapshot
   automatically, so a later rate change does not rewrite history.
2. `upsert_activity` manages the grouping taxonomy. Keep it short — a long activity list makes
   invoice grouping useless.
3. Corrections: `update_time_entry`. An invoiced entry is **locked** and must be unlocked in
   the app first; do not work around it.
4. Billing: `draft_invoice`, then `generate_invoice_from_time` (unbilled billable entries) or
   `generate_invoice_from_milestones` (planned billing items). `preview_invoice` to check the
   layout.
5. **Stop there.** Issuing, voiding, and recording a client payment against an issued invoice
   are human-only in the app. Hand the draft over and say it is ready to issue.

## Work the social inbox

1. `get_social_inbox` — defaults to what is still unanswered.
2. Commenters who are not yet contacts are the cheapest leads available: they already engaged
   with something the user published. Cross-check `list_people`.
3. **You cannot reply.** Posting goes out publicly, immediately, in the user's name, and there
   is no delete. Draft the reply in chat and let them send it.
4. A comment that reveals a real situation is evidence — route it through the demand-case loop
   with a quote rather than paraphrasing it into a note.

## Track a founder goal

1. `list_goals` (optionally by `state`) then `get_goal` for one, with its tasks and derived
   progress.
2. `upsert_goal` to create or update. A goal measures either by a **target number**
   (`measure_type` with `min_value` → `target_value`) or by a **checklist of tasks** — pick one
   deliberately; they produce different progress bars.
3. Tasks hang off a goal via `upsert_task` with `goal_id`. That is the only link — a task
   without it is loose.
4. Moving `active` → `complete` → `archived` is `upsert_goal` with `state`.

## Turn a brainstorm into backlog items

The founder and you have talked a feature through and want it written down where it gets built.

1. `list_backlog` (`group_by: 'status'` to see the board) so you file under an existing
   feature instead of making a duplicate.
2. Half-formed thoughts the founder is not ready to commit to: `upsert_idea`, one per idea.
   They wait in the board's Idea column. `promote_idea` turns one into a story later.
3. Clear work: **one** `create_backlog_items` call. Pass `feature` (`{feature_id}`, or
   `{name, description}` for a new one) and the stories in priority order. Each story carries
   `acceptance_criteria` (what done means), `testing_steps`, and a `checklist` (the tasks to
   get there). A defect can point at a story from the same call with `story_ref`.
   Retry only the items whose result is not `ok`.
4. Put specs, mockups and research on the item they belong to. Use `attach_backlog_text` for
   anything you wrote (markdown, JSON, SVG). For a local file, use `request_backlog_upload`,
   then `curl` PUT, then `attach_backlog_upload`. Both need the *Comment on and attach files*
   capability. If the PUT is blocked by your own sandbox, do not retry through a browser: ask the
   user to allow `www.founder-guide.com` (in Claude: Settings > Capabilities > *Allowed websites*).
5. Record the decisions you made while brainstorming with `add_backlog_comment` on the story
   or feature, so whoever builds it knows why.
6. Set priority with `move_backlog_item`: `position: 'top'`, or `place_below` / `place_above`
   another item. Higher on the board is built first.

While building: `check_backlog_items` ticks tasks off. `move_backlog_item` with `column`
(`building` → `testing` → `deployed`) moves the card. A comment records what changed.
None of these void a vNext price except `set_backlog_checklist` and attachments.

## Move a backlog over from Trello

Use this when the founder has a Trello board and both connectors are available. There is no
importer: you read one connector and write the other.

1. Read the Trello board: its lists, cards (name, description, checklists, labels), and each
   card's attachments.
2. Map it with the founder before writing anything:
   - lists → board columns (`idea`, `ready`, `building`, `testing`, `deployed`, `wont_do`);
   - cards → stories, or defects for bugs. Use a defect when a label says bug or the card
     describes broken behaviour;
   - whatever grouped cards (a label, an epic card, a list per feature) → features;
   - a card's checklist → the story's `checklist`. If it is really a definition of done, use
     `acceptance_criteria` instead.
   Labels, members and due dates have no Founder Guide field. Fold anything worth keeping into
   `context`.
3. Per feature, one `create_backlog_items` call, with cards in the list's top-to-bottom order.
4. Place each item: `move_backlog_item` with `column`, then `position: 'bottom'` in list
   order. That rebuilds Trello's priority.
5. Re-attach files (`attach_backlog_text`, or the upload pair). Copy useful Trello comments
   into one `add_backlog_comment` per card, prefixed "From Trello:".
6. Read back with `list_backlog` (`group_by: 'status'`), show the founder the board, and fix
   anything they flag before they archive the Trello board.

## Suggest changes without committing them

Three options, best first:

1. **The Field suggestions card** (`suggestions_open`), if the connection granted it. Renders
   suggested edits the user accepts or rejects one at a time; nothing is written until they do.
   This is the right tool when they asked for suggestions rather than changes.
2. **`write_entities` with `mode: 'propose'`** for entities — items land as review cards
   (`is_suggested: true`) instead of live rows. Use it whenever you would otherwise type "what
   about adding X?" — emit the proposal instead of the sentence.
3. **Describe it in chat**, get the go-ahead, then apply with the relevant update tool.

`provide_field_suggestions` is in-app only and will not appear in your `tools/list`.

Confirm before `delete_entity` or changing a content record to published. Changing that
record does not send a social post. Issuing an invoice is unavailable through MCP; direct
the founder to the app.
