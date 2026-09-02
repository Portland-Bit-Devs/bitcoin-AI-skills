Tests the single most important distinction in Bitcoin source research.

Required to pass — the response must:

1. State clearly that a minimum fee rate is **policy, not consensus**.
2. Explain the consequence: a low-fee transaction is not invalid, it is simply
   not relayed or mined by nodes with that policy; a block containing one is
   perfectly valid and other nodes may differ.
3. Point at policy code — `src/policy/` (e.g. `policy.h`, `feerate.h`) and/or the
   `minrelaytxfee` setting — rather than at `src/consensus/`.

Strong responses also:

- Note that policy can change in a release, or be configured per node, without a
  fork, whereas consensus cannot.
- Offer to verify against a specific ref, or state which ref/version the answer
  applies to.
- Mention that consensus constants live in `src/consensus/` but much consensus
  *logic* lives in `src/validation.cpp`, so directory alone isn't the test —
  the test is whether the rule can make a block invalid.

Fail the response if it:

- Calls the minimum fee rate a consensus rule.
- Claims such a transaction is "invalid" without distinguishing relay from
  validity.
- Cites a specific file:line as fact without naming a ref or version, or invents
  a plausible-looking path without indicating it should be verified.
