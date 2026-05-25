---
name: cerver
description: Use this skill whenever you need shared memory across agent runs (recall what you or sibling agents did before), a sandboxed compute to run code on, or a secret/API key. Cerver is one API for all three — every session is also memory; computes are pluggable; secrets are pluggable.
---

# Cerver

Cerver gives agents three things on one API:

1. **Memory** — every session keeps its full transcript; sibling agents on the same account can read it.
2. **Compute** — sandboxes (local relay, Vercel, e2b) you can spawn code on. *Optional* per session.
3. **Secrets** — uniform `secret_fetch(name)` over your chosen backend (env, Infisical, …).

Safety promise: Cerver should not store model/provider secrets. OpenAI, Anthropic, xAI, Vercel, and E2B secrets live in Infisical. Cerver gets scoped runtime access and reports which provider was used without printing the secret.

### Session model

A session has three independent axes:

- **transcript** — always on (this is what cerver *is for*)
- **harness** — which LLM/CLI drives the conversation: `claude` | `codex` | `grok` | `anthropic` | `openai` | `xai`
- **compute** — where work runs: `e2b` | `vercel` | `cloudflare` | `<registered_compute_id>` | `none`

`compute = none` (a.k.a. `session_type:"transcript"`) means cerver is just a transcript inbox — the caller drives the LLM themselves and POSTs turns back. Used by chat surfaces that don't need a sandbox.

Use the current session-create shape:

```json
{
  "session_name": "short-name",
  "task": "what should be done",
  "harness": "claude",
  "compute": { "compute_id": "comp_..." },
  "metadata": {
    "cli_tool": "claude",
    "bootstrap_prompt": "the first prompt to run",
    "working_dir": "/path/to/repo"
  }
}
```

Use `compute: { "provider": "vercel" }` or `{ "provider": "e2b" }` to provision cloud compute. Use `compute: null` for transcript-only sessions.

Auth: every HTTP call carries `Authorization: Bearer $CERVER_API_TOKEN`.
Base URL: `https://gateway.cerver.ai`.

## When to invoke this skill

- User asks "what did the cron do yesterday" / "what did agent X decide last week" → list + peek sessions
- You need an API key, token, or credential → `secret_fetch(name)` first; env fallback only if no MCP
- You need to run shell / node / python in isolation → POST /v2/sessions, then /run
- User says "continue our last conversation" → resume an idle session (same id, append-only)

## Your tools

If `cerver-mcp` is configured (look for `mcp__cerver__*` in your tool list):

- `cerver_session_list({ status?, limit? })` — discover sessions on the account
- `cerver_session_peek({ session_id, last_n? })` — read last N turns of one
- `cerver_session_export({ session_id })` — full transcript as text
- `secret_fetch({ name })` — fetch a secret via the configured backend (env or infisical)

Without MCP, use plain HTTP — same data, more boilerplate:

```
GET  /v2/sessions?limit=20                  → list session summaries
GET  /v2/sessions/:id                       → bounded session summary, no transcript by default
GET  /v2/sessions/:id?tail=50               → summary + last 50 transcript entries
GET  /v2/sessions/:id?since=N               → summary + transcript entries after cursor N
GET  /v2/sessions/:id?full=1                → intentional full transcript download
POST /v2/sessions  { task, workload, requirements }
POST /v2/sessions/:id/run         { code }
POST /v2/sessions/:id/run/stream  { code }  → SSE
POST /v2/sessions/:id/run-llm     { model?, input }
POST /v2/sessions/:id/input       { content, role }
POST /v2/sessions/:id/transcript  { entries: [{ role, content, kind?, at? }] }
POST /v2/sessions/:id/compute     { compute: { provider } | { compute_id } | null }
POST /v2/sessions/:id/switch-tool { cli_tool, harness?, compute?, content? }
POST /v2/sessions/:id/resume                → re-attach compute, status → running
DELETE /v2/sessions/:id                     → terminate
```

### Environments (apps → envs → repos)

Sessions can target a specific `app_slug` + `environment_slug` so the
sandbox provisioner knows which Infisical vault and which repos to
clone. Manage envs via the `cerver envs` CLI verb (preferred) or the
HTTP endpoints below.

CLI (preferred):

```
cerver envs                                              # list all envs across all apps
cerver envs --app SLUG                                   # filter to one app
cerver envs create --app SLUG --slug prod [--default] [--infisical ifc_...]
cerver envs update --app SLUG --env prod [--name N] [--default true|false] [--infisical ifc_…|none]
cerver envs delete --app SLUG --env prod                 # archive
cerver envs repos --app SLUG --env prod                  # list repos in an env
cerver envs repos add --app SLUG --env prod --url URL [--ref REF] [--primary]
cerver envs repos rm  --app SLUG --env prod --repo-id rep_...
```

HTTP equivalents:

```
GET    /v2/apps/:slug/environments
POST   /v2/apps/:slug/environments                       { slug, name?, is_default?, infisical_config_id? }
PATCH  /v2/apps/:slug/environments/:envSlug              { name?, is_default?, infisical_config_id? }
DELETE /v2/apps/:slug/environments/:envSlug              → archives
GET    /v2/apps/:slug/environments/:envSlug/repos
POST   /v2/apps/:slug/environments/:envSlug/repos        { repo_url, repo_ref?, is_primary? }
DELETE /v2/apps/:slug/environments/:envSlug/repos/:repoId
```

## Status enum

Three values: `running` | `ready` | `ended`.

- `running` — agent process is up
- `ready` — no agent process, but session is resumable (send another input to keep going)
- `ended` — explicitly closed; will not resume

If a session ended due to failure, check `endReason` for the cause. Sessions with no activity for >14 days auto-promote to `ended` with `endReason: "stale"` at the API boundary; the underlying record is unchanged, so a future resume still works.

> Migration note: pre-rename clients may still see the legacy value `idle` for sessions whose records haven't been re-read since the rename. Treat `idle` as a synonym for `ready` until the value disappears from your traffic.

## Common patterns

### Recall what happened

```
1. cerver_session_list({ limit: 20 })
   → scan names + statuses
2. Pick relevant by name + recency
3. cerver_session_peek({ session_id, last_n: 10 }) for cheap context,
   or cerver_session_export({ session_id }) for full history
4. Synthesize before answering the user
```

### Fetch a secret

```
1. secret_fetch({ name: "BUFFER_API_KEY" }) → { name, value, source }
2. Use value in your API call
3. NEVER log or echo the value
```

### Configure provider secrets

Use Infisical first. Do not ask the user to paste provider keys into chat unless they explicitly request it.

Expected names:

- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `XAI_API_KEY` or `GROK_API_KEY`
- `VERCEL_TOKEN`
- `E2B_API_KEY`
- `CLOUDFLARE_API_TOKEN`

When the installer has been run, prefer the guided CLI:

```bash
~/.cerver/bin/cerver-onboard
```

If the user explicitly asks you to set a secret and the Infisical CLI is logged in:

```bash
infisical secrets set OPENAI_API_KEY="VALUE" --projectId "$INFISICAL_PROJECT_ID" --env prod --path /
```

Never print the secret. If a key was pasted in chat, remind the user to rotate it when convenient.

### Run code in a sandbox

```
1. POST /v2/sessions
   {
     task: "Boot a preview and run the smoke test",
     workload: "preview",
     requirements: { runtime: "node", package_install: true, timeout_minutes: 15 }
   }
2. POST /v2/sessions/:id/run/stream  { code: "npm test" }   ← SSE
3. DELETE /v2/sessions/:id           ← when done
```

### Run the same prompt on different tools

Use this when the user asks to compare Claude Code vs Codex CLI, or wants to know which tool/model gives the best result for the same intent.

1. List computes and choose a ready local relay that advertises both tools:

```bash
curl https://gateway.cerver.ai/v2/computes \
  -H "Authorization: Bearer $CERVER_API_TOKEN"
```

Look for `provider:"cerver_local_provider"`, `status:"ready"`, and `capabilities.cli_tools` containing `claude` and `codex`.

2. Create sibling sessions with the same `task` and `metadata.bootstrap_prompt`, changing only `harness` and `metadata.cli_tool`:

```bash
curl -X POST https://gateway.cerver.ai/v2/sessions \
  -H "Authorization: Bearer $CERVER_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "session_name": "same-intent-claude-cli",
    "task": "Explain the tradeoff in 5 bullets",
    "harness": "claude",
    "compute": { "compute_id": "comp_..." },
    "metadata": {
      "cli_tool": "claude",
      "bootstrap_prompt": "Explain the tradeoff in 5 bullets",
      "comparison_group": "same-intent-demo"
    }
  }'

curl -X POST https://gateway.cerver.ai/v2/sessions \
  -H "Authorization: Bearer $CERVER_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "session_name": "same-intent-codex-cli",
    "task": "Explain the tradeoff in 5 bullets",
    "harness": "codex",
    "compute": { "compute_id": "comp_..." },
    "metadata": {
      "cli_tool": "codex",
      "bootstrap_prompt": "Explain the tradeoff in 5 bullets",
      "comparison_group": "same-intent-demo"
    }
  }'
```

3. Automatically fetch both sessions. Do not stop after creating them.

```bash
curl https://gateway.cerver.ai/v2/sessions/CLAUDE_SESSION_ID?tail=50 \
  -H "Authorization: Bearer $CERVER_API_TOKEN"

curl https://gateway.cerver.ai/v2/sessions/CODEX_SESSION_ID?tail=50 \
  -H "Authorization: Bearer $CERVER_API_TOKEN"
```

4. If the sessions are still running or have no assistant answer yet, wait briefly and fetch again. If they stay empty, report that the run started but no transcript is ready.

5. Compare the transcripts yourself and give the user a small decision table:

| Dimension | Claude Code | Codex CLI | Better |
|---|---|---|---|
| Direct answer quality | short assessment | short assessment | winner |
| Technical accuracy | short assessment | short assessment | winner |
| Work trace / tool use | short assessment | short assessment | winner |
| Practicality | short assessment | short assessment | winner |
| Cost / latency, if available | value or "not returned" | value or "not returned" | winner |

Then write:

- **Best for product copy:** `<tool>` because `<reason>`
- **Best for technical truth:** `<tool>` because `<reason>`
- **Overall winner for this intent:** `<tool or tie>`
- **Recommended next action:** `<ship / ask a follow-up / run tests / create a third session>`

Do not hide failed runs. If one tool errors, include the error in the comparison and explain whether it is a tool failure, compute failure, missing secret, or prompt issue.

This is different from switching compute. Here the compute provider can stay the same while the tool layer changes from Claude Code to Codex CLI.

### Hold a transcript without compute

For chat surfaces where work runs *outside* cerver — caller drives the LLM
themselves and just wants cerver to remember the conversation.

```
1. POST /v2/sessions
   {
     session_type: "transcript",
     session_name: "company-chat",
     harness: "claude"
   }
2. (caller hits Anthropic / OpenAI / xAI directly, gets response)
3. POST /v2/sessions/:id/transcript
   {
     entries: [
       { at, role: "user",      kind: "text", content: "hello" },
       { at, role: "assistant", kind: "text", content: "hi back" }
     ]
   }
```

Returned session has `provider:"none"`, `compute_id:null`, `sandbox_id:null`,
plus the chosen `harness`. No /run, no /resume — there's no agent process.

### Continue a paused conversation

```
1. cerver_session_list({ status: "ready" })  → find the right one
2. POST /v2/sessions/:id/resume     → same id, status → running, transcript intact
3. POST /v2/sessions/:id/input      { content: "follow-up question", role: "user" }
```

## Don't

- Don't bake secrets into your output, commits, or chat. Always fetch fresh.
- Don't create a new session when you could resume an idle one — sessions ARE memory.
- Don't poll for output; use `/run/stream` (SSE).
- Don't walk `previousSessionId` chains — sessions are append-only since the resume rewrite. One session id = one full conversation.
- Don't confuse AI/tool providers with compute providers. `harness`/`metadata.cli_tool` is the intelligence/tool layer; `compute` is where it runs.
- To continue one session with another CLI, use `/v2/sessions/:id/switch-tool` instead of creating a sibling. Cerver appends a tool-switch event, rehydrates the transcript as context, and starts the new CLI on the selected compute.

## Helpful URLs

- Full API reference: <https://cerver.ai/dashboard/api>
- Live session dashboard (humans): <https://cerver.ai/dashboard/sessions>
- Cross-agent memory docs: <https://cerver.ai/llms.txt>
