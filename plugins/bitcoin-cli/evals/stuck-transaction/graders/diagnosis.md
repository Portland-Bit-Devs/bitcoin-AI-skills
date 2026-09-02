Tests systematic diagnosis rather than jumping to "bump the fee".

Required to pass — the response must:

1. Check whether the transaction is still in the mempool at all
   (`getmempoolentry`, or `gettransaction` for a wallet transaction).
2. Compare its fee rate against current conditions — via `getmempoolentry`
   `.fees.base` with `.vsize`, and/or `getmempoolinfo` / `estimatesmartfee`.
3. Treat any fee-bumping action (`bumpfee`) as a **write** — either flagging that
   it spends money and needs confirmation of the new fee, or presenting it as a
   deliberate final step rather than the opening move.

Strong responses also:

- Note that absence from the mempool may mean it already **confirmed**, was
  evicted, or was never broadcast — not automatically failure.
- Check `confirmations == -1`, meaning conflicted/replaced, in which case it will
  never confirm.
- Check for unconfirmed ancestors (`ancestorcount`), since a low-fee parent
  holds a higher-fee child back.
- Correctly convert BTC/kvB or BTC amounts to sat/vB rather than comparing raw
  BTC figures.
- Mention that `bumpfee` requires the transaction to have signalled RBF.

Fail the response if it:

- Opens by running `bumpfee` with no diagnosis and no mention that it spends.
- Claims an unconfirmed transaction can be "cancelled" or "reversed" outright.
- Compares a BTC-denominated amount directly against a sat/vB figure.
