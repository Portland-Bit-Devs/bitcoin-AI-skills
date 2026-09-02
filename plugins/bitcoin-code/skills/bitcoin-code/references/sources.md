# Sources

> **Read this when** checking a claim, or before editing this skill.

**In this file**

| Section | Answers |
|---|---|
| Verification | **What was run, and what it produced** |
| Primary sources | The repositories, and Core's own docs |
| Corrections | Draft claims this process falsified |
| Not verified | Honest gaps |
| Editing note | Why line numbers here rot |

---

## Verification

Performed on 2026-09-02 against full local clones: `bitcoin/bitcoin` at master
`b811aeabad` (50,403 commits, 387 MB on disk) and `bitcoin/bips` at `7273e17` (210 BIP
files, 43 MB), plus `gh` against the live GitHub API.

**Repository facts** — clone sizes, commit counts, BIP file counts and the 196 `.mediawiki`
/ 14 `.md` split, per-BIP subdirectories, layer and status distributions: all counted
directly.

**Tree layout** — per-directory file counts, the contents of `src/consensus/`,
`src/policy/`, `src/script/`, `src/kernel/`, and the 282 functional tests: enumerated.

**Symbol locations** — `CheckBlock` (`validation.cpp:2319`), `ConnectBlock` (`:222`),
`CheckTransaction` (`:795`), `ContextualCheckBlockHeader` (`:2308`), `GetBlockSubsidy`
(`:1833`), `CheckProofOfWork` (`:3839`), `EvalScript` (`interpreter.cpp:417`), `MAX_MONEY`
(`consensus/amount.h:26`), `MAX_BLOCK_WEIGHT` and `COINBASE_MATURITY`
(`consensus/consensus.h:15,19`): all grepped at that ref.

**Research commands** — `git log -S`, `git log -G`, `git blame -L`, `git blame -w -C -C`,
`git tag --contains`, `git show <tag>:<path>`, `git diff <tag>..<tag> --stat`, and the
noise-filtered grep: all executed with the outputs quoted.

**GitHub commands** — `gh pr view` (36048 rendered as expected), `gh api .../reviews`,
`gh search prs`, and the `--ancestry-path` merge lookup (correctly resolved a commit to
PR #34422): all executed.

**clangd** — the three tiers were measured on a fresh clone with nothing installed.
`clangd --check` reported `bitcoin-build-config.h' file not found` on `validation.cpp`
without a CMake configure; a 3-line `compile_flags.txt` (`-std=c++20 -Isrc -I.`) took
`src/consensus/tx_check.cpp` to **0 errors**. `cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON`
was confirmed to fail without Boost. The `clangd-lsp@claude-plugins-official` plugin was
installed and confirmed **not** to take effect until Claude Code restarts — the LSP tool
still returned `No LSP server available for file type: .h` afterwards.

## Primary sources

| Source | Used for |
|---|---|
| [`bitcoin/bitcoin`](https://github.com/bitcoin/bitcoin) | Everything about the implementation |
| [`bitcoin/bips`](https://github.com/bitcoin/bips) | Specifications, statuses, test vectors |
| [`doc/bips.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/bips.md) | **The authoritative BIP → version → PR mapping** |
| [`doc/AI_POLICY.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/AI_POLICY.md) | Contribution limits for AI, quoted directly |
| [`doc/developer-notes.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/developer-notes.md) | Conventions and codebase orientation |
| [BIP 3](https://github.com/bitcoin/bips/blob/master/bip-0003.md) | The current BIP process and status vocabulary |

The repositories are the authority. Where this skill and the tree disagree, the tree wins.

## Corrections

Claims in the first draft that verification falsified — recorded because each is a mistake
a reader could otherwise repeat:

| Draft claim | Reality |
|---|---|
| `AcceptToMemoryPool` no longer exists | It is still present at `src/validation.h:273`, alongside `ProcessNewPackage` |
| BIP statuses are `Draft`/`Proposed`/`Final`/`Deployed` | BIP 3 replaced BIP 2 and the vocabulary is now `Draft`/`Complete`/`Deployed`/`Closed`; `Proposed` and `Final` no longer appear |
| `grep "BIP 341" doc/bips.md` finds the taproot entry | It does not — 340/341/342 share one grouped entry whose line reads `` [`341`] ``. Grep for `bip-0341` |
| `DEPLOYMENT_TAPROOT` is in `chainparams.cpp` | It was removed after activation; taproot is enforced unconditionally at `validation.cpp:2261` |
| `git show v28.0:...` finds `MAX_MONEY{21` | At v28.0 the declaration was `static constexpr CAmount MAX_MONEY = 21000000 * COIN;` — syntax drifted, so a master-tuned pattern finds nothing |

## Not verified

- **The LSP tool working end to end on Bitcoin source.** The plugin is installed and clangd
  is confirmed working via `--check`, but the tool itself was never exercised against the
  tree, because plugin language servers load at session start. Untested after restart.
- **Tier 2 (CMake configure).** Confirmed to *fail* without Boost; never completed, because
  the user chose not to install build dependencies. The commands come from Core's
  `doc/build-osx.md`.
- **`git log --ancestry-path` on a deep history.** It worked on one commit; not stress-tested.
- **Windows and Linux paths.** Setup guidance is macOS-flavoured.

## Editing note

**The line numbers in this skill will rot.** They were correct at master `b811aeabad` on
2026-09-02 and will drift with every release. They are included because a concrete starting
point beats a vague one — not because they can be trusted later. Any answer built on them
must re-derive and re-cite at its own ref.

The durable content is the *structure*: consensus vs. policy, the directory map, which tool
answers which question, `doc/bips.md` as the BIP→code bridge, and the pitfalls. Those have
held for years. When you update this file, re-run the commands rather than editing remembered
values, and add any newly falsified claim to the Corrections table — it is the most useful
part of this file.
