---
name: OCCA Runtime — Agent Protocol
description: How agents call back into OCCA — approvals, hiring, callbacks.
version: 0.2.0
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

## Four block-marker actions you can emit

When you can't finish a task solo, your reply can include ONE marker
block from the four below. The server parses the JSON body, acts on it,
and strips the marker before saving your reply.

1. **HIRE** — bring on a brand-new agent (a role nobody on your team has).
2. **DELEGATE** — hand a sub-task to an existing teammate (someone in
   "Available reports" of your wake message).
3. **BLOCK** — declare you can't proceed until other tasks complete.
4. **ASK** — pose a question to a teammate or the human; task parks in
   `review` until the answer comes in.

HIRE and DELEGATE go through the **approval gate** — the human approves
or rejects before any spawn happens. BLOCK and ASK take effect
immediately (no approval needed): BLOCK changes your task's status to
`blocked`, ASK posts a comment that wakes the mentioned agent.

You **never** create tasks or agents directly; the contract is
"emit marker, exit, get woken when something changes."

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

### BLOCK — wait for other tasks to complete

```
[[OCCA:BLOCK]]
{
  "blockedByTaskIds": ["<task uuid>", "..."],
  "reason": "<short why — auto-posted as a comment on this task>"
}
[[/OCCA:BLOCK]]
```

Use this when your task literally cannot proceed because another task
hasn't finished. The server parks your task in `blocked` status with
the listed task ids on `blockedByTaskIds`. When ALL of those blockers
transition to `done`, the L2 cascade flips your task back to `todo` and
re-dispatches you automatically — no manual step needed.

`reason` is optional; if provided, it's posted as a system comment on
your task so the human can see why you're parked.

### ASK — pose a question

```
[[OCCA:ASK]]
{
  "question": "<what you need to know>",
  "mentionAgentId": "<optional uuid from 'Available reports'>"
}
[[/OCCA:ASK]]
```

Posts your question as a comment on the task. If `mentionAgentId` is
provided, the server resolves the agent's name and prefixes the comment
with `@<name>` — that mention wakes the named agent on its assigned
task so they can reply. If you omit `mentionAgentId`, the human is the
implicit recipient.

Your task transitions to `review` while waiting for the response. When
the mentioned agent (or the human) replies via the comment thread, you
may be re-woken to continue.

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
  `[[/OCCA:HIRE]]`. Same for DELEGATE, BLOCK, ASK.
- **Don't BLOCK on yourself.** `blockedByTaskIds` cannot include the
  task you're currently working on — that creates a deadlock and is
  rejected.
- **Don't BLOCK on tasks from other companies.** Blocker ids that don't
  exist in your company are silently filtered out; if all are filtered,
  the BLOCK is ignored.

This skill is platform-shipped and lives on every agent. The contract
above is part of OCCA's stable runtime API — it does not change without
a migration.
