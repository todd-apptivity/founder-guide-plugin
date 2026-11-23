# Errors, Permissions & Limits

## Error codes

Tools return structured errors. React to the code, don't just retry blindly.

A failed call comes back as a normal tool result flagged `isError`, whose text is
`{ "error": { "code", "message", "details" } }` — parse it and branch on `code`.

| Code | Meaning | What to do |
|------|---------|-----------|
| `VALIDATION_ERROR` | Bad/missing params or value fails the schema | Read the issue detail; fix the argument (often a wrong `attribute_path` or enum value) and retry once. |
| `ACCESS_DENIED` | User lacks `can_use_ai` here, the token wasn't granted this tool, or the workspace id isn't theirs | **Stop.** Don't retry. Tell the user which capability/workspace is missing. |
| `AUTHENTICATION_ERROR` | Token invalid/expired | The connection needs re-authorizing — direct the user to reconnect. |
| `RATE_LIMIT_EXCEEDED` | Too many calls in the window | Back off, then continue; batch where possible. |
| `ENTITY_NOT_FOUND` / `SECTION_NOT_FOUND` / `ARTIFACT_NOT_FOUND` | The referenced id doesn't exist | Re-read with a list/get tool to get a valid id; don't invent one. |
| `TOOL_NOT_FOUND` | Tool isn't available on this connection | The user didn't grant it — don't substitute a different tool silently. |
| `EXECUTION_ERROR` / `TIMEOUT_ERROR` | Transient server-side failure | Retry once; if it persists, report it plainly. |

## The permission model (why a tool might be missing)

Three independent, server-enforced gates decide whether a tool works. The tool list you
receive already reflects all three — trust it.

1. **`can_use_ai`** — the user must be a workspace **owner or editor**. Viewers get an empty
   tool list and `ACCESS_DENIED` on every call. This is the same rule as Founder Guide's
   in-product agents.
2. **Per-token grants** — at connect time the user picked capability bundles. The token can
   only call the tools in those bundles. `list_workspaces` is always granted (it's the
   discovery entry point).
3. **OAuth scope** — every write/link/delete tool also requires the `workspace:write` scope.
   Read tools don't.

Practical consequence: if you expected a tool and it's absent, the user connected read-only or
didn't grant that bundle. Ask them to turn on the named capability for this connection (Founder
Guide: account menu, then Personal settings, then Integrations, then Permissions under External
connections) and then refresh your tool list. No reconnect is needed. Don't work around it.

## Rate limits

The MCP path is throttled **per token, 60 calls/minute** by default (`MCP_RATE_LIMIT_RPM`), in a
fixed window — unrelated to the in-app path's per-conversation limit of 50. Keep usage
efficient: read once and reuse, and batch writes with `write_entities` instead of many single
calls.

## Document upload limits

- **Supported types:** `application/pdf`, DOCX, `text/plain`, `text/csv`, `text/markdown` —
  the file extension must match `mime_type`.
- **Size:** ~3MB decoded per upload (the MCP transport rides a ~4.5MB request-body limit, and
  base64 inflates content by ~33%). The in-app uploader supports up to 50MB — send users there
  for bigger files.
- **Content encoding:** exactly one of `content` (plain text, text types only) or
  `content_base64` (required for pdf/docx).
- **Folders:** slash-separated paths, letters/digits/spaces/`-`/`_` only, ≤500 chars. Tags are
  lowercased and deduped; ≤50 chars each, ≤20 per document.
- **Async:** upload returns immediately with `version_status: "queued"`; ingestion and
  analysis happen in the background. Poll `get_document` — don't assume completion.
- **Bundle:** the write tools (`upload_document`, `update_document_metadata`,
  `trigger_document_analysis`) live in the **Manage documents** bundle; `get_document` and
  `list_documents` are part of the always-granted read bundle. Connections made before this
  bundle existed must be re-consented to gain the write tools.

## Empty states — never fabricate

An empty result is a legitimate answer. No posts, no pitch, no tasks, no journey → tell the
user "there's nothing there yet" and offer to create it. Do **not** invent data to fill a gap,
and don't treat an empty list as an error.

## Destructive actions

`delete_entity`, `op: "delete"` in `write_entities`, `unlink_*`, and publishing content are
hard or impossible to undo. Always confirm with the user before calling them, even if a bundle
technically grants the capability.

**`mode: "propose"` does not make them safe.** It only applies to `op: "create"`; an `update` or
`delete` writes live whichever mode you pass. There is no such thing as a proposed deletion.

Also irreversible in practice, though they don't read that way:

- `record_outreach_sent` copies a message into a permanent log — it records what the founder
  already sent, so never call it speculatively.
- `add_case_evidence` with `action: "revise"` **drops every supporting quote** the founder has
  not explicitly confirmed, landing the field back on `assumed`. Omitting
  `kept_support_conversation_ids` is how evidence gets silently lost.
- Issuing an invoice and `record_payment` create committed ledger entries that feed the forecast. Drafting one does not — a draft is not money.
