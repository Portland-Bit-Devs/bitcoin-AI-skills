# Sources

> **Read this when** you need to check a claim, or you are about to edit this skill and
> want to know what was actually observed.

**In this file**

| Section | Answers |
|---|---|
| Verification | **What was run, and what it produced** |
| Primary sources | The node itself, and Core's docs |
| Not verified | The honest gaps |
| Editing note | Keeping this accurate |

---

## Verification

Run against **Bitcoin Core v31.1.0** on macOS on 2026-09-02: a regtest node for the full
write path, and a mainnet node for read-only calls.

**Output shapes** — confirmed that `getblockcount` prints a bare number, `getbestblockhash`
prints a bare hash **with no JSON quotes**, and object-returning methods print indented
JSON. Piping a scalar into `jq` was confirmed to fail with `Invalid numeric literal`.

**Exit codes** — errors go to stderr with stdout empty, in the form
`error code: -N` / `error message:` / `<text>`. Measured:

| Condition | Exit |
|---|---|
| Success | `0` |
| `-18` wallet not found | `18` |
| `-5` no such transaction | `5` |
| `-3` wrong type (server-side, via `-named`) | `3` |
| `-32601` method not found | `89` — 32601 mod 256 |
| Client-side JSON parse failure | `1` |
| Could not connect | `1` |

The `abs(code) % 256` mapping and the client/server split were both established by testing
the same error two ways: `getblockhash abc` fails locally (exit `1`,
`error: Error parsing JSON: abc`) while `-named getblockhash height="notanumber"` reaches
the node and returns `-3` (exit `3`).

**Argument conversion** — confirmed that `getblock <hash>` accepts an unquoted string while
`getblockhash abc` is rejected by the client, establishing the per-method JSON-conversion
table. JSON array and object arguments were confirmed working both positionally and with
`-named`. An unquoted `[]` was confirmed to fail in the shell before reaching `bitcoin-cli`
(zsh: `no matches found`).

**Amount formatting** — this corrected a mistake in the first draft. Core's raw output uses
**fixed 8 decimal places** (`-0.00001650`, `0.00000000`); it does *not* emit scientific
notation. The `1.65e-05` originally documented came from Python's `json` module
re-serialising the value, not from Core. Separately confirmed that `jq` preserves most
literals but **renormalises some** — `{"a":0.00000000}` becomes `0E-8` while
`-0.00001650` and `1.50000000` pass through — and that naive scaling produces
`1650.0000000000002` where `| round` gives `1650`.

**Fees** — confirmed `gettransaction.fee` is present and **negative**, `getmempoolentry`
exposes `.fees.base` positive with ancestor/descendant figures, and
`getrawtransaction <txid> true` has **no** `fee` field at all. Confirmed a self-send
produces `amount: 0.00000000` at the top level while `.details[]` carries the signed
per-entry amounts.

**Transaction structure** — on a verified 1.5 BTC send, the change output (48.49998350) was
`vout[0]` and the payment (1.50000000) was `vout[1]`, confirming that output order carries
no meaning.

**Balances** — `getbalances.mine` was confirmed to split `trusted` /
`untrusted_pending` / `immature`, with mined rewards sitting in `immature`.

**Regtest workflow** — `createwallet`, `-generate`, `getnewaddress`, `sendtoaddress` with
`fee_rate`, and `gettransaction` were all run end to end. `-generate 101` produced exactly
`50.00000000` spendable, confirming the coinbase-maturity note. `estimatesmartfee` was
confirmed to return `"Insufficient data or no feerate found"` on regtest, which is why
`-fallbackfee` is required. The reorg sequence was run: invalidating the tip at height 102
dropped it to 101, mining 2 gave 103, and `reconsiderblock` left the tip at 103 — the node
follows the most-work chain rather than reverting.

**Client-only helpers** — `-getinfo`, `-netinfo`, `-addrinfo`, and `-generate` were all run
and produce the described output.

**Recipes** — every command in `recipes.md` was executed. The wallet, mempool, block, and
address recipes ran on regtest; the node-health, version, pruning, and peer recipes ran
against a mainnet node.

## Primary sources

**The node itself is the authority**, and it is always correct for its own version:

```bash
bitcoin-cli help                 # every method, grouped
bitcoin-cli help <method>        # signature, parameter types, result shape, examples
bitcoin-cli -help                # every client flag, including the client-only ones
```

`help <method>` states each parameter's type, which is what tells you whether an argument
needs JSON quoting. Prefer it to any third-party page, including this skill.

| Source | Used for |
|---|---|
| [`doc/JSON-RPC-interface.md`](https://github.com/bitcoin/bitcoin/blob/v31.0/doc/JSON-RPC-interface.md) | Parameter passing, the `args` convention, endpoint semantics |
| [`src/rpc/protocol.h`](https://github.com/bitcoin/bitcoin/blob/v31.0/src/rpc/protocol.h) | RPC error-code values behind the exit statuses |
| [Bitcoin Core RPC reference](https://bitcoincore.org/en/doc/) | `help` output rendered per release |

For the HTTP layer beneath `bitcoin-cli` — authentication, endpoints, the full error-code
tables — see the **`bitcoin-api`** skill rather than duplicating it here.

## Not verified

- **`-rpcwait` blocking behaviour.** The flag's existence and purpose are documented, but a
  clean test — client waiting while a node starts — was not completed. What *was* observed
  is that it fails immediately rather than waiting when the datadir has no cookie at all.
- **`-stdin`, `-stdinrpcpass`, `-stdinwalletpassphrase`** were confirmed present in
  `bitcoin-cli -help` but not exercised, because doing so means handling a real passphrase.
- **Remote `-rpcconnect` and SSH tunnelling** were not tested; single-machine only.
- **Multiple connected regtest nodes.** The port-separation commands are constructed from
  verified single-node behaviour, not run as a pair.
- **`bumpfee`** — not run, being a write to a wallet.
- **Fee-rate and mempool recipes on a busy mainnet mempool.** They ran, but against a
  regtest mempool that was mostly empty, so their *output* under load is unconfirmed.

## Editing note

Almost everything here is behavioural rather than transcribed, so the way to update a claim
is to **run it again** on the version you care about. Where a number appears — an exit
code, a formatted amount, a balance — it came from a real run, and replacing it with a
remembered value is how this file stops being useful.

The RPC method surface changes between major versions; the `bitcoin-cli` *program*
behaviour documented here (shapes, exit codes, quoting, client-only flags) has been stable
for many releases and is the durable content.
