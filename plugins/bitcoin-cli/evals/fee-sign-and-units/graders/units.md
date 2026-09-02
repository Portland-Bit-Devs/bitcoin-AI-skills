Tests understanding of the sign convention and of float handling for money.

Required to pass — the response must state:

1. The fee is negative because `gettransaction` reports the effect on the
   wallet's balance, and the fee is value leaving the wallet. Taking its absolute
   value (or negating) is correct.
2. Amounts are in **BTC**, not satoshis — multiply by 100,000,000 (1e8).
3. Summing BTC-denominated floating-point values is unsafe; convert each to
   integer satoshis (rounding) and sum those.

Strong responses also:

- Give a concrete correct form, e.g. `jq '(.fee * -1e8) | round'`.
- Note that naive scaling can yield artefacts like `1650.0000000000002`.
- Note this is ordinary IEEE-754 behaviour, not a Bitcoin quirk.
- Warn that `getrawtransaction` has no `fee` field, so this approach only works
  for the node's own wallet transactions.
- Mention that a self-send shows `amount: 0` while still charging a fee, so
  `.details[]` may be needed for per-entry amounts.

Fail the response if it:

- Says the negative sign indicates an error or an unusual condition.
- Treats the values as satoshis.
- Endorses summing the BTC floats directly with no precision caveat.
