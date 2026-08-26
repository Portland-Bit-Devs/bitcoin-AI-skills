---
name: bitcoin-cli
description: Use when interacting with a Bitcoin Core node from the shell — running `bitcoin-cli` or raw JSON-RPC, querying chain state, inspecting blocks and transactions, managing wallets, or setting up regtest/signet for testing. Triggers on explicit mentions of bitcoin-cli, bitcoind, Bitcoin Core RPC, getblockchaininfo, getrawtransaction, listunspent, or bitcoin.conf, and also on task descriptions that imply them without naming them — "check if my node is synced", "look up this txid", "spin up a local Bitcoin network to test against", "why is my transaction still unconfirmed", "how much is in this wallet".
---

# bitcoin-cli

> **Status: stub.** Structure is in place; the reference material is not written yet.

Working with a Bitcoin Core node means talking to `bitcoind` over JSON-RPC, usually
through the `bitcoin-cli` wrapper. This skill is for getting those calls right and
reading the responses correctly.

## Scope

- Invoking `bitcoin-cli` — connection flags, `bitcoin.conf`, cookie vs. rpcuser auth
- The RPC surface: chain/block/transaction queries, mempool, wallet, network
- Network selection: mainnet, testnet, signet, regtest
- Reading results: units (BTC vs. satoshis), hex vs. verbose forms, confirmation semantics

Out of scope: compiling Bitcoin Core (see `bitcoin-code`), and monetary theory (see `money`).

## Safety

Bitcoin operations move real, irreversible value. Before running anything:

- **Never** run commands that spend, send, or sign against mainnet on a user's behalf
  without explicit, specific confirmation of the exact amount and destination.
- Prefer regtest or signet for anything exploratory.
- Treat wallet files, descriptors, xprvs, and seed phrases as secrets — never echo them,
  never write them to logs or scratch files.

## TODO

- [ ] `references/rpc-methods.md` — the commonly used RPC calls, grouped by task
- [ ] `references/setup.md` — `bitcoin.conf`, auth, pruning, indexes (`txindex`)
- [ ] `references/regtest.md` — spin up a local chain, mine blocks, fund addresses
- [ ] `references/reading-output.md` — units, `vout` structure, fee math, confirmation depth
- [ ] `evals/evals.json` — cases covering a chain query, a wallet query, and a regtest setup
