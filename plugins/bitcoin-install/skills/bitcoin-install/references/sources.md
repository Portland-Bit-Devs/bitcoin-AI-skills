# Sources

> **Read this when** you need to check a claim, or you are about to edit this skill and
> want to know which parts were observed and which were read off a config file.

**In this file**

| Section | Answers |
|---|---|
| Verification | **What was actually run, and on what** |
| Primary sources | Homebrew manifests, Bitcoin Core docs, Polar |
| Not verified | The honest gaps |
| Editing note | Keeping this accurate as versions move |

---

## Verification

Performed on **macOS 26.5, Apple Silicon (arm64)**, on 2026-09-01.

**Homebrew — verified directly:**

- `brew info bitcoin` → formula, stable **31.1**, bottled, HEAD available. Dependencies
  `capnp`, `libevent`, `libsodium`, `zeromq`.
- `brew install bitcoin` run start to finish; `brew list bitcoin` enumerated to confirm the
  six `bin/` binaries, `libexec/bitcoin-node`, `libexec/test_bitcoin`, six man pages, and
  `share/bitcoin/rpcauth/rpcauth.py`.
- `brew info --cask bitcoin-core` → **31.1**, requires macOS ≥ 14, artifact list is exactly
  `Bitcoin-Qt.app -> Bitcoin Core.app (App)`. Confirmed by inspection that the installed
  bundle contains **one** executable (`Bitcoin-Qt`) and no command-line tools.
- Coexistence confirmed: formula symlinks in `/usr/local/bin` alongside the cask's
  `/Applications/Bitcoin Core.app`, no conflict.
- `bitcoin-cli --version` → `v31.1.0`; `bitcoind --version` → `v31.1.0`.

**The Rosetta trap — verified on this machine:**

`uname -m` reported `arm64` while `brew config` reported `HOMEBREW_PREFIX: /usr/local`,
`Rosetta 2: true`, `macOS: 26.5.2-x86_64`. Both the installed `bitcoin-cli` and the cask's
`Bitcoin-Qt` were `Mach-O 64-bit executable x86_64`, and the running node's `debug.log`
recorded `System: macOS 26.5, x86_64-little_endian-lp64`. An earlier native run on the same
machine had logged `arm64-little_endian-lp64`, confirming the log line distinguishes them.

**The regtest workflow — every command in `cli-tools.md` was executed:**

`bitcoind -regtest -datadir=… -daemon` with `-rest=1`, `-fallbackfee`, `-zmqpubrawblock`
and `-zmqpubsequence`; then `createwallet`, `getnewaddress`, `generatetoaddress 101`, and
`getbalance` → **50.00000000 BTC**, confirming the 101-block coinbase-maturity note.
`getzmqnotifications` listed both endpoints with `hwm 1000`. The REST interface answered on
the regtest RPC port. `bitcoin-cli stop` shut it down cleanly.

**Bitcoin-Qt — partly verified.** The `server=1` requirement and the resulting behaviour
were confirmed on a real GUI node earlier in the same session: with `server=1` set,
`bitcoin-cli` and `curl` both reached it on port 8332 using cookie auth. The Debug Console
claim is from Bitcoin Core's own documentation (*"the headless daemon `bitcoind` has the
JSON-RPC API enabled by default, the GUI `bitcoin-qt` has it disabled by default"*) rather
than from a side-by-side test.

**Polar — configuration-derived, not run.** The Docker daemon was stopped, so no container
was started. Everything in `polar.md` was read from a real Polar install:

- `~/.polar/networks/1/docker-compose.yml` — the generated Compose project, giving the full
  `bitcoind` command line, the image tag `polarlightning/bitcoind:28.0`, container name
  `polar-n1-backend1`, project name `polar-network-1`, the bind mount to
  `/home/bitcoin/.bitcoin`, and the host↔container port mapping including the
  18444→**19444** and 28335→**29335** remaps and the unmapped `hashblock` port.
- `~/.polar/networks/networks.json` — per-node assigned ports, confirming the same values.
- The `polaruser` / `polarpass` credentials and `rpcport=18443` were confirmed by
  inspecting strings in Polar's application bundle as well as the generated config.
- Requirements, supported implementations, and version ranges are from Polar's README.

## Primary sources

| Source | Used for |
|---|---|
| [Homebrew formula `bitcoin`](https://github.com/Homebrew/homebrew-core/blob/HEAD/Formula/b/bitcoin.rb) | Formula contents, dependencies, caveats |
| [Homebrew cask `bitcoin-core`](https://github.com/Homebrew/homebrew-cask/blob/HEAD/Casks/b/bitcoin-core.rb) | Cask artifact list and requirements |
| [bitcoincore.org downloads](https://bitcoincore.org/en/download/) | Official archives; zip vs. tar.gz distinction |
| [`doc/JSON-RPC-interface.md`](https://github.com/bitcoin/bitcoin/blob/v31.0/doc/JSON-RPC-interface.md) | The `bitcoind` vs. `bitcoin-qt` RPC default |
| [Polar](https://github.com/jamaljsr/polar) | Requirements, supported implementations, distribution |

For the RPC interface itself — auth, endpoints, error codes — see the `bitcoin-api` skill
rather than duplicating it here.

## Not verified

Stated plainly so nobody trusts these more than they should:

- **No Polar network was started.** Container behaviour, the mine button, channel
  operations, and whether the documented ports actually accept connections at runtime are
  all inferred from configuration.
- **Polar's port-assignment scheme across multiple networks.** Only network 1 existed on the
  install inspected. The values given are that network's real ones; the pattern for
  networks 2+ is not established here — read `networks.json`.
- **Bitcoin-Qt's first-run dialog** and the exact `settings.json` it writes were not
  re-tested for this skill.
- **Linux and Windows.** This skill is macOS-specific throughout.

## Editing note

Version numbers age fast — Homebrew's formula and cask both tracked 31.1 at the time of
writing, and Polar's supported range topped out at Bitcoin Core v30.0. Prefer commands that
*ask* (`brew info`, `getnetworkinfo`, `networks.json`) over hard-coded versions, and when
you update a version here, say where you checked it.

The structural claims are the durable ones and are what this skill is really for: the
formula/cask split, `server=1` for the GUI, coexistence, one-node-per-datadir, and Polar's
static regtest credentials. Those have not changed in years.
