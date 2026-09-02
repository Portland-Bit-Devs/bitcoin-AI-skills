Negative case. "Check if my node is synced", "look up this txid", and "still
isn't confirmed" are all conditional triggers for this skill — conditional on
Bitcoin being the subject, which here it explicitly is not.

Required to pass — the response must:

1. Recognise the subject is Ethereum, not Bitcoin Core, and say so plainly.
2. Not offer `bitcoin-cli` commands, `getblockchaininfo`, `gettransaction`,
   `getmempoolentry`, or BTC/satoshi unit conversions as the answer.

Strong responses also:

- Name the correct tool family for the actual question (`geth`, `cast`, an RPC
  client) without pretending to expertise this skill doesn't carry.
- Note that confirmation semantics and fee units differ between the two chains,
  so the Bitcoin material would actively mislead.

Fail the response if it:

- Issues or suggests any `bitcoin-cli` invocation.
- Applies Bitcoin's mempool, RBF, or fee-rate reasoning to Ethereum.
- Notices the mismatch but answers with Bitcoin material anyway.
