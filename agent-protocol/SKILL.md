---
name: agent-protocol
description: How agents call back into OCCA — approvals, delegation, callbacks.
version: 0.3.0
---

# OCCA Runtime — Agent Protocol

You are running inside OCCA OS. This skill defines how you talk back to
the OCCA server when you need to do things that require a structured
channel (most importantly: delegating work to a teammate).

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

## Two block-marker actions you can emit

When you can't finish a task solo, your reply can include ONE marker
block from the two below. The server parses the JSON body, acts on it,
and strips the marker before saving your reply.

1. **DELEGATE** — hand a sub-task to an existing teammate (someone in
   "Available reports" of your wake message).
2. **BLOCK** — declare you can't proceed until other tasks complete.

DELEGATE goes through the **approval gate** — the human approves or
rejects before any spawn happens. BLOCK takes effect immediately (no
approval needed): it changes your task's status to `blocked`.

You **never** create tasks or agents directly; the contract is
"emit marker, exit, get woken when something changes." Adding new
agents to the team is a user-driven action — you cannot request it.

## How to submit a request — primary path: BLOCK MARKERS

Embed a marker block anywhere in your reply text. The server parses the
JSON body, validates it, and creates the approval row for you. This is
the recommended channel — it doesn't need outbound HTTP, just text.

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
finish what you can yourself and flag the gap to the human in your
reply text.

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

### Rules for marker blocks

- Emit **at most ONE** marker block per turn.
- The block can sit anywhere in your reply (start, middle, end).
- The server strips the block from the persisted reply, so end users
  never see the raw marker.
- Markdown code-fences around the JSON are tolerated:
  ` ```json … ``` ` inside the block parses fine.

## Mid-task clarifications — RequestInfo (not a marker)

For mid-task questions to the human, do **not** emit a marker — POST
to the typed-action HTTP back-channel:

`POST {apiUrl}/api/agents/me/actions/emit`

```json
{
  "type": "RequestInfo",
  "question": "<what you need to know>"
}
```

The server posts your question as a comment on the task AND parks the
task in `review` so the human is unblocked of the kanban card. You will
be re-woken when the human replies via the comment thread.

## How to submit a delegate request — alternative path: HTTP API

If you have outbound HTTP capability and prefer a synchronous response,
you may POST directly. The marker channel above is preferred because it
works in environments where outbound HTTP isn't reliable, but the API
path is supported for completeness.

`POST {apiUrl}/api/agents/me/approvals`

```json
{
  "actionType": "delegate",
  "payload": {
    /* same shape as the DELEGATE marker body */
  }
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

## Verify before refusing

Before claiming a capability is missing — "no integration available",
"no tool for this", "I need credentials", "I can't do that here" — you
MUST verify against what is actually wired up to you on this wake:

1. Scan the assigned-skills block above. Each skill's full content is
   inlined. If any skill's name or description matches the kind of work
   the task is asking for, read that skill and follow it.
2. If a skill instructs you to call a discovery endpoint to enumerate
   capabilities, call it before refusing. Anything operator-managed
   (credentials, third-party integrations, custom servers) is only
   knowable by asking the server at runtime.
3. Only when both checks come up empty do you have grounds to surface
   a missing-capability blocker.

Refusing without verifying is a process failure. The cost of an extra
discovery call is trivial; the cost of a wrong "I can't do this" is a
manual task return-for-review and broken trust.

## Reading the company's documents

Every task your team completes is auto-saved as a document. Articles
your News Writer published, briefs your Head Research filed, anything
a teammate produced — it's all in the company document store and you
can read any of it on demand.

This is your shared memory. Use it before asking, before guessing, and
before refusing on "I don't have the source." The document IS the source.

### List recent documents

```
GET {apiUrl}/api/me/agent/documents?limit=25
Authorization: Bearer {apiKey}
```

Optional filter:

```
GET {apiUrl}/api/me/agent/documents?tags=technology,markets&limit=10
```

Response (titles + metadata only — content is fetched separately):

```json
{
  "documents": [
    {
      "id": "uuid",
      "title": "Vitalik outlines Ethereum's native privacy roadmap",
      "format": "markdown",
      "tags": [],
      "taskId": "uuid",
      "deploymentId": "uuid",
      "createdAt": "2026-05-21T05:19:10Z"
    }
  ]
}
```

### Fetch one document in full

```
GET {apiUrl}/api/me/agent/documents/{id}
Authorization: Bearer {apiKey}
```

Returns the document with full `content`. Format is usually `markdown`.

### When to use it

- You're writing a tweet about an article a teammate just published →
  list documents, find the one whose title matches the brief, fetch
  it, read the actual piece before composing the tweet.
- You're asked to follow up on prior work → list documents and find
  the relevant one instead of guessing.
- Someone references "the piece Juno filed" → look it up.

You always have access. There is no per-document permission inside a
company. If `apiKey` is valid, you can read any document the company
owns.

## Common mistakes to avoid

- **Don't include yourself as `targetAgentId`.** Self-delegate is rejected.
- **Don't repeat a request** if you've already submitted one for the same
  work — the human sees a queue, duplicates create noise.
- **Don't try to call /api/tasks or /api/agents directly.** Agents cannot
  create tasks or agents; delegation goes through the approval gate.
- **Don't fabricate `targetAgentId`.** Use the IDs in the wake preamble.
- **Don't emit unclosed markers.** Always close `[[OCCA:DELEGATE]]` with
  `[[/OCCA:DELEGATE]]`. Same for BLOCK.
- **Don't BLOCK on yourself.** `blockedByTaskIds` cannot include the
  task you're currently working on — that creates a deadlock and is
  rejected.
- **Don't BLOCK on tasks from other companies.** Blocker ids that don't
  exist in your company are silently filtered out; if all are filtered,
  the BLOCK is ignored.

This skill is platform-shipped and lives on every agent. The contract
above is part of OCCA's stable runtime API — it does not change without
a migration.
