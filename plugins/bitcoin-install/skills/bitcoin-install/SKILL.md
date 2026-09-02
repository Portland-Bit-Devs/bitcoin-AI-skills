---
name: bitcoin-install
description: Use when installing, upgrading, uninstalling, or first running Bitcoin Core on macOS — choosing between the Homebrew formula and the Homebrew cask, getting `bitcoin-cli` and `bitcoind` onto your PATH, running the Bitcoin-Qt desktop node and enabling its RPC server, or spinning up a disposable regtest Lightning network with Polar and Docker. Triggers on "install bitcoin", "brew install bitcoin", "install Bitcoin Core", "bitcoin-cli command not found", "bitcoind not found", "I have the app but no command line tools", "set up a bitcoin node", "how do I run bitcoind", "Bitcoin-Qt", "where is bitcoin.conf", "which datadir", "upgrade Bitcoin Core", "uninstall bitcoin", "brew services start bitcoin", "Polar", "polarlightning", "polaruser/polarpass", "regtest network", "bitcoin in docker". ONLY once Bitcoin is named or already established in context, also triggers on "my node syncs really slowly", "how do I test without real money", "I want a local chain I can mine on". Bitcoin Core on macOS only — do NOT load for Ethereum, Solana, or any other chain, for a Lightning implementation on its own, or for generic Docker, server, or node questions. Not for using the tools once installed — that is `bitcoin-cli` for the shell and `bitcoin-api` for HTTP/ZMQ; not for reading Core's source — that is `bitcoin-code`.
---

# bitcoin-install

## Preconditions

This skill is about **Bitcoin Core on macOS**. Confirm that before using it.

Stop and say so in one line if the subject is another chain (Ethereum, Solana, Litecoin,
Monero), a Lightning implementation on its own, a non-blockchain node or server, or Linux
or Windows. The package names, install paths, datadir locations, and the formula/cask
distinction here are macOS-and-Bitcoin-Core-specific and are wrong elsewhere — translating
them across is worse than not answering. On Linux, point at the distribution's packages or
bitcoincore.org and stop.

If it's genuinely ambiguous which chain or OS is meant, ask rather than assume.

Getting Bitcoin Core onto a Mac, and running it the way you actually intend to.

The single most common mistake is installing the wrong Homebrew package. **There are two,
they are not alternatives, and neither one is a superset of the other:**

| You run | You get | You do **not** get |
|---|---|---|
| `brew install bitcoin` — the **formula** | `bitcoind`, `bitcoin-cli`, `bitcoin-tx`, `bitcoin-util`, `bitcoin-wallet`, `bitcoin` | Any GUI |
| `brew install --cask bitcoin-core` — the **cask** | The Bitcoin-Qt desktop app, and nothing else | **`bitcoin-cli`, `bitcoind`, or any other binary** |

Installing the cask and then finding `bitcoin-cli: command not found` is not a broken
install — the cask simply does not ship it. Its Homebrew artifact list is one line: the
app. Install both if you want both; they coexist cleanly and share a data directory.

## Scope

- The two Homebrew packages, in depth: what each installs, where, and how to verify
- Running the command-line node: `bitcoind`, config, first run, regtest
- Running the desktop node: Bitcoin-Qt, and enabling its RPC server (off by default)
- Polar + Docker: a disposable regtest Bitcoin and Lightning network
- Upgrading, uninstalling, and the Apple Silicon / Rosetta trap

Out of scope: non-macOS platforms — this skill is macOS-specific. On Linux, prefer your
distribution's packages or the official binaries from bitcoincore.org.

## Related skills

This skill owns **getting the software onto the machine and running it the first time**.
It is the front door: every other Bitcoin skill here assumes binaries exist and a node is
reachable. Once that is true, hand off and stop.

| When the question moves to… | Hand off to |
|---|---|
| Composing commands, quoting arguments, reading output, `jq` | **`bitcoin-cli`** |
| `curl`, HTTP, application code, REST, ZMQ | **`bitcoin-api`** |
| What Core's source actually does, or why | **`bitcoin-code`** |
| Whether any of this is *money* | **`money`** |

Coming the other way, expect to be handed **back** here when: `bitcoin-cli: command not
found` (the cask ships no binaries — see `references/homebrew.md`), a node can't be reached
because Bitcoin-Qt's RPC server is off by default (`references/qt-gui.md`), or two nodes
are fighting over one datadir.

**Shared topic — regtest.** *Starting* a regtest node lives here
(`references/cli-tools.md`), and the Lightning-flavoured version lives in
`references/polar.md`. The regtest *workflow* — mine, fund, spend, control time, tear
down — belongs to `bitcoin-cli`. Don't restate it here.

## Which do you want?

| Goal | Install | Read |
|---|---|---|
| Scripting, `bitcoin-cli`, a headless node | Formula | `references/cli-tools.md` |
| A node with a window, wallet UI, and a progress bar | Cask | `references/qt-gui.md` |
| Both — GUI node, driven from the terminal | **Both** | Both of the above |
| Throwaway chain to develop and test against | Formula (regtest) | `references/cli-tools.md` |
| Local **Lightning** network to develop against | Polar + Docker | `references/polar.md` |

Running your own regtest node with the formula is lighter than Polar and needs no Docker.
Choose Polar when you want Lightning nodes wired up to that chain — that is what it is for,
and building the equivalent by hand is a bad afternoon.

## Verify what you actually installed

Do this before debugging anything else. It answers "which package do I have" in one step:

```bash
brew list --formula | grep '^bitcoin$'   # the CLI tools
brew list --cask    | grep bitcoin-core  # the GUI app
command -v bitcoin-cli bitcoind          # silence means the formula is missing
```

## Safety

Installing is low-risk; what you do next is not.

- **Never run a second node against a data directory that is already in use.** The
  formula's post-install message suggests `brew services start bitcoin`. If the Bitcoin-Qt
  app is running, that starts a *competing* `bitcoind` on the same default datadir and
  port. Pick one node per datadir.
- **Do not delete a data directory to "start clean"** without confirming what is in it. It
  holds wallets. A resync costs hours; a lost wallet is permanent.
- **Prefer regtest or signet for anything exploratory.** Neither uses real money.
- **Verify official downloads.** If you install from bitcoincore.org rather than Homebrew,
  check the release against `SHA256SUMS` and its signatures. Homebrew does this for you.
- Treat wallet files, descriptors, and seed phrases as secrets. Never echo or copy them.

## Reference material

Load on demand. Each file opens with a "Read this when" note and its own contents table.

| File | Read it when | Verified? |
|---|---|---|
| `references/homebrew.md` | Choosing, installing, upgrading, or removing either package — **and the Rosetta trap** | **yes** |
| `references/cli-tools.md` | You want `bitcoind`/`bitcoin-cli` running, including a regtest chain | **yes** |
| `references/qt-gui.md` | You want the desktop node, or need its RPC server reachable | partly |
| `references/polar.md` | You want a local Lightning network in Docker | config only |
| `references/sources.md` | Checking a claim, or editing this skill | — |

**yes** means performed on a real macOS machine; **config only** means read from Polar's
own generated configuration rather than from a running container. `sources.md` has details.

## Status

Written and verified on macOS 26.5 (Apple Silicon) against **Bitcoin Core v31.1** —
Homebrew formula `bitcoin` 31.1_1 and cask `bitcoin-core` 31.1 — and **Polar v4** network
configuration. Homebrew package contents, install paths, the regtest workflow, and the
architecture check were all exercised directly.
