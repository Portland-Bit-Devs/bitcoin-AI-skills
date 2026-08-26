# bitcoin-AI-skills

A marketplace of self-owned, openly licensed [Agent Skills](https://code.claude.com/docs/en/skills)
for working with Bitcoin — the software, the protocol, and the money.

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

| Plugin | What it covers | Status |
|---|---|---|
| [`bitcoin-cli`](plugins/bitcoin-cli) | Driving a Bitcoin Core node from the shell — RPC calls, wallets, regtest | 🚧 stub |
| [`money`](plugins/money) | What money is: the three functions, the monetary properties, how to evaluate a money | 🚧 stub |
| [`bitcoin-code`](plugins/bitcoin-code) | Reading the Bitcoin Core source tree, with a read-only source-reader agent | 🚧 stub |

🚧 **stub** means the structure and skill descriptions are in place but the reference
material has not been written yet. See each skill's `TODO` section.

## Contributing

New skills are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) — the short version is that
a skill needs a description good enough for Claude to know when to load it, and that
contributions are GPL-3.0.

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

## License

GPL-3.0. See [LICENSE](LICENSE).
