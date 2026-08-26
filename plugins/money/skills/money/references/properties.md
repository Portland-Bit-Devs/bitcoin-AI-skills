# The Properties of Money

Six properties determine how well a good serves the monetary functions. The monetary
medium a society uses has changed as new materials and technologies emerged that scored
better — but *what* is being scored has stayed constant.

## The six

### 1. Scarcity
Limited in supply relative to other goods. *(Serves: store of value.)*
The single most important property — see the extended discussion below.

### 2. Durability
Can be used repeatedly without losing functionality. *(Serves: store of value.)*
Physical durability for a commodity; for a digital money, the durability of the record and
of the network that maintains it.

### 3. Acceptability
Is used by others, and therefore accepted widely. *(Serves: medium of exchange.)*
The reflexive property: it is a fact about other people, not about the good. This is what
makes network effects decisive and what makes incumbents hard to dislodge.

### 4. Portability
Can be moved across distances. *(Serves: medium of exchange.)*
Where gold's weakness became decisive — expensive to move and to guard, which is exactly
what pushed societies toward paper claims on stored gold, and from there toward
centralization.

### 5. Divisibility
Can be divided into smaller units of value. *(Serves: unit of account.)*
Determines the range of transaction sizes a money can price — a good that can't be divided
can't measure small values.

### 6. Fungibility
One unit is regarded as identical to, and interchangeable with, another.
*(Serves: unit of account.)*
Without it there is no single price for "one unit," because units differ. Fungibility is the
most context-dependent of the six and the easiest to get wrong — see the extended treatment
below.

## Scarcity: the rate, not the amount

The most common error in reasoning about monetary supply is focusing on the total rather
than the **rate of increase**.

The share-issuance analogy makes it clean. A company worth $100 issuing 100 shares gives
each share a value of $1. Had it issued 200 instead, each would be worth $0.50 — and
nothing real would have changed. The initial number is arbitrary. But if it issues 100
shares and then issues 100 *more* a year later, existing holders are diluted by 50%. That
is a real transfer.

Money works identically. The quantity of money at inception carries no information; the
change in quantity over time is what matters, because new units are claims on the same
underlying goods, and someone receives them first.

This is why hard-to-produce goods have been sought as money. Gold's above-ground supply has
grown roughly 1.5–2.5% per year — enough certainty that holders don't expect sudden
dilution. Newly issued government money has generally grown considerably faster.

The mechanism to reason about is not "printing causes prices to rise" in the abstract, but
that whoever receives new units first spends them at old prices, and holders further down
the chain absorb the difference.

## Fungibility: a property of context, not of the good

Fungibility is the property people most often assert as binary — a money either is or isn't
fungible — and it is the one where that framing fails hardest. A unit's interchangeability
is not a fact about the unit. It is a fact about the unit **within a particular settlement
context**, and the same unit can be perfectly fungible in one context and discounted or
refused in another.

The clearest way to reason about it is in three layers.

| Layer | Question it answers | Where it can break |
|---|---|---|
| **Protocol** | Does the system's own rule set treat all units identically? | Consensus rules that read a unit's history or encumbrance |
| **Encumbrance** | Can a specific unit be restricted in how it may move next? | Script-level conditions attached to particular units |
| **Social / economic** | Will counterparties accept this unit at par? | Surveillance, taint scoring, blacklists, regulation |

A money can pass at one layer and fail at another. Gold is a useful calibration: a bar with
suspect provenance trades at a discount (social layer fails), but melting it restores
interchangeability completely (protocol layer never failed, and there is no encumbrance
layer). Physical cash carries serial numbers — a latent tracking mechanism that is almost
never enforced, which is precisely why cash is fungible in practice despite being
distinguishable in principle.

The general lesson: **fungibility degrades when history becomes both legible and
actionable.** Legibility alone is not fatal. Someone must also be willing to act on it.

## Fungibility applied to Bitcoin

### Protocol layer: fungible by construction

Bitcoin's consensus rules do not read history. A node validating a transaction checks that
inputs exist, are unspent, and are correctly authorized — nothing else. No consensus rule
assigns a coin a provenance score, and no valid satoshi is worth less to a node than any
other. At this layer bitcoin is fully fungible, and that is a deliberate design property,
not an accident.

### Social layer: broken in practice

At the same time, every bitcoin carries a complete, permanent, public transaction history.
That history is *legible* by design — it is how double-spending is prevented without a
trusted third party. The consequence is a fungibility failure at the social layer:

- Exchanges and custodians run deposits through chain-analytics tools (Chainalysis,
  Elliptic, TRM Labs) that score where the coins have been.
- Coins scoring poorly can be quarantined, frozen, or refused — sometimes several hops
  removed from any flagged activity, affecting holders who had no knowledge of it.
- There is **no industry standard** for which tainting method to use, what score triggers
  action, or how many hops to trace. One venue accepts a deposit another rejects.

That last point is the crux. When acceptance varies by venue, a bitcoin's value depends on
where you try to spend it — which is exactly what non-fungibility means. The protocol's
indifference to history does not save you if your counterparty is not indifferent.

The sanctions dimension makes it concrete. The 2022 sanctioning of Tornado Cash (an Ethereum
mixer, but the mechanism is chain-agnostic) led to industry-wide freezing of anything that
had touched it; OFAC delisted it in March 2025 after losing in court. The episode showed
both that taint can propagate across an entire industry at once, and that the criteria are
contestable rather than settled.

### What was the intent?

Bitcoin's design does not treat the public ledger and fungibility as being in tension —
because the whitepaper assigned fungibility to a *different mechanism*.

Section 10 of the whitepaper addresses this directly. The traditional banking model gets
privacy by restricting who sees transactions; publishing every transaction forecloses that
route, so privacy is preserved instead "by breaking the flow of information in another
place: by keeping public keys anonymous." The public sees that an amount moved, without
information linking it to anyone — explicitly analogized to a stock exchange tape, which
publishes time and size but not counterparties.

The operational instruction follows: *"a new key pair should be used for each transaction to
keep them from being linked to a common owner."*

So the intent was **fungibility through unlinkability, not through opacity.** History would
be public but unattributable. Coins would be distinguishable from one another while their
owners remained pseudonymous, and an unattributable history is not actionable.

Satoshi also flagged the leak that would eventually break this: multi-input transactions
"necessarily reveal that their inputs were owned by the same owner," and if one key is ever
tied to an identity, that linkage propagates. What the design did not anticipate was an
industry built on exactly that heuristic, combined with mandatory identity collection at the
points where bitcoin meets the banking system. Address reuse, KYC'd on- and off-ramps, and
industrialized clustering turned an acknowledged edge case into the default.

**The honest summary:** bitcoin's fungibility was designed to rest on a privacy practice
that most users do not follow and that regulated intermediaries are structurally required to
defeat. The protocol kept its side of the bargain; the surrounding context did not.

### Covenants: a possible fungibility change at the encumbrance layer

A **covenant** is a script condition that restricts not merely *who* may spend an output but
*where it may go next* — the spending transaction must itself satisfy conditions set by the
current one. The legitimate uses are real: vaults that force a delay and an abort path
before funds can leave, congestion-control batching, and various scaling constructions.

The fungibility question turns on **recursion**:

- **Non-recursive** covenants constrain the next hop only. The restriction expires once
  spent. `OP_CHECKTEMPLATEVERIFY` (BIP 119) is the canonical example — it commits to a
  specific template for the next transaction.
- **Recursive** covenants can re-impose themselves on every subsequent spend, indefinitely.

Recursion is what would introduce a genuinely new failure mode: coins that are permanently
restricted at the protocol level — a class of bitcoin that can only ever move within a
defined set. That would be a fungibility divergence at the **encumbrance** layer, meaningfully
different from today's social-layer taint. Today's discount is a matter of counterparty
policy that another counterparty can decline to enforce. A recursive encumbrance would be
enforced by every node, with no counterparty to appeal to.

**The counter-argument, which is serious:** covenants are opt-in and *visible before
acceptance*. You can inspect an output's conditions and simply refuse to receive encumbered
coins, and nothing lets anyone retroactively wrap already-free coins in a covenant. On this
reading the market prices encumbered coins below unencumbered ones, nobody accepts them at
par, and the failure mode never materializes.

**The rebuttal:** the concern was never voluntary adoption. It is issuance-point coercion —
regulated custodians made to release only covenant-bound coins, so that "just don't accept
them" stops being available to anyone who acquires bitcoin through regulated channels. There
is also a diffuse cost: if encumbrances become common, every recipient must check them,
which raises verification cost for everyone.

**There is no developer consensus on this.** Treat it as an open dispute, not a resolved
question, and represent both sides when it comes up.

**Status as of August 2026 — verify before relying on it.** No covenant proposal has been
activated on mainnet. `OP_CAT` (BIP 347) reached "Complete" specification status in March
2026 with no mainnet activation parameters, and has been tested heavily on signet. CTV+CSFS
has emerged as the frontrunner combination among core developers, with CTV's own client
pointing at 2027. A competing bundle (BIP 448) entered Draft in March 2026, and several
credible proposals now compete for the same soft-fork slot, which may itself delay any
activation. **This section dates quickly — check current status rather than citing it.**

### Fungibility can also be repaired

The direction is not one-way, which is worth saying because the topic attracts fatalism:

- **CoinJoin / PayJoin** break the common-input-ownership heuristic that most clustering
  depends on — attacking the specific inference, not the visibility.
- **Lightning** moves most transfers off-chain, leaving far less history to analyze.
- **Taproot** (activated 2021) made key-path spends indistinguishable regardless of the
  policy behind them, enlarging the anonymity set. The improvement is partial — script-path
  spends remain distinguishable from key-path — but it is a consensus change that *increased*
  fungibility, which is the direct counterexample to the assumption that protocol changes can
  only erode it.

## Answering fungibility questions

Never answer "is X fungible?" with yes or no. Name the layer:

1. **Protocol** — do the rules treat all units identically? (Bitcoin: yes.)
2. **Encumbrance** — can specific units be restricted in how they move? (Bitcoin: not
   meaningfully today; the covenant debate is about whether that should change.)
3. **Social** — will counterparties accept the unit at par? (Bitcoin: unevenly, and this is
   where its fungibility actually degrades.)

Then say which layer the questioner is actually asking about — usually the third, often
while using language that implies the first.

## The "backing" fallacy

A recurring confusion worth addressing directly whenever it comes up.

**Money does not need to be backed by anything. It needs to have the properties.**

Gold wasn't chosen because something backed it — it was chosen because it scored well
across all six. Paper scores well on portability and divisibility and terribly on scarcity
and (as an object) durability. So paper had to be made redeemable for gold: "backing" was
the mechanism by which paper *borrowed* monetary properties it did not have, in exchange
for being easier to handle.

Which reframes the question. When someone says a currency is "backed by the government,"
that is not a claim about monetary properties at all. Ask instead: what are this money's
properties, and who controls its supply?

Anything lacking the properties must be backed by something that has them. Anything
possessing them needs no backing.

## The seventh property: immutability through decentralization

*(Serves: store of value — alongside scarcity and durability.)*

This is the distinctive thesis of Eric Yakes's *The 7th Property*, and should be presented
as an argued position rather than settled consensus.

It belongs in the **time** dimension with the other two store-of-value properties, and it
is the one they depend on. Scarcity that a controlling party can revoke was never scarcity;
a ledger that party can rewrite was never durable. Immutability is what converts the other
two from claims into properties — which is also why it is easy to overlook when the
controlling party happens not to be exercising the power.

**The claim:** the six classical properties are incomplete, because they say nothing about
whether a money's supply and ownership records can be *altered by whoever controls them*.
We call that property **immutability through decentralization**, and the name is the
argument in miniature: *immutability* is the property being sought, and **decentralized
production and storage** is the mechanism that delivers it. The two halves are not
separable. A money can be immutable in practice because no one currently chooses to alter
it — but that is a promise, not a property. Only decentralization of who may produce units
and who holds the record turns it into something structural.

Use the full phrase on first mention in any explanation; "the seventh property" is fine as
shorthand afterward.

**The historical argument:** early monies were immutable more or less by accident. They
were produced in a decentralized way (anyone could mine or refine) and stored in a
decentralized way (each holder kept their own). Centralization proceeded in two stages —
first production (state mints, and the coin debasement that followed), then storage (banks,
and eventually central banks). Each stage bought efficiency and paid for it in mutability.
Today's money is highly mutable in both dimensions.

**The tradeoff:** in every prior form of money, immutability through decentralization cost efficiency — hard money
was expensive to produce, move, and verify. That's precisely why societies kept trading it
away. The thesis is that a money which is *natively* decentralized in production and
storage could offer immutability without that penalty, and that this is the next
evolutionary step.

**The digital caveat that sharpens it:** software is nearly free to replicate and can be
controlled remotely, which cuts both ways. A state-issued digital currency would concentrate
control further than anything before it — every holder maintaining an account directly with
the issuing authority. The property is therefore contested territory, not a foregone
conclusion.

**Where to be careful.** Treat immutability through decentralization as a *proposal* to
extend the framework, not as a received part of it. Most textbook treatments still list six
properties or fewer, and a critic could reasonably argue it is scarcity plus durability
restated institutionally rather than a genuinely separate axis. A second objection targets
the name directly: decentralization is a matter of degree that can erode over time, so a
money's score on this property is a moving measurement rather than a fixed characteristic
of the good. Present the argument and both objections.

## Applying the framework

To evaluate any candidate money, score it on all six (or seven) properties and identify
which functions it can therefore serve. Some worked-through observations:

- **Gold** — excellent scarcity and durability, poor portability at scale, imperfect
  divisibility. Its portability failure is what drove the centralization that followed.
- **Fiat currency** — excellent portability, divisibility, and (by mandate) acceptability;
  scarcity is a policy choice rather than a property of the good.
- **Bitcoin** — the case its proponents make rests on scarcity and on immutability through
  decentralization;
  the criticisms rest on acceptability and on volatility while it is still early in the
  function sequence.

A complete answer names the tradeoff rather than declaring a winner. No good has ever
scored perfectly on all of them — which is precisely why the monetary medium has changed
across history, and why arguments about it are genuinely arguments rather than
misunderstandings.
