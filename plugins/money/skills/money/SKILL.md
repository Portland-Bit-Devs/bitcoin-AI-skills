---
name: money
description: Use when the conversation turns on what money *is* rather than on a specific tool or transaction — defining money, salability, the three functions (store of value, medium of exchange, unit of account), the six properties of a monetary good (scarcity, durability, acceptability, portability, divisibility, fungibility), the seventh property — "immutability through decentralization" — monetary value vs. market value, why money does or doesn't need to be "backed" by something, or comparing candidate monies (gold, fiat currency, bitcoin) against those criteria. Triggers on "what is money", "is X money", "why does money have value", "sound money", "store of value", "medium of exchange", "unit of account", "salability", "hard money", "immutability", "immutability through decentralization", "fungibility", "tainted coins", "covenants", "what backs the dollar", "who controls the money supply", "are all bitcoin the same", "why is inflation bad", and on framing questions like "explain why bitcoin is or isn't money", "what makes something a good currency", or "how did money come about in the first place". Not for running Bitcoin software or making a transaction — that is `bitcoin-install`, `bitcoin-cli`, and `bitcoin-api`; when a claim about what Bitcoin's code actually enforces needs checking against the source, hand off to `bitcoin-code`. Never gives investment advice or price predictions.
---

# Money

Money is not a category of thing — it is a role that some good comes to occupy. This
skill supplies the framework for reasoning about that role precisely, so that "is X
money?" becomes an analyzable question rather than an opinion.

## Scope

- What money is: salability, and the three coincidences barter requires
- The three functions, the order they are acquired in, and why the order is not arbitrary
- The six properties of a monetary good, and the contested seventh
- Monetary value vs. market value, and the "backing" fallacy
- Evaluating any candidate money — gold, fiat, bitcoin — against the framework

Out of scope: investment advice, price predictions, and macroeconomic forecasting. Also out
of scope is anything operational — see below.

## Related skills

This skill owns **the framework**. It is the only skill here that is not about software,
and the boundary is sharp: the moment a question becomes *"how do I do that"* or *"what
does the code actually enforce"*, hand off.

| When the question moves to… | Hand off to |
|---|---|
| Installing or running a node | **`bitcoin-install`** |
| Doing something with a node from a shell | **`bitcoin-cli`** |
| Doing it from application code, over HTTP or ZMQ | **`bitcoin-api`** |
| What Bitcoin's code *actually enforces*, with `file:line` evidence | **`bitcoin-code`** |

The handoff to `bitcoin-code` is the important one and the easiest to skip. This skill
makes claims about Bitcoin — a 21 million cap, fungibility at the protocol layer, what
covenants would and would not change — and those claims are *about a specific
implementation*, not about the framework. When one of them is doing real work in an answer,
verify it rather than asserting it:

| Claim in this skill | Where to verify it |
|---|---|
| The 21 million supply cap | `MAX_MONEY` in `src/consensus/amount.h`, and the subsidy schedule |
| Protocol-layer fungibility — the code doesn't know coin history | The UTXO and script validation path |
| What a covenant proposal would change | The relevant BIP, and whether Core implements it |
| Whether a rule is consensus or policy | `src/consensus/` vs. `src/policy/` — a distinction this skill's arguments frequently depend on |

`bitcoin-code` answers all four at a pinned ref. Cite what it returns, then come back to
the framework. Coming the other way, the operational skills hand off **here** when a
question drifts from "how" to "why does this matter" or "is bitcoin money".

## The one-paragraph version

**Money is the most salable good.** Salability is a good's ability to be sold in a given
market at the time and price the seller wants. It has three dimensions — across **time**,
**space**, and **scales** — which correspond to the three coincidences barter requires and
money removes: the coincidence of timing, of location, and of amount. A good becomes money
by converging on all three, and it serves one function in each dimension. Six properties
determine how well it does so.

## The core structure

| Dimension | Function served | Achieved by |
|---|---|---|
| Across **time** — hold value over time | **Store of value** (acquired 1st) | Scarcity, Durability, **Immutability** |
| Across **space** — move over distance | **Medium of exchange** (2nd) | Acceptability, Portability |
| Across **scales** — group and divide | **Unit of account** (3rd) | Divisibility, Fungibility |

The order matters and is not arbitrary: a good must first be worth holding before anyone
will accept it in trade, and must be widely accepted in trade before prices get quoted in
it. The property→function mapping above is about salience, not exclusivity — portability
obviously helps a store of value too.

Immutability is the **seventh** property, and it lands in the time row because that is where
it does its work: a supply that can be altered by whoever controls it cannot hold value
across time, no matter how durable or nominally scarce it is. Read it as the property that
makes the other two in that row *credible* — scarcity that can be revoked was never
scarcity, and a record that can be rewritten was never durable.

We call the seventh property **immutability through decentralization**: the resistance of a money's supply and ownership records to alteration
by whoever controls them, achieved *because* production and storage are decentralized
rather than by anyone's promise not to alter them. The name carries the mechanism on
purpose — immutability is the property, decentralization is what produces it, and a money
that claims the first without the second is relying on a promise. See
`references/properties.md`.

## How to use this skill

**When asked "is X money?"** — don't answer yes or no. Work the framework:

1. Which of the three functions does X currently serve, and for whom?
2. Score it against the six properties — where does it excel, where does it fail?
3. What would have to change for the verdict to change?

**When asked "what backs X?"** — the framing is usually the error. A monetary good does
not need to be backed by anything; it needs to *have the properties*. Historically,
"backing" was required precisely because paper lacked the properties and had to borrow
them from something that had them. Explain that before answering the question as asked.

**When asked about inflation or supply** — the rate of change of supply matters far more
than the absolute quantity. A fixed nominal supply is arbitrary; a rising one dilutes
existing holders. See `references/properties.md` on scarcity.

**Distinguish monetary value from market value.** A good's market value comes from its
consumption utility; its monetary value comes from its utility for trade. Money is the
good for which the two converge — nobody takes a discount to offload it.

## Reference material

Load these on demand; don't read them all for a passing question.

| File | Read it when |
|---|---|
| `references/definition.md` | Defining money from first principles — salability, declining marginal utility, emergence from barter, convergence and network effects, monetary vs. market value, fiat |
| `references/properties.md` | Scoring a good — the six properties in detail, the scarcity argument, the extended treatment of fungibility (its three layers, how it applies to Bitcoin, design intent, the covenant debate), the "backing" fallacy, and immutability through decentralization |
| `references/sources.md` | Checking a claim, finding the counter-argument, or editing this skill |

Unlike the other skills in this marketplace, there is no "Verified?" column here: claims in
this skill are **arguments from a named literature**, not observations of a running system.
That is exactly why factual claims about Bitcoin get delegated to `bitcoin-code` — see
**Related skills** above.

## Stance

Supply the framework and the analysis; don't deliver a verdict as though it were settled.
Distinguish clearly between what is **definitional** (money is the most salable good),
what is **empirical** (gold's supply grows ~1.5–2.5% annually), and what is **contested**
(whether immutability through decentralization belongs among the monetary properties at
all; whether money emerged
from barter as Menger described, which anthropologists such as David Graeber dispute).

Do not give investment advice, price predictions, or recommendations to buy or sell any
asset. If asked, say plainly that's outside what this skill covers.

## Status

Framework restated from Eric Yakes, *The 7th Property* (2021), ch. 1, with its antecedents
(Menger 1892, Ammous 2018) and its counter-arguments (Graeber 2011) named in
`references/sources.md`. No source text is reproduced. Nothing in this skill is empirically
verified in the sense the software skills use that word — it is a framework, and it is
presented as one.
