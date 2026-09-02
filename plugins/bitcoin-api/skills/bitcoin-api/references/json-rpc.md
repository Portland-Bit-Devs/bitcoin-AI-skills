# The JSON-RPC wire protocol

> **Read this when** you are writing or debugging the HTTP calls themselves — the request
> body, the response envelope, or an error code you don't recognise. For getting
> *connected* in the first place (auth, ports, config), read `connection.md` instead.

**In this file**

| Section | Answers |
|---|---|
| The two endpoints | `/` vs. `/wallet/<name>`, and why to always use the latter |
| Parameter passing | Positional, named, and the `args` convention |
| JSON-RPC 1.1 vs. 2.0 | Which one you're speaking, and the HTTP-status trap |
| Batching | Many calls in one round trip |
| Error codes | HTTP statuses and the full `RPC_*` tables — **start with the quick table** |
| Versioning and deprecation | Asking the node what it is |
| Consistency guarantees | What two successive calls do and don't promise |

---

Every call is an HTTP `POST` with a JSON body to one of two endpoints, authenticated with
HTTP Basic. There is no other shape to learn.

```bash
curl --user "$(cat "$DATADIR/.cookie")" \
     --data-binary '{"jsonrpc":"2.0","id":"1","method":"getblockcount","params":[]}' \
     -H 'content-type: application/json' \
     http://127.0.0.1:8332/
```

Send `content-type: application/json`. Core v31 does **not** actually enforce it — a
missing header or `application/x-www-form-urlencoded` is accepted — but it costs nothing,
it is what the documented examples use, and relying on the leniency is how you get burned
by a proxy or a future release that does check.

## The two endpoints

### `/`

Always active. Services every non-wallet method, and services wallet methods **only when
exactly one wallet is loaded**.

### `/wallet/<walletname>`

Active when the wallet component is compiled in. Services both wallet and non-wallet
methods. **Required** when two or more wallets are loaded. This is what `bitcoin-cli` uses
under the hood when you pass `-rpcwallet=`.

```bash
curl --user "$(cat "$DATADIR/.cookie")" \
     --data-binary '{"jsonrpc":"2.0","id":"1","method":"getbalance","params":[]}' \
     -H 'content-type: application/json' \
     http://127.0.0.1:8332/wallet/mywallet
```

**Address `/wallet/<name>` unconditionally** in any application that touches a wallet.
Code that works against the bare `/` endpoint is code that silently depends on the node
having exactly one wallet loaded, and it breaks the day someone loads a second one — with
a `-19` that looks like a code change rather than an environment change.

URL-encode names containing special characters. The default wallet created by older
versions has the empty name, addressed as `/wallet/`.

## Parameter passing

Three forms, all supported.

**By position** — a JSON array, arguments in declaration order:

```json
{"method": "createwallet", "params": ["mywallet", false, false, "", false, false, true]}
```

**By name** — a JSON object. Order is irrelevant and you skip the defaults you don't care
about, which is why this is the right choice for anything non-trivial:

```json
{"method": "createwallet", "params": {"wallet_name": "mywallet", "load_on_startup": true}}
```

**Mixed** — every method accepts a special named parameter `args` holding an array of
leading positional values, combined with named values for the rest:

```json
{"method": "createwallet", "params": {"args": ["mywallet"], "load_on_startup": true}}
```

Positional calls are the ones that break on upgrade: a parameter inserted in the middle of
a signature silently shifts everything after it. Named calls don't have that failure mode.

## JSON-RPC 1.1 vs. 2.0

Core speaks both. A request is treated as 2.0 **if and only if** the body contains
`"jsonrpc": "2.0"`; otherwise it falls back to the legacy 1.1 behavior, which was the only
option through v27.0.

| | 1.1 | 2.0 |
|---|---|---|
| Request marker | `"version": "1.1"`, or nothing | `"jsonrpc": "2.0"` |
| Response marker | none | `"jsonrpc": "2.0"` |
| `error` and `result` fields | **both** present, one is null | **only one** present |
| HTTP status on an RPC error | non-200 (invalid params, method not found, …) | **200** — only genuine HTTP failures are non-200 |
| Notifications (no reply) | not supported | supported: omit `id`, get `204 No Content` |

One exception to the status row: a **JSON parse error escapes the 2.0 rules entirely**.
The server cannot see your `"jsonrpc": "2.0"` marker in a body it failed to parse, so it
answers `500` with a 1.1-shaped response (`result` and `error` both present, no `jsonrpc`
key) no matter what you sent. Don't let a strict 2.0 response parser choke on it.

That HTTP-status row is the one that bites. Under 1.1, a client can branch on the status
code. Under 2.0, an application-level error such as "insufficient funds" arrives as
`200 OK` with an `error` object in the body — a client that only checks `resp.ok` will
treat a failed send as a success.

**Always inspect the body's `error` field.** Under 2.0 it is the only signal.

Prefer 2.0 for new code: one unambiguous field, standards-conformant, and no null
`result` alongside a populated `error`.

## Batching

Send an array of request objects; get back an array of responses. The `id` field is how
you match them up — **responses are not guaranteed to arrive in request order.**

```bash
curl --user "$(cat "$DATADIR/.cookie")" \
     --data-binary '[
       {"jsonrpc":"2.0","id":"h","method":"getblockcount","params":[]},
       {"jsonrpc":"2.0","id":"i","method":"getblockchaininfo","params":[]}
     ]' \
     -H 'content-type: application/json' http://127.0.0.1:8332/
```

One failing item does not fail the batch. Each element carries its own success or error,
and you must check every one — the outer HTTP status tells you only that the batch was
delivered.

Batching amortizes HTTP and auth overhead, which matters when walking a range of blocks.
It does not make the node execute anything in parallel.

## Error codes

Two independent layers: the HTTP status, and Bitcoin's own numeric code in the response
body's `error.code`. **Branch on the numeric code.** Error *messages* are prose and change
between releases; the codes are stable, and are defined in `src/rpc/protocol.h`.

```json
{"result": null, "error": {"code": -18, "message": "Requested wallet does not exist or is not loaded"}, "id": "1"}
```

### Start here: the codes you'll actually hit

Ninety percent of real debugging is one of these. The full tables below are for the rest.

| Code | Means | Do this |
|---|---|---|
| `-28` | Node is still warming up | Retry with backoff — **not** a failure |
| `-18` | Zero wallets loaded, or the wallet in your URL isn't loaded | `loadwallet`, or fix the path |
| `-19` | Two or more wallets loaded, bare `/` used | Address `/wallet/<name>` |
| `-5` | Usually "tx not in mempool and no `txindex`" | Enable `txindex`, or pass a block hash |
| `-8` | Bad parameter value — including an unknown *named* parameter | Check spelling and the method's signature |
| `-3` | Wrong parameter *type* | Number sent as a string, usually |
| `-1` | Catch-all — e.g. `"Block not available (pruned data)"` | Read the message; it is specific |
| `-32601` | No such method | Typo, or a method your version/build lacks |
| `-32600` | Malformed request object → HTTP 400 | Missing `method`, or bad `params` type |
| `-32700` | Body wasn't valid JSON → HTTP 500 | Your serialisation is broken |

### HTTP status codes

| Code | Meaning |
|---|---|
| 200 | OK — under JSON-RPC 2.0, *also* how application errors arrive |
| 204 | No Content — response to a 2.0 notification |
| 400 | Bad Request — a malformed *request object* (`-32600`): missing `method`, non-string `method`, bad `params` type |
| 401 | Unauthorized — bad or stale credentials |
| 403 | Forbidden — client IP not permitted by `rpcallowip` |
| 404 | Not Found — unknown endpoint, or `-32601` under 1.1. **Not** how an unloaded wallet is reported |
| 405 | Method Not Allowed — you sent GET to the JSON-RPC endpoint (REST is the GET interface) |
| 500 | Internal Server Error — under 1.1, the generic application-error status; **also a JSON parse error (`-32700`) under either version** |
| 503 | Service Unavailable — work queue full |

> **Editing note.** The four tables below are a transcription of the `RPCErrorCode` enum
> in [`src/rpc/protocol.h`](https://github.com/bitcoin/bitcoin/blob/v31.0/src/rpc/protocol.h),
> and they keep that file's grouping and order on purpose so the two can be diffed. If you
> update them, re-derive them from the header for the version you care about rather than
> hand-editing rows. The *"Meaning"* column is ours and is free to improve.

### Standard JSON-RPC codes

| Code | Name | Meaning |
|---|---|---|
| -32600 | `RPC_INVALID_REQUEST` | Malformed request object → HTTP 400 |
| -32601 | `RPC_METHOD_NOT_FOUND` | No such method → HTTP 404 |
| -32602 | `RPC_INVALID_PARAMS` | Parameters don't match the signature |
| -32603 | `RPC_INTERNAL_ERROR` | A genuine bug or corruption in the node |
| -32700 | `RPC_PARSE_ERROR` | Body was not valid JSON → HTTP **500**, not 400 |

### General application codes

| Code | Name | Meaning |
|---|---|---|
| -1 | `RPC_MISC_ERROR` | Catch-all exception during command handling |
| -3 | `RPC_TYPE_ERROR` | Wrong type passed for a parameter |
| -5 | `RPC_INVALID_ADDRESS_OR_KEY` | Bad address or key — **also** "no such transaction" from `getrawtransaction` without `txindex` |
| -7 | `RPC_OUT_OF_MEMORY` | Out of memory |
| -8 | `RPC_INVALID_PARAMETER` | Invalid, missing, or duplicate parameter |
| -20 | `RPC_DATABASE_ERROR` | Database error |
| -22 | `RPC_DESERIALIZATION_ERROR` | Couldn't parse a raw hex structure |
| -25 | `RPC_VERIFY_ERROR` | General failure submitting a transaction or block |
| -26 | `RPC_VERIFY_REJECTED` | Rejected by network rules — the usual `sendrawtransaction` failure |
| -27 | `RPC_VERIFY_ALREADY_IN_UTXO_SET` | Transaction is already confirmed |
| -28 | `RPC_IN_WARMUP` | **Node still starting up. Retry with backoff.** |
| -32 | `RPC_METHOD_DEPRECATED` | Re-enable with `-deprecatedrpc=` or migrate |

### P2P and chain codes

| Code | Name | Meaning |
|---|---|---|
| -9 | `RPC_CLIENT_NOT_CONNECTED` | Node has no P2P connections |
| -10 | `RPC_CLIENT_IN_INITIAL_DOWNLOAD` | Still doing IBD |
| -23 / -24 | `RPC_CLIENT_NODE_ALREADY_ADDED` / `NOT_ADDED` | `addnode` bookkeeping |
| -29 | `RPC_CLIENT_NODE_NOT_CONNECTED` | Peer to disconnect not found |
| -30 | `RPC_CLIENT_INVALID_IP_OR_SUBNET` | Bad IP or subnet |
| -31 | `RPC_CLIENT_P2P_DISABLED` | No connection manager |
| -33 | `RPC_CLIENT_MEMPOOL_DISABLED` | No mempool instance |
| -34 | `RPC_CLIENT_NODE_CAPACITY_REACHED` | Connection slots full |

### Wallet codes

| Code | Name | Meaning |
|---|---|---|
| -4 | `RPC_WALLET_ERROR` | Unspecified wallet problem, e.g. key not found |
| -6 | `RPC_WALLET_INSUFFICIENT_FUNDS` | Not enough funds |
| -11 | `RPC_WALLET_INVALID_LABEL_NAME` | Invalid label |
| -12 | `RPC_WALLET_KEYPOOL_RAN_OUT` | Call `keypoolrefill` |
| -13 | `RPC_WALLET_UNLOCK_NEEDED` | Encrypted wallet is locked — `walletpassphrase` first |
| -14 | `RPC_WALLET_PASSPHRASE_INCORRECT` | Wrong passphrase |
| -15 | `RPC_WALLET_WRONG_ENC_STATE` | e.g. encrypting an already-encrypted wallet |
| -16 | `RPC_WALLET_ENCRYPTION_FAILED` | Encryption failed |
| -17 | `RPC_WALLET_ALREADY_UNLOCKED` | Already unlocked |
| -18 | `RPC_WALLET_NOT_FOUND` | **Wallet not loaded, or wrong name in the URL path** |
| -19 | `RPC_WALLET_NOT_SPECIFIED` | **Several wallets loaded — you must use `/wallet/<name>`** |
| -35 | `RPC_WALLET_ALREADY_LOADED` | Already loaded |
| -36 | `RPC_WALLET_ALREADY_EXISTS` | A wallet by that name exists |

`-2` is retired (`RPC_FORBIDDEN_BY_SAFE_MODE`) and must not be reused.

### The ones worth handling explicitly

- **`-28`** on startup. Not an error condition — the node is loading the block index. Any
  client that starts alongside the node must retry with backoff rather than crash.
- **`-18` vs `-19`** are distinguished by how many wallets are loaded, and the messages
  say which: **zero** wallets loaded gives `-18` *"No wallet is loaded"*; a
  `/wallet/<name>` path naming a wallet that isn't loaded gives `-18` *"Requested wallet
  does not exist or is not loaded"*; **two or more** loaded with a bare `/` gives `-19`.
  In every case the fix is the URL, not the call.
- **`-5` from `getrawtransaction`** usually means the transaction isn't in the mempool and
  `txindex=1` isn't enabled, rather than that the txid is wrong. The message says so
  outright: *"No such mempool transaction. Use -txindex or provide a block hash…"*

On a **pruned** node, note that `getblockhash` still succeeds for any height — the header
chain is complete — while `getblock` on the returned hash fails with `-1` *"Block not
available (pruned data)"*. So a successful `getblockhash` is not evidence the block body
is retrievable, and code that chains the two must handle `-1` on the second call.

## Versioning and deprecation

The RPC interface is implicitly versioned on Core's major version. Deprecated features are
normally re-enablable for a one-major-version grace period via `-deprecatedrpc=<feature>`;
release notes name each one. After that, they're gone.

Get the version from the node rather than assuming:

```json
{"jsonrpc":"2.0","id":"1","method":"getnetworkinfo","params":[]}
```

`version` is a packed integer (e.g. `310000`); `subversion` is the human-readable user
agent (e.g. `/Satoshi:31.0.0/`). `subversion` is also how you detect that you are talking
to a fork such as Bitcoin Knots rather than Core — their RPC surfaces are close but not
identical, and Knots adds methods and options Core does not have.

## Consistency guarantees

Documented, and narrower than people assume:

- Any RPC reflects at least the chain state immediately before the call.
- Mempool state is self-consistent and consistent with the chain at call time, but is
  **not** guaranteed current with the live mempool.
- Wallet state is consistent with the chain, but may lag the mempool. A wallet transaction
  replaced in the mempool may not yet show as replaced.

So: don't build a system that assumes two successive RPCs saw the same mempool.
