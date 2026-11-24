# Tool Catalog

Every tool below is exposed over the Founder Guide MCP server. **All of them except
`list_workspaces` (and the vNext team's `list_vnext_workspaces`) require a
`knowledge_graph_id` argument** (from `list_workspaces`). The
exact JSON Schema for each tool's parameters is delivered by the MCP `tools/list` response —
this catalog is the orientation map, not a substitute for those schemas.

**Availability is per-connection**, and this catalog is grouped by the *capability bundle* the
user ticks at consent for exactly that reason. A tool only works if its bundle was granted and
(for writes and links) the connection carries the `workspace:write` scope. If a tool is not in
your `tools/list`, the user did not grant it — name the bundle when you ask, e.g. "I need the
*Manage outreach* permission to log that." Do not retry a tool that returned
`ACCESS_DENIED`; see `errors-and-limits.md`.

Bundles marked **always on** are granted to every connection. Everything else is opt-in.

## Start here, every session

1. `list_workspaces` — the only tool that needs no `knowledge_graph_id`. Pick one and pass its
   id on every subsequent call.
2. `get_section_state` / `list_entities` — read before you write. Always.
3. Then the section below that matches the job.

If you do not have an id for something, you are looking for a `list_*` tool, not guessing a
uuid. `list_entities` is the general one; most areas have a specific one.

---

# Read tools

None of these change anything, so reading widely before acting is free and expected.

## `Read business profile` — always on

The plan itself: dimensions, entities, documents.

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_workspaces` | `limit?` | **Start here.** Workspaces where the user can use AI, each with id + name. |
| `get_section_state` | `entity_type`, `fields?`, `instance_id?` | A section's TypeScript schema **and** current data. Singletons return one props object; multi-instance types (`JobToBeDone`, `Offer`, `ElevatorPitch`, `TeamMember`) return an array. `fields:[...]` returns values only, no schema — use it for a quick pre-write check. `entity_type: "personas"` fetches all personas. |
| `get_section_completeness` | `section_type` | The completeness ring for one dimension (`problem-market`, `customer-contacts`, `offering-value`, `messaging-positioning`, `acquisition-conversion`, `delivery-retention`, `operations-economics`, `team`, or `overview` for the whole graph). Returns `completeness_percent` + per-dimension scores. |
| `read_entity` | `entity_id`, `depth?` | One entity, optionally with its relationships and adjacent entities. Works for any entity kind. |
| `list_entities` | `types?`, `name_contains?`, `include_suggested?`, `limit?`, `offset?` | **Discovery.** Ids by type, newest-updated first. Returns id/name/types/timestamps, never props. Use it before creating anything that might already exist. Paged, with `total` + `has_more`. |
| `list_documents` | `file_kind?`, `analysis_status?`, `title_contains?`, `folder_path?`/`folder_prefix?`, `tag?`, `include_facets?`, `limit?`, `offset?` | The document library, newest first. `include_facets=true` returns the folders and tags already in use — reuse them rather than inventing new ones. |
| `get_document` | `document_id`, `include_insights?` | One document: metadata, latest version status (poll after `upload_document` until `ready`), and analysis progress/results. **`next_step` tells you what to do next.** |
| `list_document_links` | `document_id` OR (`target_type` + `target_id`) | What a document is attached to, or what is attached to a target. Check before adding evidence. |
| `describe_app_feature` | `feature_id` | In-depth docs for a Founder Guide feature — what it does and where the user clicks. No user data. Use it when the user asks what the app can do or where something lives, rather than guessing at the UI. |

## `Read content library` — on by default

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `get_content_items` | `status?`, `platform?`, `format?`, `author_type?`, `title_contains?`, `author_contains?`, `tags?`, `theme_ids?`, `theme_assigned?`, `scheduled_between?`/`scheduled_after?`/`scheduled_before?`, `published_*`, `limit?`, `offset?`, `order_by?` | Filtered, paged posts (default 50, max 100). For calendar questions use a **narrow** `scheduled_between` window. `theme_assigned: false` finds unthemed posts. |
| `get_content_item` | `content_id` | One item in full: strategy, post brief, creative, distribution, performance, learning. |
| `get_tone_style_guidelines` | `resolve_effective?`, `platform?`, `social_connection_id?`, `author_type?` | Reusable tone/style guidelines. `resolve_effective=true` returns the single guideline that applies to a draft. |
| `get_voice_profile` | — | The whole layered Voice Profile: every layer with its traits, plus annotated examples. For **inspecting or editing** the voice. |
| `resolve_voice_profile` | `author?`, `length?`, `channel?`, `account?`, `funnel_stage?` | The voice **composed for one draft**: rules by content dimension, the negative-space `avoid` list, and context-ranked exemplars. **Call this before drafting or substantially rewriting a post**, and follow what it returns. |
| `get_themes` | `status?`, `include_angles?` | Content themes (title, summary, why_it_matters, status, order). `include_angles=true` adds each theme's sub-topics. `status="all"` includes archived. |
| `get_theme_detail` | `theme_id`, `include_archived_angles?` | One theme with its angles **plus `content_usage`** — published/planned counts per theme and angle, so you can see which angles are worked and which are stale. |
| `get_theme_coverage` | `theme_ids?`, `window_days?` | Per-theme coverage over a window (default 30 days): a `silent`/`low`/`medium`/`high` bucket, pipeline counts, share of themed output, last reinforced. Window totals separate `excluded` (deliberately unthemed) from `unthemed` (the real backlog). Use it to find neglected themes before proposing content. |
| `get_weekly_focus` | `week_start_date?`, `include_history?`, `history_weeks?` | The week's four buckets: reinforce / explore / experiment / avoid, each with a theme and rationale. |
| `get_audience_observations` | `include_dismissed?`, `limit?` | What the audience is responding to, derived from engagement events, with confidence scores and linked themes. |
| `get_social_inbox` | `state?`, `kind?`, `unresolved_only?`, `limit?` | Inbound comments, DMs, mentions and reviews on connected accounts. Defaults to what is unanswered. **Commenters who are not yet contacts are the cheapest leads there are** — they already engaged with something the user published. You can read this; you cannot reply (see the bottom of this file). |

## `Read contacts`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_people` | `segment?`, `limit?`, `offset?` | People contacts. Use it to find a `person_id` before attributing anything to someone. |
| `list_organizations` | `industry?`, `limit?`, `offset?` | Organizations (name, industry, size, notes). |
| `list_notes` | `category?`, `limit?`, `offset?` | Notes (title, content, metadata). |

## `Read tasks & goals`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_goals` | `state?` | The founder's goals (`active`/`complete`/`archived`): id, title, measure type, due date, target vs current. |
| `get_goal` | `goal_id` | One goal with its tasks and derived progress — progress percent, label, target/current. |
| `get_task` | `task_id` | One task in full: markdown content (steps, checklists, templates), status, effort, due date, related sections, reasoning, `goalId`, reflection data. |

## `Read journey`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `read_journey` | `journey_id` | The assembled journey: stages, steps (tasks/outcomes/decisions/examples/fences/features/friction/inbox/prompt), arrows, and persona. **Read this before proposing structural changes** so recommendations are grounded in what is there. |

## `Read mind maps`

A mind map is a freeform brainstorm tree, not a business record. Nodes are labels and notes —
not assumptions, personas, risks or features — and **nothing on a map counts until the founder
accepts it**. To promote something you read here, use `write_entities` in `propose` mode.

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `read_mind_map` | `mind_map_id?` | Omit the id to list every map in the workspace (a map id is not an entity id, so `list_entities` will not find one). With an id — or on a mind map page, where `pageContext.entityRefs.mind_map` supplies it — returns that map as an indented `outline` plus the structured `doc`. **Read the outline, not the tree**, and read before editing so you are working from current ids. |

## `Read demand cases`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_demand_cases` | `status?` | Every case: status, which is active, how many people are named, whether a buyer ever put a number on the problem, and the provenance state of the four fields. |
| `get_demand_case` | `case_id` | One case in full: the four fields with text **and provenance state**, the C.O.S.T. of the problem with who said each figure, named people, the smallest offer, and the campaigns testing it. `assumed` means nobody outside the building has said it. |

## `Read outreach`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_outreach_campaigns` | `status?`, `demand_case_id?` | Campaigns with per-campaign counts (queued/sent/replied/disqualified/due today), reply rate, disqualification reasons, and which case each tests. |
| `get_outreach_queue` | `campaign_id`, `limit?` | Who is **due today** on one campaign, oldest first: name, email or LinkedIn, the signal that put them on the list, sequence position, and the next planned message. |
| `get_meeting_playbook` | `opportunity_id` OR `demand_case_id`, `generate?`, `the_ask?`, `lead_offer?`, `call_length_minutes?`, `who_leads?` | The sheet for a live call: qualification bar, ranked learning targets, 8–12 past-tense discovery questions each labelled with the case field it tests, what to listen for, the bridge to the offer, the ask, anticipated objections, and the post-call capture. Reads cache by default — **`generate: true` costs LLM tokens**. |

## `Read cash flow`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `get_cash_forecast` | `scenario_id?`, `horizon_weeks?`, `safety_floor?`, `as_of?` | Weekly inflows, outflows, closing balances, the cash trough, and any safety-floor breach — optionally under a what-if scenario. |
| `get_project_margin` | `project_id`, `as_of?` | Contract value, cost-to-date, estimate-to-complete, margin at completion, and the bottom-up vs pace-implied hours divergence (optimism risk). |
| `get_ar_aging` | `as_of?` | Invoiced-but-unpaid items with days overdue and each one's trough impact, plus total outstanding. |
| `preview_invoice` | `invoice_id` | Invoice layout data (seller, client, line items, totals) without issuing it. |

## `Read time tracking`

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_time_entries` | `project_id?`, `resource_id?`, `billable?`, `invoiced?`, `from_date?`, `to_date?` | Time entries matching the filters. |
| `get_project_hours` | `project_id`, `from?`, `to?`, `by_resource?` | Hours summed by project and activity, with billable amounts from rate snapshots — the shape an invoice wants. |

## `Read backlog & vNext`

The product backlog (features, user stories, defects) and, on a workspace with vNext, the
vNext side of it. `list_backlog` works without vNext; the Inbox and request tools need it.

| Tool | Key params | What it returns |
|------|-----------|-----------------|
| `list_backlog` | `lens?` (`open`\|`stories`\|`defects`\|`vnext`\|`all`), `types?` (`story`\|`defect`\|`idea`), `statuses?`, `query?`, `feature_id?`, `vnext_state?`, `group_by?` (`status`), `limit?` | Features, then stories and defects in priority order with status, `rank`, feature, who delivers it, the vNext state, and the latest released estimate. Ideas only when `types` asks. **`group_by: 'status'` returns the board**: one column per status (idea, ready, building, testing, deployed), each top-priority first. |
| `get_backlog_item` | `entity_id` | One story (all its fields), defect (error, steps, expected, actual) or feature (its stories, decisions), plus vNext state, estimate, attachments (each with a 15-minute `download_url`), the latest comments and, for stories and defects, the checklist (ids, text, done). **Read this before changing an item.** |
| `list_backlog_comments` | `entity_id`, `order?` (`newest`\|`oldest`), `limit?`, `cursor?` | The comments on an item, with who wrote each (founder, teammate, or an assistant and its client name). `get_backlog_item` already carries the latest five. |
| `list_vnext_inbox` | `waiting_on?`, `entity_id?`, `cursor?` | Conversations with the vNext team. `waiting_on: workspace` means vNext waits for the team. `needs_you_count` also counts open requests. |
| `get_vnext_thread` | `thread_id` | One conversation's messages. A message flagged as a possible credential comes back hidden. |
| `list_vnext_requests` | `open_only?` | vNext's requests for a key (`request_value`) or for access (`request_access`), and sends. Metadata only. A `request_value` is answered **only in Founder Guide** — never ask the user to paste a secret into chat. |

---

# Write tools

All of these need `workspace:write` **plus** their bundle. Two rules across every one of them:

- **Read the current state first.** These tools write into a real business record, not a
  scratchpad.
- **Propose rather than commit unless the user authorised the write.** Where a tool has a
  `mode`, `propose` is the default and is the right default.

## `Edit business profile` — risk: high

The plan's own fields and entities. High risk because it rewrites what the user wrote.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `write_entities` | `mode` (`propose`\|`commit`), `items[]` | **Preferred batch writer** for multi-instance and journey entities — create/update/delete across ~20 kinds in one call. See `write-entities.md`. `propose` makes review cards; `commit` needs explicit authorisation ("do it", "yes, add it"). |
| `create_entity` | `entity_type` (`JobToBeDone`\|`Offer`\|`ElevatorPitch`\|`TeamMember`), `props?` | Create one multi-instance entity. For batches prefer `write_entities`. Returns `entity_id`. |
| `create_persona` | `name`, `persona_type` (`b2b`\|`b2c`\|`network`), `role?`, `seniority?`, `goals?`, `pains?`, `motivations?`, `decision_influences?`, `anti_signals?`, … | Create a buyer/user persona. |
| `update_section_attribute` | `entity_type`, `attribute_path` (dot notation), `value`, `entity_id?`, `merge_arrays?` | Update one attribute. Take `attribute_path` from `get_section_state`. Arrays are **replaced** unless `merge_arrays=true`; objects always deep-merge. **Only after explicit approval** — for suggestions, use the Field suggestions card. |
| `unlink_relationship` | `from_id`, `to_id`, `predicate`, `mode?` | Remove an edge. `propose` (default) only removes suggested edges; `commit` removes confirmed ones and needs authorisation. For journey step-to-step arrows use `write_entities` with kind `journey_arrow` instead — those are inline, not edge rows. |

## `Quick capture` — risk: low

Append-mostly. The safest place to put something the user just said.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `add_assumption` / `add_question` / `add_risk` | `entity_type` OR `section_id`, + `description`/`question`, importance/severity/status | Append a validation item to a section. |
| `upsert_assumption` | `assumption_id?`, `entity_type`/`section_id`, `description`, `importance?`, `validation_status?` | Create or update an assumption (omit the id to create). |
| `upsert_question` | `question_id?`, …, `question`, `suggested_respondent?`, `priority?`, `status?`, `answer?` | Create or update a question. |
| `upsert_risk` | `risk_id?`, …, `description`, `severity?`, `mitigation?`, `status?` | Create or update a risk. |
| `upsert_task` | `task_id?`, `title`, `status`, `description?`, `effort_estimation?`, `due_by?`, `section_ids?`, `goal_id?`, `parent_type?`/`parent_id?`, `subtasks?` | Create or update a task. Pass `goal_id` to hang it under a founder goal. |
| `upsert_goal` | `goal_id?`, `title`, `description?`, `due_date?`, `measure_type?`, `unit?`, `min_value?`, `target_value?`, `current_value?`, `state?` | Create or update a founder goal. Goals measure by a target number (min→target) or by a checklist of tasks, and move `active` → `complete` → `archived`. Add tasks under it with `upsert_task` + `goal_id`. |

## `Author content` — risk: low

| Tool | Key params | What it does |
|------|-----------|--------------|
| `create_content_item` | `platform`, `format`, `status`, `title?`, `short_description?`, `scheduled_for?`, `creative?`, `post_brief?`, `strategy?`, `tags?`, `theme_id?`, `angle_id?`, `theme_excluded?`, `author_type?` | Create a draft/scheduled/published post. Assign a theme at creation from `get_themes`. |
| `update_content_item` | `content_id` + any field | Update a post. Unsent fields are preserved, and JSONB fields (`creative`, `strategy`, `post_brief`, `distribution`, `performance`, `learning`, `effort`) are **shallow-merged** — send only the keys you are changing. `creative.draft` merges too, so updating the body leaves hook_type and tone alone. `theme_id: null` clears the theme; `theme_excluded: true` marks it intentionally unthemed so it stops counting against coverage. Every call snapshots a version. |
| `request_asset_upload` | `content_id`, `mime_type`, `size_bytes?` | Step 1 for any image/video/PDF bigger than a few KB (tool arguments are capped near 50KB, so bytes can't go inline). Returns `upload_url` (on `www.founder-guide.com`), `upload_path`, and a ready `curl` line: **PUT the raw bytes** with the shown `Content-Type`. URL expires in 2h. |
| `attach_content_asset` | `content_id` + one of `upload_path` \| `source_url` \| `content_base64`, `alt_text?`, `file_name?`, `mime_type?` | Adds the file to the post's `creative.assets`. `upload_path` after a PUT from `request_asset_upload`; `source_url` when the image already sits at a public https URL (server downloads it); `content_base64` only for tiny files. PDF/slides are LinkedIn-only; LinkedIn images max 8MB. Don't hand-write `creative.assets` — use this. |
| `update_content_status` | `content_id`, `status` | Move a post between statuses. Validated — `published` requires `distribution.post_url`. |
| `resolve_linkedin_mention` | `person_id` OR `linkedin_url`, `display_name?` | **Required to tag anyone on LinkedIn.** Plain `@Name` tags nobody, and neither does the `creative.draft.mentions` array. Call this and paste the returned `mention_markup` (`@[Name](urn:li:person:…)`) inline into `creative.draft.body` verbatim. |
| `upsert_tone_style_guideline` | `social_connection_id?` \| `is_workspace_default?` \| `platform?` \| `author_type?`, + `tone?`, `style_guidelines?`, `examples?` | Create or update a tone/style guideline at one of four scopes. Needs at least one of tone/guidelines/examples. |
| `upsert_voice_layer` | scope dims (`author?`, `length?`, `channel?`, `account?`, `funnel_stage?`), `traits?`, `status?`, `source?` | Create or update one layer of the Voice Profile. Array dims match when the draft's value is **in** the array; omit a dim to leave it a wildcard; omit all for the root voice. `traits` **replace** the layer's rules — use content_dimension `negative_space` for `avoid` rules. More specific layers win at draft time. |
| `upsert_voice_example` | `content`, context dims, `demonstrated_traits?`, `source?`, `source_content_id?` | Add an annotated example to the Voice Profile, tagged with the context it represents. Retrieved by context at draft time. |

## `Manage content themes` — risk: medium

No hard delete anywhere here; archiving is reversible and leaves assigned posts alone.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `upsert_theme` | `theme_id?`, `title`, `summary`, `why_it_matters?`, `status?`, `angles?` | Create or update a theme, or archive/restore via `status`. Omit `theme_id` to create — `title` and `summary` are then required and you may include up to 10 initial angles. **There is a cap on active themes**; when you hit it, update or archive one instead. |
| `upsert_angle` | `angle_id?`, `theme_id?`, `label`, `prompt?`, `example_post_direction?`, `status?` | Create or update an angle inside a theme. **Deliberately refuses to move an angle between themes** — that is `reparent_angle`. |
| `reparent_angle` | `angle_id`, `target_theme_id` | Move an angle to another (active) theme. It keeps its id, label, prompt and history; posts assigned to it move with it. |

## `Edit journey` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `update_journey_canvas` | `journey_id`, `op` (`reorder_stages`), `ordered_stage_ids` | Reorder stages — pass the **complete** id list in the new order. To add or remove stages, use `write_entities` with kind `journey_stage`. |

## `Edit mind maps` — risk: medium

The canvas is the founder's thinking space. Put things on it when they ask you to — not to
record your own conclusions, and never to promote anything into the business profile (that is
`write_entities` in `propose` mode).

| Tool | Key params | What it does |
|------|-----------|--------------|
| `update_mind_map` | `mind_map_id?` (falls back to `pageContext.entityRefs.mind_map`) + any of `add_nodes`, `update_nodes`, `move_nodes`, `remove_nodes`, `fold_nodes`, `place_nodes`, `release_nodes`, `add_links`, `update_links`, `remove_links`, `add_boundaries`, `update_boundaries`, `remove_boundaries`, `add_summaries`, `update_summaries`, `remove_summaries`, `layout`, `branch_shape` | One call carries several op arrays, applied structure → appearance → map-level. **`remove_nodes` deletes each node AND ITS WHOLE SUBTREE** plus any cross-links touching it; the central node cannot be deleted. Layout is computed, so leave positions alone unless the founder asked for an arrangement — `place_nodes` pins, `release_nodes` and `layout` drop pins. Folding is their view of the map. Send no version: a lost race is re-applied server-side. Stale ids come back as `warnings`, not a failure, so the rest of the batch still lands. Call `read_mind_map` first. |

## `Manage documents` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `upload_document` | `file_name`, `mime_type` (pdf\|docx\|txt\|csv\|md), `content` (text) OR `content_base64` (pdf/docx), `title?`, `file_kind?`, `folder_path?`, `tags?`, `analyze?` | Upload a file, max ~3MB decoded — bigger files go through the in-app uploader. `file_kind` drives analysis: `evidence`/`customer_signal` auto-analyze, `reference` needs a manual trigger, `operating_doc` never analyzes. Ingestion is async — **poll `get_document`**. |
| `update_document_metadata` | `document_id` + `title?`, `description?`, `file_kind?`, `analysis_mode?`, `retrieval_eligibility?`, `folder_path?` (null to unfile), `tags?` (full replace) | Edit metadata and filing. `origin` is immutable. |
| `trigger_document_analysis` | `document_id` | Start multi-agent analysis manually (version must be `ready`). **Runs an LLM pipeline — uses the user's token allowance.** Poll `get_document`. |

## `Draft demand cases` — risk: medium

A demand case is one buyer's situation written down so it can be tested: the outcome they
want, why now, what else they are considering, what is missing from those options, what the
problem costs them (C.O.S.T.), and the smallest offer that finishes the job.

These three write **the founder's own words**. Nothing is quoted, every field lands `assumed`,
and the case stays `untested` so the user can rewrite all of it.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `ensure_demand_case` | `label`, `outcome?`, `role?`, `starting_assumption?`, `case_id?` | **Start here.** Returns the workspace's case, creating one only if there is nothing to use. `adopted` non-null means you are holding a case somebody else wrote — say so rather than reading it back as the user's own words. |
| `update_case_draft` | `case_id` + any of `label`, `role`, `outcome`, `why_now`, `what_else`, `whats_missing`, `smallest_offer`, `signal`, `starting_assumption`, `cost_of_problem[]`, `named_people[]` | Fill the case in a piece at a time. `cost_of_problem` and `named_people` **merge** with what is there — send only what is new. |
| `draft_demand_case` | `label`, `outcome` + the same optional fields, `lead_offer_id?` | The whole case in one call. **Omit `interview_session_id`** — that belongs to the in-app interview. This **overwrites** the text of every field you pass, so read the case first. |

## `Manage demand cases` — risk: medium

The other half: moving a field's provenance state. Every call here demands the buyer's
**verbatim quote**, and that is the entire difference between evidence and an assertion.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `propose_case_update` | `case_id`, `conversation_id`, `field`, `action` (`support`\|`contradict`), `quote`, `speaker?`, `proposed_text?` | Queue a change for the user to accept or dismiss. **Prefer this** unless they told you to file evidence directly. Nothing changes until they decide. |
| `add_case_evidence` | `case_id`, `action` (`support`\|`contradict`\|`cost_stated`\|`revise`), `field`/`dimension`, `quote`, `person_id?`, `speaker?`, `source?`, `document_id?`, `new_text?`, `reason?`, `kept_support_conversation_ids?` | Applies immediately. Carry `person_id` and `speaker`: the ladder counts **distinct people**, so two unattributed quotes in one thread only count once. `revise` is the user changing their mind — it needs their `reason` in their words, refuses the `outcome` field, and drops every supporter they have not confirmed. |

Rules that will bite you:

- **Do not create a second case.** Call `ensure_demand_case` and use what it returns.
- **A case that reached `testing` refuses draft writes**, and the error says to use
  `add_case_evidence`. That is not a bug to route around.
- **You cannot set a field's state**, make a case active, or mark a cost as buyer-stated.
  Those come from evidence and from the user.
- **Never derive evidence from aggregates.** A reply rate is not something a buyer said.

## `Manage contacts` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `upsert_contact` | `contact_id?`, `name` OR `first_name`/`last_name`, `email?`, `linkedin_url?`, `phone?`, `title?`, `company?`, `relationship?`, `notes?`, `create_new?` | Add or edit a PERSON in the contacts (not personas or team members — never use `write_entities`/`create_entity` for this). Without `contact_id` it matches first (email → LinkedIn → name + company) and returns `created`, `matched` (blank fields filled, nothing overwritten — see `not_overwritten`), or `ambiguous` (nothing saved). **Always pass `company` and `email` when known** — they prevent duplicates. |

- **On `ambiguous`, ask the user** which candidate they mean, then call again with that
  `contact_id`, or `create_new: true` only if they say it is none of them.
- **With `contact_id`, passed fields overwrite.** `relationship` is set exactly — only when the
  user says so. `company` is ignored: this tool never moves a contact to another employer.

## `Manage outreach` — risk: medium

**Founder Guide never sends anything.** The user sends from their own inbox; these tools plan
the list and record what actually happened.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `create_outreach_campaign` | `name`, `demand_case_id`, `goal?`, `signal?`, `weekly_target?`, `calls_target?`, `priced_offers_target?` | Start a prospect list worked against one demand case. **`demand_case_id` is required** — a list that is not testing a case cannot produce evidence about one. Reuses a same-named campaign rather than duplicating. |
| `import_outreach_targets` | `campaign_id` OR `campaign_name`, `rows[]`, `demand_case_id?`, `start_date?` | Add people: creates missing contacts and companies, links each person to their company, and queues them with planned messages. Re-importing the same people is safe — nobody doubles, edited message plans are never overwritten, and an existing customer is never relabelled a lead. |
| `record_outreach_sent` | `target_id`, `channel`, `body`, `subject?`, `sent_at?`, `external_ref?` | Log a message that went out. Copies the exact body to the permanent log, advances the sequence, schedules the next touch, and returns how much of the weekly target is left. Pass `external_ref` to make retries safe. |
| `log_outreach_reply` | `target_id`, `outcome` (`replied`\|`no_reply`\|`bounced`\|`not_interested`), `body?`, `note?` | Record what came back, keeping their verbatim words. `no_reply` changes nothing — they stay in the sequence. A reply stops it. |
| `update_outreach_target` | `target_id`, `status?`, `disqualified_reason?`, `signal?`, `messages?`, `next_message_due?`, `follow_up_reminder_at?`, `notes?` | Disqualify with a reason, hold someone out of Today's focus until a date, rewrite planned messages, or fix the signal. **Editing the plan never changes what was already sent.** |
| `convert_outreach_target` | `target_id`, `title?`, `person_name?` | Turn a target who replied into an opportunity. The deal inherits the campaign's demand case, the signal, and the contact as champion, and opens at Qualify. The case's cost of the problem is seeded as **inferred**, never as stated. |

## `Manage cash flow` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `upsert_project` | `project_id?`, `name`, `organization_id?`, `engagement_type?`, `contract_value?`, `start_date?`, `end_date?`, `payment_terms_net_days?`, `estimated_total_hours?`, `status?` | Create or update a client project. |
| `upsert_resource` | `resource_id?`, `name`, `role?`, `cost_rate?`, `bill_rate?`, `payroll_schedule?`, `payroll_amount?`, `invoice_cadence?`, `invoice_lag_days?`, `payment_terms_net_days?` | Create or update a founder, employee, or contractor with their rates and terms. |
| `set_opening_balance` | `opening_balance`, `opening_balance_date`, `name?`, `currency?`, `safety_floor_amount?` | Set the cash account's opening balance, currency, and safety floor. Creates the single cash account if none exists. |
| `upsert_billing_item` | `billing_schedule_id?`, `project_id?`/`contract_id?`, `trigger_type`, `amount_source`, `amount?`/`pct_of_contract?`, `cadence?`, `expected_date?`, `phase_name?`, `payment_terms_net_days?` | Create or update a **planned** invoice — recurring, milestone-date, or milestone-completion. |
| `record_payment` | `billing_schedule_id`, `paid_date?` | Mark it paid and settle the ledger entry on the actual date (feeds payment-timing learning). |
| `update_estimate_to_complete` | `project_id`, `phases[]` | Update remaining-hours estimates and phase status — this is what drives the margin pace check. |
| `upsert_cost_item` | `cost_schedule_id?`, `type`, `project_id?`/`resource_id?`, `amount`, `cadence?`, `invoice_lag_days?`, `net_terms_days?` | Create or update a contractor bill, payroll, or pass-through expense. |
| `upsert_scenario` | `scenario_id?`, `name`, `is_base?`, `overlay?` | Create or update a what-if overlay (client pays late, delay a cost) to stress-test the forecast. |
| `draft_invoice` | `contract_id?`, `project_id?`, `currency?`, `notes?` | Create a draft invoice, no number assigned. |
| `generate_invoice_from_time` | `invoice_id`, `from_date?`, `to_date?`, `project_id?`, `group_by?`, `replace_existing?` | Add line items from unbilled billable time. |
| `generate_invoice_from_milestones` | `invoice_id`, `billing_schedule_ids?`, `replace_existing?` | Add line items from planned billing milestones. |

**Issuing, voiding, and recording a client payment against an issued invoice are human-only.**
You draft; the user issues in the app.

## `Manage time tracking` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `log_time` | `resource_id`, `project_id?`, `task_id?`, `activity_id?`, `entry_date`, `hours`, `billable?`, `notes?` | Log time, including **on another resource's behalf** — this is how a contractor's hours get in. Rates snapshot automatically. |
| `update_time_entry` | `time_entry_id` + any field | Edit an unlocked entry. An invoiced entry is locked and must be unlocked in the app first. |
| `upsert_activity` | `activity_id?`, `name`, `default_billable?`, `sort_order?`, `is_archived?` | Create, rename, or archive an activity — the invoice-grouping taxonomy. **Keep the list short**; a long one makes grouping useless. |

## `Generate prompt` — risk: medium

| Tool | Key params | What it does |
|------|-----------|--------------|
| `generate_prompt` | `target_id`, `target_kind` (`journey`\|`step`\|`feature`\|`friction`\|`idea`\|`defect`), `purpose`, `text`, `built_from?` | Persist a structured Prompt for a journey target. **You** write the markdown body; this stores it with `about → target` and `built_from → sources` edges. Use for "build prompt", "refactor prompt", "fix prompt". |

## `Manage backlog` — risk: medium

Arrays (`acceptance_criteria`, `steps_to_reproduce`, ...) replace the whole list: read the item
first and send the full list. An item vNext holds frozen refuses changes (`reason: locked`):
recall it first. Its status follows vNext while vNext delivers it. Nothing here deletes.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `upsert_feature` | `feature_id?`, `name`, `description?`, `status?` | Create (no `feature_id`) or update a feature. |
| `upsert_user_story` | `story_id?`, `feature_id?`, `title?`, `body?`, `context?`, `acceptance_criteria?`, `out_of_scope?`, `affected_areas?`, `testing_steps?`, `status?` | Create (title or idea required) or update a story. Changing a priced story's words voids the price. `feature_id: null` takes it off every feature. |
| `upsert_defect` | `defect_id?`, `feature_id?`, `title`, `error_message?`, `steps_to_reproduce?`, `expected?`, `actual?`, `status?`, `archived?` | File (title required) or update a defect. vNext refuses a defect with no steps to reproduce. |
| `upsert_idea` | `idea_id?`, `title?`, `body?`, `feature_id?` (create only), `archived?` | Capture a raw idea while brainstorming, before it is clear enough to be a story, or edit one. |
| `create_backlog_items` | `feature?` (`{feature_id}` or `{name, description?}`), `items[]` (≤25; `kind: 'story'` with story fields, or `kind: 'defect'` with defect fields + `story_ref?`; each with `checklist?`) | **The brainstorm-to-backlog call.** A feature and its stories in one go. Each result is `ok` with its id or the error; a failed item does not undo the rest, so retry only those. |
| `move_backlog_item` | `entity_id`, `column?` (`idea`\|`ready`\|`building`\|`testing`\|`deployed`\|`wont_do`), one of `place_below?` / `place_above?` / `position?` (`top`\|`bottom`) | Drag a card on the board: change its column (sets status; defects map to open/fixing/fixed/wont_fix), its priority, or both. An idea moved out of `idea` becomes a story. A vNext item can be reordered, not moved across columns. |
| `promote_idea` | `idea_id`, `feature_id?`, `status?` | Turn an idea into a user story (title, words, rank, attachments carry over; the idea is archived). Safe to retry. |
| `set_backlog_checklist` | `entity_id`, `items[]` (`{id?, text}`, ≤100) | Write a story's or defect's checklist: the tasks to get it done. Replaces the list; send `id` to keep an item and its tick. A change to the item: voids a vNext price, refused while frozen. |
| `check_backlog_items` | `entity_id`, `item_ids[]`, `done` | Tick checklist items off (or back on) as work lands. Progress only: never voids a price, works while vNext delivers. |

## `Comment on and attach files to backlog items` — risk: low

Off by default: the user ticks it. A comment is not a change to the item: it works on frozen
items and never voids a price. An attachment is item content, so it is refused while vNext
holds the item frozen, and adding one voids a price vNext published. The vNext team can read
comments on items it delivers: never put a secret in one.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `add_backlog_comment` | `entity_id`, `body` (markdown) | Log a decision, progress note or question on an item. Shown with your client's name. |
| `update_backlog_comment` | `entity_id`, `comment_id`, `body` | Rewrite one of your own comments. |
| `delete_backlog_comment` | `entity_id`, `comment_id` | Delete one of your own comments. |
| `attach_backlog_text` | `entity_id`, `file_name`, `content` (≤45,000 chars), `mime_type?` (markdown default; text, csv, json, svg) | Attach a text file you wrote (spec, notes, fixture, SVG mockup) in one call. |
| `request_backlog_upload` | `entity_id`, `mime_type`, `size_bytes?` | Step 1 for a local file (screenshot, PDF, video, zip): a signed PUT URL on `www.founder-guide.com` and a ready `curl`. Needs a shell that can reach that host. |
| `attach_backlog_upload` | `entity_id`, `upload_path`, `file_name` | Step 2: attach the bytes you PUT. For an image, returns `embed_markdown` to place in a story's body. |
| `detach_backlog_attachment` | `entity_id`, `document_id` | Remove an attachment. `get_backlog_item` lists them, each with a 15-minute `download_url`. |

## `Work with vNext` — risk: medium

The vNext flow without money: hand items to vNext, send them for a price, answer vNext's
questions, recall to change. **Approving a price, the agreement, cancelling approved work and
disputes happen only in Founder Guide**, with the owner. So does answering a request for a
secret value. Moves are rate limited per member and per workspace.

Typical loop: `assign_vnext_items` (`to: vnext`) → `submit_vnext_items` → vNext prices it
(`awaiting_approval`, the owner approves in the app) or asks questions (`awaiting_answers`:
`reply_vnext_thread` and/or edit the item, then `resubmit_vnext_items`).

| Tool | Key params | What it does |
|------|-----------|--------------|
| `assign_vnext_items` | `ids`, `to` (`vnext`\|`founder`) | Choose who delivers each item. Nothing is sent to vNext yet. |
| `submit_vnext_items` | `ids`, `confirmed_not_secret_fields?` | Send for a price estimate. Per-id refusals: `blocked`, `payment_required` (owner has no card), `looks_like_secret` (returns `fields`; ask the user, then retry naming every field). |
| `resubmit_vnext_items` | `ids`, `confirmed_not_secret_fields?` | Send back for a price after answering vNext's questions. |
| `recall_vnext_items` | `ids` | Take items back before approval to change them. **Voids any published price.** |
| `start_vnext_thread` | `subject`, `body`, `entity_id?`, `confirmed_not_secret?` | Ask the vNext team a question in the Inbox. |
| `reply_vnext_thread` | `thread_id`, `body`, `confirmed_not_secret?` | Reply in an open conversation. |
| `close_vnext_thread` | `thread_id` | Close a conversation; it takes no more replies. |
| `confirm_vnext_access` | `request_id`, `note?` | Tell vNext an access invite was sent (a `request_access` only, after the user sent it). |
| `decline_vnext_request` | `request_id` | Decline a request or withdraw a send. Cannot be undone; confirm with the user. |

## `Delete entity` — risk: high

| Tool | Key params | What it does |
|------|-----------|--------------|
| `delete_entity` | `entity_id`, `entity_type` (`assumption`\|`question`\|`risk`\|`task`\|`note`) | **Irreversible.** Founder Guide cannot undo it. Only when explicitly authorised, and confirm with the user first. |

## `vNext operator` — vNext team only, risk: medium

Only an admin account can be granted this bundle, and only on a workspace with the vNext
add-on. Founders never see it. These tools carry **metadata and question text only**: no tool
reveals, sends or accepts a Secure Handoff value. A value is revealed in `/admin/vnext`, in a
browser, never in a tool result. Start with `list_vnext_workspaces` instead of
`list_workspaces`: the operator is not a member of these workspaces.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `list_vnext_workspaces` | `limit?` | Workspaces with the vNext add-on: id, name, `enabled_at`. Needs no `knowledge_graph_id`. |
| `request_handoff` | `kind` (`variable`\|`access`), `title`, `description`, `reason`, `instructions`, `variable_name` (variable) or `invite_email` + `access_level` (access), `entity_id?`, `expires_in_days?` (1, 3, 7, 14; default 7) | Ask the workspace for an environment variable or for access. Returns id, status, `expires_at`. The founder answers in Founder Guide. |
| `list_handoffs` | `status?`, `limit?` | The workspace's handoffs: what was asked, status, `has_note`, `has_passphrase`, and when and by which role each step happened. `ready` means "reveal it in /admin/vnext". Never a value, never the confirmation note. |
| `ask_vnext_question` | `subject`, `body`, `entity_id?` | Ask the workspace a question in its Inbox, on the whole workspace or one backlog item. A body that looks like a credential is refused: ask for credentials with `request_handoff`. |
| `list_vnext_threads` | `status?`, `waiting_on?`, `entity_id?`, `limit?` | Inbox questions with their messages. A message flagged as a possible credential comes back as `[hidden: flagged as a possible credential]`. |
| `list_vnext_items` | `state?`, `include_shipped?`, `limit?` (default 100, max 200) | The workspace's backlog items with vNext: id, kind, title, delivery state, when vNext was asked, a pending cancel, release dates. Defaults to items still with the vNext team. Read only. |
| `get_vnext_item` | `entity_id` | One vNext backlog item: what the founder wrote, its estimates and approvals, its delivery dates. Text that looks like a credential comes back hidden: ask for it with `request_handoff`. Read only. |

---

# Link tools

Relationships between things. Need `workspace:write` and the `Edit business profile` bundle.

| Tool | Key params | What it does |
|------|-----------|--------------|
| `link_artifact` | `artifact_id`, `target_type` (`entity`\|`section`\|`assumption`\|`question`\|`risk`), `section_type?`, `entity_id?` | Attach a document as evidence to a section or validation item. |
| `link_document_target` | `document_id`, `target_type` (`task`\|`entity`\|`kg_section`\|`milestone_block`), `target_id` | Link a document to a target. Idempotent — re-linking returns the existing relationship. |
| `unlink_document_target` | `document_id`, `target_type`, `target_id` | Remove the link. The document stays in the workspace. |

---

# Interactive cards

Two bundles render a card in the Founder Guide UI instead of returning data. The user acts
inside the card; you do not write on their behalf. Each bundle grants the render tool **and**
the callbacks its buttons fire, because granting the render tool alone produces a card whose
every button fails.

| Bundle | Render tool | What it does |
|--------|-------------|--------------|
| `Field suggestions card` | `suggestions_open` | Shows suggested profile edits the user accepts or rejects one at a time. **Nothing is written until they accept** — this is the right tool when they asked for suggestions rather than changes. |
| `Contact card` | `contact_card_open` | Shows one person or organization as a card they can correct in place (title/segment; industry/website/HQ; email/phone/notes). Only the fields they save are written. |

---

# Not available over MCP (do not reach for these)

Deliberately withheld — they will not appear in `tools/list`, and calling them returns
`ACCESS_DENIED` rather than working.

- **Stateful interview flows** — `start_idea_evaluation`, `evaluate_interview_answer`,
  `generate_interview_report`, `get_interview_progress`, `start_demand_case_interview`,
  `evaluate_demand_answer`. Long-lived sessions that do not fit a stateless MCP call. The
  demand case those sessions produce **is** reachable — see *Draft demand cases* above.
- **Replying to inbound social** — `reply_to_inbound`. You can read the inbox with
  `get_social_inbox`; you cannot post. A reply goes out publicly, immediately, in the user's
  name, and Founder Guide has no delete to undo it with. Draft the reply in chat and let them
  send it.
- **`web_search`** — costs real money per call and the external path has no wallet gate. Use
  your own host's search if you have one.
- **Issuing and voiding invoices, and recording a payment against an issued one.** There are no
  tools for these at all — they are human-only actions in the app. Draft with `draft_invoice`
  and hand it over.
- **The sales pipeline and fundraising** — opportunities, deals, rounds, investor activity and
  data-room sharing are internal-only for now. `convert_outreach_target` is the one door into
  the pipeline, and it only opens a deal.
- **`provide_field_suggestions`** — relays cards over an in-app SSE channel external clients do
  not have. Use the *Field suggestions card* bundle instead, or just describe the changes.
- **Workspace administration** — deleting or archiving a workspace, sharing and collaborators,
  billing, and auth/account settings are permanently off-limits to all agents. Direct the user
  to the Founder Guide UI.
