# BIPs: the specifications

> **Read this when** the question is about what was *specified* rather than what Core
> implements — reading a BIP, checking its status, or mapping a BIP to the code.

**In this file**

| Section | Answers |
|---|---|
| What a BIP is and isn't | **The distinction people get wrong** |
| The repository | Layout and file formats |
| The preamble | Reading BIP metadata |
| **Statuses** | **The vocabulary changed — old write-ups are stale** |
| Finding a BIP | By number, topic, or layer |
| **BIP → code** | **`doc/bips.md`, the authoritative mapping** |
| Code → BIP | The other direction |

---

## What a BIP is and isn't

A BIP is a **proposal document**. It is not a standard, not a specification the network is
obliged to follow, and not evidence that anything implements it. Anyone may open one.

Three independent facts get conflated constantly:

| Question | Answered by |
|---|---|
| Was it proposed and written up? | The BIP exists |
| Did the process accept it? | Its `Status` field |
| Does Bitcoin Core implement it? | **`doc/bips.md` in the Core tree** |
| Is it active on mainnet? | Activation parameters in `src/kernel/chainparams.cpp` |

A `Deployed` BIP that Core implements may still not be active on a given network, and a
`Draft` BIP may be widely implemented elsewhere. Answer the question that was asked, and
say which of these you checked.

## The repository

```bash
git clone https://github.com/bitcoin/bips.git
```

Small — ~22 MB transferred, ~43 MB on disk, 4,777 commits, **210 BIP files** at the ref
checked. Layout:

```
README.mediawiki        the index table of every BIP
bip-0341.mediawiki      the BIP itself
bip-0341/               optional directory: test vectors, diagrams, reference code
CONTRIBUTING.md
```

**Two file formats coexist.** Verified counts: 196 `.mediawiki` and 14 `.md`. Older BIPs
are MediaWiki; newer ones are Markdown. Always glob for both:

```bash
ls bip-0341.* 2>/dev/null       # → bip-0341.mediawiki
ls bip-0003.* 2>/dev/null       # → bip-0003.md  and  bip-0003/
```

The per-BIP subdirectory is easy to miss and often holds the most useful material —
`bip-0341/` carries `wallet-test-vectors.json` and a diagram. If you need to *implement*
or *verify* something, look there before re-deriving it.

## The preamble

Every BIP opens with a metadata block. Verified, from `bip-0341.mediawiki`:

```
BIP: 341
Layer: Consensus (soft fork)
Title: Taproot: SegWit version 1 spending rules
Authors: Pieter Wuille, Jonas Nick, Anthony Towns
Status: Deployed
Type: Specification
Assigned: 2020-01-19
License: BSD-3-Clause
Discussion: <mailing-list links>
Requires: 340
```

`Requires` and `Replaces` matter for research: BIP 341 requires BIP 340 (Schnorr), so
answering a taproot question from 341 alone misses the signature scheme. Follow the graph.

`Layer` tells you how consequential it is. Distribution across the repo:

| Layer | Count | Means |
|---|---|---|
| Applications | 94 | Wallet/app conventions — no network rules |
| Consensus (soft fork) | 50 | Tightens rules; old nodes still accept |
| Peer Services | 29 | P2P protocol |
| Consensus (hard fork) | 12 | Would split the network |
| API/RPC | 4 | Interface conventions |

## Statuses

**The vocabulary changed.** [BIP 3](https://github.com/bitcoin/bips/blob/master/bip-0003.md)
("Updated BIP Process", Status: `Deployed`) **replaced BIP 2**, which is now `Closed`.
Current values, with counts verified across the repo:

| Status | Count | Means |
|---|---|---|
| `Deployed` | 78 | In use on the network |
| `Closed` | 57 | Withdrawn, rejected, or superseded |
| `Draft` | 53 | Proposed, under discussion |
| `Complete` | 22 | Specification finished; adoption is a separate question |

Older commentary — and plenty of blog posts — cite BIP 2's vocabulary: `Proposed`,
`Final`, `Active`, `Deferred`, `Rejected`, `Withdrawn`, `Replaced`, `Obsolete`. **Those
values no longer appear** in current BIP headers. If a source describes a BIP as "Final",
it is using a retired vocabulary; check the current header rather than repeating it.

```bash
grep -A1 "^  Status:" bip-0341.mediawiki
```

## Finding a BIP

```bash
# By number — remember both extensions and the subdirectory
ls bip-0341.* bip-0341/ 2>/dev/null

# By topic, across all BIPs
grep -ril "silent payments" bip-*.mediawiki bip-*.md | head

# By title
grep -h "^  Title:" bip-*.mediawiki bip-*.md | grep -i taproot

# All consensus soft forks
grep -l "^  Layer: Consensus (soft fork)" bip-*.mediawiki bip-*.md | head
```

`README.mediawiki` holds the master index table — layer, number, title, owner, type,
status — and is worth reading directly when browsing rather than targeting.

## BIP → code

**`doc/bips.md` in the Bitcoin Core tree is the authoritative mapping**, and it is far
better than guessing: it names the BIP, the Core version that implemented it, and the PR.
Verified format:

> `BIP 16`: The pay-to-script-hash evaluation rules have been implemented since **v0.6.0**,
> and took effect on *April 1st 2012* ([PR #748](https://github.com/bitcoin/bitcoin/pull/748)).

**Grep for the file name, not the phrase.** Related BIPs are grouped into one multi-line
entry, so `grep "BIP 341"` misses taproot entirely — the line reads just `` [`341`] ``:

```bash
grep -n "bip-0341" "$BITCOIN_SRC/doc/bips.md"          # reliable
grep -n -B2 -A4 "bip-0341" "$BITCOIN_SRC/doc/bips.md"  # with the surrounding entry
```

Verified: BIP 340, 341 and 342 share a single entry stating that taproot validation rules
"are implemented as of **v0.21.0** ([PR 19953](https://github.com/bitcoin/bitcoin/pull/19953))".

Three things that file gives you for free: whether Core implements it at all, the **version**
it landed in, and the **PR** carrying the rationale. That is a complete research trail in
one line, and it beats grepping the source for a concept name.

**If a BIP is not listed in `doc/bips.md`, Core does not implement it** — which is a
finding, not a dead end. Say so rather than hunting for code that isn't there.

## Code → BIP

Going the other way, from an implementation detail to its specification:

```bash
# Core's source cites BIPs directly in comments
grep -rn "BIP341\|BIP 341" "$BITCOIN_SRC/src" --include="*.cpp" --include="*.h" | head
# src/key.h:169  "in BIP341:"

# Functional tests are named after features
ls "$BITCOIN_SRC/test/functional/" | grep -i taproot
# feature_taproot.py
# wallet_taproot.py
```

**Script verification flags are the most direct bridge.** Each `SCRIPT_VERIFY_*` in
`src/script/interpreter.h` corresponds to a soft fork, and the flag name finds both the
rule and its enforcement.

### A worked warning: don't grep for a deployment that no longer exists

The obvious move for "where is taproot activated" is to look for a BIP 9 deployment. That
returns **nothing**, and the wrong conclusion is "taproot isn't in this code":

```bash
grep -rn "DEPLOYMENT_TAPROOT" "$BITCOIN_SRC/src"     # no results
```

Taproot's BIP 9 deployment was **removed after activation**. Buried deployments in
`src/consensus/params.h` now stop at `DEPLOYMENT_SEGWIT`, and taproot is enforced
unconditionally — verified at `src/validation.cpp:2261`, where `SCRIPT_VERIFY_TAPROOT` is
part of the base flag set rather than gated on any deployment. Core's own docs were even
updated to say so (PR #35183, "recommend script_flags instead of deployments.taproot",
merged 2026-05-11).

The general lesson, and the reason this skill insists on pinning a ref: **an activated soft
fork's activation machinery gets deleted.** Absence of a deployment constant is evidence
the fork is long since active, not evidence it is missing. Search for the *rule*
(`SCRIPT_VERIFY_*`), not the *activation*.
