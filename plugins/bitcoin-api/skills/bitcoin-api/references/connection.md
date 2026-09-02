# Connecting and authenticating

> **Read this when** you cannot reach the node at all, or are setting one up for the first
> time. Once your calls are getting *through* and you're debugging what comes back, you
> want `json-rpc.md` instead.

**In this file**

| Section | Answers |
|---|---|
| Is the RPC server even on? | The `bitcoin-qt` gotcha that wastes the most time |
| Where bitcoin.conf lives | Paths per platform, and the network-section trap |
| Ports | Every network's RPC and P2P port |
| Authentication | Cookie vs. `rpcauth` vs. `rpcuser` — and which to pick |
| Binding and remote access | `rpcbind`, `rpcallowip`, and why not to use them |
| Throughput settings | `rpcthreads`, `rpcworkqueue`, and the file-descriptor crash |
| Troubleshooting | **Symptom-first table — start here if something is broken** |

---

## Is the RPC server even on?

`bitcoind` has JSON-RPC **enabled** by default. `bitcoin-qt` has it **disabled** by
default. Both are changed with `-server`.

This is the single most common cause of "connection refused" for people running the
graphical node: the GUI's own Debug Console executes RPC methods internally, without
going over HTTP, so the console working tells you nothing about whether port 8332 is
listening.

```conf
# bitcoin.conf — required for bitcoin-qt, redundant for bitcoind
server=1
```

Check from outside the node:

```bash
lsof -nP -iTCP:8332 -sTCP:LISTEN
```

## Where bitcoin.conf lives

| Platform | Data directory |
|---|---|
| macOS | `~/Library/Application Support/Bitcoin/` |
| Linux | `~/.bitcoin/` |
| Windows | `%APPDATA%\Bitcoin\` |

`bitcoin.conf` goes in that directory and is **not created for you** — a fresh install has
no config file at all, which the debug log records as `Config file: ... (not found, skipping)`.

Non-mainnet networks nest their data under a subdirectory (`testnet3/`, `testnet4/`,
`signet/`, `regtest/`), and network-specific options belong in a section header:

```conf
server=1

[regtest]
rpcport=18443
```

Options set before any section header apply to all networks. This trips people up
constantly: an `rpcport=` at the top of the file applies to mainnet *and* regtest.

## Ports

| Network | RPC | P2P | Flag |
|---|---|---|---|
| mainnet | 8332 | 8333 | *(default)* |
| testnet3 | 18332 | 18333 | `-testnet` |
| testnet4 | 48332 | 48333 | `-testnet4` |
| signet | 38332 | 38333 | `-signet` |
| regtest | 18443 | 18444 | `-regtest` |

The REST interface shares the RPC port. ZMQ uses separate ports you choose yourself
(28332/28333 are conventional but carry no special meaning).

## Authentication

HTTP Basic auth, always. There are three ways to get credentials, in descending order of
preference.

### 1. The cookie file — the default, and the right choice for local tools

When no `rpcpassword` is configured, Core generates a fresh random credential on every
startup and writes it to `.cookie` in the data directory (or the network subdirectory),
readable only by the user running the node.

The file contains a single line, already in `user:password` form:

```
__cookie__:8f3a1b...
```

So it drops straight into curl:

```bash
curl --user "$(cat "$HOME/Library/Application Support/Bitcoin/.cookie")" ...
```

Properties worth knowing:

- **It changes on every restart.** A long-lived client must re-read the file, not cache
  the value. A `401` from a client that worked yesterday is almost always this.
- **It is deleted on clean shutdown.** If `.cookie` is absent, the node is not running.
  This is a reliable liveness check that costs nothing.
- It lives in the *network* subdirectory for non-mainnet: `regtest/.cookie`.
- Relocate it with `-rpccookiefile=<path>` when the client can't read the data directory.

Treat it as a credential: don't echo it, log it, or write it into scratch files.

### 2. `rpcauth` — static credentials, password never stored on the node

Generate with the script shipped in Core's source tree:

```bash
python3 share/rpcauth/rpcauth.py alice
```

It prints an `rpcauth=` line for `bitcoin.conf` and, separately, the password. The config
line holds only a salted HMAC:

```conf
rpcauth=alice:e7f9c0...$b8a3...
```

The node never stores the plaintext password, and you can define several `rpcauth` lines
for several clients — which means you can revoke one without disturbing the others. This
is the correct choice for an application with its own credentials.

### 3. `rpcuser` / `rpcpassword` — plaintext, avoid

```conf
rpcuser=alice
rpcpassword=hunter2
```

Works, stores the password in cleartext in a config file, and gives every client the same
credential. Use `rpcauth` instead. Note that setting `rpcpassword` **disables cookie
generation**, so this choice breaks any local tool that was relying on the cookie.

## Binding and remote access

By default the RPC server binds to localhost only and rejects everything else. Two
separate settings govern opening that up, and you need both:

```conf
rpcbind=10.0.0.5      # which interface to listen on
rpcallowip=10.0.0.0/24  # which clients may connect
```

`rpcbind` has no effect unless `rpcallowip` is also set — Core refuses to start with
`rpcbind` alone, on the theory that you probably didn't mean it.

**Do not expose this to the internet.** JSON-RPC is unencrypted HTTP; Basic auth
credentials cross the network in the clear, and Core's documentation states plainly that
the interface "has not been hardened to withstand arbitrary Internet traffic." The
supported answers for remote access are an SSH tunnel:

```bash
ssh -N -L 8332:127.0.0.1:8332 user@node-host
```

or a VPN. Not `rpcallowip=0.0.0.0/0`.

Under Docker, the naive `-p 8332:8332` publishes the port on **all** host interfaces.
Bind it explicitly: `-p 127.0.0.1:8332:8332`.

## Throughput settings

```conf
rpcthreads=4       # worker threads handling calls
rpcworkqueue=16    # depth of the queue in front of them
```

A `503` with `Work queue depth exceeded` means the queue filled: requests arrived faster
than `rpcthreads` could retire them. Raise `rpcworkqueue` if the load is bursty, raise
`rpcthreads` if it is sustained — but first check whether you should be using ZMQ or REST
instead of hammering JSON-RPC.

Both interfaces share a documented failure mode: opening many hundreds of simultaneous
HTTP connections can exhaust the process's file descriptors and **crash the node**. Pool
and reuse connections; do not fan out unboundedly.

## Troubleshooting, symptom first

| Symptom | Likely cause | Fix |
|---|---|---|
| `Connection refused` | Node not running, or `server=1` unset under `bitcoin-qt` | Start node; add `server=1` |
| `Connection refused`, node *is* running | Wrong port for the network | Check the port table above |
| `401 Unauthorized` | Stale cookie after restart | Re-read `.cookie` |
| `401 Unauthorized`, `rpcauth` in use | Password mismatch, or `$` eaten by the shell | Single-quote the `rpcauth` value |
| `403 Forbidden` | Client IP not in `rpcallowip` | Add the CIDR; check you're not hitting a proxy |
| RPC error `-18` on `/wallet/foo` | Wallet not loaded (**not** an HTTP 404 — it is `200` under 2.0, `500` under 1.1) | `loadwallet`, or check `listwallets` |
| `404` on every path | JSON-RPC 2.0 method-not-found, or `-rest` off | See `json-rpc.md` on 1.1 vs. 2.0 |
| RPC error `-28` | Node still warming up | Retry with backoff — expected on startup |
| RPC error `-18` on `/` | **Zero** wallets loaded | `loadwallet` |
| RPC error `-19` on `/` | **Two or more** wallets loaded | Address `/wallet/<name>` |
| `503 Work queue depth exceeded` | Too many concurrent calls | Raise `rpcworkqueue`; batch; use ZMQ |
| Node exits immediately on start | Consent gate, bad config, or corrupt state | Read `debug.log` — it always says why |

`debug.log` in the data directory is the authoritative account of what the node did on
startup, including the exact config file it read and every option it applied. Read it
before guessing.
