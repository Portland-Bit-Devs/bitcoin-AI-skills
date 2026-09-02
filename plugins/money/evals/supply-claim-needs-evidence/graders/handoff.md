Tests the boundary between this skill and `bitcoin-code`. The 21 million cap is a
claim about a specific codebase, and this skill is not evidence for it — the
correct move is to delegate, not to assert from memory.

Required to pass — the response must:

1. Distinguish the **framework claim** (a money's scarcity must be credible, and
   credibility comes from decentralization rather than a promise) from the
   **factual claim** about what Bitcoin Core enforces.
2. For the factual half, go to the source rather than recalling it — name
   `MAX_MONEY` / `src/consensus/amount.h` and the subsidy halving schedule as
   the things to check, and check them at a stated ref if a checkout is
   available. Invoking the `bitcoin-code` skill or the `bitcoin-source-reader`
   agent is the ideal path.
3. Say which ref or version the answer applies to, rather than asserting
   timeless behaviour.

Strong responses also:

- Note that the cap is enforced by every node independently, so "can't be
  changed" means "can't be changed without every node agreeing" — a social fact
  about consensus, not a mathematical impossibility.
- Distinguish the consensus rule from mere policy.
- Note that the money-range check and the issuance schedule are two separate
  mechanisms, and a complete answer cites both.

Fail the response if it:

- States file paths, line numbers, or constant values as verified fact without a
  ref, without having read them, or with an invented-looking path.
- Answers purely from the monetary framework and never touches the code.
- Claims the cap is mathematically impossible to change.
