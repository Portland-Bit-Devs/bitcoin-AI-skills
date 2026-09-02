# The REST interface

> **Read this when** you want read-only chain data without authentication, or in raw
> binary — typically for indexing or bulk chain walking. Anything involving a wallet, or
> any write, has to go through JSON-RPC instead.
>
> **Status: verified.** Every path below was exercised against a live Bitcoin Core v31.1
> regtest node with `-rest=1` — see `sources.md`.

**In this file**

| Section | Answers |
|---|---|
| What it is for / cannot do | Why there is no wallet access, ever |
| Security | The unauthenticated-plus-browser risk |
| Paths | Every `/rest/` endpoint, by category |
| Consistency and limits | Shared guarantees, and the shared crash |

---

Enabled with `-rest=1`. Runs on the **same port as JSON-RPC** — 8332 on mainnet — but
answers `GET` requests at `/rest/…` paths, with **no authentication at all**.

```bash
curl http://127.0.0.1:8332/rest/chaininfo.json
```

If `-rest` is **off**, every `/rest/` path returns **HTTP 404 with an empty body** — no
JSON, no error message. An empty-bodied 404 on a path you're sure is correct means the
interface is disabled, not that your path is wrong.

## What it is for

Read-only chain data, cheaply. No auth handshake, no JSON-RPC envelope, and — importantly
— the option of raw binary output, so you can pull a block without paying for hex encoding
and JSON parsing. That makes it the right interface for indexers and anything walking the
chain in bulk.

## What it deliberately cannot do

**There is no wallet access, and never will be.** That is the design: an unauthenticated
interface must not reach anything that holds keys or spends money. If you need wallet
data, you need JSON-RPC.

It also cannot write. No `sendrawtransaction`, no `generatetoaddress`, no configuration.

## Security

Anything that can reach the port can read everything REST serves. There is no credential
to get wrong, which cuts both ways.

The specific documented risk is a browser: because REST is plain `GET` on localhost, a
malicious web page you visit on the same machine can issue requests like
`<script src="http://127.0.0.1:8332/rest/tx/<txid>.json">` and learn things about your
node. Core names this as a real risk of running a browser on a REST-enabled node. Leave
`-rest` off unless something needs it.

## Paths

Every path takes a format suffix — `.bin` (raw serialized), `.hex`, or `.json` — except
those noted as JSON-only.

### Transactions

```
GET /rest/tx/<TX-HASH>.<bin|hex|json>
```

**Searches the mempool only, by default.** To retrieve a confirmed transaction you need
`txindex=1`. Returns `404` if not found — and "not found" here usually means "not indexed"
rather than "does not exist". Confirmed: on a node without `txindex`, requesting a
*confirmed* transaction that certainly exists still returns `404`.

### Blocks

```
GET /rest/block/<BLOCK-HASH>.<bin|hex|json>
GET /rest/block/notxdetails/<BLOCK-HASH>.<bin|hex|json>
GET /rest/blockpart/<BLOCK-HASH>.<bin|hex>?offset=<OFFSET>&size=<SIZE>
```

`notxdetails` returns transaction hashes instead of full transaction objects, and affects
the JSON response only. `blockpart` returns a byte range — useful for large blocks when
you need only a slice.

Request and response are handled **entirely in memory**. A few thousand concurrent
full-block requests is a memory problem.

### Headers

```
GET /rest/headers/<BLOCK-HASH>.<bin|hex|json>?count=<COUNT=5>
```

Returns `COUNT` headers walking **upward** (toward the tip) from the given hash. Empty
result if the block doesn't exist or isn't in the active chain — note that's an empty
result, not a `404`.

The older positional form `/rest/headers/<COUNT>/<BLOCK-HASH>.<…>` is deprecated as of
24.0 but still works.

### Block filters (BIP157/158)

```
GET /rest/blockfilter/<FILTERTYPE>/<BLOCK-HASH>.<bin|hex|json>
GET /rest/blockfilterheaders/<FILTERTYPE>/<BLOCK-HASH>.<bin|hex|json>?count=<COUNT=5>
```

`FILTERTYPE` is `basic` in practice. Requires `blockfilterindex=1` on the node — without
it both endpoints return **HTTP 400**, not 404. A 400 here means "index disabled", not
"bad request".
The positional-count form of `blockfilterheaders` is likewise deprecated since 24.0.

### Block hash by height

```
GET /rest/blockhashbyheight/<HEIGHT>.<bin|hex|json>
```

The REST equivalent of `getblockhash`, and the usual entry point when iterating by height.

### Spent transaction outputs

```
GET /rest/spenttxouts/<BLOCK-HASH>.<bin|hex|json>
```

One list of spent outputs per transaction in the block — the undo data. `404` if the
block's undo data isn't available, which is the normal case on a **pruned** node for
blocks outside the retained window.

### UTXO set queries

```
GET /rest/getutxos/<TXID>-<N>/<TXID>-<N>/….<bin|hex|json>
GET /rest/getutxos/checkmempool/<TXID>-<N>/<TXID>-<N>/….<bin|hex|json>
```

Query a set of outpoints against the UTXO set; the `checkmempool` variant also considers
unconfirmed spends. Multiple outpoints in one request, slash-separated.

The response carries a `bitmap` — a string of `1`/`0` characters, one per requested
outpoint, saying which were found. The `utxos` array contains **only the hits**, so you
must use the bitmap to line results back up with your request. Getting this wrong silently
misattributes outputs.

Binary and hex serialization follow
[BIP64](https://github.com/bitcoin/bips/blob/master/bip-0064.mediawiki).

### Chain and deployment info — JSON only

```
GET /rest/chaininfo.json
GET /rest/deploymentinfo.json
GET /rest/deploymentinfo/<BLOCKHASH>.json
```

Mirror `getblockchaininfo` and `getdeploymentinfo`.

### Mempool — JSON only

```
GET /rest/mempool/info.json
GET /rest/mempool/contents.json?verbose=<true|false>&mempool_sequence=<false|true>
```

Mirror `getmempoolinfo` and `getrawmempool`. The `verbose` and `mempool_sequence` query
parameters exist from 25.0 onward; `contents.json` defaults to `verbose=true`, which on a
busy mainnet mempool is a very large response — pass `verbose=false` unless you need the
fee data.

## Consistency and limits

REST offers the [same consistency guarantees as JSON-RPC](json-rpc.md#consistency-guarantees).

It shares the same file-descriptor exhaustion bug, too: several hundred simultaneous
connections can **crash the node**. Reuse connections and bound your concurrency.
