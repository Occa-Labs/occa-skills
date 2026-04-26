---
name: OCCA Runtime — Agent Protocol
description: How agents call back into OCCA — approvals, hiring, callbacks.
version: 0.1.0
---

# OCCA Runtime — Agent Protocol

You are running inside OCCA OS. This skill defines how you talk back to
the OCCA server when you need to do things that require a structured
channel (most importantly: hiring another agent).

## Your runtime credentials

Every wake message ends with a preamble like:

```
OCCA runtime:
  apiUrl: <base>
  apiKey: <bearer token>
  agentId: <your external agent id>
  traceId: <current trace>
```

`apiKey` is short-lived (one trace) and authenticates you as **this
agent** to the OCCA server. Send it as `Authorization: Bearer {apiKey}`
on every call below.

## Two ways to grow capacity

When work is too big or out of your scope, you have two options:

1. **DELEGATE** — assign work to an existing agent in your subtree.
   Use this when someone on your team can already do the job.
2. **HIRE** — bring an entirely new agent onto the team.
   Use this when no one on your team has the right role for the work.

Both go through the **same approval gate** — you submit a request, the
human owner approves or rejects, and only after approval does the work
get created and dispatched. You **never** create tasks or agents
directly; the contract is "request, exit, get woken when done."

## How to submit a request — primary path: BLOCK MARKERS

Embed a marker block anywhere in your reply text. The server parses the
JSON body, validates it, and creates the approval row for you. This is
the recommended channel — it doesn't need outbound HTTP, just text.

### HIRE — bring on a new agent

```
[[OCCA:HIRE]]
{
  "targetRole": "<one of: ceo, cto, cmo, eng, researcher>",
  "targetName": "<a name for the new agent, e.g. Aria, Bolt>",
  "title": "<short task title for their first assignment>",
  "description": "<full detail of what they should do>",
  "acceptanceCriteria": "<optional: what 'done' looks like>"
}
[[/OCCA:HIRE]]
```

**Role catalog** — `targetRole` must be one of:
`ceo`, `cto`, `cmo`, `eng`, `researcher`. Each role has its
own pre-built workspace template and skill set.

When the human approves, the server:
1. Creates a new agent on the OpenClaw gateway (~10–30s provisioning).
2. Seeds their workspace with role-specific markdown templates.
3. Auto-assigns role-eligible skills (installs run async in background).
4. Creates a task assigned to them with the title/description above.
5. Dispatches the task immediately — they start working with default
   tools while domain skills finish installing in parallel.

Their `parentAgentId` is set to **you**, so they become your direct
report. On future turns you can DELEGATE further work to them.

### DELEGATE — assign to an existing agent in your subtree

```
[[OCCA:DELEGATE]]
{
  "targetAgentId": "<uuid from 'Available reports' in the wake preamble>",
  "title": "<short task title>",
  "description": "<full detail>",
  "acceptanceCriteria": "<optional>"
}
[[/OCCA:DELEGATE]]
```

You may delegate to any agent in your subtree — direct reports and
their reports recursively. The wake preamble lists this set under
"Available reports" with names, roles, and IDs. Picking an agent
outside that list is rejected.

If "Available reports" is empty, you have no team to delegate to —
use HIRE instead.

### Rules for marker blocks

- Emit **at most ONE** marker block per turn.
- The block can sit anywhere in your reply (start, middle, end).
- The server strips the block from the persisted reply, so end users
  never see the raw marker.
- Markdown code-fences around the JSON are tolerated:
  ` ```json … ``` ` inside the block parses fine.

## How to submit a request — alternative path: HTTP API

If you have outbound HTTP capability and prefer a synchronous response,
you may POST directly. The marker channel above is preferred because it
works in environments where outbound HTTP isn't reliable, but the API
path is supported for completeness.

`POST {apiUrl}/api/agents/me/approvals`

```json
{
  "actionType": "hire",
  "payload": { /* same shape as the HIRE marker body */ }
}
```

Send your `apiKey` (from the wake preamble) as
`Authorization: Bearer {apiKey}`. The response is
`{ "approvalId": "<uuid>" }`. The downstream behaviour is identical to
the marker path.

## After requesting

Submitting an approval (either channel) is a complete action for this
turn. Finish your reply normally and exit. You will be woken when the
spawned task progresses, or when the human rejects and you need to
choose a different path.

## Common mistakes to avoid

- **Don't include yourself as `targetAgentId`.** Self-delegate is rejected.
- **Don't repeat a request** if you've already submitted one for the same
  work — the human sees a queue, duplicates create noise.
- **Don't try to call /api/tasks or /api/agents directly.** Agents cannot
  create tasks or agents; both go through the approval gate.
- **Don't fabricate `targetAgentId`.** Use the IDs in the wake preamble.
- **Don't fabricate `targetRole`.** Use a value from the catalog above.
- **Don't emit unclosed markers.** Always close `[[OCCA:HIRE]]` with
  `[[/OCCA:HIRE]]`. Same for DELEGATE.

This skill is platform-shipped and lives on every agent. The contract
above is part of OCCA's stable runtime API — it does not change without
a migration.
