# Polar: regtest Bitcoin and Lightning in Docker

> **Read this when** you want a local Lightning network to develop against — several
> lightning nodes wired to a shared regtest bitcoind, with channels, invoices, and a mine
> button. If you only need a Bitcoin regtest chain, `bitcoind -regtest` is lighter and
> needs no Docker; see `cli-tools.md`.
>
> **Status: configuration-derived.** The details below were read from Polar's own generated
> `docker-compose.yml` and network metadata on a real install, not from a running
> container. See `sources.md`.

**In this file**

| Section | Answers |
|---|---|
| What Polar is | And when it beats plain regtest |
| Requirements | Docker, and the Linux exception |
| Install | App plus Docker |
| What a network actually is | Files on disk, containers, images |
| **Connecting to the bitcoind** | **Credentials and ports — the part you came for** |
| How Polar configures bitcoind | Every flag, and why several matter |
| Lifecycle and gotchas | Ports, resets, and versions |

---

## What Polar is

A desktop app that builds regtest Lightning networks out of Docker containers. You draw
the topology — some LND nodes, a Core Lightning node, one bitcoind backing them — press
start, and it generates the Docker Compose project and wires everything together. It gives
you channel management, invoices, node terminals, streaming logs, and a button to mine
blocks and hand out regtest coins.

Reach for it when you need **Lightning**. Assembling the equivalent by hand — matching
implementations and versions, generating configs, funding nodes, opening channels — is a
genuinely bad afternoon. For a plain Bitcoin regtest chain, it is far more machinery than
you need.

## Requirements

**Docker is mandatory.** Polar's own documentation: *"Polar requires that you have Docker
installed to create the local networks."*

- **macOS / Windows:** Docker Desktop.
- **Linux:** Docker Server — **not** Docker Desktop. Polar states Docker Desktop is *"currently
  not supported due to a significant change in how it handles file sharing between host and
  container."* This trips up Linux users who install the familiar option.

The Docker **daemon must be running** before Polar can start a network. A stopped daemon is
the most common "Polar won't start anything" cause.

## Install

```bash
brew install --cask docker   # if you don't have Docker Desktop
```

Polar itself is distributed as pre-built binaries from
[GitHub releases](https://github.com/jamaljsr/polar/releases) — DMG for macOS, DEB /
AppImage / RPM for Linux, EXE for Windows. Install the app, start Docker, then create a
network in the UI.

Polar pulls container images on first use, so the first start of a given node version
downloads before it runs.

## What a network actually is

Everything lives under `~/.polar/`:

```
~/.polar/
├── settings.json
├── logs/
└── networks/
    ├── networks.json              # every network's metadata, including assigned ports
    └── 1/
        ├── docker-compose.yml     # the generated Compose project
        └── volumes/
            ├── bitcoind/backend1/ # the node's real datadir, on your disk
            └── c-lightning/alice/
```

Two consequences worth internalising:

- **`docker-compose.yml` is generated, and readable.** When you need to know a port, a
  credential, or a flag, read that file — it is the ground truth for that network.
- **`volumes/` is the chain data**, bind-mounted into the container at
  `/home/bitcoin/.bitcoin`. It persists across container restarts and is what "reset
  network" deletes.

Containers are named `polar-n<networkId>-<nodeName>`, and the Compose project is
`polar-network-<networkId>`:

```bash
docker ps --filter name=polar-
docker logs -f polar-n1-backend1
docker exec -it polar-n1-backend1 bitcoin-cli -regtest getblockchaininfo
```

The bitcoind image is `polarlightning/bitcoind:<version>`.

## Connecting to the bitcoind

Polar uses **static credentials, identical on every network** — it is a development tool,
and that is deliberate:

| | |
|---|---|
| **User** | `polaruser` |
| **Password** | `polarpass` |
| **Network** | regtest |

They are passed to bitcoind as an `-rpcauth=` hash, so the plaintext password is never in
the config file — but it is the same well-known pair every time. **This is fine for
regtest and unacceptable anywhere else.** Never reuse this pattern on a node holding real
funds.

**Ports are assigned per network** and listed both in the UI and in `networks.json`. For
network 1 on a real install:

| Purpose | Host port | Container port |
|---|---|---|
| JSON-RPC | **18443** | 18443 |
| P2P | **19444** | 18444 |
| ZMQ raw block | **28334** | 28334 |
| ZMQ raw tx | **29335** | 28335 |

Note the remapping: P2P and ZMQ-tx are *not* the same on both sides, because additional
networks would otherwise collide. Read the actual values rather than assuming:

```bash
python3 -c "
import json,pathlib
d=json.loads(pathlib.Path.home().joinpath('.polar/networks/networks.json').read_text())
for n in d.get('networks', []):
    for b in n.get('nodes', {}).get('bitcoin', []):
        print(n['id'], b['name'], b['ports'])"
```

So from the host:

```bash
curl -s --user polaruser:polarpass \
  -H 'content-type: application/json' \
  --data-binary '{"jsonrpc":"2.0","id":"1","method":"getblockchaininfo","params":[]}' \
  http://127.0.0.1:18443/
```

or with the CLI, if you installed the formula:

```bash
bitcoin-cli -regtest -rpcport=18443 -rpcuser=polaruser -rpcpassword=polarpass getblockcount
```

## How Polar configures bitcoind

The generated command line, from a real network:

```
bitcoind -server=1 -regtest=1 -rpcauth=polaruser:<salt>$<hash>
  -debug=1
  -zmqpubrawblock=tcp://0.0.0.0:28334
  -zmqpubrawtx=tcp://0.0.0.0:28335
  -zmqpubhashblock=tcp://0.0.0.0:28336
  -txindex=1 -dnsseed=0 -upnp=0
  -rpcbind=0.0.0.0 -rpcallowip=0.0.0.0/0 -rpcport=18443
  -rest -listen=1 -listenonion=0
  -fallbackfee=0.0002
  -blockfilterindex=1 -peerblockfilters=1
```

Several of these are worth noticing:

- **`-rest`** — the unauthenticated REST interface is **on**. `http://127.0.0.1:18443/rest/chaininfo.json`
  works with no credentials at all.
- **`-txindex=1`** — every transaction is indexed, so `getrawtransaction` works for any
  txid without a block hash. Real nodes usually lack this.
- **`-blockfilterindex=1 -peerblockfilters=1`** — BIP157/158 filters are available, so the
  REST blockfilter endpoints work here and 400 on a default node.
- **`-rpcbind=0.0.0.0 -rpcallowip=0.0.0.0/0`** — wide open, which is *correct inside a
  container* on a private Docker network and *catastrophic* on a host. Do not copy this
  into a real `bitcoin.conf`.
- **`-fallbackfee=0.0002`** — regtest has no fee history; without this, wallet sends fail.

Together these make a Polar bitcoind an unusually convenient target for exercising the
`bitcoin-api` skill: REST, ZMQ, txindex, and block filters are all enabled at once, on a
disposable chain.

**One asymmetry:** `zmqpubhashblock` is published on container port 28336 but is **not**
mapped to a host port. Raw block and raw tx are reachable from the host; `hashblock` is
reachable only from inside the Docker network. Subscribe to `rawblock` from the host, or
run your subscriber as a container on the same network.

## Lifecycle and gotchas

- **Port collisions.** Polar's bitcoind claims host 18443 — the standard regtest RPC port.
  A `bitcoind -regtest` you started yourself will already hold it. Run one or the other,
  or move yours with `-rpcport`.
- **Version ranges are bounded.** Polar supports a specific window of node versions
  (Bitcoin Core v26.0–v30.0 as of v4; also LND, Core Lightning, Eclair, Taproot Assets,
  Lightning Terminal). It does not track Core's latest release immediately, so the bitcoind
  in your network is often older than the one Homebrew installs. Check `getnetworkinfo`
  rather than assuming parity with your host install.
- **"Reset network" deletes the chain.** It wipes `volumes/`, which is the point — but any
  regtest coin, channel, or wallet in that network goes with it.
- **Containers keep running** if you quit the app carelessly. `docker ps --filter name=polar-`
  tells you the truth; stop the network from the UI, or `docker compose -p polar-network-1 down`.
- **Images are downloaded per version.** Adding a node with a version you have not used
  before pulls an image first, which looks like a hang on a slow connection.
