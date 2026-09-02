# bitcoin-cli

Drive a Bitcoin Core node from the command line.

**Status:** v0.1 — verified against Bitcoin Core v31.1.0 on macOS.

```bash
claude plugin install bitcoin-cli@bitcoin-ai-skills
```

Provides the `bitcoin-cli` skill: invocation and connection flags, network and wallet
selection, the client-only helpers (`-getinfo`, `-netinfo`, `-addrinfo`, `-generate`),
argument quoting and the per-method JSON conversion rule, output shapes and exit codes,
BTC-vs-satoshi units and fee math, the regtest workflow, and task-oriented recipes.

## The split with `bitcoin-api`

These two overlap by subject and not by scope. **`bitcoin-api` owns the wire; this skill
owns the program.**

| | `bitcoin-cli` | `bitcoin-api` |
|---|---|---|
| Auth | *"it reads the cookie for you"* | Cookie vs. `rpcauth` vs. `rpcuser`, how each works |
| Errors | How the CLI surfaces one — stderr, exit status | What every numeric code **means** |
| Endpoints | `-rpcwallet=` | `/` vs. `/wallet/<name>`, and why |
| Audience | A person at a terminal | Application code over HTTP |

If you find yourself copying an error-code table into this skill, that belongs in
`bitcoin-api` instead.

## Editing this skill

```
skills/bitcoin-cli/
├── SKILL.md             always loaded — routing, safety, the three gotchas
└── references/
    ├── invocation.md    flags: network, datadir, wallet, waiting, secrets
    ├── arguments.md     the JSON-conversion rule and quoting
    ├── output.md        shapes, exit codes, units, fees, jq
    ├── regtest.md       the private-chain workflow
    ├── recipes.md       task-oriented commands
    └── sources.md       verification log
```

Only `SKILL.md` is always in context; `references/` loads on demand. A new reference file
needs a row in `SKILL.md`'s routing table or it will never be read.

**This skill is behavioural, not transcribed.** Nearly every claim came from running the
command — exit codes, output shapes, formatted amounts, balances. To update one, **run it
again** on the version you care about rather than editing from memory. That discipline
already caught one error during authoring: the first draft claimed Core emits scientific
notation for amounts, when in fact Core prints fixed 8-decimal values and the `1.65e-05`
came from Python's JSON module.

**If you change a factual claim, update `sources.md`.** Its Verification section is the
record of what was actually observed.

## Safety

`bitcoin-cli` spends money with no confirmation prompt and no undo. The skill instructs
Claude never to run a spending or signing command on mainnet without explicit per-command
confirmation of amount and destination, never to read or echo private keys, never to put a
passphrase in argv (use the `-stdin` family), and to prefer regtest for anything
exploratory.

## Evals

Three cases under `evals/`, in the `prompt.md` + `graders/` form:

- `argument-parse-failure` — explaining the JSON-conversion rule, not a quoting superstition
- `stuck-transaction` — systematic diagnosis before reaching for `bumpfee`
- `fee-sign-and-units` — the negative fee convention and safe satoshi conversion

Not executed: `claude plugin eval` is early-access and unavailable on the authoring account.

Licensed GPL-3.0. See the [repository root](../../README.md).
