# The command-line node: `bitcoind` and `bitcoin-cli`

> **Read this when** you want a node you drive from a terminal — headless, scriptable, or
> a throwaway regtest chain. For the desktop app, read `qt-gui.md`.

**In this file**

| Section | Answers |
|---|---|
| Install | One command |
| First run on mainnet | What actually happens, and how long |
| Configuration | `bitcoin.conf`, pruning, and the options worth setting |
| Talking to the node | Cookie auth, and why it just works |
| Running as a service | `brew services`, and when not to |
| **A regtest chain in 60 seconds** | **The bit you'll use most** |
| Stopping cleanly | Why `kill -9` is a bad habit here |

---

## Install

```bash
brew install bitcoin
```

See `homebrew.md` for exactly what this puts on disk. Verify:

```bash
bitcoin-cli --version && bitcoind --version
```

## First run on mainnet

```bash
bitcoind -daemon
```

That is the whole command — Bitcoin Core needs no configuration to start. It creates
`~/Library/Application Support/Bitcoin/`, connects to peers, and begins initial block
download from genesis.

Set expectations honestly: IBD downloads and **verifies** every block. It is bound by CPU
(signature verification) and disk, not usually by bandwidth. Expect hours at best, and
substantially longer on a translated x86_64 build (see `homebrew.md` on Rosetta) or a slow
disk. Watch it:

```bash
bitcoin-cli -getinfo
tail -f "$HOME/Library/Application Support/Bitcoin/debug.log"
```

`bitcoin-cli` will report `-28 Loading block index` for the first moments after start.
That is normal — the node is up but not ready. Retry rather than treating it as failure.

## Configuration

`bitcoin.conf` lives in the data directory and **is not created for you**:

```bash
"$HOME/Library/Application Support/Bitcoin/bitcoin.conf"
```

A reasonable starting point:

```conf
# Keep only the most recent ~20 GB of blocks instead of the full ~700 GB.
# Any non-zero value makes this a pruned node; see the caveat below.
prune=20000

# Index every transaction so getrawtransaction works for any txid.
# NOT compatible with prune. Costs significant extra disk.
# txindex=1

# Accept JSON-RPC over HTTP. Already the default for bitcoind; required for bitcoin-qt.
server=1
```

Two things worth knowing before you commit:

- **`prune` and `txindex` are mutually exclusive.** A pruned node has discarded the blocks
  an index would point into. Decide which you need; changing your mind later means a
  reindex or a resync.
- **Pruning is not retroactive-free.** Turning pruning on later discards data you already
  downloaded; turning it off requires downloading everything again.

Network-specific options belong under a section header, and options above any header apply
to *every* network:

```conf
server=1

[regtest]
rpcport=18443
```

## Talking to the node

Nothing to configure. Bitcoin Core writes a fresh random credential to `.cookie` in the
data directory on every start, and `bitcoin-cli` reads it from the same default location:

```bash
bitcoin-cli getblockchaininfo
bitcoin-cli -getinfo
```

If your node uses a non-default data directory, tell both sides:

```bash
bitcoind  -datadir=/path/to/data -daemon
bitcoin-cli -datadir=/path/to/data getblockcount
```

For talking to the node over HTTP from application code — including static `rpcauth`
credentials rather than the rotating cookie — see the `bitcoin-api` skill. The
`rpcauth.py` generator ships with the formula at
`$(brew --prefix bitcoin)/share/bitcoin/rpcauth/rpcauth.py`.

## Running as a service

```bash
brew services start bitcoin    # starts now and at login
brew services stop bitcoin
brew services list
```

**Do not do this while the Bitcoin-Qt app is running.** Two nodes cannot share a data
directory; the second will fail on the lock. Pick one.

For a machine that should always have a node up and no GUI, the service is the right
answer. For a laptop where you sometimes open the app, it is a trap.

## A regtest chain in 60 seconds

Regtest is a private chain where you mine blocks instantly and coins are worthless. It is
the correct place to develop, and it costs nothing to throw away.

> **Where this stops.** This section gets a regtest node *running*. The workflow on top of
> it — funding a wallet, why it takes 101 blocks and not 100, spending, controlling time,
> reorgs, connecting two nodes, tearing down — belongs to the **`bitcoin-cli`** skill and
> its regtest reference. Go there next rather than expecting it here. For a regtest chain
> with **Lightning** nodes attached, stay in this skill and see `references/polar.md`.

Use a dedicated data directory so it can never touch your real one:

```bash
export RT=/tmp/btc-regtest && mkdir -p "$RT"

bitcoind -regtest -datadir="$RT" -daemon -fallbackfee=0.0002
```

`-fallbackfee` matters: regtest has no fee history, so wallet sends fail without it.

```bash
bitcoin-cli -regtest -datadir="$RT" -named createwallet wallet_name=testwallet
ADDR=$(bitcoin-cli -regtest -datadir="$RT" -rpcwallet=testwallet getnewaddress)
bitcoin-cli -regtest -datadir="$RT" generatetoaddress 101 "$ADDR"
bitcoin-cli -regtest -datadir="$RT" -rpcwallet=testwallet getbalance
```

**101 blocks, not 100** — coinbase outputs need 100 confirmations before they are
spendable, so block 1's reward matures only once block 101 exists. You end with 50 BTC of
spendable regtest coin.

Useful additions for development work:

```bash
bitcoind -regtest -datadir="$RT" -daemon \
  -fallbackfee=0.0002 \
  -rest=1 \
  -txindex=1 \
  -zmqpubrawblock=tcp://127.0.0.1:28332 \
  -zmqpubsequence=tcp://127.0.0.1:28333
```

That gives you the unauthenticated REST interface on the regtest RPC port (18443) and ZMQ
notifications — the two things hardest to experiment with on a mainnet node. See the
`bitcoin-api` skill for how to use them.

Throw it away when done:

```bash
bitcoin-cli -regtest -datadir="$RT" stop && rm -rf "$RT"
```

## Stopping cleanly

```bash
bitcoin-cli stop
```

Bitcoin Core flushes the chainstate on shutdown, and that can take a while on a busy node
— `stop` returns immediately while the process keeps working. Wait for it to exit rather
than killing it.

`kill -9` on a node mid-flush risks a corrupted chainstate and a multi-hour reindex. The
cookie file's disappearance is a reliable "fully stopped" signal:

```bash
bitcoin-cli stop
while [ -f "$HOME/Library/Application Support/Bitcoin/.cookie" ]; do sleep 1; done
echo "stopped"
```
