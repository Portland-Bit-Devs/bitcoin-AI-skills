# ZMQ notifications

> **Read this when** you would otherwise poll the node in a loop. ZMQ tells you *that*
> something changed; you still confirm *what* over JSON-RPC.
>
> **Status: documentation-derived.** The verification node had ZMQ compiled in but no
> endpoints configured, so none of the topics or framing below were observed — see
> `sources.md`.

**In this file**

| Section | Answers |
|---|---|
| Enabling | Config, and telling "not compiled in" from "not configured" |
| Topics | The five topics and their message framing |
| Subscribing | The empty-subscription trap, with a working example |
| Delivery is not guaranteed | **The part people get wrong** |
| Interaction with assumeutxo | A silence you might not expect |

---

ZeroMQ publish/subscribe sockets that push a message when a block or transaction arrives,
so you don't have to poll. Read-only, unauthenticated, one-way: the node publishes, you
subscribe, and nothing you do can affect the node through this interface.

## Enabling

ZMQ is **not compiled in by default** when building from source, though release binaries
normally include it. A node built without it silently ignores every `-zmqpub*` option,
which is a confusing failure — no messages, no error. Check with:

```json
{"jsonrpc":"2.0","id":"1","method":"getzmqnotifications","params":[]}
```

Read the result carefully, because the two failure modes look different:

- **`-32601 Method not found`** — ZMQ is not compiled into this build. No configuration
  will help; you need a different binary.
- **`[]` (empty array)** — ZMQ *is* compiled in, but no `-zmqpub*` endpoints are
  configured. Add them to `bitcoin.conf` and restart.
- **A populated array** — working; it lists each active topic and address.

If you are building yourself, compile with `cmake -B build -DWITH_ZMQ=ON`.

Then configure endpoints, in `bitcoin.conf` or on the command line:

```conf
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashblock=tcp://127.0.0.1:28332
zmqpubsequence=tcp://127.0.0.1:28334
```

The same address may serve several notification types, and the same type may be published
to several addresses. `unix:/tmp/bitcoind.tx.raw` works too. Each has an optional
high-water-mark option (`-zmqpubrawtxhwm=n`, …) capping how many messages queue for a slow
subscriber before they are dropped.

Bind these to localhost. There is no authentication, and Core's guidance is that the port
should be exposed only to trusted entities via firewalling.

## Topics

| Topic | Body | Fires when |
|---|---|---|
| `hashblock` | 32-byte block hash, reversed | Chain tip updated |
| `rawblock` | serialized block | Chain tip updated |
| `hashtx` | 32-byte txid, reversed | Transaction enters mempool, **or** arrives in a block |
| `rawtx` | serialized transaction | Same as `hashtx` |
| `sequence` | see below | Every mempool add/remove and block connect/disconnect |

Every message has three parts: **topic**, **body**, **sequence number** (4-byte
little-endian unsigned int, counted separately per topic).

Hashes are in **reversed byte order** — the display order used by the RPC interface and
block explorers, not the internal hashing order. If your txids come out backwards, this is
why.

### `sequence` — the one to use for mempool tracking

The body is a hash followed by a one-character type tag, and for the mempool events an
8-byte little-endian mempool sequence number:

| Body | Meaning |
|---|---|
| `<32-byte block hash>C` | Block **c**onnected |
| `<32-byte block hash>D` | Block **d**isconnected |
| `<32-byte txid>A<8-byte LE uint>` | Transaction **a**dded to mempool |
| `<32-byte txid>R<8-byte LE uint>` | Transaction **r**emoved from mempool, not via block inclusion |

`sequence` is more informative than the `*block` topics because it reports **every** block
connection and disconnection. The `hashblock` and `rawblock` topics only announce the new
tip: during a reorg you are told where the chain ended up, not what was undone. If you
care about reorgs, subscribe to `sequence`.

### Duplicate transaction notifications are normal

`rawtx` and `hashtx` fire when a transaction enters the mempool *and again* for each block
that includes it. A transaction can legitimately be announced several times. Deduplicate
on txid; don't treat a repeat as a new event.

## Subscribing

The subscriber must set `ZMQ_SUBSCRIBE` to a topic prefix. **An empty subscription
receives nothing** — this is ZeroMQ semantics, not a Core quirk, and it's the most common
reason a first attempt sees silence. A prefix works: subscribing to `hash` gets both
`hashblock` and `hashtx`.

```python
import zmq

ctx = zmq.Context()
sock = ctx.socket(zmq.SUB)
sock.setsockopt_string(zmq.SUBSCRIBE, "hashblock")
sock.connect("tcp://127.0.0.1:28332")

while True:
    topic, body, seq = sock.recv_multipart()
    print(topic.decode(), body.hex(), int.from_bytes(seq, "little"))
```

Core ships a fuller example at `contrib/zmq/zmq_sub.py`.

Sockets are self-connecting and self-healing: either end can start or stop in any order
and the connection re-establishes. You do not need reconnect logic, but you **do** need to
handle the gap that an outage leaves.

## Delivery is not guaranteed — design for it

This is the part people get wrong. ZMQ notifications can be lost: a subscriber that is
slow or disconnected past the high-water mark simply misses messages, and there is no
replay, no acknowledgement, and no backfill.

The per-topic sequence number exists precisely so you can **detect** the loss — a gap
means you missed something. It does not help you recover the content.

So the correct pattern is:

> **Treat a notification as a hint that something changed, then confirm the actual state
> over RPC or REST.**

Never treat a ZMQ message as the authoritative record of an event, and never build
accounting on the assumption that you saw every message. Core's own documentation warns
that subscribers should validate received data since it "may be out of date, incomplete or
even invalid" — the node publishes what it received from the P2P network, and publication
is not a claim of validity.

On startup, and after any detected gap, reconcile against the node rather than assuming
you are in sync.

## Interaction with assumeutxo

When `assumeutxo` is in use, `rawblock` and `hashblock` are **not** issued for historical
blocks connected to the background validation chainstate. A client watching those topics
during background sync will see nothing for that work.
