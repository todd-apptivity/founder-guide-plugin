# write_entities — the universal batch writer

`write_entities` creates, updates, and deletes multiple knowledge-graph entities in one call,
including hierarchies. Prefer it over many single `create_*` calls whenever you're writing more
than one related entity.

Granted by the **Edit business profile** bundle, and needs `workspace:write`.

## Shape

```jsonc
{
  "knowledge_graph_id": "<from list_workspaces>",
  "mode": "propose",           // DEFAULT. "commit" writes live rows — see below.
  "items": [
    {
      "kind": "journey",       // see allowed kinds below
      "op": "create",          // "create" | "update" | "delete". Defaults to "create".
      "local_id": "j1",        // your temporary id, referenced by LATER items in this call
      "id": "<existing-id>",   // required for update/delete; omit for create
      "payload": { },          // fields for this kind — names vary, see below
      "parent_refs": { }       // e.g. { "journey_id": "j1" }. Keys end in _id.
    }
  ]
}
```

## `propose` vs `commit` — both of them write

This is the one thing to get right, because the names are misleading in two separate ways.

**`propose` is not a dry run.** There is no dry run. It persists — it just persists something
the user still has to accept.

**And `mode` only applies to `op: "create"`.** This is the part that bites:

| `op` | under `mode: "propose"` | under `mode: "commit"` |
|------|-------------------------|------------------------|
| `create` | writes a row with `is_suggested: true` — a review card | writes a live row |
| `update` | **writes live. Immediately.** | writes live |
| `delete` | **deletes. Immediately. Irreversibly.** | deletes |

So:

- **`mode: "propose"` does not make an update or a delete safe to try.** There is no such thing
  as a proposed deletion here. Get explicit confirmation for every `update` and `delete`
  regardless of the mode you are passing, exactly as you would for `delete_entity`.
- **Never "preflight with `propose`, then re-send with `commit`."** For creates that writes
  everything twice — once as a card, once live. For updates and deletes it applies them twice.
- **`mode` defaults to `propose`.** Omit it on a batch of creates and you have written review
  cards, not live rows. Do not tell the user their journey was created when it is sitting in a
  queue — read `data.mode` back and say which happened. Omit it on a batch of deletes and
  the default protects nothing.
- Use `propose` for creates whenever you would otherwise type "what about adding X?" — emit the
  proposal instead of the sentence. Use `commit` only when the user explicitly authorised it
  ("do it", "just add it", "yes do that").

The response splits along the same line: only proposed *creates* land in `data.proposals`.
Updates and deletes always land in `data.committed`, whatever mode you asked for.

## Ordering and `local_id`

`local_id` lets a child reference a parent created in the same call, before that parent has a
real id. **Parent-before-child ordering is mandatory**, not an alternative to `local_id`: refs
resolve against items already processed in this batch, so a child listed first passes its
parent's `local_id` through as a literal id and fails with a not-found error.

`local_id` values are yours and local to the call — `"j1"`, `"stage_discover"`, anything.

**The response does not echo `local_id`.** `committed` is `[{ id, kind, op }]` and `proposals`
is `[{ id, kind, is_suggested, payload, parent_refs }]` — neither carries your temporary id. If
you need to report a specific new id back to the user, keep batches small enough to correlate
by `kind` and order, or write the parent in its own call first and use the real id.

## Partial failure is normal — the batch is not atomic

Items are written **one at a time, each in its own error boundary.** A batch of ten where item
six fails still wrote items one to five.

Which failures are which matters, because they need opposite responses:

- **Anything a schema can see is rejected wholesale, before a single item is written.** That
  includes a malformed `items` array, an unknown `kind`, a `payload` missing a required field,
  a misspelled payload field, and a wrong `parent_refs` key — all of it, for *any* item, kills
  the whole call with `VALIDATION_ERROR: Invalid parameters`. Read `details` for the Zod issues;
  the message alone does not say which item. **Nothing was written, so fix and re-send the whole
  batch.**
- **Only failures that need the database are per-item**: a parent id that does not exist, a kind
  this connection may not write, a conflict. These are recorded and the loop continues.

So a malformed batch is safe to retry wholesale, and a *half-applied* batch never comes from bad
field names — it comes from bad ids.

When anything failed, the call comes back as an error — but `data.committed` and
`details.committed` list what **did** land. Read them before retrying, or you will double-write
the items that succeeded. Say what partially landed rather than reporting the whole batch lost.

## Allowed kinds over MCP

Only these `kind` values are accepted on the external path (the server injects this
allow-list). Anything else comes back as a per-item error.

**Multi-instance:** `job_to_be_done`, `offer`, `elevator_pitch`, `team_member`

**Persona:** `persona`

**Journey cockpit:** `journey`, `journey_stage`, `journey_step`, `journey_feature`,
`journey_user_story`, `journey_decision`, `journey_friction_note`, `journey_arrow`,
`journey_inbox_item`, `journey_prompt`, `journey_task`, `journey_outcome`, `journey_fence`,
`journey_example`

**Document-analysis insights:** `pattern`, `recommendation`

For single-attribute edits on a *section* (problem statement, customer profile, etc.) use
`update_section_attribute` instead — sections aren't `write_entities` kinds.

## Payload and parent_refs field names

**Do not guess these.** Each `kind` has its own payload schema, and the names are not uniform —
a stage has a `name`, a step has a `title`. Every kind's real schema and its `parent_refs` keys
arrive in the `tools/list` JSON Schema for `write_entities`, and each kind's description states
its refs. Read them there rather than pattern-matching from another kind.

The common journey kinds, as a sanity check:

| Kind | Required payload | `parent_refs` |
|------|------------------|---------------|
| `journey` | `name` | — |
| `journey_stage` | `name` | `journey_id` |
| `journey_step` | `title` | `journey_id`, `stage_id` |
| `journey_feature` | `name` | `journey_step_id` |
| `journey_arrow` | `from_step_id`, `to_step_id` | `journey_id` |

Note `journey_step.parent_refs` is `stage_id`, not `journey_stage_id`. Keys are named after the
column they set, not after the kind they point at.

## Worked example: a journey with two stages and a step

```json
{
  "knowledge_graph_id": "kg_123",
  "mode": "commit",
  "items": [
    {
      "kind": "journey",
      "op": "create",
      "local_id": "j1",
      "payload": { "name": "Buyer onboarding" }
    },
    {
      "kind": "journey_stage",
      "op": "create",
      "local_id": "s1",
      "payload": { "name": "Awareness" },
      "parent_refs": { "journey_id": "j1" }
    },
    {
      "kind": "journey_stage",
      "op": "create",
      "local_id": "s2",
      "payload": { "name": "Evaluation" },
      "parent_refs": { "journey_id": "j1" }
    },
    {
      "kind": "journey_step",
      "op": "create",
      "local_id": "st1",
      "payload": { "title": "Reads a comparison post" },
      "parent_refs": { "journey_id": "j1", "stage_id": "s1" }
    }
  ]
}
```

One call, correctly parented, in an order that resolves. `mode: "commit"` here because the
example assumes the user asked for it — drop it and you get four review cards instead.

## Tips

- **Read before writing.** `list_entities` to check whether the journey already exists,
  `read_journey` to see what is on it. A second "Buyer onboarding" is not an error anyone sees.
- Keep batches focused. Not because a big one is rejected — it isn't — but because a partial
  failure in a big batch is much harder to reconcile.
- Building a whole journey? One call beats node-by-node by a lot of round-trips.
- Deletes (`op: "delete"`) are irreversible and **ignore `mode`** — confirm with the user first,
  same as `delete_entity`. Same for `update`.
