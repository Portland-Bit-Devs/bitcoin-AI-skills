# money

What money is, as an analytical framework rather than a verdict.

```bash
claude plugin install money@bitcoin-ai-skills
```

Provides the `money` skill:

- **Definition** — money as the most salable good; salability across time, space, and
  scales; emergence from barter; monetary value vs. market value
- **The three functions** — store of value, medium of exchange, unit of account, and why
  a good acquires them in that order
- **The six properties** — scarcity, durability, acceptability, portability, divisibility,
  fungibility — mapped to the functions each one serves
- **The "backing" fallacy** — why a monetary good needs properties, not backing
- **The seventh property** — *immutability through decentralization*, presented as an
  argued thesis rather than settled consensus

The framework follows Eric Yakes, *The 7th Property* (2021), ch. 1, which builds in turn on
Menger's *On the Origins of Money* (1892) and Ammous's *The Bitcoin Standard* (2018). It is
restated here in our own words with attribution — no source text is reproduced. See
[`references/sources.md`](skills/money/references/sources.md), which also points at the
anthropological counter-argument (Graeber) so the skill doesn't present one side of a live
disagreement as settled.

## Relationship to the other skills

`money` is the only skill in this marketplace that isn't about software, and the boundary
is deliberate. It supplies the *framework*; it does not run anything and it does not
adjudicate what Bitcoin's code does.

- Operational questions — installing, running, spending, querying — hand off to
  [`bitcoin-install`](../bitcoin-install), [`bitcoin-cli`](../bitcoin-cli), and
  [`bitcoin-api`](../bitcoin-api).
- Factual claims about Bitcoin — the 21 million cap, protocol-layer fungibility, what
  covenants would change — hand off to [`bitcoin-code`](../bitcoin-code) for a `file:line`
  citation at a pinned ref. The skill delegates these rather than asserting them from
  memory, and its reference files carry explicit *"verify, don't assert"* callouts at each
  such claim.

Coming the other way, the software skills hand off here when a question drifts from *how*
to *why does this matter* or *is bitcoin money*.

Licensed GPL-3.0. See the [repository root](../../README.md).
