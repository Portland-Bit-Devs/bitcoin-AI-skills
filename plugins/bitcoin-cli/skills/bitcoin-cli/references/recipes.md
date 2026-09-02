# Recipes

> **Read this when** you have a task rather than a question. Each entry is the command
> plus the part people get wrong.

All recipes are **read-only** unless marked. See the Safety section of `SKILL.md` before
running anything that spends.

**In this file**

| Section | Task |
|---|---|
| Node health | Is it up, is it synced, what version |
| Blocks | Look up by height or hash, walk a range |
| Transactions | Look up a txid, find the fee, decode |
| Mempool | What's pending, at what fee rate |
| Wallets | Balances, UTXOs, history, addresses |
| Unconfirmed payments | **Why is my transaction stuck** |
| Peers and network | Connections, bans, traffic |

---

## Node health

```bash
bitcoin-cli -getinfo                      # the one-screen summary
```

**Is it running at all** — without authenticating. The cookie exists only while the node
runs:

```bash
test -f "$HOME/Library/Application Support/Bitcoin/.cookie" && echo up || echo down
```

**Is it synced?** Don't compare `blocks` to `headers` alone — a node can have all headers
long before it has verified the blocks. `initialblockdownload` is the honest answer:

```bash
bitcoin-cli getblockchaininfo | jq -r '
  if .initialblockdownload
  then "syncing: \(.blocks)/\(.headers) (\(.verificationprogress*100|floor)%)"
  else "synced at \(.blocks)" end'
```

**Which implementation and version am I actually talking to?**

```bash
bitcoin-cli getnetworkinfo | jq -r '.subversion'      # e.g. /Satoshi:31.1.0/
```

`subversion` is also how you spot a fork such as Knots, whose RPC surface differs.

**Is it pruned, and from where?**

```bash
bitcoin-cli getblockchaininfo | jq '{pruned, pruneheight, size_on_disk}'
```

## Blocks

```bash
bitcoin-cli getblockhash 800000                       # height → hash
bitcoin-cli getblock "$(bitcoin-cli getblockhash 800000)"        # hash → block
bitcoin-cli getblock "$HASH" 2                        # verbosity 2: full transactions
```

`getblock` takes a **hash**, not a height — the two-step above is the usual dance.
Verbosity `0` gives raw hex, `1` (default) gives txids, `2` gives decoded transactions.

**On a pruned node, `getblockhash` succeeds for any height but `getblock` may not** — the
header chain is complete while the block bodies are gone. Expect
`-1 Block not available (pruned data)` and handle it.

Walk a range of heights:

```bash
for h in $(seq 800000 800009); do
  bitcoin-cli getblock "$(bitcoin-cli getblockhash $h)" \
    | jq -r '"\(.height)\t\(.time|todate)\t\(.nTx) txs"'
done
```

That is two RPCs per block. For anything larger, batch over HTTP — see `bitcoin-api`.

## Transactions

```bash
bitcoin-cli getrawtransaction "$TXID" true            # decoded, if retrievable
```

**The most common failure**: `-5 No such mempool transaction. Use -txindex…`. By default a
node can only retrieve transactions **in the mempool**. For a confirmed one you need either
`txindex=1` on the node, or the containing block hash as a third argument:

```bash
bitcoin-cli getrawtransaction "$TXID" true "$BLOCKHASH"
```

**What fee did it pay?** Depends what you have:

```bash
# Your own wallet transaction — fee is present, and negative
bitcoin-cli -rpcwallet=w1 gettransaction "$TXID" | jq '.fee'

# Still in the mempool — fee is present and positive
bitcoin-cli getmempoolentry "$TXID" | jq '{fee: .fees.base, vsize, sat_per_vb: (.fees.base*1e8/.vsize)}'
```

A confirmed transaction that isn't yours has **no** retrievable fee from
`getrawtransaction` — you would have to fetch every input's previous output and subtract.

**Decode something you already have as hex:**

```bash
bitcoin-cli decoderawtransaction "$HEX"
bitcoin-cli decodescript "$SCRIPTHEX"
```

## Mempool

```bash
bitcoin-cli getmempoolinfo | jq '{size, bytes, mempoolminfee, fullrbf}'
bitcoin-cli getrawmempool | jq length                  # just the txids
```

`getrawmempool true` returns full entries for **every** transaction — on a busy mainnet
node that is a very large response. Filter rather than fetching it repeatedly:

```bash
# Transactions paying more than 20 sat/vB
bitcoin-cli getrawmempool true \
  | jq -r 'to_entries[] | select(.value.fees.base*1e8/.value.vsize > 20) | .key'

# Current fee-rate distribution, roughly
bitcoin-cli getrawmempool true \
  | jq '[.[] | .fees.base*1e8/.vsize] | sort | {min: .[0], median: .[length/2|floor], max: .[-1]}'
```

**Fee estimates** for a target confirmation depth:

```bash
bitcoin-cli estimatesmartfee 6 | jq '{btc_per_kvb: .feerate, sat_per_vb: (.feerate*1e5)}'
```

Returns `"Insufficient data or no feerate found"` on regtest and on a freshly started node
— it needs observed history.

## Wallets

```bash
bitcoin-cli listwallets                                # loaded now
bitcoin-cli listwalletdir                              # exists on disk
bitcoin-cli -named loadwallet filename=w1              # load one
```

**Balances.** `getbalance` is a single number; `getbalances` is the honest breakdown:

```bash
bitcoin-cli -rpcwallet=w1 getbalances | jq '.mine'
# { "trusted": 149.99998350, "untrusted_pending": 0.00000000, "immature": 5000.00001650 }
```

`trusted` is what you can spend now. `untrusted_pending` is incoming and unconfirmed.
`immature` is coinbase output that hasn't reached 100 confirmations. A "missing" balance is
almost always sitting in one of the latter two.

**UTXOs and history:**

```bash
bitcoin-cli -rpcwallet=w1 listunspent
bitcoin-cli -rpcwallet=w1 listunspent 6                # ≥6 confirmations only
bitcoin-cli -rpcwallet=w1 listtransactions "*" 25
bitcoin-cli -rpcwallet=w1 listtransactions "*" 25 | jq -r '.[] | "\(.category)\t\(.amount)\t\(.confirmations)"'
```

**Addresses:**

```bash
bitcoin-cli -rpcwallet=w1 -named getnewaddress address_type=bech32m
bitcoin-cli -rpcwallet=w1 getaddressinfo "$ADDR" | jq '{ismine, solvable, desc}'
```

## Unconfirmed payments

"Why is my transaction stuck?" — work through it in this order:

```bash
# 1. Is it even in the mempool?
bitcoin-cli getmempoolentry "$TXID" 2>/dev/null || echo "not in mempool"
```

Not in the mempool means it was never broadcast, was evicted, or **was already confirmed**.
Check for confirmation before assuming failure.

```bash
# 2. What is it paying, versus what the mempool currently demands?
bitcoin-cli getmempoolentry "$TXID" | jq '.fees.base*1e8/.vsize'   # its sat/vB
bitcoin-cli getmempoolinfo         | jq '.mempoolminfee*1e5'       # floor, sat/vB
bitcoin-cli estimatesmartfee 6     | jq '.feerate*1e5'             # ~6-block target
```

```bash
# 3. Is it stuck behind an unconfirmed parent?
bitcoin-cli getmempoolentry "$TXID" | jq '{ancestorcount, ancestorsize, fees}'
```

An `ancestorcount` above 1 means it cannot confirm until its parents do — miners select by
ancestor package, so a low-fee parent holds a high-fee child back.

```bash
# 4. Was it replaced or double-spent?
bitcoin-cli -rpcwallet=w1 gettransaction "$TXID" | jq '{confirmations, "replaced_by_txid"}'
```

`confirmations: -1` means **conflicted** — a conflicting transaction confirmed instead.
That transaction is never going to confirm.

**Fixing it writes to the chain — confirm the fee with the user before running either:**

```bash
bitcoin-cli -rpcwallet=w1 bumpfee "$TXID"                     # RBF, if signalled
bitcoin-cli -rpcwallet=w1 -named bumpfee txid="$TXID" options='{"fee_rate": 25}'
```

## Peers and network

```bash
bitcoin-cli -netinfo                     # dashboard; add 0-4 for detail level
bitcoin-cli -netinfo 4                   # most verbose peer table
bitcoin-cli getconnectioncount
bitcoin-cli getpeerinfo | jq -r '.[] | "\(.addr)\t\(.subver)\t\(.conntype)"'
bitcoin-cli getnettotals | jq '{totalbytesrecv, totalbytessent}'
```

`-netinfo` is a `bitcoin-cli` feature with no RPC equivalent — you cannot get that table
over HTTP, only the raw `getpeerinfo` behind it.

```bash
bitcoin-cli listbanned
bitcoin-cli getnodeaddresses 10 | jq -r '.[].address'
```
