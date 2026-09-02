Tests whether the model right-sizes the tooling instead of defaulting to the
tool the user named. The user described a plain Bitcoin need, not Lightning.

Required to pass — the response must:

1. State that Polar requires Docker.
2. Point out that for a plain Bitcoin regtest chain, Polar is not necessary, and
   `bitcoind -regtest` needs no Docker.
3. Give a usable regtest starting point — at minimum `bitcoind -regtest`, plus
   `generatetoaddress` for mining.

Strong responses also:

- Ask what the app actually needs, or state the deciding factor explicitly:
  Polar earns its complexity when **Lightning** nodes are involved.
- Mention using a separate `-datadir` to keep regtest away from mainnet data.
- Note that 101 blocks are needed before coinbase output is spendable
  (100-block coinbase maturity).
- Mention `-fallbackfee` being required for wallet sends on regtest.

Fail the response if it:

- Simply walks through installing Polar without noting the lighter alternative.
- Claims Polar can run without Docker.
- Recommends testnet or signet as equivalent to regtest for a chain the user
  wants to *mine on themselves* (they cannot mine on demand there).
