# bitcoin-api

Talk to a Bitcoin Core node over its own network interfaces, rather than through the
`bitcoin-cli` wrapper.

**Status:** v0.1 — reference material written against Bitcoin Core v31.0 documentation
and source.

```bash
claude plugin install bitcoin-api@bitcoin-ai-skills
```

Provides the `bitcoin-api` skill, covering all three interfaces a node exposes:

| Interface | Transport | Auth | For |
|---|---|---|---|
| **JSON-RPC** | HTTP POST, port 8332 | HTTP Basic, required | The full API |
| **REST** | HTTP GET, same port, `-rest=1` | none | Read-only chain data |
| **ZMQ** | pub/sub, separate port | none | Block and transaction notifications |

Includes: the `.cookie` / `rpcauth` / `rpcuser` authentication methods, the `/` vs.
`/wallet/<name>` endpoints, positional and named parameter passing, JSON-RPC 1.1 vs. 2.0
(and why the difference will bite you), batching, the complete RPC error-code table, every
`/rest/` path, ZMQ topics and message framing, and worked examples in shell, Python,
JavaScript, and Postman.

## Relationship to the other skills

- **`bitcoin-cli`** — the shell wrapper. Use that for a human at a terminal; use this for
  application code and for debugging the wire.
- **`bitcoin-code`** — how Core *implements* any of this.
- **`money`** — monetary theory. Unrelated.

Third-party explorer APIs (mempool.space, Esplora) are deliberately out of scope. This
skill is about your node.

## Safety

The RPC interface can spend your money. The skill instructs Claude never to call a
spending or signing method on mainnet without explicit per-call confirmation of amount and
destination, never to read or echo private keys or the cookie credential, and never to
recommend exposing the RPC port to the internet.

## Editing this skill

```
skills/bitcoin-api/
├── SKILL.md              always loaded — keep it short; it routes to the rest
└── references/
    ├── connection.md     can't reach the node
    ├── json-rpc.md       reaching it, debugging the response
    ├── rest.md           unauthenticated read-only
    ├── zmq.md            push instead of poll
    ├── recipes.md        runnable examples
    └── sources.md        provenance + verification log
```

Only `SKILL.md` is always in context. Everything under `references/` is loaded on demand,
which is why detail belongs there and `SKILL.md` stays a map. If you add a reference file,
add a row to the routing table in `SKILL.md` — otherwise it will never be read.

**Know what you're editing.** Three kinds of content live here, and they have different
rules:

| Kind | Where | Rule |
|---|---|---|
| Transcribed from Core | The error-code tables in `json-rpc.md`; the path lists in `rest.md`; the topic table in `zmq.md` | Re-derive from upstream rather than hand-editing. The error tables deliberately mirror the grouping and order of [`src/rpc/protocol.h`](https://github.com/bitcoin/bitcoin/blob/v31.0/src/rpc/protocol.h) so the two can be diffed. Marked with an editing note in the file. |
| Observed on a real node | Anything `sources.md` lists under Verification | Don't "correct" these from memory or from a web page — several were originally wrong *because* they came from memory. Re-test against a node. |
| Ours | Every "Meaning" column, all prose, the recommendations, every code example | Edit freely. Improving the explanations is the easiest way to make this skill better. |

**If you change a factual claim, update `sources.md`.** Its Verification section is the
record of what has actually been tested, and it is what stops the next person from
trusting an unverified claim.

**Re-verifying against your own node.** Point read-only calls at it and compare. The gaps
worth closing first are the two marked *docs only* in `SKILL.md`: enable `-rest=1` and
confirm the paths in `rest.md`, and configure a `-zmqpub*` endpoint to confirm the topics
and framing in `zmq.md`. Error `-19` also needs two wallets loaded to reproduce. Prefer
regtest — see the repository's safety rules in
[CONTRIBUTING.md](../../CONTRIBUTING.md).

## Evals

Three cases under `evals/`, in the `prompt.md` + `graders/` form:

- `multi-wallet-endpoint` — diagnosing RPC error `-19` as an endpoint problem
- `auth-connection-failure` — knowing `bitcoin-qt` disables the HTTP RPC server by default
- `push-vs-poll` — recommending ZMQ *and* getting its delivery guarantees right

These have not been executed: `claude plugin eval` is currently early-access and was not
available on the authoring account.

Licensed GPL-3.0. See the [repository root](../../README.md).
