---
name: bitcoin-code
description: Use when researching how Bitcoin actually works at the implementation and specification level — locating where a consensus rule, policy check, or behaviour lives in the Bitcoin Core source, tracing a code path, finding when and why something changed, reading a BIP, or mapping a BIP onto the code that implements it. Covers setting up local clones of bitcoin/bitcoin and bitcoin/bips, wiring clangd for go-to-definition, the source tree layout, git archaeology (`git log -S`, `git blame`, release tags), pull-request and review history via `gh`, and Core's own `doc/bips.md` and `doc/AI_POLICY.md`. Triggers on mentions of bitcoin/bitcoin, Bitcoin Core source, validation.cpp, script interpreter, mempool policy, consensus rules in code, a symbol like CheckBlock/ConnectBlock/EvalScript/MAX_MONEY, a BIP number, a Core PR number, and on questions phrased without any of those — "where is the 21 million cap enforced", "when did this rule change and why", "how does Core decide to relay a transaction", "show me how signature verification actually works", "which release shipped taproot", "what does this PR change", "is this behaviour consensus or policy", "clone the bitcoin source", "set up a language server for the bitcoin code". Read-only research: it never opens or drives pull requests, per Core's `doc/AI_POLICY.md`. Not for running or driving a node — that is `bitcoin-cli`, `bitcoin-api`, and `bitcoin-install`; not for monetary theory — that is `money`.
---

# bitcoin-code

Research tooling for the Bitcoin implementation and its specifications. Two repositories
answer most questions:

| Repository | Is | Answers |
|---|---|---|
| [`bitcoin/bitcoin`](https://github.com/bitcoin/bitcoin) | The implementation | What the network **actually does** |
| [`bitcoin/bips`](https://github.com/bitcoin/bips) | The specifications | What was **proposed and agreed** |

When they disagree, **the code is what the network enforces** — a BIP is a description,
and the deployed software is the thing miners and nodes run. Say which one you are quoting.

## Scope

- Setting up local clones, and wiring clangd for real code intelligence
- The source tree: where consensus, policy, script, wallet, net, and RPC live
- Answering "where is X enforced" with `file:line` evidence at a pinned ref
- Git archaeology: when a rule appeared, which release shipped it, who changed it and why
- Pull requests and review history via `gh`
- BIPs: reading them, and mapping BIP ↔ implementing code

## Related skills

This skill owns **the source and the specifications** — what the code says, at a named ref,
with `file:line` evidence.

| When the question moves to… | Hand off to |
|---|---|
| Running a node, composing commands, reading output | **`bitcoin-cli`** |
| The RPC / REST / ZMQ interfaces as *interfaces* | **`bitcoin-api`** |
| Installing Bitcoin Core in order to *use* it | **`bitcoin-install`** |
| Whether any of this is *money* | **`money`** |

This is the **terminal skill** for evidence: every other skill in this marketplace hands
off here when an answer stops being "what does the tool do" and becomes "what does the code
enforce, and since when". Concretely — `bitcoin-cli` and `bitcoin-api` send you here for
the rule behind a result or an error, and `money` sends you here whenever a claim about
Bitcoin's supply, fungibility, or immutability needs checking against the source instead of
being asserted. Answer those with a citation, then hand the conversation back.

The clone this skill sets up is a research asset the whole marketplace shares — once it
exists, prefer it over recalling Core's behaviour from memory anywhere.

## Ground rules

**Cite or don't claim.** Every statement about what the code does carries a `file:line`
reference at a named ref. Bitcoin's behaviour is too consequential to describe from memory,
and memory of this codebase goes stale every release.

**Pin the version.** Always state which ref you read — `master` at a given sha, or a
release tag like `v31.1`. "Bitcoin Core does X" without a ref is an unfinished answer.

```bash
git -C "$BITCOIN_SRC" rev-parse --short HEAD
```

**Distinguish consensus from policy.** A rule in `src/consensus/` or reached from
`ConnectBlock` makes blocks invalid for everyone. A rule in `src/policy/` only governs what
*this node* relays or mines — other nodes may differ, and it can change without a fork.
Conflating them is the most common substantive error in Bitcoin source answers.

**Distinguish the spec from the implementation.** A BIP's status — `Draft`, `Complete`,
`Deployed`, or `Closed` under the current process — says nothing about whether Bitcoin Core
implements it. `doc/bips.md` in the Core tree is the authoritative mapping. Note the status
vocabulary itself changed when BIP 3 replaced BIP 2, so older write-ups cite values like
`Proposed` and `Final` that no longer exist. See `references/bips.md`.

**Read-only.** This skill reads upstream source. It never commits, pushes, fetches into a
user's checkout, changes their checked-out ref, or builds.

## Core's AI policy — read this before contributing anything

Bitcoin Core ships [`doc/AI_POLICY.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/AI_POLICY.md).
It permits AI as a coding tool but draws hard lines that this skill must respect:

- **"Pull requests should not be opened or driven by autonomous agents."** A human must
  choose the work, understand the change, and be responsible for it.
- **"Do not include agents as authors or co-authors of your commits."**
- **AI-written comments to maintainers are not acceptable** — issue and review comments are
  expected to be written by humans, and may be moderated if they appear AI-generated.
- Only open a PR if you know the language, could have written the code yourself, and
  understand the surrounding code.
- If you include AI-derived context in a comment, disclose it and add human commentary.

**Research and reading are entirely fine — that is what this skill is for.** Drafting a
contribution *for a human who will understand and own it* is fine. Opening or driving a PR,
or ghost-writing review comments, is not. If a user asks for that, say so plainly and point
them at the policy rather than complying.

## Safety

This skill reads; it does not write. The risks are to the user's checkout and to their
standing with upstream, not to their money.

- **Never modify a user's clone.** No commits, no pushes, no `fetch` into their checkout,
  no changing their checked-out ref, no builds. If a stale clone is a problem, say so and
  let them update it. A user may have work in progress you cannot see.
- **Never open, comment on, or drive a pull request or issue upstream**, per Core's
  `doc/AI_POLICY.md` — see above. Drafting for a human who will own the result is fine.
- **Never state a `file:line`, a constant, or a behaviour without having read it** at a
  named ref. Inventing a plausible path is worse than saying you don't know: this codebase
  is quoted in arguments about money.
- **Cloning is a multi-hundred-MB fetch.** Ask first, and ask where it should go.

## How to use this skill

**Prefer a local clone.** `Grep` and `Read` over a checkout are faster, complete, and
rate-limit free, and they unlock `git log`/`blame`, which is where most research value
lives. If none exists, ask before cloning — it is a multi-hundred-MB fetch — and ask *where
to put it*. See `references/setup.md`.

**Search before you assume.** The tree is large and names are reused. Start from
`references/tree-map.md`, then grep. Exclude `src/test/`, `src/bench/`, and `src/qt/` from a
first pass unless the question is about them.

**Answer "why" with git, not with prose.** `git log -S` finds the commit that introduced a
constant; `git blame` names the change behind a line; the merge commit names the PR, and
the PR carries the rationale and the review. See `references/research.md` and
`references/prs-and-history.md`.

**Use the `bitcoin-source-reader` agent** for searches that would otherwise dump a lot of
file content into the conversation. It is read-only and returns findings with citations.

## Reference material

Load on demand. Each file opens with a "Read this when" note and its own contents table.

| File | Read it when | Verified? |
|---|---|---|
| `references/setup.md` | Cloning, or wiring clangd for go-to-definition | **yes** |
| `references/tree-map.md` | You need to know *where to look* | **yes** |
| `references/research.md` | Finding a rule, or when and why it changed | **yes** |
| `references/prs-and-history.md` | Following a change to its PR and review | **yes** |
| `references/bips.md` | Reading a BIP, or mapping BIP ↔ code | **yes** |
| `references/sources.md` | Checking a claim, or editing this skill | — |

## Status

Written and verified against a full local clone of `bitcoin/bitcoin` (50,403 commits,
master at 2026-09-02) and `bitcoin/bips` (210 BIP files), with `gh` against the live
GitHub API. Symbol locations and line numbers move between releases — re-derive them rather
than trusting the ones quoted here.
