# Setting up: clones and code intelligence

> **Read this when** there is no local checkout yet, or when go-to-definition and
> find-references don't work on the Bitcoin source.

**In this file**

| Section | Answers |
|---|---|
| Finding an existing checkout | Look before you clone |
| **Interviewing the user** | **What to ask before a multi-hundred-MB fetch** |
| Cloning | Full vs. shallow, and why history matters for research |
| Code intelligence | clangd, and the three setup tiers |
| **The cheap tier** | **compile_flags.txt — real navigation, zero dependencies** |
| The full tier | CMake configure, and what it costs |
| Keeping current | Updating without disturbing the user's work |

---

## Finding an existing checkout

Look before asking. Common locations:

```bash
for d in ~/src/bitcoin ~/IdeaProjects/bitcoin ~/Projects/bitcoin ~/code/bitcoin ~/bitcoin; do
  [ -d "$d/.git" ] && git -C "$d" remote get-url origin 2>/dev/null | grep -q bitcoin && echo "found: $d"
done
```

Confirm it is really Core and not something else with the same directory name — check the
remote and a landmark file:

```bash
git -C "$D" remote -v | head -1
test -f "$D/src/validation.cpp" && test -f "$D/src/consensus/amount.h" && echo "looks like Core"
```

A directory called `bitcoin` is very often a user's own project rather than upstream.

## Interviewing the user

A Core clone is a **multi-hundred-MB fetch** and a long-lived directory on their disk.
Never clone silently. Ask, and ask enough to only do it once:

1. **Do you want a local checkout at all?** Without one, source questions fall back to
   fetching individual files from GitHub — workable for a known file, poor for searching,
   and it cannot do `git log` or `blame` at all.
2. **Where should it go?** Offer concrete paths and check for collisions first. Do not
   assume `~/bitcoin` is free; a personal project of the same name is common. Never clone
   into a directory that already contains something.
3. **Full history or shallow?** This is the consequential one — see below.
4. **Also clone the BIPs?** It is small (~40 MB) and answers a different class of question.
5. **How far to set up the build?** Only matters for code intelligence — see the tiers.

Report the paths back afterwards so the user knows what landed where.

## Cloning

```bash
git clone https://github.com/bitcoin/bitcoin.git "$DEST"
git clone https://github.com/bitcoin/bips.git   "$BIPS_DEST"
```

Measured on a full clone: `bitcoin/bitcoin` is ~316 MB transferred and **~387 MB on disk**
with 50,403 commits; `bitcoin/bips` is ~22 MB transferred, ~43 MB on disk, 210 BIP files.

**Full history is the point for research.** A shallow clone (`--depth 1`) is ~5× smaller
and much faster, but it gives up exactly the capabilities that distinguish research from
code reading:

| Capability | Full | Shallow |
|---|---|---|
| Read current code | ✅ | ✅ |
| `git log -S` — when was this introduced | ✅ | ❌ |
| `git blame` — who changed this line and why | ✅ | ❌ |
| Check out a release tag | ✅ | ❌ |
| Find the merge commit naming a PR | ✅ | ❌ |

A middle option preserves history while skipping old file blobs until needed:

```bash
git clone --filter=blob:none https://github.com/bitcoin/bitcoin.git "$DEST"
```

A shallow clone can be upgraded later with `git fetch --unshallow`.

## Code intelligence

The language server for C++ is **clangd**. In Claude Code it is wired in by a plugin:

```bash
claude plugin install clangd-lsp@claude-plugins-official
```

That plugin declares clangd for `.c/.h/.cpp/.cc/.cxx/.hpp/.hxx` and runs it with
`--background-index`. It requires the `clangd` binary on your `PATH` — on macOS with Xcode
installed you already have one (`xcrun -f clangd`); otherwise `brew install llvm`.

Two things to know before you count on it:

- **`ENABLE_LSP_TOOL=1`** must be set in `~/.claude/settings.json` for the LSP tool to exist
  at all.
- **Installing the plugin is not enough for the current session.** Language servers are
  loaded at session start, so the LSP tool keeps reporting
  `No LSP server available for file type: .h` until Claude Code is restarted. Verified.

How well clangd works then depends entirely on whether it can figure out the compiler flags
for each file. Three tiers:

| Tier | Setup | Cost | Result |
|---|---|---|---|
| 0 | Nothing | — | Wrong C++ standard, broken includes. Near useless on Core. |
| 1 | `compile_flags.txt` | seconds, no dependencies | **Works for most files.** Fails only where a generated header is included. |
| 2 | `cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | ~1–2 min + `brew install boost capnp` | Full intelligence across the tree. |

A **full build is not required** for any of this. Tier 2 is a *configure*, not a compile.

## The cheap tier: compile_flags.txt

Core uses angle-bracket includes rooted at `src/` (`#include <consensus/amount.h>`) and
requires C++20. Tell clangd both, in a file at the repository root:

```bash
printf -- '-std=c++20\n-Isrc\n-I.\n' > "$DEST/compile_flags.txt"
```

Verified effect on a fresh clone with nothing installed:

| File | Before | After |
|---|---|---|
| `src/consensus/tx_check.cpp` | broken includes | **0 errors** |
| `src/consensus/amount.h` | `inline variables are a C++17 extension` | standard resolved |
| `src/validation.cpp` | — | still fails: `'bitcoin-build-config.h' file not found` |

That last row is the whole limitation. `bitcoin-build-config.h` is generated by CMake from
`cmake/bitcoin-build-config.h.in`, so any file including it — which is most `.cpp` under
`src/` — needs tier 2. Headers and self-contained sources work fine at tier 1.

Check any file yourself without starting a server:

```bash
clangd --check=src/consensus/tx_check.cpp 2>&1 | grep "All checks completed"
```

**Keep it out of the user's `git status`.** `compile_flags.txt` is not in Core's
`.gitignore`, so it shows up as untracked and pollutes every `git status` they run. Exclude
it locally instead of editing a tracked file:

```bash
printf 'compile_flags.txt\n.cache/\nbuild/\n' >> "$DEST/.git/info/exclude"
```

`.git/info/exclude` is per-clone and never committed — the right tool for local-only
artefacts in someone else's repository.

## The full tier: CMake configure

```bash
brew install boost capnp          # cmake is also required
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
ln -sf build/compile_commands.json compile_commands.json
```

This generates `bitcoin-build-config.h` and a `compile_commands.json` with exact flags for
every translation unit, which is what clangd really wants. **Verified: the configure step
fails without Boost** — `Could not find a package configuration file provided by "Boost"` —
so the `brew install` is not optional.

Building (`cmake --build build`) adds nothing for navigation. Do it only if you want to run
tests.

## Keeping current

The user's checkout is theirs. Read-only git is always fine; anything that moves refs or
touches the working tree is not.

```bash
git -C "$DEST" fetch origin          # safe: updates remote refs only
git -C "$DEST" log --oneline -5 origin/master
```

**Do not** `git pull`, `checkout`, `reset`, or `clean` in a checkout you did not create —
they may have work in progress. To read an older release without disturbing anything, use
`git show` with a ref instead of checking it out:

```bash
git -C "$DEST" show v28.0:src/consensus/amount.h | head -30
```

That reads any file at any tag with no working-tree change at all, and is the right way to
compare two releases.
