# The Bitcoin Core source tree

> **Read this when** you know what you're looking for but not where to look. Start here,
> then grep — don't grep the whole tree blind.

**In this file**

| Section | Answers |
|---|---|
| **Consensus vs. policy** | **The distinction that matters most** |
| Directory map | What lives where, with sizes |
| Landmark files | The handful worth knowing by name |
| Entry points | Where the well-known code paths start |
| Tests | And why they're often the best documentation |
| Search tactics | Cutting the noise |

---

## Consensus vs. policy

Get this wrong and the answer is wrong, however good the citation.

| | Consensus | Policy |
|---|---|---|
| Lives in | `src/consensus/`, and paths reached from `ConnectBlock` | `src/policy/` |
| Violating it means | The block is **invalid to every node** | This node won't relay or mine it — others may |
| Changing it needs | A fork | A release, or a config option |
| Example | `MoneyRange`, block subsidy, script validity | Minimum relay fee, dust threshold, TRUC/v3 rules |

`src/consensus/` is deliberately small — ten files — because everything in it is
fork-relevant:

```
amount.h      MAX_MONEY, COIN, MoneyRange()
consensus.h   MAX_BLOCK_WEIGHT, COINBASE_MATURITY, and friends
params.h      Consensus::Params — per-network rule parameters and activation heights
tx_check.cpp  CheckTransaction — context-free transaction validity
tx_verify.cpp Contextual checks: inputs, sequence locks, fees
merkle.cpp    Merkle root computation
validation.h  Result types shared with the validation engine
```

Note the asymmetry: consensus *constants* live here, but much consensus *logic* lives in
`src/validation.cpp`. "It isn't in `src/consensus/`" does not mean "it isn't consensus."
The reliable test is whether the check can make a block invalid.

## Directory map

Under `src/`, by rough file count on a recent master:

| Directory | Files | Contains |
|---|---|---|
| `consensus/` | 10 | Fork-relevant rules and constants |
| `policy/` | 22 | Relay and mempool policy: `feerate`, `rbf`, `packages`, `truc_policy`, `ephemeral_policy` |
| `script/` | 22 | Script engine: `interpreter`, `script`, `descriptor`, `miniscript`, `solver`, `sigcache`, `sign` |
| `primitives/` | 5 | The core data types: block, transaction |
| `crypto/` | 37 | SHA256, RIPEMD160, ChaCha20, AES, HMAC |
| `node/` | 65 | Node-level services above validation |
| `kernel/` | 29 | The extracted validation engine and its option structs |
| `wallet/` | 85 | Descriptors, coin selection, signing, the wallet database |
| `rpc/` | 28 | RPC dispatch and method implementations |
| `index/` | 13 | `txindex`, `blockfilterindex`, `coinstatsindex` |
| `net*` (top level) | — | `net.cpp` (connections), `net_processing.cpp` (message handling) |
| `util/` | 82 | General utilities |
| `qt/` | 136 | The GUI. Almost never the answer — exclude it from searches |
| `test/` | 324 | Unit tests |
| `secp256k1/` | 104 | Vendored subtree — the EC library. Upstream is a **separate repo** |
| `leveldb/`, `minisketch/`, `crc32c/`, `univalue/` | — | Vendored subtrees; don't attribute their code to Core |

Outside `src/`:

| Path | Contains |
|---|---|
| `doc/` | **`bips.md`** (BIP → version → PR), `AI_POLICY.md`, `developer-notes.md`, `build-*.md`, release notes |
| `test/functional/` | 282 Python integration tests driving real nodes |
| `test/fuzz/`, `src/test/fuzz/` | Fuzz harnesses |
| `contrib/` | Helper scripts, including `contrib/guix` for reproducible builds |
| `cmake/` | Build system, including `bitcoin-build-config.h.in` |

## Landmark files

The few worth knowing by name:

| File | Why |
|---|---|
| `src/validation.cpp` | The largest and most important file. Block and transaction validation, chain state, the mempool acceptance path |
| `src/net_processing.cpp` | P2P message handling — what the node does with what peers send |
| `src/script/interpreter.cpp` | The script engine. `EvalScript` is the opcode loop |
| `src/consensus/tx_check.cpp` | Context-free transaction checks, including the money-range test |
| `src/kernel/chainparams.cpp` | Network parameters, activation heights, checkpoints, seeds |
| `src/init.cpp` | Startup, and the authoritative list of config options |
| `src/rpc/`, `src/wallet/rpc/` | Where every RPC method is defined, with its help text |

## Entry points

Where the well-known paths start. **Line numbers move every release — re-derive them**;
these were read at master on 2026-09-02:

| Concept | Symbol | Location then |
|---|---|---|
| Block validity | `CheckBlock` | `src/validation.cpp:2319` |
| Applying a block | `ConnectBlock` | `src/validation.cpp:222` |
| Transaction validity | `CheckTransaction` | `src/validation.cpp:795` |
| Header context checks | `ContextualCheckBlockHeader` | `src/validation.cpp:2308` |
| Block subsidy / halving | `GetBlockSubsidy` | `src/validation.cpp:1833` |
| Proof of work | `CheckProofOfWork` | `src/validation.cpp:3839` |
| Script execution | `EvalScript` | `src/script/interpreter.cpp:417` |
| The 21M cap | `MAX_MONEY` | `src/consensus/amount.h:26` |

Mempool acceptance has more than one entry point, and which one matters depends on the
question. Verified in `src/validation.h` at that ref:

| Symbol | Role |
|---|---|
| `AcceptToMemoryPool` | Single-transaction acceptance — still present, despite frequent rumours of its removal |
| `ProcessNewPackage` | Package (multi-transaction) acceptance |
| `MemPoolAccept` | The internal class both go through |

```bash
grep -n "MemPoolAccept\|AcceptToMemoryPool\|ProcessNewPackage" src/validation.h
```

Check which exists in the version you are answering about rather than assuming — this area
has been restructured repeatedly, which is exactly why this skill insists on citing a ref.

## Tests as documentation

Often the fastest way to learn what a rule *means* is the test that pins it down.

- `test/functional/` — Python, drives real nodes, named by feature: `feature_taproot.py`,
  `feature_rbf.py`, `mempool_*.py`, `p2p_*.py`, `wallet_*.py`. Highly readable, and they
  state expected behaviour explicitly.
- `src/test/` — C++ unit tests, closer to the code, better for a specific function.

```bash
ls test/functional/ | grep -i taproot
```

## Search tactics

Cut the noise on a first pass — tests, benchmarks, GUI, and vendored subtrees are usually
not the answer:

```bash
grep -rn "MoneyRange" src \
  --include="*.cpp" --include="*.h" \
  | grep -vE "src/(test|bench|qt|secp256k1|leveldb|minisketch|univalue|crc32c)/"
```

Other useful moves:

```bash
# Where is a constant defined, as opposed to used?
grep -rn "constexpr.*MAX_BLOCK_WEIGHT" src/consensus/

# Which RPC method implements this, and what's its help text?
grep -rn "\"getblockchaininfo\"" src/rpc/ src/wallet/rpc/

# Which config option controls this?
grep -rn "\-txindex\|\"txindex\"" src/init.cpp src/index/
```

When a name is too common to grep usefully, that is the moment for clangd's
`findReferences` rather than more grep — see `setup.md`.
