# Arguments and quoting

> **Read this when** a command won't parse, you're passing JSON, or you're wondering why
> one argument needs quotes and the next one doesn't.

**In this file**

| Section | Answers |
|---|---|
| **The conversion rule** | **Why `getblock <hash>` works but `getblockhash abc` doesn't** |
| Positional arguments | The default form, and its upgrade hazard |
| Named arguments | `-named`, and why to prefer it |
| Mixing the two | The `args` convention |
| JSON arguments | Two levels of quoting, and the shell traps |
| Empty and default arguments | Skipping a middle parameter |
| Diagnosing a parse failure | Is it your shell, the client, or the node? |

---

## The conversion rule

This one mechanism explains nearly every confusing `bitcoin-cli` argument error.

**`bitcoin-cli` carries a per-method table of which arguments are JSON and which are
strings.** For each argument of each method it either

- **parses it as JSON** — numbers, booleans, arrays, objects — or
- **passes it through as a plain string**, untouched.

Consequences, both verified:

```bash
bitcoin-cli getblock 0f9188f13cb7b2c71f2a335e3a4fc328bf5beb436012afca590b1a11466e2206
#  → works. getblock's first argument is a string; no parsing, no quotes needed.

bitcoin-cli getblockhash abc
#  → error: Error parsing JSON: abc
#  getblockhash's argument is numeric, so the client tries to JSON-parse "abc" and fails.
```

Note where each failed. The first would be rejected by the *node*; the second never left
the client. That distinction is the fastest way to diagnose a bad command — see the last
section.

The practical rule: **string arguments need no quotes; anything structured must be valid
JSON.** You do not have to memorise which is which, because `help <method>` states each
parameter's type:

```bash
bitcoin-cli help getblockhash    # Argument #1 - height: numeric, required
```

## Positional arguments

The default form. Arguments go in declaration order:

```bash
bitcoin-cli getblock <hash> 2
bitcoin-cli gettxout <txid> 0 true
```

Compact, and fine interactively. The hazard is upgrades: if a future release inserts a
parameter into the middle of a signature, every positional call silently shifts and starts
passing values to the wrong parameters. Nothing errors; the meaning just changes.

For anything you save — a script, a cron job, a runbook — prefer named.

## Named arguments

```bash
bitcoin-cli -named getblock blockhash=<hash> verbosity=2
bitcoin-cli -named createwallet wallet_name=mywallet load_on_startup=true
```

Order stops mattering, you skip defaults you don't care about, and the command documents
itself. It is immune to the parameter-insertion hazard above.

`bitcoin rpc` is exactly `bitcoin-cli -named`, if you have the unified wrapper:

```bash
bitcoin rpc getblock blockhash=<hash> verbosity=2
```

An unknown parameter name is caught by the node with `-8 Unknown named parameter`, which is
a far better failure than a positional call quietly doing the wrong thing.

## Mixing the two

Every method also accepts a special `args` parameter holding leading positional values,
combined with named ones:

```bash
bitcoin-cli -named createwallet mywallet load_on_startup=true
```

Here `mywallet` fills the first positional slot and `load_on_startup` is named. Useful when
the first argument is obvious and the rest are not.

## JSON arguments

Some methods take arrays or objects. There are **two levels of quoting**: the shell's, and
JSON's. Wrap the whole value in single quotes so the shell passes it through intact, and
use double quotes inside for JSON:

```bash
bitcoin-cli createrawtransaction '[]' '{"data":"48656c6c6f"}'

bitcoin-cli -named createrawtransaction \
  inputs='[{"txid":"<txid>","vout":0}]' \
  outputs='[{"<address>":0.01}]'
```

**Without the single quotes, your shell mangles it before `bitcoin-cli` ever sees it.** In
zsh, an unquoted `[]` fails with `no matches found` — the shell tried to glob it. In bash
the failure is different but no less real. This is a shell problem, not a Bitcoin problem,
and the error message will not mention Bitcoin.

When a value contains a shell variable, you cannot use single quotes for the whole thing.
Build it with a tool that escapes correctly rather than by hand:

```bash
OUTPUTS=$(jq -nc --arg addr "$ADDR" '[{($addr): 0.01}]')
bitcoin-cli -named createrawtransaction inputs='[]' outputs="$OUTPUTS"
```

`jq -n` builds the JSON, `--arg` inserts the variable safely, and `-c` keeps it on one
line. String concatenation with `echo` will eventually bite you on an address or label
containing something surprising.

## Empty and default arguments

To reach a later positional parameter you must fill the earlier ones. `""` is the usual
placeholder for "default":

```bash
bitcoin-cli -rpcwallet=w1 getnewaddress "" bech32m     # empty label, explicit type
```

This is exactly the awkwardness `-named` removes:

```bash
bitcoin-cli -named -rpcwallet=w1 getnewaddress address_type=bech32m
```

## Diagnosing a parse failure

Read *which* component complained. The three failures look similar and have different
fixes:

| Message | Who rejected it | Fix |
|---|---|---|
| `zsh: no matches found: [...]` (or shell-specific noise) | **Your shell** | Quote the argument in single quotes |
| `error: Error parsing JSON: <value>` | **`bitcoin-cli`, locally** | That parameter is JSON-typed; give it valid JSON |
| `error code: -8` / `-3` / `-1` with a message | **The node** | The command reached the node; the value is wrong. See `bitcoin-api`'s error tables |

A local parse error exits `1`. A node error exits with the RPC code — see `output.md`.
