# Reading the output

> **Read this when** you have a result and need to interpret it, or you are piping
> `bitcoin-cli` into something and the shapes aren't what you expected.

**In this file**

| Section | Answers |
|---|---|
| Output shapes | Bare values vs. pretty JSON, and what that means for `jq` |
| Exit codes and stderr | **Scripting `bitcoin-cli` correctly** |
| Amounts | BTC vs. satoshis, and the float trap |
| Fees | Where the fee is, where it isn't, and its sign |
| Transaction structure | `vin`, `vout`, and which output is yours |
| Confirmations | Including the negative case |
| jq patterns | The ones worth knowing |

---

## Output shapes

`bitcoin-cli` prints results in three different shapes, and mixing them up breaks
pipelines. All verified:

```bash
$ bitcoin-cli getblockcount
101                                    # bare number

$ bitcoin-cli getbestblockhash
79eeb65da067bc03…                      # bare string, NO surrounding quotes

$ bitcoin-cli getblockchaininfo
{
  "chain": "regtest",                  # pretty-printed, indented JSON
  "blocks": 101,
  …
}
```

**Scalar results are printed bare** — a string comes out without JSON quotes. This is
deliberate and convenient: you can use them directly in shell substitution with no parsing.

```bash
HASH=$(bitcoin-cli getbestblockhash)          # ready to use
bitcoin-cli getblock "$HASH"
```

The corollary is that **piping a scalar into `jq` fails**, because a bare hash is not
valid JSON:

```bash
bitcoin-cli getbestblockhash | jq .           # parse error
bitcoin-cli getblockchaininfo | jq -r .bestblockhash   # do this instead
```

Objects and arrays *are* valid JSON and pipe into `jq` cleanly.

## Exit codes and stderr

Errors go to **stderr**; stdout stays empty. The format:

```
error code: -32601
error message:
Method not found
```

The exit status distinguishes three failure classes, which is what makes `bitcoin-cli`
scriptable:

| Exit | Meaning |
|---|---|
| `0` | Success |
| `1` | **Client-side failure** — couldn't connect, couldn't parse an argument, bad flag |
| *n* | **Node returned an RPC error**, where *n* is derived from the RPC error code |

For node errors the status is the absolute value of the RPC error code, wrapped into a byte
(`abs(code) % 256`). Verified: `-18` → exit `18`, `-5` → exit `5`, `-3` → exit `3`, and
`-32601` → exit `89`, because 32601 mod 256 is 89.

Two practical consequences:

- **Don't test for a specific code above 255**; the wrap makes it unintuitive. Test
  `-eq 0` for success, and parse stderr when you need to distinguish a specific failure.
- **Exit `1` means the command never reached the node.** That is a different problem from
  any RPC error, and it is the one to check first.

```bash
if ! out=$(bitcoin-cli getblockcount 2>&1); then
  echo "failed ($?): $out" >&2
  exit 1
fi
```

Note `set -e` will abort on the first non-zero exit, which for `bitcoin-cli` includes
perfectly expected conditions like "wallet not loaded". Guard the calls you expect to fail.

## Amounts

**Every amount in the RPC interface is in BTC, never satoshis.** `0.00001650` is 1650
satoshis.

Core prints amounts at a **fixed 8 decimal places** — `-0.00001650`, `0.00000000` — so the
raw CLI output is unambiguous. The hazard is downstream: these are JSON *numbers*, and the
moment a parser reads one into a binary float, precision and formatting are no longer
Core's business. Verified on the same value:

| Consumer | Result |
|---|---|
| `bitcoin-cli` raw output | `-0.00001650` |
| `jq -r .fee` (passthrough) | `-0.00001650` — literal preserved |
| Python `json.load()` | `-1.65e-05` — re-serialised as a float |
| `jq '.fee * -1e8'` | `1650.0000000000002` — arithmetic error |
| `jq '(.fee * -1e8) \| round'` | `1650` — correct |

This is ordinary IEEE-754 behaviour, not a Bitcoin quirk — `jq -n '0.1 + 0.2'` gives
`0.30000000000000004` too. But it matters more here, because the values are money.

**Don't rely on textual round-tripping either.** `jq` preserves most literals but
renormalises some, and zero is the one you will meet:

```bash
$ echo '{"a":0.00000000,"b":-0.00001650,"c":1.50000000}' | jq .
{
  "a": 0E-8,          # renormalised
  "b": -0.00001650,   # preserved
  "c": 1.50000000     # preserved
}
```

So a zero balance can reach a downstream consumer as `0E-8`. Anything that string-matches
on amounts, or feeds them to a parser expecting plain decimals, needs to cope with that.
Converting to integer satoshis avoids the whole class of problem.

**Convert to integer satoshis as early as possible and do arithmetic there:**

```bash
bitcoin-cli -rpcwallet=w1 getbalance | awk '{printf "%.0f\n", $1 * 100000000}'
bitcoin-cli -rpcwallet=w1 gettransaction "$TXID" | jq '(.fee * -1e8) | round'
```

Never accumulate BTC-denominated floats for accounting.

## Fees

**Where the fee is depends on which call you make**, and this trips people up constantly:

| Call | Fee available? |
|---|---|
| `gettransaction` (wallet) | **Yes** — `.fee`, and it is **negative** |
| `getmempoolentry` | **Yes** — `.fees.base`, positive, plus ancestor/descendant figures |
| `getrawtransaction <txid> true` | **No** — verified: there is no `fee` field |

`getrawtransaction` describes the transaction as it exists on the chain, and a transaction
does not record its fee — the fee is inputs minus outputs, and the inputs are in *other*
transactions. To compute it you must fetch every input's previous output. This is why
`txindex` or a block explorer exists.

The sign convention in `gettransaction` is a genuine trip hazard: **`.fee` is negative**
because it is money leaving the wallet, as are `send`-category amounts in `.details[]`.

```bash
bitcoin-cli -rpcwallet=w1 gettransaction "$TXID" | jq '{fee: .fee, amount: .amount}'
# { "fee": -0.00001650, "amount": 0.00000000 }
```

That `amount: 0` is not a bug. `gettransaction.amount` is the **net** effect on the wallet,
and this was a self-send — the money came back. For what actually moved, read `.details[]`,
where each entry has its own `category` (`send`, `receive`, `generate`, `immature`) and
signed `amount`.

Fee *rates* are quoted in **BTC/kvB**, not sat/vB. Multiply by 100,000 to convert:
`0.00001000 BTC/kvB` is `1 sat/vB`. Sizes are in **vbytes** (`vsize`), with `weight` being
four times that.

## Transaction structure

```bash
bitcoin-cli getrawtransaction "$TXID" true
```

- `vin[]` — inputs, each referencing a previous `txid` and `vout` index. A coinbase input
  has a `coinbase` field instead.
- `vout[]` — outputs, each with `.n` (index), `.value` **in BTC**, and `.scriptPubKey`
  carrying `.type`, `.address` (when the type has one), and `.desc`.

**Output order is not meaningful.** A spend typically produces two outputs — the payment
and the change — and which is which is deliberately not marked. In a verified example the
change (48.4999835 BTC) was `vout[0]` and the 1.5 BTC payment was `vout[1]`. Never assume
the payment is index 0. Identify outputs by address, and use the wallet's own view
(`gettransaction`, `listunspent`) when you need to know what belongs to you.

## Confirmations

`confirmations` appears on wallet transactions and verbose raw transactions:

| Value | Meaning |
|---|---|
| `0` | In the mempool, not yet mined |
| `n > 0` | Included in a block, with `n-1` blocks on top |
| **`-1`** | **Conflicted** — a conflicting transaction was confirmed instead |

The negative case is the one to handle. A transaction that has been replaced (RBF) or
double-spent shows `-1`, and treating "not positive" as "still pending" will leave a
payment waiting forever.

Coinbase outputs carry the `immature` category until 100 confirmations, after which they
become spendable — hence mining 101 blocks in regtest to get one spendable reward.

## jq patterns

```bash
# Sync status, human-readable
bitcoin-cli getblockchaininfo | jq -r '"\(.blocks)/\(.headers) — \(.verificationprogress*100|floor)%"'

# Balance in satoshis, safely
bitcoin-cli -rpcwallet=w1 getbalances | jq '(.mine.trusted * 1e8) | round'

# Every output of a transaction with its address
bitcoin-cli getrawtransaction "$TXID" true \
  | jq -r '.vout[] | "\(.n)\t\(.value)\t\(.scriptPubKey.address // .scriptPubKey.type)"'

# Total value of your spendable UTXOs, in satoshis
bitcoin-cli -rpcwallet=w1 listunspent | jq '[.[] | .amount * 1e8] | add | round'

# Mempool transactions paying above 10 sat/vB
bitcoin-cli getrawmempool true \
  | jq -r 'to_entries[] | select(.value.fees.base * 1e8 / .value.vsize > 10) | .key'
```

Note `.scriptPubKey.address // .scriptPubKey.type` — `address` is absent for output types
that have none, and `//` supplies the fallback rather than printing `null`.
