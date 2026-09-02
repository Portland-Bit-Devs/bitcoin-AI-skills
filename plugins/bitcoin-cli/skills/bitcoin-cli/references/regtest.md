# The regtest workflow

> **Read this when** you want a private chain you control — to mine on, spend on, break,
> and throw away. This is where anything exploratory belongs.

**In this file**

| Section | Answers |
|---|---|
| Why regtest | And how it differs from testnet and signet |
| Start a chain | Isolated from your real node |
| Fund a wallet | And why it's 101 blocks, not 100 |
| Spend | The `-fallbackfee` requirement |
| Control time and blocks | Mining on demand, reorgs |
| Multiple nodes | Connecting two regtest nodes |
| Tear down | Leave nothing behind |

---

## Why regtest

Regtest is a private Bitcoin network with the same consensus rules and a trivial
proof-of-work target, so you mine a block instantly on your own machine. Coins are
worthless, the chain is yours alone, and you can reset it whenever you like.

| | regtest | signet | testnet3/4 |
|---|---|---|---|
| Mine on demand | **Yes** | No | Impractical |
| Coins | Worthless, self-minted | Faucet | Faucet |
| Peers | None (or ones you choose) | Public | Public |
| Resettable | **Instantly** | No | No |
| Good for | Development, tests, CI | Realistic pre-release testing | Legacy |

Choose regtest when you want *control*, and signet when you want *realism*. Testnet3 has a
long history of instability; testnet4 replaced it.

## Start a chain

Always use a dedicated data directory so a regtest node can never touch your real one:

```bash
export RT=/tmp/btc-regtest && mkdir -p "$RT"
bitcoind -regtest -datadir="$RT" -daemon -fallbackfee=0.0002
bitcoin-cli -regtest -datadir="$RT" -rpcwait getblockcount     # → 0
```

`-rpcwait` avoids the race between `-daemon` returning and the RPC server accepting
connections.

Typing `-regtest -datadir="$RT"` on every command gets old immediately. Define a helper:

```bash
btc() { bitcoin-cli -regtest -datadir="$RT" "$@"; }
btc getblockcount
```

In **zsh**, note that `$VAR` holding a command string does *not* word-split — a shell
function like the above works, but `CMD="bitcoin-cli -regtest"; $CMD getblockcount` does
not. Use a function, not a variable.

## Fund a wallet

```bash
btc -named createwallet wallet_name=w1
btc -rpcwallet=w1 -generate 101
btc -rpcwallet=w1 getbalance          # → 50.00000000
```

`-generate` is a `bitcoin-cli` convenience: it creates an address and mines to it in one
step, returning both the address and the block hashes. The explicit form is:

```bash
ADDR=$(btc -rpcwallet=w1 getnewaddress)
btc generatetoaddress 101 "$ADDR"
```

**Why 101 and not 100.** Coinbase outputs require 100 confirmations before they can be
spent. Mining 101 blocks means block 1's reward has 100 blocks on top of it and is mature —
so you end with exactly one spendable 50 BTC reward. Mine 100 and your balance is zero,
with every reward still `immature`. Verified: after 101 blocks, `getbalance` returns
`50.00000000` while `getbalances` shows the rest under `mine.immature`.

## Spend

```bash
DEST=$(btc -rpcwallet=w1 getnewaddress "" bech32m)
TXID=$(btc -rpcwallet=w1 -named sendtoaddress address="$DEST" amount=1.5 fee_rate=10)
btc -rpcwallet=w1 -generate 1          # confirm it
btc -rpcwallet=w1 gettransaction "$TXID" | jq '{fee, confirmations}'
```

**`-fallbackfee` is required.** Regtest has no fee history, so `estimatesmartfee` returns
`"Insufficient data or no feerate found"` and wallet sends fail without a fallback. Set it
at startup as above, or pass an explicit `fee_rate` (in sat/vB) on every send.

## Control time and blocks

Mining is instant and on demand, which is the whole point:

```bash
btc -generate 6                        # bury a transaction under confirmations
btc getblockcount
```

To test reorg handling, invalidate a block and mine a competing chain:

```bash
HASH=$(btc getbestblockhash)
btc invalidateblock "$HASH"            # drop the tip
btc -generate 2                        # build a longer chain
btc reconsiderblock "$HASH"            # clear the invalid mark
```

`reconsiderblock` removes the invalid mark, but the node still follows the most-work chain
— so if the replacement chain you mined is longer, the tip stays where it is. Verified:
invalidating at height 102 dropped the tip to 101, mining 2 gave 103, and reconsidering
left it at 103.

This is the cleanest way to exercise code that must cope with a chain reorganisation —
something you cannot arrange on a public network.

## Multiple nodes

Two regtest nodes on one machine need separate datadirs and non-colliding ports:

```bash
bitcoind -regtest -datadir=/tmp/rt-a -daemon -port=19000 -rpcport=19001 -fallbackfee=0.0002
bitcoind -regtest -datadir=/tmp/rt-b -daemon -port=19010 -rpcport=19011 -fallbackfee=0.0002

bitcoin-cli -regtest -datadir=/tmp/rt-b -rpcport=19011 addnode "127.0.0.1:19000" add
bitcoin-cli -regtest -datadir=/tmp/rt-b -rpcport=19011 getpeerinfo | jq length
```

If you also run Polar, note it claims host port **18443** — the default regtest RPC port —
so move yours with `-rpcport` rather than fighting over it. See the `bitcoin-install` skill.

## Tear down

```bash
btc stop
rm -rf "$RT"
```

`stop` returns immediately while the node finishes flushing; wait for the process to exit
before deleting the directory. The cookie file disappearing is a reliable signal:

```bash
btc stop
while [ -f "$RT/regtest/.cookie" ]; do sleep 1; done
rm -rf "$RT"
```

Because the datadir is disposable, `rm -rf` here is safe in a way it never is for mainnet —
which is exactly why the dedicated `-datadir` at the top matters.
