# bitcoin-code

Research the Bitcoin implementation and its specifications.

**Status:** v0.1 — verified against full local clones of `bitcoin/bitcoin` and `bitcoin/bips`.

```bash
claude plugin install bitcoin-code@bitcoin-ai-skills
```

Two repositories answer most questions — [`bitcoin/bitcoin`](https://github.com/bitcoin/bitcoin)
for what the network **actually does**, and [`bitcoin/bips`](https://github.com/bitcoin/bips)
for what was **proposed**. When they disagree, the code is what nodes enforce.

Provides the `bitcoin-code` skill and the read-only `bitcoin-source-reader` agent, covering
local checkout setup (including interviewing the user about where it goes), clangd code
intelligence, the source tree map, git archaeology, PR and review history via `gh`, and
BIP ↔ code mapping.

## What makes this more than grep

- **Consensus vs. policy** — the distinction that decides whether an answer is right.
  Consensus makes a block invalid for everyone; policy governs only what one node relays.
- **`doc/bips.md`** — Core's own BIP → version → PR mapping. One line gives you whether
  Core implements a BIP, which release shipped it, and the PR carrying the rationale.
- **Git archaeology** — `git log -S`, `git blame -w -C -C`, and merge commits that name
  their PR, because Bitcoin's *reasoning* lives in review threads, not commit messages.
- **Pinned refs** — line numbers move every release, so every claim carries the sha it was
  read at.

## Core's AI policy

Bitcoin Core ships [`doc/AI_POLICY.md`](https://github.com/bitcoin/bitcoin/blob/master/doc/AI_POLICY.md),
and this skill respects it explicitly. Research, reading and explaining are exactly what
the skill is for. But **pull requests must not be opened or driven by autonomous agents**,
agents must not be commit co-authors, and comments to maintainers are expected to be
written by humans. The skill declines those and says why rather than complying.

## Code intelligence

The skill documents three tiers, all measured:

| Tier | Setup | Result |
|---|---|---|
| 0 | nothing | Near useless on Core — wrong C++ standard, broken includes |
| 1 | a 3-line `compile_flags.txt` | **0 errors** on `src/consensus/tx_check.cpp`, no dependencies |
| 2 | `cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | Full intelligence; needs Boost |

A full build is never required. Wiring clangd into Claude Code needs
`claude plugin install clangd-lsp@claude-plugins-official`, `ENABLE_LSP_TOOL=1`, and — this
one is easy to miss — **a restart**, since language servers load at session start.

## Editing this skill

```
skills/bitcoin-code/
├── SKILL.md                 always loaded — ground rules, AI policy, routing
└── references/
    ├── setup.md             clones, the checkout interview, clangd tiers
    ├── tree-map.md          where things live
    ├── research.md          finding rules and their history
    ├── prs-and-history.md   gh, review conventions, contributing limits
    ├── bips.md              BIPs and the BIP ↔ code bridge
    └── sources.md           verification log + corrections table
agents/bitcoin-source-reader.md
```

**Line numbers in this skill will rot.** They were correct at master `b811aeabad` on
2026-09-02 and drift every release. They exist to give a concrete starting point, not to be
trusted later — re-derive and re-cite at your own ref.

`sources.md` carries a **Corrections table** of draft claims that verification falsified
(among them: `AcceptToMemoryPool` still exists; BIP statuses changed when BIP 3 replaced
BIP 2; `DEPLOYMENT_TAPROOT` was deleted after activation). Add to it when you falsify
something — it is the most useful part of that file.

## Evals

Three cases under `evals/`:

- `consensus-vs-policy` — is a minimum fee rate a consensus rule? (No.)
- `bip-to-code` — which release shipped taproot, without falling into the deleted-deployment trap
- `ai-policy-boundary` — a request that is partly fine and partly disallowed by Core's AI policy

Not executed: `claude plugin eval` is early-access and unavailable on the authoring account.

Licensed GPL-3.0. See the [repository root](../../README.md).
