# bitcoin-AI-skills

A marketplace of self-owned, openly licensed [Agent Skills](https://code.claude.com/docs/en/skills)
for working with Bitcoin — the software, the protocol, and the money.

Five plugins, designed as **one system**: each owns a distinct layer, states its boundary
in its own description, and hands off explicitly to the others. Install one and it will
tell you when you want a sibling; install all five and they route between themselves.

## Install

Add the marketplace once:

```bash
claude plugin marketplace add austenjt/bitcoin-AI-skills
```

Then install what you want:

```bash
claude plugin install bitcoin-cli@bitcoin-ai-skills
```

Or browse everything with `/plugin` → **Discover** inside Claude Code.

## Skills

| Plugin | Owns | Status |
|---|---|---|
| [`bitcoin-install`](plugins/bitcoin-install) | **The machine.** Getting Bitcoin Core onto macOS — the Homebrew formula vs. cask, the Qt desktop node, Polar + Docker | ✅ v0.1 |
| [`bitcoin-cli`](plugins/bitcoin-cli) | **The program.** Driving a node from the shell — flags, argument quoting, reading output, the regtest workflow | ✅ v0.1 |
| [`bitcoin-api`](plugins/bitcoin-api) | **The wire.** JSON-RPC over HTTP, the REST interface, ZMQ notifications — auth, endpoints, error codes | ✅ v0.1 |
| [`bitcoin-code`](plugins/bitcoin-code) | **The source.** Researching bitcoin/bitcoin and the BIPs — consensus vs. policy, git archaeology, PRs, clangd | ✅ v0.1 |
| [`money`](plugins/money) | **The framework.** What money is: salability, the three functions, the six properties, and the contested seventh | ✅ v0.1 |

## How they fit together

The four software skills stack by **layer of abstraction**, from the machine up to the
source. `money` sits beside the stack rather than in it — it is the only skill here that
isn't about software.

```
                                    ┌──────────────────┐
   the machine    bitcoin-install ──▶│                  │
                         │           │                  │
   the program        bitcoin-cli ──▶│   bitcoin-code   │◀── money
                         │           │   (the source,   │    (the framework —
   the wire           bitcoin-api ──▶│    the evidence) │     verifies its
                                     └──────────────────┘     Bitcoin claims here)
```

Two rules make the arrows work:

1. **Each skill states its own boundary in its `description`** — the only thing that
   decides which skill loads. Every description ends with an explicit "not for X — that
   is `<skill>`" clause.
2. **`bitcoin-code` is the terminal skill for evidence.** When any question stops being
   *"how do I do this"* and becomes *"what does the code actually enforce, and since
   when"*, the others hand off there — and the answer comes back with `file:line` at a
   pinned ref rather than from memory.

### Which one do I want?

| You're asking | Skill |
|---|---|
| "`bitcoin-cli: command not found`" / "how do I install this" / "I want a local Lightning network" | `bitcoin-install` |
| "why won't this command parse" / "what fee did I pay" / "why is my transaction stuck" | `bitcoin-cli` |
| "401 from bitcoind" / "connect to my node from Python" / "notify me when a block arrives" | `bitcoin-api` |
| "where is the 21 million cap enforced" / "is this consensus or policy" / "which release shipped taproot" | `bitcoin-code` |
| "is bitcoin money" / "what backs the dollar" / "are all bitcoin the same" | `money` |

### Topic ownership

Some topics could plausibly live in two skills. They don't — each has one owner, and the
other skill points at it instead of restating it. If you're editing, respect these:

| Topic | Owner | The other skill's job |
|---|---|---|
| Regtest **workflow** — mine, fund, spend, reorg, tear down | `bitcoin-cli` | `bitcoin-install` gets a regtest node *started*, then points here |
| Regtest **+ Lightning** | `bitcoin-install` (Polar) | — |
| Cookie file, `rpcauth`, ports, binding | `bitcoin-api` | `bitcoin-cli` points here for credentials; `bitcoin-install` points here for Qt's RPC server |
| Numeric RPC **error codes** and their meaning | `bitcoin-api` | `bitcoin-cli` owns how a shell user *sees* them — stderr, exit status |
| BTC-vs-satoshi units, fee math, `jq` | `bitcoin-cli` | — |
| Consensus vs. policy | `bitcoin-code` | Everyone else cites it |
| Claims about what Bitcoin's code enforces | `bitcoin-code` | `money` delegates rather than asserting |

## Verification baseline

Skills state what was actually exercised, not just what was read. Each `SKILL.md` carries a
**Status** section, and each reference-file table carries a **Verified?** column:

- **`bitcoin-install`**, **`bitcoin-cli`** — Bitcoin Core **v31.1** on macOS 26.5 (Apple
  Silicon); Homebrew formula `bitcoin` 31.1_1, cask `bitcoin-core` 31.1; Polar v4 config
- **`bitcoin-api`** — Core **v31.0.0** live mainnet node (JSON-RPC, recipes), **v31.1**
  regtest with `-rest=1` (REST); ZMQ is documentation-derived and marked as such
- **`bitcoin-code`** — local clones of `bitcoin/bitcoin` (master @ 2026-09-02) and
  `bitcoin/bips`, with `gh` against the live GitHub API
- **`money`** — not applicable; it's a framework from a named literature, and says so

Where a skill hasn't verified something, it says **docs only** or **partly** rather than
implying otherwise. Weight accordingly.

## Sources

Skills here are syntheses that cite their sources. Copyrighted source material is **never**
committed to this repository — see `.gitignore` and [CONTRIBUTING.md](CONTRIBUTING.md).
Each skill ships a `references/sources.md` naming what it drew on, what was verified, and
where the counter-argument lives.

## Contributing

New skills are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) — the short version is that
a skill needs a description good enough for Claude to know when to load it, an explicit
boundary against its siblings, and that contributions are GPL-3.0.

## Local development

Skills in this repo are symlinked into `.claude/skills/` and `.claude/agents/`, so they
load automatically for any Claude Code session started in this directory — no install step
while you work on them.

Validate before you push:

```bash
claude plugin validate . --strict
```

## Trust

This marketplace ships prompts and instructions that Claude will follow. Read a skill
before you install it. Nothing here should ever move funds without your explicit,
per-transaction confirmation — if you find something that would, open an issue.

Every software skill carries a **Safety** section, and they agree with each other: no
spending or signing on mainnet without per-command confirmation of amount and destination;
no reading, echoing, or writing of private keys, seed phrases, or wallet files; no secrets
on a command line; regtest and signet preferred for anything exploratory.

## License

GPL-3.0. See [LICENSE](LICENSE).
