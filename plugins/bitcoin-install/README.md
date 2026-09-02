# bitcoin-install

Install and first-run Bitcoin Core on macOS, and stand up a disposable regtest network.

**Status:** v0.1 — verified on macOS 26.5 against Bitcoin Core v31.1 and Polar v4.

```bash
claude plugin install bitcoin-install@bitcoin-ai-skills
```

Provides the `bitcoin-install` skill. The distinction it exists to teach:

| Command | You get | You do **not** get |
|---|---|---|
| `brew install bitcoin` (formula) | `bitcoind`, `bitcoin-cli`, `bitcoin-tx`, `bitcoin-util`, `bitcoin-wallet`, `bitcoin` | Any GUI |
| `brew install --cask bitcoin-core` (cask) | The Bitcoin-Qt desktop app | **`bitcoin-cli` or any binary** |

Covers both paths separately, plus running a local Lightning network with Polar and Docker,
enabling the GUI node's RPC server (off by default), a 60-second regtest recipe, upgrading
and uninstalling, and the Apple Silicon / Rosetta trap.

## Editing this skill

```
skills/bitcoin-install/
├── SKILL.md            always loaded — the formula/cask table and routing
└── references/
    ├── homebrew.md     choosing, installing, upgrading, removing; Rosetta
    ├── cli-tools.md    bitcoind + bitcoin-cli, config, regtest
    ├── qt-gui.md       the desktop node and its RPC server
    ├── polar.md        Lightning in Docker
    └── sources.md      provenance + verification log
```

Only `SKILL.md` is always in context; `references/` loads on demand. A new reference file
needs a row in `SKILL.md`'s routing table or it will never be read.

**Version numbers age fastest.** Prefer commands that ask (`brew info`, `getnetworkinfo`,
`~/.polar/networks/networks.json`) over hard-coded versions. The structural claims — the
formula/cask split, `server=1` for the GUI, one node per datadir, Polar's static regtest
credentials — are the durable content and rarely change.

**If you change a factual claim, update `sources.md`.** Its Verification section records
what was actually run versus read off a config file, and is what stops the next person
trusting an unverified claim.

The clearest gap to close: no Polar network was started, because the Docker daemon was
stopped. Starting one and confirming the documented ports accept connections would upgrade
`polar.md` from *configuration-derived* to verified.

## Relationship to the other skills

- **`bitcoin-cli`** — using the CLI once it's installed.
- **`bitcoin-api`** — talking to the node over HTTP. A Polar bitcoind runs with `-rest`,
  `-txindex`, and ZMQ all enabled, which makes it a convenient target for that skill.
- **`bitcoin-code`** — the Core source tree.

## Safety

Installing is low-risk; the skill's safety section covers what isn't — never running two
nodes against one datadir (including the `brew services` trap), never deleting a datadir
without checking for wallets, preferring regtest/signet, and verifying non-Homebrew
downloads against `SHA256SUMS`.

## Evals

Three cases under `evals/`, in the `prompt.md` + `graders/` form:

- `cask-has-no-cli` — the formula/cask confusion, phrased as "did my install fail?"
- `qt-rpc-disabled` — the Debug Console is not evidence that port 8332 is open
- `polar-vs-regtest` — right-sizing the tool when the user names Polar but describes a
  plain Bitcoin need

Not executed: `claude plugin eval` is early-access and unavailable on the authoring account.

Licensed GPL-3.0. See the [repository root](../../README.md).
