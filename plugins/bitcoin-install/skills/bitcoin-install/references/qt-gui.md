# The desktop node: Bitcoin-Qt

> **Read this when** you want Bitcoin Core with a window — a wallet UI, a sync progress
> bar — or when something can't reach the RPC server of a node you started from the app.

**In this file**

| Section | Answers |
|---|---|
| Install | The cask, and the app's real name |
| First run | Datadir choice and pruning, decided once |
| **The RPC server is off by default** | **The single biggest gotcha** |
| The Debug Console | Useful, and why it proves nothing about port 8332 |
| Launching with flags | Running the GUI on regtest or signet |
| GUI vs. daemon | What actually differs |

---

## Install

```bash
brew install --cask bitcoin-core
```

Installs as **`/Applications/Bitcoin Core.app`** — the cask renames the bundle, though the
executable inside is still `Bitcoin-Qt`. Requires macOS 14+.

```bash
open -a "Bitcoin Core"
```

It ships **no command-line tools**. If you want `bitcoin-cli` as well, that is the separate
Homebrew formula — see `homebrew.md`.

## First run

On first launch the app asks where to keep block data and offers to limit its size. Both
answers are worth thinking about once:

- **Data directory.** The default is `~/Library/Application Support/Bitcoin`. Everything
  else — `bitcoin.conf`, `debug.log`, the cookie, your wallets — lives here. If you point
  it at an external disk, keep in mind the node stops when the disk detaches.
- **Pruning.** The dialog's size limit sets `prune` in `settings.json`. Pruning discards
  old blocks after verifying them; the node remains fully validating. It cannot be
  combined with `txindex`, and reversing it means resyncing.

The GUI writes its choices to `settings.json`, which Bitcoin Core manages itself. Your own
options belong in `bitcoin.conf`, which the app does **not** create for you.

## The RPC server is off by default

**`bitcoind` enables JSON-RPC over HTTP by default. `bitcoin-qt` does not.**

This is the difference that wastes the most time. A node started from the app is a real,
fully validating node — but nothing outside the app can reach it until you add:

```conf
# ~/Library/Application Support/Bitcoin/bitcoin.conf
server=1
```

Then quit and reopen the app. `bitcoin.conf` is read at startup only.

Afterwards, `bitcoin-cli` works against the GUI node with no further configuration — both
sides find the same data directory and the same cookie file:

```bash
bitcoin-cli getblockcount
```

Confirm the port is actually open before blaming anything else:

```bash
lsof -nP -iTCP:8332 -sTCP:LISTEN
```

## The Debug Console proves nothing about port 8332

The app has a built-in console at **Window → Console** that runs RPC methods and is
genuinely useful — `getblockchaininfo`, `getpeerinfo`, and `help` all work there, and it
is the fastest way to inspect a GUI node.

But it calls those methods **inside the process**. It does not go over HTTP. So the console
answering `getblockcount` tells you nothing about whether `server=1` is set or whether
anything external can connect. People reasonably conclude "RPC works, so the problem is my
client" and debug the wrong end for an hour.

If the console works but `curl` and `bitcoin-cli` get `Connection refused`, the answer is
`server=1`.

## Launching with flags

The app accepts the same command-line options as `bitcoind`. Run the executable inside the
bundle directly:

```bash
"/Applications/Bitcoin Core.app/Contents/MacOS/Bitcoin-Qt" -regtest -server=1
"/Applications/Bitcoin Core.app/Contents/MacOS/Bitcoin-Qt" -signet
```

Or, if you also installed the formula, the unified wrapper does the same thing:

```bash
bitcoin gui -regtest
```

Running the GUI on regtest gives you a visual wallet against a private chain — a good way
to explore wallet behaviour without risking anything. Mine into it with `bitcoin-cli
-regtest generatetoaddress`, or use the console.

## GUI vs. daemon

Same validation, same consensus rules, same data directory format — a chain synced by one
is usable by the other. What differs:

| | `bitcoin-qt` | `bitcoind` |
|---|---|---|
| RPC over HTTP | **off** unless `server=1` | on by default |
| Interface | Window, wallet UI, console | None; RPC only |
| Lifecycle | Quits with the app | `bitcoin-cli stop`, or a service |
| Headless / remote | No | Yes |

You can alternate: sync with the app, then stop it and start `bitcoind` against the same
datadir. What you must not do is run both at once — see `homebrew.md` on the
`brew services` collision.
