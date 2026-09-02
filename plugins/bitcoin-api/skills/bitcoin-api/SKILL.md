---
name: bitcoin-api
description: Use when talking to a Bitcoin Core node over its network interfaces rather than through the `bitcoin-cli` wrapper — building or debugging JSON-RPC calls over HTTP, the unauthenticated REST interface, or ZMQ notifications, from curl, Postman, or application code in any language. Covers RPC authentication (the `.cookie` file, `rpcauth`, `rpcuser`/`rpcpassword`), the `/` and `/wallet/<name>` endpoints, positional vs. named parameters and the `args` convention, JSON-RPC 1.1 vs. 2.0 differences, batching, HTTP status and RPC error codes (-28 warmup, -18 wallet not found, -19 wallet not specified, -5, -8, -32601), `rpcbind`/`rpcallowip`/`rpcworkqueue` configuration, `/rest/` paths, and `-zmqpubrawblock`/`-zmqpubsequence` topics. Triggers on "bitcoin RPC", "bitcoind JSON-RPC", "curl my bitcoin node", "connect to Bitcoin Core from Python/Node/Go", "Postman bitcoin", "401 Unauthorized from bitcoind", "connection refused on 8332", "which port is regtest RPC", "how do I authenticate to my node", "getblockchaininfo over HTTP", "REST interface", "rest/chaininfo.json", "ZMQ notifications", "get notified when a new block arrives", "watch the mempool from my app", and on phrasings that never name the protocol — "my app can't reach my node", "how do I read my node from a script", "poll for new transactions".
---

# bitcoin-api

A Bitcoin Core node exposes three network interfaces. This skill is about speaking to
them directly — from `curl`, from Postman, from application code — rather than through
the `bitcoin-cli` wrapper.

| Interface | Transport | Auth | What it is for |
|---|---|---|---|
| **JSON-RPC** | HTTP POST, default port 8332 | HTTP Basic, **required** | The full API. Everything `bitcoin-cli` can do. |
| **REST** | HTTP GET, *same port*, `-rest=1` | **None** | Read-only chain data. No wallet access, by design. |
| **ZMQ** | TCP/IPC publish-subscribe, separate port | **None** | Push notifications for new blocks and transactions. |

`bitcoin-cli` is a thin client over the first of these. Anything it does, an HTTP request
can do — and understanding the wire protocol is what lets you debug the cases where it
doesn't work.

## Scope

- Connecting and authenticating: `bitcoin.conf`, the cookie file, `rpcauth`, ports, binding
- The JSON-RPC wire protocol: request/response shape, endpoints, parameters, batching
- Reading errors: HTTP status codes and Bitcoin's own RPC error codes
- The REST interface: paths, formats, and what it deliberately cannot do
- ZMQ notifications: topics, message framing, and their delivery guarantees (there are none)
- Calling all of the above from application code

Out of scope: driving the node from a shell with `bitcoin-cli` (see `bitcoin-cli`), how
the node implements any of this (see `bitcoin-code`), monetary theory (see `money`), and
third-party block-explorer APIs such as mempool.space or Esplora — this skill is about
**your** node.

## The 30-second version

Cookie auth, which is the default and needs no configuration:

```bash
curl --user "$(cat "$HOME/Library/Application Support/Bitcoin/.cookie")" \
     --data-binary '{"jsonrpc":"2.0","id":"1","method":"getblockchaininfo","params":[]}' \
     -H 'content-type: application/json' \
     http://127.0.0.1:8332/
```

Three things go wrong most often, in this order:

1. **Connection refused** — the node isn't running, or `server=1` isn't set. `bitcoind`
   enables RPC by default; **`bitcoin-qt` does not.**
2. **401 Unauthorized** — stale cookie. It is regenerated on every restart; re-read it.
3. **`-28` "Loading block index"** — the node is up but still warming up. Retry, don't fail.

See `references/connection.md`.

## How to use this skill

**Pick the right interface.** Reach for REST for plain read-only chain data in a
performance-sensitive path — it skips JSON-RPC's auth and envelope, and serves raw binary.
Reach for ZMQ when you'd otherwise poll. Everything else is JSON-RPC.

**Never poll `getblockcount` in a loop** when ZMQ is available. Even with ZMQ, treat a
notification as a *hint* that something changed and confirm via RPC — ZMQ has no delivery
guarantee and no replay.

**Read the error code, not the error string.** Strings change between releases; the
numeric codes in `references/json-rpc.md` are stable and are what you should branch on.

**Check which wallet you're talking to.** The bare `/` endpoint services wallet calls only
when *exactly one* wallet is loaded. With zero or several, you get `-18` or `-19` and the
fix is to address `/wallet/<name>` explicitly. Doing that unconditionally is the better
habit — see `references/json-rpc.md`.

**State the version.** The RPC interface is implicitly versioned on Core's major version,
and methods do get deprecated. When an answer depends on behavior that has changed, say
which version you checked against and get it from `getnetworkinfo` → `version` /
`subversion` rather than assuming.

## Safety

The RPC interface can spend your money. Core's own documentation is blunt about this: it
allows other programs "to spend funds from your wallets, affect consensus verification,
read private data."

- **Never call a spending or signing method on mainnet without explicit, per-call
  confirmation of the exact amount and destination.** That covers `sendtoaddress`,
  `sendmany`, `send`, `sendall`, `walletcreatefundedpsbt`, `signrawtransactionwithwallet`,
  `signrawtransactionwithkey`, and `sendrawtransaction`. Read-only methods are fine to run
  freely.
- **Treat the cookie file and `rpcauth` string as secrets.** Pass them to `curl` via
  `--user` reading from the file; never echo them into terminal output, log files,
  scratch files, artifacts, or commit them.
- **Never `dumpwallet`, `dumpprivkey`, or `listdescriptors true`** on a user's behalf, and
  never write their output anywhere. These return private keys.
- **Never suggest exposing the RPC port to the internet.** It is unencrypted HTTP Basic
  auth — credentials cross the wire in the clear. If remote access is genuinely needed,
  the answer is an SSH tunnel or a VPN, not `rpcallowip=0.0.0.0/0`.
- **The REST interface is unauthenticated.** Enabling `-rest` means anything that can
  reach the port can read your chain data. It is also reachable from a browser, which
  makes it an XSS target — Core documents this risk directly.
- Prefer regtest or signet for anything exploratory.

## Reference material

Load these on demand; don't read them all for a passing question. Each file opens with a
"Read this when" note and a table of its own contents.

| File | Read it when | Verified? |
|---|---|---|
| `references/connection.md` | You **can't reach the node** — auth, ports, `bitcoin.conf`, binding, and a symptom-first troubleshooting table | partly |
| `references/json-rpc.md` | Calls get through and you're **debugging what comes back** — endpoints, parameters, 1.1 vs. 2.0, batching, error codes | **yes** |
| `references/rest.md` | You want **unauthenticated read-only** chain data, or raw binary | docs only |
| `references/zmq.md` | You'd otherwise **poll in a loop** — topics, framing, and delivery caveats | docs only |
| `references/recipes.md` | You want **something that runs** — shell, Python, JavaScript, Postman, a health check | **yes** |
| `references/sources.md` | You need to **check a claim** upstream, or you're about to edit this skill | — |

The "Verified?" column matters: **yes** means exercised against a live Bitcoin Core
v31.0.0 node; **docs only** means transcribed from Core's documentation but never observed
running. Weight your confidence accordingly, and say which you're relying on when it
matters. `sources.md` has the details.

## Status

Written against **Bitcoin Core v31.0** — `doc/JSON-RPC-interface.md`,
`doc/REST-interface.md`, `doc/zmq.md`, and `src/rpc/protocol.h`.

The JSON-RPC material and every code example were verified against a live mainnet Core
v31.0.0 node, which corrected several claims. REST and ZMQ were not: that node had `-rest`
disabled and no `-zmqpub*` endpoints. The table above marks which is which, and
`references/sources.md` records exactly what was tested, what changed, and what wasn't.

Where a detail is version-sensitive, the reference files say so.
