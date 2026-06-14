---
name: crypto-news-domain
description: Domain judgment for writing crypto news and research. Use whenever covering on-chain activity, protocols, tokens, or markets. Defines how to read on-chain metrics (what is signal vs noise), name protocols and chains correctly, source against primary on-chain data, and frame a piece for an analyst audience without hype. Layers on top of the generic journalism (news-writing) skill.
source: OCCA (internal)
installed: 2026-05-19
---

# Crypto news domain

Your journalism skills cover the craft: checking facts, verifying
sources, editing. This skill covers the part that is specific to crypto.
It is judgment, not a fact sheet. Specific numbers and protocol states
come from your data tools at the moment you write; what follows is how
to read them.

This skill is about understanding what the numbers mean before you write
them. `news-writing` covers the craft and the rule that every number is
read directly from a tool result, never typed from memory.

## 1. Reading on-chain metrics

Know what the core metrics measure:

- **TVL** (total value locked) — capital deposited in a protocol or
  chain. A proxy for committed liquidity, not for usage.
- **DEX volume** — value traded through on-chain exchanges. A proxy for
  active engagement.
- **Fees / revenue** — what users paid to use a protocol. The closest
  on-chain proxy for real demand.
- **Market cap** — token price times supply. A market opinion, not an
  on-chain fact. Treat it as the weakest of these.

Signal vs noise. A number is a story only when it is one of:

- **Large** relative to the metric's own recent range.
- **Sustained** across several consecutive observations, not one print.
- **Divergent** from comparable protocols or chains.

Daily TVL and volume routinely vary a few percent in each direction. A
single-day move inside that range is not a lead. A move becomes one when
it persists across three or more observations or breaks a multi-week
range.

A number alone is never news. `$43B TVL` means nothing until you give
the reader a baseline: down from what, flat for how long, an all-time
high or low. Always anchor the figure.

Check a peer before claiming a move is specific to one protocol or
chain. If Ethereum TVL falls, look at Solana over the same window. A
move that every comparable shares is a market story; a move only one
shows is that one's story. Say which it is.

State windows exactly. "Week-over-week" means the same metric measured
seven days apart. Do not silently switch between point-in-time and
trailing-average comparisons.

## 2. Naming protocols, chains, and tokens

Use canonical names and casing: Ethereum, Solana, Bitcoin, Base,
Arbitrum; Uniswap, Aave, Lido. Tickers uppercase: ETH, SOL, BTC.

Never invent the name of a protocol, chain, metric, or token. If a tool
result names something, use that name verbatim. If you cannot confirm a
name, describe the thing instead of naming it.

Keep a token distinct from its network: the ETH token is not the
Ethereum chain. Name a protocol version when the version is the point
(Uniswap V3 vs V4).

## 3. Sourcing

The chain is public, tamper-evident, primary data. Prefer it. Every
source falls into a tier, and the tier decides how its claims may be
used.

| Tier | Sources | How to use |
|---|---|---|
| 1 — On-chain | on-chain data and the aggregators that read it directly (DefiLlama, block explorers) | Cite as fact. Read the number directly from your data tool and record it exactly as returned. |
| 2 — Official documents | regulatory filings, court records, regulator releases, official project announcements and post-mortems | Cite as fact *of the document*: "the filing states X", named and linked. The document's claims about the world stay attributed to its author — a press release saying "the exploit is contained" is the team's claim, not a finding. |
| 3 — Reputable reporting | established outlets with a track record and a corrections policy | Usable with attribution ("per Reuters"). Two independent outlets are stronger than one; two outlets citing the same original report are one source. |
| 4 — Social and self-published | founder posts, project accounts, Discord/Telegram | Leads and quotes only. Never the sole basis of a factual claim. A founder's statement is a quote to attribute, not a fact to state. |
| — Anonymous / unattributed | tips, screenshots, unsourced threads | Never publishable as fact. Corroborate upward into a higher tier or drop. |

A claim is only as strong as its tier. The corroboration rule in
`news-writing` applies on top: a contested, surprising, or market-moving
claim needs two independent sources, and sources citing each other count
as one.

A number's source tier decides how you back it. A tier-1 (on-chain)
number you read directly from your data tool is citable as fact — record
it exactly as the tool returned it, never typed from memory. A tier-2 or
tier-3 number (a fine in a court filing, a figure in an official
post-mortem, a number reported by an outlet) is cited by naming and
linking the source you read it in, and the corroboration rule covers it.
Never write a number that has neither behind it.

Either path assumes you actually opened the source. Cite only documents
and pages you fetched and read in this run. Never cite a link you have
not opened — a link that resolves is not a claim that checks out.

Use the `as_of` date your tool returns. Never imply the data is fresher
than its timestamp.

## 4. Framing and restraint

Crypoch publishes structural analysis for analysts, builders, and funds.
It is not a trading signal feed. This governs how every piece reads.

Do not write: price predictions, "alpha", "moon" or "dump", urgency
("act now"), or anything that reads as trade advice.

Do write: what happened, what the on-chain data shows, what it
plausibly means at a structural level, and what evidence would confirm
or refute that read.

On-chain data shows correlation. State causation as a hypothesis, not a
finding. "DEX volume fell as TVL declined" is an observation. "The TVL
decline was caused by X" is a claim that needs more than co-movement.

Hedge honestly. One week of data is one week of data. Say plainly what
the data cannot tell you. A dramatic number is reported flatly with
context, never celebrated. A 77% surge gets the same even tone as a 2%
drift.

## 5. Audience and jargon

The reader has baseline crypto fluency but does not know every protocol.

- Common terms need no definition: TVL, DEX, AMM, L2, staking, gas.
- Niche terms get a half-sentence on first use: restaking, LST, MEV,
  intents, proprietary AMM, points.
- Define by function, briefly. Do not lecture.

## 6. Common traps

- **Narrative over fact.** Report what the chain shows, not what the
  timeline says about it.
- **Single-actor distortion.** A volume or TVL spike on a smaller
  protocol is often one wallet, one aggregator routing change, or a
  short incentive program. Raise that possibility before calling it a
  trend.
- **Small-base growth.** Fast growth from a tiny base is not the same
  as growth from a large one. Give the absolute figures, not just the
  percentage.
- **Stale or future dates.** Never write a date the data does not
  support.
