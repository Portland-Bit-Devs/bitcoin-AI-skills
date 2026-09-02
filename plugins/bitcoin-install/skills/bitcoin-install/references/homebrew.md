# Installing with Homebrew

> **Read this when** you are choosing, installing, upgrading, or removing Bitcoin Core on
> macOS — or when a binary you expected isn't there.

**In this file**

| Section | Answers |
|---|---|
| Formula vs. cask | **The distinction everything else depends on** |
| The formula in depth | Exactly what lands on disk, and where |
| The cask in depth | Why it ships no binaries, and the app's real name |
| Running both | They coexist — with one collision to avoid |
| The Rosetta trap | Why your node may be running x86_64 on Apple Silicon |
| Upgrading | And why upgrading the app can restart your node |
| Uninstalling | Removing packages without destroying your chain data |
| Not using Homebrew | Official binaries, and verifying them |

---

## Formula vs. cask

Homebrew ships **two independent packages** for Bitcoin Core. Both are called some form of
"bitcoin", both are version 31.1, and they contain entirely different things.

```bash
brew install bitcoin              # formula → command-line tools
brew install --cask bitcoin-core  # cask    → the desktop GUI app
```

Neither includes the other. This is the source of nearly every "I installed Bitcoin Core
but `bitcoin-cli` isn't found" report: the cask's complete artifact list, from its own
Homebrew manifest, is

```
==> Artifacts
Bitcoin-Qt.app -> Bitcoin Core.app (App)
```

One app. No `bin/`. Nothing on your `PATH`.

## The formula in depth

```bash
brew install bitcoin
```

Bottled, so it is a download rather than a source build — installation takes seconds.

**Dependencies:** `capnp`, `libevent`, `libsodium`, `zeromq`. The `zeromq` dependency
matters: it means **this build has ZMQ notification support compiled in**, which not every
Bitcoin Core build does. Confirm on a running node with `bitcoin-cli getzmqnotifications`
— an empty array `[]` means compiled-in but unconfigured, whereas `-32601 Method not
found` would mean absent.

**What lands in `bin/`,** symlinked onto your `PATH`:

| Binary | Purpose |
|---|---|
| `bitcoind` | The node daemon |
| `bitcoin-cli` | RPC client — the one people come for |
| `bitcoin-tx` | Build and edit raw transactions offline |
| `bitcoin-util` | Miscellaneous helpers (e.g. `grind`) |
| `bitcoin-wallet` | Offline wallet tool — create, info, salvage, dump |
| `bitcoin` | Unified wrapper: `bitcoin node`, `bitcoin gui`, `bitcoin rpc` |

The `bitcoin` wrapper is newer and worth knowing: `bitcoin rpc` is exactly equivalent to
`bitcoin-cli -named`, so named parameters come for free.

**Also installed, and easy to miss:**

- `share/bitcoin/rpcauth/rpcauth.py` — generates static `rpcauth` credentials, so you do
  not have to clone the Core source to get it. See `bitcoin-api`'s `connection.md`.
- `share/man/man1/*.1` — real man pages for all six binaries.
- `libexec/bitcoin-node`, `libexec/test_bitcoin` — the multiprocess node binary and the
  unit-test runner.

**Where it lives.** Under your Homebrew prefix — `/opt/homebrew` on a native Apple Silicon
install, `/usr/local` on Intel *or* on a Rosetta install (see below):

```bash
brew --prefix bitcoin        # e.g. /usr/local/opt/bitcoin
ls "$(brew --prefix bitcoin)/bin"
```

## The cask in depth

```bash
brew install --cask bitcoin-core
```

Requires macOS 14 or newer. It installs one thing: the desktop app.

**It is renamed on install.** The downloaded bundle is `Bitcoin-Qt.app`, but the cask
places it as **`/Applications/Bitcoin Core.app`** — with a space. Scripts and
`open -a` calls referencing `Bitcoin-Qt.app` will not find it:

```bash
open -a "Bitcoin Core"                                   # works
"/Applications/Bitcoin Core.app/Contents/MacOS/Bitcoin-Qt" -regtest   # direct, with flags
```

Note the executable *inside* the bundle is still called `Bitcoin-Qt`. Only the bundle is
renamed.

## Running both

They coexist without conflict. The formula owns `bin/` symlinks; the cask owns an
`/Applications` bundle. Nothing overlaps, and `bitcoin-cli` from the formula will happily
drive a node started by the GUI — they share the same default data directory and
authenticate through the same cookie file.

**The one collision:** the formula's post-install caveat suggests

```
brew services start bitcoin
```

Do not run that while the GUI app is running. It launches a second `bitcoind` against the
same datadir and the same port; the second process will fail on the data directory lock,
and if you have changed ports to work around that, you now have two nodes competing for
one chainstate. Run one node per data directory.

## The Rosetta trap

On Apple Silicon, check what you actually installed:

```bash
uname -m                                    # arm64 = Apple Silicon hardware
brew config | grep -E 'HOMEBREW_PREFIX|Rosetta'
file "$(brew --prefix bitcoin)/bin/bitcoin-cli"
```

If `uname -m` says `arm64` but Homebrew's prefix is `/usr/local` and `Rosetta 2: true`,
you are running the **x86_64** Homebrew under translation — and every bottle it installs,
including Bitcoin Core, will be an x86_64 binary. `file` will say
`Mach-O 64-bit executable x86_64` rather than `arm64`.

A translated node still works, but initial block download is dominated by signature
verification and hashing, which is exactly what Rosetta translation slows down. A running
node reports its own build in `debug.log`:

```bash
grep -a "System:" "$HOME/Library/Application Support/Bitcoin/debug.log" | tail -1
# x86_64-little_endian-lp64  ← translated
# arm64-little_endian-lp64   ← native
```

To get native binaries, install Homebrew natively at `/opt/homebrew` and use that, or skip
Homebrew and take the official `arm64-apple-darwin` release. Mixing the two prefixes is
allowed but confusing — make sure your `PATH` puts the one you want first.

This applies to the cask too: the app it installs follows the same Homebrew architecture.

## Upgrading

```bash
brew upgrade bitcoin              # CLI tools
brew upgrade --cask bitcoin-core  # GUI app
```

**Upgrading the cask replaces a running app's bundle.** Quit the node first and restart it
deliberately, rather than discovering mid-sync that the binary underneath changed. Bitcoin
Core does not require a reindex for a normal minor upgrade, but a *downgrade* can, because
a newer version may have written chainstate the older one cannot read.

Check what is actually running rather than what is installed — they are different
questions:

```bash
bitcoin-cli getnetworkinfo | grep -E 'version|subversion'
```

## Uninstalling

```bash
brew uninstall bitcoin
brew uninstall --cask bitcoin-core
```

**Neither touches your data directory.** Chain data, `bitcoin.conf`, and — critically —
your **wallets** all survive under `~/Library/Application Support/Bitcoin/`, which is
correct behaviour: uninstalling software should not destroy funds.

If you genuinely want the data gone, back up any wallets first and remove it deliberately.
Do not do this to reclaim disk space while a node is running; set `prune` instead.

## Not using Homebrew

The official builds live at [bitcoincore.org](https://bitcoincore.org/en/download/). Two
things to know:

- **Pick the right archive.** The macOS `.zip` contains *only* `Bitcoin-Qt.app` — no
  command-line tools. The `.tar.gz` contains `bin/` with all six binaries. Downloading the
  zip and expecting `bitcoin-cli` is the same mistake as installing the cask.
- **Verify before installing.** Check the download against `SHA256SUMS` and verify
  `SHA256SUMS.asc` against the release signing keys. Homebrew does this on your behalf;
  a manual download does not.

Match the architecture to your hardware — `arm64-apple-darwin` for Apple Silicon, and see
the Rosetta section above for why that matters.
