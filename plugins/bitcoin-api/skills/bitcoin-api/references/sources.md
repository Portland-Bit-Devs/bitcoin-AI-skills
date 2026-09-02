# Sources

> **Read this when** you need to check a claim against upstream, or you are about to edit
> this skill and want to know which parts are transcribed from Core and which are ours.

**In this file**

| Section | Answers |
|---|---|
| Primary | Core's own docs and headers — the authority for everything here |
| The node itself | Why `help` beats any web page |
| Verification | **What was tested live, what was corrected, what was not tested** |
| Secondary / Prior art | Useful surrounding material, including the Postman collection |
| Forks | Why Knots may not match |
| Licensing note | What we may and may not reproduce |

---

This skill is a synthesis of Bitcoin Core's own documentation and source, restated rather
than reproduced. Everything here is verifiable against the links below — go to them when
an answer matters.

## Primary — Bitcoin Core, MIT licensed

Read at tag **v31.0**. The RPC interface is implicitly versioned on Core's major version,
so check the tag matching the node you are actually talking to.

| Document | Covers |
|---|---|
| [`doc/JSON-RPC-interface.md`](https://github.com/bitcoin/bitcoin/blob/v31.0/doc/JSON-RPC-interface.md) | Endpoints, parameter passing, 1.1 vs. 2.0, security, consistency guarantees |
| [`doc/REST-interface.md`](https://github.com/bitcoin/bitcoin/blob/v31.0/doc/REST-interface.md) | Every `/rest/` path, formats, limitations, XSS risk |
| [`doc/zmq.md`](https://github.com/bitcoin/bitcoin/blob/v31.0/doc/zmq.md) | Topics, message framing, sequence numbers, delivery caveats |
| [`src/rpc/protocol.h`](https://github.com/bitcoin/bitcoin/blob/v31.0/src/rpc/protocol.h) | The authoritative `RPCErrorCode` and `HTTPStatusCode` enums |
| [`share/rpcauth/rpcauth.py`](https://github.com/bitcoin/bitcoin/blob/v31.0/share/rpcauth/rpcauth.py) | Generating static `rpcauth` credentials |
| [`contrib/zmq/zmq_sub.py`](https://github.com/bitcoin/bitcoin/blob/v31.0/contrib/zmq/zmq_sub.py) | A working ZMQ subscriber |

The error-code tables in `json-rpc.md` are transcriptions of the `RPCErrorCode` enum. When
they disagree with the header, **the header is right** — regenerate them from it.

## The node itself

The most reliable reference is the node in front of you, and it is always current for its
own version:

```bash
# Every method, grouped by category
btc help

# Full signature, parameters, result shape, and examples for one method
btc help '"getblockchaininfo"'
```

`help` output is the same text that reaches the RPC documentation sites, generated from
the same source. Prefer it to any third-party page.

## Verification

The JSON-RPC reference material was exercised against a live **mainnet Bitcoin Core
v31.0.0** node (`/Satoshi:31.0.0/`, pruned, `getnetworkinfo` → `version: 310000`) on
2026-09-01. Read-only calls only.

**Confirmed as documented:** the 1.1 vs. 2.0 envelope difference (2.0 returns exactly one
of `result`/`error`; 1.1 returns both); 2.0 application errors arriving as HTTP `200`
while 1.1 returns `404` for `-32601` and `500` for application errors; 2.0 notifications
(no `id`) returning `204` with an empty body; positional, named, and `args`-convention
parameter passing; batching with an error in one item not failing the batch; `GET` to the
JSON-RPC endpoint returning `405 JSONRPC server handles only POST requests`; the
`/wallet/<name>` endpoint also servicing non-wallet methods; missing or bad credentials
returning `401` with an empty body; and the `-3`, `-5`, `-8`, `-18`, `-32600`, `-32601`,
`-32700` codes.

**Corrected as a result** — these were wrong in the first draft:

| Claim | Reality on the node |
|---|---|
| Core rejects non-JSON content types | Not enforced at all; a missing header and `application/x-www-form-urlencoded` both succeed |
| Malformed JSON → HTTP `400` | `-32700` → HTTP **`500`**, in a 1.1-shaped response even when the body carried `"jsonrpc": "2.0"` — the marker is unreadable in a body that failed to parse |
| Unloaded wallet → HTTP `404` | `-18` at HTTP `200` under 2.0, `500` under 1.1 |
| `-18`/`-19` both mean "wrong number of wallets" | `-18` = zero loaded, or a named wallet not loaded; `-19` = two or more loaded |
| Empty `getzmqnotifications` means ZMQ may not be compiled in | Compiled-in-but-unconfigured returns `[]`; a build without ZMQ omits the method entirely (`-32601`) |

**Example code:** the shell helper, the batch example, the Python client, the JavaScript
client, and the monitoring health check in `recipes.md` were all executed against that node
and produce the output they claim, including their error paths. Running the health check
also surfaced a bug since fixed: jq's `halt_error` emits its string without a trailing
newline.

**Not verified — documentation-derived only:**

- Every `/rest/` path. The node had `-rest` disabled; all paths returned an empty-bodied
  `404`, which confirms only the disabled-interface signature.
- All ZMQ topics and message framing. ZMQ was compiled in (`Zmq` appears in `help`) but no
  `-zmqpub*` endpoints were configured.
- Error `-19`, which requires two or more wallets loaded. The verification node had none,
  and loading wallets on someone's mainnet node is a write.
- Every code in the error tables not listed above. Those are transcribed from
  `src/rpc/protocol.h` at v31.0 and are as reliable as that file.

For reference, v31.0 exposes **151 methods** across nine `help` categories: Blockchain,
Control, Mining, Network, Rawtransactions, Signer, Util, Wallet, Zmq.

## Secondary

- **[Bitcoin Core RPC API reference](https://bitcoincore.org/en/doc/)** — the `help` output
  rendered per release. Convenient; pin the version selector to your node's version.
- **[BIP64](https://github.com/bitcoin/bips/blob/master/bip-0064.mediawiki)** — outpoint
  serialization used by the REST `getutxos` endpoint.
- **[JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification)** — the base
  protocol, including the by-position and by-name parameter structures Core implements.
- **[ZeroMQ API](https://libzmq.readthedocs.io/en/zeromq4-x/)** — endpoint specifiers and
  socket options for the ZMQ interface.

## Prior art

- **[StevenBlack/bitcoin-postman](https://github.com/StevenBlack/bitcoin-postman)** — a
  Postman collection for querying a node. The request shape it establishes (POST to a bare
  host:port, Basic auth from environment variables, a raw JSON body) is the pattern the
  Postman section of `recipes.md` follows. Worth knowing about; treat its contents with
  care. It covers roughly 35 methods, its `Rawtransactions` and `Util` folders are empty,
  at least one request body does not match its own name, and its README states it is
  "under development, not ready for general consumption." It also predates the
  `/wallet/<name>` endpoint handling that multi-wallet nodes require.

## Forks

Bitcoin Knots and other forks expose RPC surfaces that are close to Core's but not
identical — extra methods, extra options, occasionally different defaults. Nothing here is
guaranteed for a non-Core node. Check `getnetworkinfo` → `subversion` to find out what you
are actually talking to, then consult that project's documentation.

## Licensing note

Bitcoin Core is MIT licensed. This skill restates its documented interface — protocol
shapes, endpoint paths, numeric error codes, and configuration option names — which are
factual API surface rather than creative expression. No substantial passages are
reproduced. The example code in `recipes.md` is original and ships under this repository's
GPL-3.0.
