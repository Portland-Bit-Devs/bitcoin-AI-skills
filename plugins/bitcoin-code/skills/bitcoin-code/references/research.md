# Research techniques

> **Read this when** the question is not "what does this code do" but "when did this
> change", "why is it like this", or "which release shipped it".

**In this file**

| Section | Answers |
|---|---|
| The four questions | A workflow for most research |
| Where is X enforced | Finding the rule |
| When did X change | `git log -S` and pickaxe searching |
| Why is X like this | `git blame` → commit → PR |
| Which release shipped X | Tags, `--contains`, release notes |
| Comparing releases | Reading two refs without checking out |
| Pitfalls | The mistakes that produce confident wrong answers |

---

## The four questions

Almost every source question is one of these, and each has a tool:

| Question | Tool |
|---|---|
| Where is X enforced? | `grep`, then clangd `findReferences` |
| When did X change? | `git log -S` / `git log -G` |
| Why is it like this? | `git blame` → merge commit → PR discussion |
| Which release has it? | `git tag --contains`, `doc/release-notes/` |

Chain them. A complete answer usually cites a location, a commit, and a release.

## Where is X enforced

Start from `tree-map.md` to pick a directory, then grep with the noise filtered out.
Worked example — the 21 million cap, verified at master on 2026-09-02:

```bash
grep -rn "MAX_MONEY" src/consensus/amount.h
# src/consensus/amount.h:26: inline constexpr CAmount MAX_MONEY{21'000'000 * COIN};
# src/consensus/amount.h:27: inline bool MoneyRange(const CAmount& nValue) { ... }
```

The constant alone is not the answer. **The enforcement is wherever it is checked**, and
for consensus purposes that is the transaction-validity path:

```bash
grep -rn "MoneyRange" src/consensus/
# src/consensus/tx_check.cpp:40   — context-free output check
# src/consensus/tx_verify.cpp:192 — input values
# src/consensus/tx_verify.cpp:209 — fee
```

Note there is no single "21 million" line anywhere. The cap is a *consequence* of the
subsidy schedule plus the range check — `GetBlockSubsidy` halving every
`nSubsidyHalvingInterval` blocks, with `MoneyRange` bounding every value. Saying "it's
enforced at `amount.h:26`" is a citation without an explanation; give both.

## When did X change

`git log -S` (the "pickaxe") finds commits where the **number of occurrences** of a string
changed — that is, where it was introduced or removed:

```bash
git log -S "MAX_MONEY" --oneline --reverse -- src/ | head -3
# 84c3fb07b0 directory re-organization (keeps the old build system)
# ba4081c1fc move back to original directory structure
# 65786afb05 Add various tests for CTransaction::CheckTransaction()
```

Note what that output teaches: the earliest hits are **file moves**, not the introduction
of the concept. The pickaxe tracks a string in a path, and Bitcoin's tree has been
reorganised repeatedly. Widen the path or follow renames:

```bash
git log -S "MAX_MONEY" --oneline --reverse --follow -- src/consensus/amount.h
git log -S "MAX_MONEY" --oneline --reverse            # whole repo, no path filter
```

Use `-G` instead of `-S` to match a **regex appearing in the diff** at all, rather than a
change in occurrence count — better for "when did this line get modified":

```bash
git log -G "MoneyRange.*nValueOut" --oneline -- src/consensus/
```

Add `-p` to see the actual diffs, and `--reverse` to read oldest-first.

## Why is X like this

`git blame` names the commit behind a line. Anchor it with a regex range so it survives
line-number drift:

```bash
git blame -L '/MAX_MONEY{21/,+1' --date=short src/consensus/amount.h
# fab74a0e922 (MarcoFalke 2026-08-04 26) inline constexpr CAmount MAX_MONEY{21'000'000 * COIN};
```

That gives a commit, an author, and a date — but often it is a refactor rather than the
decision. Two moves get past that:

```bash
git log --follow -p -L '/MAX_MONEY{21/,+1':src/consensus/amount.h | head -60   # line history
git blame -w -C -C --date=short src/consensus/amount.h                        # ignore whitespace & moved code
```

`-w -C -C` is the important one: it looks through whitespace changes and code movement to
find where a line *really* came from, which is what defeats "the last commit was a
reformat".

Then follow the commit to its pull request — see `prs-and-history.md`.

## Which release shipped X

Bitcoin Core tags every release. Given a commit:

```bash
git tag --contains 2702711c3a | grep -E '^v[0-9]+\.[0-9]+$' | sort -V | head -3
# v31.0
# v31.1
```

**An empty result is meaningful, not a failure.** Verified: `git tag --contains fab74a0e922`
returns nothing, because that commit is dated 2026-08-04 while `v31.1` was tagged
2026-07-06 — it is on `master` and has shipped in no release. "It's in the code" and "it's
in a release" are different claims, and this is how you tell them apart.

Filter to real releases with `grep -E '^v[0-9]+\.[0-9]+$'`; the repo also carries rc and
assorted build tags that make raw output noisy.

Available tags:

```bash
git tag --list 'v3*' --sort=-v:refname | head
# v31.1  v31.1rc1  v31.0  v31.0rc4 ...
```

Release notes are in the tree and are written for humans:

```bash
ls doc/release-notes/ | tail -5
grep -rn "taproot" doc/release-notes/release-notes-22.0.md | head -3
```

For a rule with an *activation height* rather than a release, the parameters are in
`src/kernel/chainparams.cpp` — a feature can ship in one release and activate much later.
Keep "shipped" and "active" separate.

## Comparing releases

Read any file at any ref **without touching the working tree** — essential when working in
a user's checkout:

```bash
git show v28.0:src/consensus/amount.h | head -30
diff <(git show v28.0:src/policy/policy.h) <(git show v31.1:src/policy/policy.h) | head -40
git diff v28.0..v31.1 --stat -- src/consensus/
```

That last one is a good opening move for "what changed in consensus between these
releases" — the answer is usually "very little", which is itself informative.

## Pitfalls

**Don't confuse a constant with its enforcement.** Find where it is *checked*.

**Don't trust line numbers — or syntax.** Both drift. Verified on the same constant:

```
v28.0   src/consensus/amount.h:26   static constexpr CAmount MAX_MONEY = 21000000 * COIN;
master  src/consensus/amount.h:26   inline constexpr CAmount MAX_MONEY{21'000'000 * COIN};
```

Same file, same line number, different declaration — so a grep pattern tuned to master
(`MAX_MONEY{21`) finds **nothing** at `v28.0`, and it looks like the constant is absent
rather than spelled differently. Search for the bare identifier first, then narrow. Cite
`file:line` *at a stated ref*, and re-derive rather than reusing a number from memory or
from this file.

**Watch for vendored subtrees.** `src/secp256k1/`, `src/leveldb/`, `src/minisketch/`,
`src/univalue/`, and `src/crc32c/` are imported from other projects. Their history in this
repo is squashed subtree merges; real history lives upstream. Don't attribute that code to
Core contributors or expect meaningful `blame`.

**Watch for file moves.** The tree was reorganised several times early on, so plain
pickaxe results often surface moves. Use `--follow`, or search without a path filter.

**Check whether a name still exists** in the version you're answering about. Symbols get
renamed and split; a confident answer about `AcceptToMemoryPool` is wrong if the question is
about a version that structures it differently. Verify against the ref you cite.

**Tests often answer "what does this rule mean"** faster than the implementation. See
`tree-map.md` on `test/functional/`.
