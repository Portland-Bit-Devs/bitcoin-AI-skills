---
name: bitcoin-code
description: Use when a question is about the Bitcoin Core implementation itself — where a rule lives in the source, how a code path works, what a specific function or file does, what changed in a release, or how a BIP is actually implemented. Triggers on mentions of bitcoin/bitcoin, Bitcoin Core source, validation.cpp, consensus rules in code, script interpreter, mempool policy, wallet descriptors, or a specific symbol like CheckBlock/ConnectBlock/AcceptToMemoryPool, and on questions phrased without those names — "where is the 21 million cap enforced", "how does Core decide to relay a transaction", "show me how signature verification actually works", "what does this Core PR change".
---

# bitcoin-code

> **Status: stub.** Structure is in place; the reference material is not written yet.

Questions about Bitcoin's rules are ultimately questions about code: the consensus
rules are whatever the implementation does. This skill answers them by reading
`github.com/bitcoin/bitcoin` and citing what it finds.

## Scope

- Locating code in the Bitcoin Core tree: consensus, validation, policy, script, wallet, RPC, net
- Explaining a code path end to end, with `file:line` citations
- Mapping a BIP or a described behavior onto the code that implements it
- Reading a specific commit, tag, or pull request

Out of scope: running a node (see `bitcoin-cli`), and monetary theory (see `money`).

## How to use this skill

Dispatch the `bitcoin-code:bitcoin-source-reader` agent for anything that requires
actually searching the tree. It is read-only and returns findings with citations.
Answer directly from this skill only for orientation questions ("which directory
holds X") that don't need the source in front of you.

## Ground rules

- **Cite or don't claim.** Every statement about what the code does must carry a
  `file:line` reference at a named commit or tag. Bitcoin's behavior is too consequential
  to describe from memory — and memory of this codebase goes stale every release.
- **Pin the version.** Always state which ref you read (`master` at a given sha, or a
  release tag like `v28.0`). "Bitcoin Core does X" without a ref is an unfinished answer.
- **Read-only.** This skill reads upstream source. It never modifies a Bitcoin Core
  checkout, opens PRs, or pushes anything.

## TODO

- [ ] `references/tree-map.md` — directory-by-directory map of the Core source tree
- [ ] `references/key-paths.md` — the well-known code paths (block validation, tx relay, script eval)
- [ ] `references/bip-index.md` — BIP → implementing files
- [ ] `evals/evals.json` — cases covering a "where is X enforced" and a "explain this path" question
