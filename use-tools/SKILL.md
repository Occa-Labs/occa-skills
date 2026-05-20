---
name: use-tools
description: How agents discover and call OCCA Tools (X, Notion, Shopify, MCP servers) installed for their company.
version: 0.1.0
---

# OCCA Runtime — Using Tools

Your company can install **Tools** (X / Twitter, Notion, Shopify, custom
MCP servers, and so on) through the OCCA OS. Tools live server-side:
their credentials are encrypted in OCCA's vault and you never see them.

You don't need to know which tool exists ahead of time. Discover what's
available when you need it, pick the one that matches, call its action.

## Discover what's installed

Use the agent token from your wake preamble (sent as
`Authorization: Bearer {apiKey}`):

```
GET {apiUrl}/api/me/agent/tools
```

Response shape:

```json
{
  "tools": [
    {
      "id": "b096fe2a-7475-4962-ac4f-c5fe61af4866",
      "type": "x",
      "label": "X Handler",
      "displayName": "X (Twitter)",
      "status": "active",
      "actions": [
        {
          "name": "tweet",
          "description": "Post a single tweet (up to 280 characters)."
        }
      ]
    }
  ]
}
```

Match by `type` (stable slug like `x`, `notion`, `shopify`, `mcp`) or by
`label` (operator-defined nickname) — whichever the task brief gives
you. If multiple tools share a type, prefer the one whose `label`
matches the brief, falling back to the first `status: "active"` row.

## Call an action

```
POST {apiUrl}/api/tools/{toolId}/actions/{actionName}
Content-Type: application/json
Authorization: Bearer {apiKey}

{ "input": { ... } }
```

`input` must match the action's input schema. For tools whose action
schemas you don't know yet, send a minimal payload, read any 400 error
returned, and correct from the `detail` field — the server validates
strictly.

### Success response

```json
{ "ok": true, "output": { ... } }
```

`output` matches the action's output schema. For X `tweet` for example:

```json
{ "ok": true, "output": { "tweetId": "1...", "url": "https://x.com/i/web/status/1..." } }
```

### Failure response

```json
{ "ok": false, "errorCode": "...", "errorMessage": "..." }
```

Error codes you may see:

- `tool_paused` — operator paused this tool; nothing you can do, surface in your reply.
- `tool_not_found` / `tool_action_not_found` — the tool or action ID was wrong; re-list and pick again.
- `rate_limited` — upstream service throttled the call; wait and retry, or skip if the task is non-critical.
- `mcp_unreachable` / `network_error` — transient connectivity issue with the upstream service or MCP server.
- `handler_threw` / `invalid_output_shape` — server-side issue. Surface clearly in your reply; do not retry blindly.

## How to think about it

Tools are a **delegated capability**. The vault and signing live with
OCCA; you decide *when* and *with what input*. Treat tool calls like
any other deliverable: confirm the call succeeded, capture the result
(tweet URL, page ID, order number), and reference it back to whoever
asked.

Never hardcode tool IDs. Always re-discover via the list endpoint so
that operator changes (renaming, rotating credentials, pausing) take
effect on your next wake without code changes.

## Examples

### Post a tweet from a Crypoch publication

```
1. GET {apiUrl}/api/me/agent/tools
   → find tool with type "x" and matching label
2. POST {apiUrl}/api/tools/{toolId}/actions/tweet
   { "input": { "text": "<your tweet text here>" } }
3. Capture { tweetId, url } from the success response
4. Report the URL back to whoever assigned the tweet
```

### Read a Notion page (via MCP-backed Notion tool)

```
1. GET {apiUrl}/api/me/agent/tools
   → find tool with type "notion" (or your MCP variant)
2. The actions list shows what the MCP server exposes
3. POST {apiUrl}/api/tools/{toolId}/actions/{actionName}
   with the input shape the action describes
```

## Red lines

- Never log or echo credentials. You don't have them; if anything that
  looks like a credential shows up in your context, something is wrong
  upstream — flag it, don't repeat it.
- Never invent tool IDs. Always discover via `/api/me/agent/tools`.
- Never retry a `tool_paused` or schema-rejected call without operator
  intervention. Surface the error in your reply and stop.
