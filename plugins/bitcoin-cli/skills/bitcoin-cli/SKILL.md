---
name: bitcoin-cli
description: Use when driving a Bitcoin Core node from a terminal or a shell script with `bitcoin-cli` — composing a command, quoting its arguments, reading what comes back, or building a pipeline with `jq`. Covers connection and network flags (`-regtest`, `-signet`, `-testnet4`, `-datadir`, `-rpcwallet`, `-rpcwait`), the client-only helpers (`-getinfo`, `-netinfo`, `-addrinfo`, `-generate`), positional vs. `-named` arguments and the per-method JSON conversion that decides which need quotes, passing secrets via `-stdin`/`-stdinrpcpass`/`-stdinwalletpassphrase`, exit codes and stderr, output shapes, BTC-vs-satoshi units and float-precision hazards, fee math, confirmation semantics, and the regtest mine-and-fund workflow. Triggers on "bitcoin-cli", "bitcoind", "getblockchaininfo", "getrawtransaction", "listunspent", "generatetoaddress", "bitcoin-cli error parsing JSON". ONLY once Bitcoin is named or already established in context, also triggers on "check if my node is synced", "look up this txid", "how much is in this wallet", "why is my transaction still unconfirmed", "what fee did I pay", "mine some coins to test with". Bitcoin Core only — do NOT load for Ethereum, Solana, or any other chain, or for non-blockchain nodes and servers. Not for the HTTP wire protocol, auth, or RPC error-code tables — that is `bitcoin-api`; not for installing Core — that is `bitcoin-install`; not for how Core implements a rule — that is `bitcoin-code`.
---

# bitcoin-cli

## Preconditions

This skill is about the **`bitcoin-cli` binary talking to Bitcoin Core**. Confirm that
before using it.

Stop and say so in one line if the subject is another chain's CLI (`geth`, `solana`,
`lncli`, `bitcoin-cash`…), a non-blockchain node or server, or a generic "my node won't
sync" question with no Bitcoin in it. The flags, argument-quoting rules, BTC-vs-satoshi
units, exit codes, and confirmation semantics here are Bitcoin Core's; applying them to
another tool produces confident, wrong answers.

If it's genuinely ambiguous which chain is meant, ask rather than assume.

`bitcoin-cli` is a thin JSON-RPC client for `bitcoind`, plus a handful of conveniences the
node itself does not provide. This skill is about using it well from a shell: composing
commands, surviving the quoting rules, and reading the output correctly.

## Scope

- Invocation: connection flags, network selection, wallet selection, secrets
- The client-only helpers: `-getinfo`, `-netinfo`, `-addrinfo`, `-generate`
- Arguments: positional vs. `-named`, and **why some need quotes and others don't**
- Output: shapes, exit codes, units, `vout`, fees, confirmations, `jq` pipelines
- The regtest workflow: mine, fund, spend, throw away

## Related skills

This skill owns **the program**: the `bitcoin-cli` binary, its flags, its argument
handling, and the text it prints.

| When the question moves to… | Hand off to |
|---|---|
| The HTTP wire protocol, auth mechanisms, RPC error-code tables | **`bitcoin-api`** |
| Installing Core, `command not found`, first run, Polar | **`bitcoin-install`** |
| How Core implements the rule behind a result | **`bitcoin-code`** |
| Whether any of this is *money* | **`money`** |

The split with `bitcoin-api` is worth stating plainly: **that skill owns the wire, this one
owns the program.** When a call fails with a numeric RPC code, the code's meaning lives in
`bitcoin-api`; how `bitcoin-cli` surfaces it — stderr, exit status — lives here. Two
practical consequences: send a user to `bitcoin-api` the moment they leave the terminal for
application code, and send them here the moment they come back.

Coming the other way, `bitcoin-install` hands off here as soon as the formula is installed,
and `bitcoin-api` hands off here when someone reaching for `curl` would be better served by
one `bitcoin-cli` invocation.

**Shared topic — regtest.** This skill owns the regtest **workflow**:
`references/regtest.md` is the canonical treatment. `bitcoin-install` owns only *getting a
regtest node started*, and Polar.

## The 30-second version

```bash
bitcoin-cli getblockchaininfo          # mainnet, cookie auth, zero configuration
bitcoin-cli -getinfo                   # the human summary
bitcoin-cli -regtest -generate 101     # private chain: mine 101 blocks to a new address
bitcoin-cli help sendtoaddress         # authoritative, version-correct, always available
```

`bitcoin-cli` finds the data directory and the cookie on its own. If the node is up and on
the default network, nothing needs configuring.

Three things account for most confusion, and each has a reference file:

1. **Quoting.** `getblock <hash>` needs no quotes, `getblockhash abc` fails locally with
   `error parsing JSON`, and JSON arguments need quotes *inside* shell quotes. There is a
   rule — see `references/arguments.md`.
2. **Output is not always JSON.** Single values print bare, with no quotes, which is a
   feature for shell use and a surprise for `jq`. See `references/output.md`.
3. **`-getinfo` and friends are not RPC methods.** They are client-side, so `help` won't
   describe them and they can't be called over HTTP. See `references/invocation.md`.

## How to use this skill

**Use `help` before guessing.** `bitcoin-cli help <method>` is generated from the running
node's own source, so it is correct for *that* version — better than any documentation
including this skill. When a signature matters, read it there.

**Prefer `-named`.** Positional arguments break silently when a parameter is inserted into
a signature between releases; named ones don't. `bitcoin rpc` is shorthand for
`bitcoin-cli -named`.

**Always pass `-rpcwallet=` when a wallet is involved**, even with one wallet loaded. It
costs nothing and makes the command keep working when a second wallet appears.

**Read amounts carefully.** RPC amounts are in **BTC**, not satoshis. Core prints them at
a fixed 8 decimal places (`-0.00001650`), but they are JSON *numbers* — the moment a parser
reads one into a binary float, you can get `-1.65e-05` and arithmetic error. Convert to
integer satoshis and round. See `references/output.md`.

**Reach for regtest.** Anything exploratory, anything destructive, anything you want to run
twice belongs on a private chain. See `references/regtest.md`.

## Safety

`bitcoin-cli` spends money. It has no confirmation prompt and no undo.

- **Never run a spending or signing command against mainnet without explicit, per-command
  confirmation of the exact amount and destination.** That covers `sendtoaddress`,
  `sendmany`, `send`, `sendall`, `sendrawtransaction`, `signrawtransactionwithwallet`,
  `signrawtransactionwithkey`, `walletcreatefundedpsbt`, and `bumpfee`. Read-only queries
  need no such ceremony.
- **`-generate` and `generatetoaddress` are regtest tools.** Always pass `-regtest`.
- **Never run `dumpwallet`, `dumpprivkey`, `listdescriptors true`, or
  `gettransaction`-with-private-data on a user's behalf**, and never write their output
  anywhere. They return private keys.
- **Never put a passphrase or RPC password in a command line.** It lands in your shell
  history and in `ps` output for every user on the machine. Use `-stdin`,
  `-stdinrpcpass`, and `-stdinwalletpassphrase` — see `references/invocation.md`.
- **`walletpassphrase` unlocks for a duration.** Lock it again with `walletlock` rather
  than leaving a wallet open.
- Prefer regtest or signet for anything exploratory.

## Reference material

Load on demand. Each file opens with a "Read this when" note and its own contents table.

| File | Read it when | Verified? |
|---|---|---|
| `references/invocation.md` | Choosing flags — network, datadir, wallet, waiting, secrets | **yes** |
| `references/arguments.md` | A command won't parse, or you're quoting JSON | **yes** |
| `references/output.md` | Reading results — shapes, exit codes, units, fees, `jq` | **yes** |
| `references/regtest.md` | You want a private chain to mine on and throw away | **yes** |
| `references/recipes.md` | You have a task, not a question | **yes** |
| `references/sources.md` | Checking a claim, or editing this skill | — |

**yes** means the commands and their output were run against a real Bitcoin Core v31.1
node. `sources.md` records exactly what.

## Status

Written and verified against **Bitcoin Core v31.1.0** on macOS — read-only commands against
a mainnet node, and the full write path (wallets, mining, spending, fee inspection) against
a regtest node. Where behaviour is version-sensitive, the reference files say so.
