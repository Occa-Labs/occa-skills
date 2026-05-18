---
name: verifiable-claims
description: Required output format for any story, research note, or report that cites numbers (TVL, trading volume, price, market cap, fees, percentages). Use whenever writing a document that contains quantitative claims. Defines the <!--occa:claims--> block that the OCCA verification gate re-fetches and diffs against live sources before the document is accepted for publication.
source: OCCA (internal)
installed: 2026-05-18
---

# Verifiable claims

Every document you publish that contains numbers passes through an
automated verification gate. The gate re-fetches each number from its
source and compares it to what you wrote. It is plain code, not a
reviewer you can persuade. If a number does not match its source, the
document is rejected and sent back to you with the exact discrepancy.

This skill defines the structured `<!--occa:claims-->` block the gate
reads. A document with quantitative claims and no valid block is
rejected before it is even checked.

## The non-negotiable workflow

Do these in order. Do not reorder them.

1. **Gather.** Call your data tools. Read the actual tool results.
2. **Record.** Build the `<!--occa:claims-->` block now, copying each
   value directly out of the tool result you just read.
3. **Write.** Write the prose story *from the block*. Every number in
   the prose must already exist as a row in the block.

You do not pick a headline and then look for numbers to support it. You
record what the data says, then write what the data shows. If the data
does not support the story you imagined, the story changes — not the
numbers.

## The hard rules

- **Every value is copied verbatim from a tool result.** Never type a
  number from memory, never round it, never estimate it. If you cannot
  point to the exact figure in a tool result, you do not have it.
- **No retrieval, no claim.** If you did not fetch a number with a tool,
  it cannot appear in the story. There is no such thing as a number you
  "know."
- **Every number in the prose has a matching claim row.** A figure in
  the body with no row in the block fails the gate.
- **`as_of` is the envelope date.** Your task prompt begins with a
  header like `[Mon 2026-05-18 07:22 UTC]`. That bracket is today's
  date. Use it. Never derive a date from a Unix timestamp, a data
  point, or anything inside a payload — you will get it wrong.

## Block format

Place one HTML comment at the very end of the document. It stays
invisible in the rendered article.

```
<!--occa:claims
{
  "as_of": "2026-05-18",
  "claims": [ ... ]
}
-->
```

`as_of` is `YYYY-MM-DD`, taken from the envelope header. Each entry in
`claims` is one of two kinds.

### DefiLlama claim

A number read straight from a DefiLlama endpoint.

```json
{
  "source": "defillama",
  "id": "meteora_dlmm_7d",
  "label": "Meteora DLMM 7d volume",
  "endpoint": "/overview/dexs/solana",
  "selector": "protocols[name=Meteora DLMM].total7d",
  "value": 963681427
}
```

- `id` — short identifier, letters/digits/underscore only (no spaces, no `-`).
- `label` — human description of the metric.
- `endpoint` — the DefiLlama REST path (see the map below).
- `selector` — path into the JSON payload to a single number (see below).
- `value` — the exact number from the tool result.
- `abs_tolerance` — add this for percentage / change fields, e.g.
  `"abs_tolerance": 2`. Omit it for absolute dollar amounts; the gate
  allows 5% drift on those by default.

### Calculated claim

A number you computed from other claims (only `+` and `-` are allowed).

```json
{
  "source": "calculated",
  "id": "meteora_combined_7d",
  "label": "Meteora combined 7d volume",
  "expression": "meteora_dlmm_7d + meteora_damm_v2_7d + meteora_dbc_7d",
  "value": 1087656591
}
```

`expression` references other claim `id`s. The gate recomputes it from
the verified inputs, so every input must be its own claim row first.

## Endpoint map

Your DefiLlama MCP tool calls correspond to these REST paths. Put the
REST path in `endpoint`, not the tool name.

| MCP tool you called | `endpoint` value |
|---|---|
| `defillama__dex_overview` (chain `solana`) | `/overview/dexs/solana` |
| `defillama__fees_overview` (chain `solana`) | `/overview/fees/solana` |
| `defillama__protocol_summary` (metric `dexs`, protocol `X`) | `/summary/dexs/X` |
| `defillama__protocol_summary` (metric `fees`, protocol `X`) | `/summary/fees/X` |
| `defillama__chain_tvl` (chain `Solana`) | `/v2/historicalChainTvl/Solana` |

## Selector syntax

A selector walks the JSON payload to one number.

- `total7d` — a top-level field. Use for a `protocol_summary` claim.
- `change_7dover7d` — a top-level field (week-over-week % change).
- `protocols[name=Meteora DLMM].total7d` — find the array element whose
  `name` is exactly `Meteora DLMM`, then read `total7d`. Use for a claim
  sourced from `dex_overview` / `fees_overview`. The name must match
  exactly, including capitalization.
- `[last].tvl` — the last element of a root-level array (the historical
  TVL endpoint returns a bare array), then read `tvl`.

Which tool for which number:

- Week-over-week change (`change_7dover7d`) — take it from
  `dex_overview` / `fees_overview` (the `protocols[...]` selector). A
  `protocol_summary` does not always carry it.
- A single protocol's absolute volume or fees — `protocol_summary` is
  smallest; `dex_overview` / `fees_overview` also carry it.

## Worked example

A story reporting Meteora DLMM's weekly volume and its week-over-week
change ends with:

```
<!--occa:claims
{
  "as_of": "2026-05-18",
  "claims": [
    {
      "source": "defillama",
      "id": "dlmm_7d",
      "label": "Meteora DLMM 7d volume",
      "endpoint": "/overview/dexs/solana",
      "selector": "protocols[name=Meteora DLMM].total7d",
      "value": 963681427
    },
    {
      "source": "defillama",
      "id": "dlmm_change",
      "label": "Meteora DLMM week-over-week change",
      "endpoint": "/overview/dexs/solana",
      "selector": "protocols[name=Meteora DLMM].change_7dover7d",
      "value": -2.91,
      "abs_tolerance": 2
    }
  ]
}
-->
```

Note that `value` for `dlmm_change` is `-2.91` because that is what the
`change_7dover7d` field returned. If the data shows a decline, the story
reports a decline. You do not write `+53` because a rise reads better.

## When the gate rejects your document

You receive a list of mismatches: each says the claim id, the value you
wrote, and the value the source actually has. Fix the document by
correcting the numbers to match the source — and correct the prose
around them, since a wrong number usually means the story's framing was
wrong too. Resubmit. Do not argue with the numbers.
