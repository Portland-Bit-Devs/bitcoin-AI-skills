# Invoking bitcoin-cli

> **Read this when** you are choosing flags — which network, which data directory, which
> wallet, how to wait for a node, or how to pass a secret safely.

**In this file**

| Section | Answers |
|---|---|
| The zero-config case | Why it usually just works |
| Network selection | Every network flag and its port |
| Data directory and config | `-datadir`, `-conf`, and non-default layouts |
| Wallet selection | `-rpcwallet`, and why to always pass it |
| Connecting to a remote node | `-rpcconnect`, and the SSH tunnel you should use instead |
| Waiting for a node | `-rpcwait`, `-rpcwaittimeout` |
| **Passing secrets** | **`-stdin` family — never put a passphrase in argv** |
| Client-only helpers | `-getinfo`, `-netinfo`, `-addrinfo`, `-generate` |

---

## The zero-config case

```bash
bitcoin-cli getblockchaininfo
```

No flags needed. `bitcoin-cli` defaults to mainnet, looks in the platform's default data
directory, and reads the `.cookie` credential the node wrote there at startup. Client and
server agree on all three by default, which is why nothing needs configuring when both run
as the same user on one machine.

It stops being automatic as soon as any of those differ — a non-default datadir, a
different network, a remote node, or static credentials instead of the cookie.

## Network selection

| Flag | Network | RPC port | Datadir subdirectory |
|---|---|---|---|
| *(none)* | mainnet | 8332 | *(the datadir itself)* |
| `-testnet` | testnet3 | 18332 | `testnet3/` |
| `-testnet4` | testnet4 | 48332 | `testnet4/` |
| `-signet` | signet | 38332 | `signet/` |
| `-regtest` | regtest | 18443 | `regtest/` |

`-chain=<name>` is the general form (`main`, `test`, `testnet4`, `signet`, `regtest`).

**The flag must match the node.** `bitcoin-cli` with no flag talks to port 8332; a node
started with `-regtest` listens on 18443. Mismatched flags produce `Connection refused`,
not a helpful message — and it is the single most common cause of that error when a node
is demonstrably running.

The cookie also lives in the network subdirectory, so `-regtest` changes *where the
credential is read from* as well as the port. Passing the network flag handles both.

## Data directory and config

```bash
bitcoin-cli -datadir=/path/to/data getblockcount
bitcoin-cli -conf=/path/to/bitcoin.conf getblockcount
```

`-datadir` is what you usually want: it locates both the config file and the cookie. Give
the *base* directory, not the network subdirectory — `bitcoin-cli -regtest -datadir=/x`
looks for the cookie in `/x/regtest/.cookie` on its own.

Point both the node and the client at the same one:

```bash
bitcoind     -regtest -datadir="$RT" -daemon
bitcoin-cli  -regtest -datadir="$RT" getblockcount
```

Related, occasionally useful:

- `-rpccookiefile=<path>` — read the cookie from somewhere other than the datadir
- `-rpcclienttimeout=<n>` — seconds before the client gives up (default 900); raise it for
  long calls such as `rescanblockchain` or `dumptxoutset`

## Wallet selection

```bash
bitcoin-cli -rpcwallet=mywallet getbalance
```

This sets the HTTP endpoint to `/wallet/mywallet`. Without it, wallet calls go to the bare
endpoint, which only works when **exactly one** wallet is loaded — zero gives `-18`, two or
more gives `-19`.

**Pass it whenever a wallet is involved, even with one wallet loaded.** It costs nothing,
and it means your script keeps working the day someone loads a second wallet. Code that
omits it has an invisible dependency on the node's wallet count.

```bash
bitcoin-cli listwallets          # what's loaded right now
bitcoin-cli listwalletdir        # what exists on disk, loaded or not
```

## Connecting to a remote node

```bash
bitcoin-cli -rpcconnect=10.0.0.5 -rpcport=8332 -rpcuser=alice -rpcpassword=... getblockcount
```

**Don't.** JSON-RPC is unencrypted HTTP, so those credentials cross the network in the
clear, and Core's documentation states the interface is not hardened against arbitrary
traffic. The supported approach is a tunnel, after which the node is local as far as
`bitcoin-cli` is concerned:

```bash
ssh -N -L 8332:127.0.0.1:8332 user@node-host &
bitcoin-cli -rpcconnect=127.0.0.1 getblockcount
```

You still need credentials the remote node accepts — the local cookie file won't be the
remote one. See `bitcoin-api`'s `connection.md` for generating `rpcauth` credentials.

## Waiting for a node

```bash
bitcoin-cli -rpcwait getblockcount
bitcoin-cli -rpcwait -rpcwaittimeout=60 getblockcount
```

`-rpcwait` blocks until the RPC server answers instead of failing immediately — the right
flag in a startup script that launches `bitcoind` and then immediately queries it.
`-rpcwaittimeout` bounds the wait; without it, it waits indefinitely.

It waits for the *connection*. It does not conjure credentials: if the datadir has no
cookie yet because the node has never started, the command fails rather than waiting. And a
node that answers may still be warming up — `-28 Loading block index` is a successful
connection to a node that isn't ready. Retry on that separately.

## Passing secrets

**Never put a passphrase or RPC password in a command line.** Arguments are visible in
`ps` output to every user on the machine, and they land in your shell history.

Three flags read from stdin instead:

| Flag | Reads |
|---|---|
| `-stdin` | All arguments, one per line |
| `-stdinrpcpass` | The **first line** is the RPC password; remaining lines are arguments |
| `-stdinwalletpassphrase` | The **first line** is the wallet passphrase |

```bash
# Unlock an encrypted wallet for 60 seconds without exposing the passphrase
echo "my wallet passphrase" | \
  bitcoin-cli -stdinwalletpassphrase -rpcwallet=w1 walletpassphrase "" 60

# Lock it again as soon as you're done
bitcoin-cli -rpcwallet=w1 walletlock
```

Read the passphrase from a password manager or a file with restrictive permissions rather
than typing it into a pipeline that your shell will record.

## Client-only helpers

These are implemented **in `bitcoin-cli` itself**, not on the node. They do not appear in
`help`, cannot be called over HTTP, and several make multiple RPC calls and format the
result:

| Flag | Does |
|---|---|
| `-getinfo` | One-screen summary: chain, blocks, sync progress, network, balance |
| `-netinfo` | Peer connection dashboard; takes a detail level `0`–`4` |
| `-addrinfo` | Counts of known addresses by network (ipv4, ipv6, onion, i2p) |
| `-generate [n]` | Regtest only: create an address and mine `n` blocks to it |

```bash
bitcoin-cli -getinfo
bitcoin-cli -netinfo 4                       # most verbose peer table
bitcoin-cli -regtest -rpcwallet=w1 -generate 101
```

`-getinfo` respects `-rpcwallet`, and reports that wallet's balance. `-generate` is a
convenience wrapper over `getnewaddress` plus `generatetoaddress`; it returns the address
it used along with the block hashes, so you can fund from it immediately.

Because these are client-side, an error in one can look different from an RPC error — see
`output.md` on exit codes.
