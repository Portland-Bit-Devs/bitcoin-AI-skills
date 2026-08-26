# Contributing

Skills live in this repository. To add one, open a pull request that adds a plugin under
`plugins/` and an entry in `.claude-plugin/marketplace.json`.

## License

Contributions are licensed **GPL-3.0**, same as the repository. By opening a pull request
you agree your contribution ships under that license. If that doesn't work for you, say so
in the PR and we'll talk — but don't submit code or prose you can't license this way.

## Layout

```
plugins/<plugin-name>/
├── .claude-plugin/plugin.json
├── skills/<skill-name>/SKILL.md
│                       └── references/*.md     # optional, loaded on demand
├── agents/*.md                                 # optional
├── evals/                                      # cases for `claude plugin eval`
└── README.md
```

One plugin per coherent topic. A plugin may ship several related skills — users install
"the Lightning stuff," not fourteen separate line items.

## The description is the product

A skill only helps if Claude loads it at the right moment, and the *only* thing that
decides that is the `description` in the `SKILL.md` frontmatter. This is where most skills
fail. A description must:

- **Say when to use the skill**, not just what it is about. Start with "Use when…".
- **List trigger phrasings**, including the ones that don't use the jargon. A fee skill
  should trigger on "why is my transaction stuck," not only on "fee estimation."
- **Name the concrete symbols** a user might mention — RPC methods, file names, BIP
  numbers, function names.
- **Say what's out of scope**, and point at the skill that does cover it.

Descriptions can be long. Err that way.

## Checklist

Before you open a PR:

- [ ] `claude plugin validate . --strict` passes
- [ ] `SKILL.md` frontmatter has a `name` and a trigger-rich `description` (see above)
- [ ] Scope section says what the skill does *not* cover
- [ ] Entry added to `.claude-plugin/marketplace.json` with `description`, `category`,
      `keywords`, `version`, `author`, and `license`
- [ ] Marketplace entry `description` and `plugin.json` `description` agree
- [ ] `category` appears **only** in the marketplace entry — `plugin.json` doesn't take it,
      and `--strict` fails if it's there
- [ ] A symlink added under `.claude/skills/` (and `.claude/agents/` if the plugin ships agents)
- [ ] At least one eval case under `evals/`, ideally three
- [ ] A plugin `README.md`
- [ ] Root `README.md` skills table updated

## Names are permanent

The `name` in a marketplace entry is an immutable slug — people have it installed under
that name, and changing it breaks their install. Pick carefully. If a rename is genuinely
unavoidable, add a `renames` entry at the top level of `marketplace.json` so existing
installs migrate automatically.

## Source material and copyright

This repository is public and GPL-3.0. Publishing a skill here is **distribution**.

- **Never commit copyrighted source material** — books, papers, or purchased PDFs. `.gitignore`
  blocks `*.pdf`/`*.epub` by default; don't work around it.
- **Restate, don't reproduce.** Frameworks, taxonomies, and ideas are not copyrightable and are
  fair to synthesize. Extended excerpts, transcribed passages, and summaries dense enough to
  substitute for the original are not.
- **Attribute.** Name the source and chapter in a `references/sources.md`, and point readers at
  the original. If a skill leans heavily on one book, say so and tell people to buy it.
- **You can't relicense someone else's expression** as GPL-3.0. Only contribute what you can.
- **Cite the other side.** Where a framework is contested, name the counter-argument rather than
  presenting one school as settled fact.

## Safety rules for Bitcoin skills

This is money. Skills in this repo must not instruct Claude to:

- spend, send, or sign anything on mainnet without explicit per-transaction confirmation
  of amount and destination
- read, echo, or write seed phrases, xprvs, descriptors containing private keys, or
  wallet files
- disable or work around a user's confirmation prompts

Prefer regtest and signet for anything a skill demonstrates. PRs that violate this get
closed, not revised.

## Versioning

Bump the `version` in **both** `plugin.json` and the marketplace entry — they must agree.
Tag releases with `claude plugin tag plugins/<plugin-name>`.
