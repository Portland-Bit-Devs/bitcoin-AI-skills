# Worked examples

> **Read this when** you want something that runs, rather than an explanation. Every
> example here was executed against a live Bitcoin Core v31.0.0 node, error paths included.

All examples are **read-only**. See the Safety section of `SKILL.md` before adapting any of
them into something that spends.

**In this file**

| Section | Gives you |
|---|---|
| Shell | A `btc()` helper — **later shell snippets assume it** |
| Batching a block range | Many heights in one round trip, matched by `id` |
| Python client | A real client: cookie re-reading, `-28` retry, error branch |
| JavaScript / Node | The same, with `fetch` and no dependencies |
| REST | Unauthenticated `GET`s, including raw binary |
| Postman | Environment variables and the cookie-goes-stale caveat |
| Node health check | A monitoring script with correct exit codes |

Paths in these examples are macOS (`~/Library/Application Support/Bitcoin`). On Linux use
`~/.bitcoin`; for a non-mainnet network, append the network subdirectory (`regtest`,
`signet`, `testnet3`, `testnet4`) and change the port — see `connection.md`.

---

## Shell

Set up once:

```bash
# macOS. Linux: ~/.bitcoin  •  regtest: append /regtest
export BTC_DATADIR="$HOME/Library/Application Support/Bitcoin"
export BTC_RPC="http://127.0.0.1:8332"

btc() {
  local method="$1"; shift
  curl -s --user "$(cat "$BTC_DATADIR/.cookie")" \
    -H 'content-type: application/json' \
    --data-binary "{\"jsonrpc\":\"2.0\",\"id\":\"1\",\"method\":\"$method\",\"params\":[$*]}" \
    "$BTC_RPC/"
}
```

```bash
btc getblockcount
btc getblockchaininfo | jq '{chain, blocks, headers, verificationprogress, pruned}'
btc getblockhash 800000
btc getblock '"00000000000000000002a7c4c1e48d76c5a37902165a270156b7a8d72728a054"' 1 | jq '.tx | length'
```

Note the nested quoting: string parameters need JSON quotes *inside* the shell quotes.
This is the single most common mistake when hand-writing these calls.

Am I synced?

```bash
btc getblockchaininfo | jq -r '
  if .result.initialblockdownload then "syncing: \(.result.verificationprogress * 100 | floor)%"
  else "synced at height \(.result.blocks)" end'
```

Is the node alive at all, without authenticating? The cookie file's existence answers it —
it is written on startup and removed on clean shutdown:

```bash
test -f "$BTC_DATADIR/.cookie" && echo "node is running" || echo "node is not running"
```

## Batching a block range

One round trip instead of a hundred. Match responses by `id`, since order is not
guaranteed:

```bash
python3 - <<'PY'
import json, os, subprocess

datadir = os.path.expanduser("~/Library/Application Support/Bitcoin")
cookie = open(os.path.join(datadir, ".cookie")).read().strip()

batch = [
    {"jsonrpc": "2.0", "id": str(h), "method": "getblockhash", "params": [h]}
    for h in range(800000, 800100)
]
out = subprocess.run(
    ["curl", "-s", "--user", cookie, "-H", "content-type: application/json",
     "--data-binary", json.dumps(batch), "http://127.0.0.1:8332/"],
    capture_output=True, text=True,
).stdout

by_id = {r["id"]: r for r in json.loads(out)}
for h in range(800000, 800005):
    print(h, by_id[str(h)]["result"])
PY
```

## Python client

No dependency beyond `requests`. Handles the two things a real client must handle: cookie
re-reading, and `-28` on startup.

```python
import json, os, time
import requests

class BitcoinRPC:
    def __init__(self, datadir, url="http://127.0.0.1:8332", wallet=None):
        self.datadir = datadir
        self.url = url + (f"/wallet/{wallet}" if wallet else "/")
        self.session = requests.Session()   # reuse the connection; see the fd limit
        self._id = 0

    def _auth(self):
        # Re-read every call: the cookie is regenerated on every node restart.
        with open(os.path.join(self.datadir, ".cookie")) as f:
            user, _, password = f.read().strip().partition(":")
        return (user, password)

    def call(self, method, *args, **kwargs):
        params = kwargs if kwargs else list(args)
        if args and kwargs:
            params = {"args": list(args), **kwargs}   # Core's mixed-parameter convention
        self._id += 1
        resp = self.session.post(
            self.url,
            auth=self._auth(),
            headers={"content-type": "application/json"},
            data=json.dumps({
                "jsonrpc": "2.0", "id": str(self._id),
                "method": method, "params": params,
            }),
        )
        # Under JSON-RPC 2.0 an application error still arrives as HTTP 200,
        # so the body's "error" field is the only reliable signal.
        body = resp.json()
        if body.get("error"):
            raise BitcoinRPCError(body["error"]["code"], body["error"]["message"])
        return body["result"]

    def wait_until_ready(self, timeout=300):
        """Block until the node is out of warmup. -28 is expected, not fatal."""
        deadline = time.time() + timeout
        delay = 1
        while time.time() < deadline:
            try:
                return self.call("getblockchaininfo")
            except BitcoinRPCError as e:
                if e.code != -28:
                    raise
                time.sleep(delay)
                delay = min(delay * 2, 15)
        raise TimeoutError("node did not finish warming up")

class BitcoinRPCError(Exception):
    def __init__(self, code, message):
        super().__init__(f"[{code}] {message}")
        self.code, self.message = code, message


rpc = BitcoinRPC(os.path.expanduser("~/Library/Application Support/Bitcoin"))
info = rpc.wait_until_ready()
print(info["chain"], info["blocks"])

# Named parameters — resilient to signature changes across versions
print(rpc.call("getblockhash", height=800000))
```

## JavaScript / Node

```javascript
import { readFileSync } from "node:fs";
import { join } from "node:path";

const DATADIR = join(process.env.HOME, "Library/Application Support/Bitcoin");

async function rpc(method, params = [], wallet = null) {
  const cookie = readFileSync(join(DATADIR, ".cookie"), "utf8").trim();
  const url = `http://127.0.0.1:8332/${wallet ? `wallet/${wallet}` : ""}`;

  const res = await fetch(url, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: "Basic " + Buffer.from(cookie).toString("base64"),
    },
    body: JSON.stringify({ jsonrpc: "2.0", id: "1", method, params }),
  });

  // Do NOT branch on res.ok alone: under JSON-RPC 2.0 an application-level
  // failure is still HTTP 200 with an error object in the body.
  const body = await res.json();
  if (body.error) {
    const err = new Error(body.error.message);
    err.code = body.error.code;
    throw err;
  }
  return body.result;
}

console.log(await rpc("getblockchaininfo"));
```

## REST — no auth needed

```bash
curl -s http://127.0.0.1:8332/rest/chaininfo.json | jq '{chain, blocks}'
curl -s http://127.0.0.1:8332/rest/mempool/info.json | jq
curl -s "http://127.0.0.1:8332/rest/blockhashbyheight/800000.json" | jq -r .blockhash

# Raw binary block straight to disk — no hex, no JSON
curl -s -o block.dat "http://127.0.0.1:8332/rest/block/$(
  curl -s http://127.0.0.1:8332/rest/blockhashbyheight/800000.json | jq -r .blockhash
).bin"
```

## Postman

Configure an environment with `protocol`, `server`, `port`, `username`, `password`, then:

- **Method**: `POST`
- **URL**: `{{protocol}}://{{server}}:{{port}}/`
- **Auth**: Basic, `{{username}}` / `{{password}}`
- **Headers**: `content-type: application/json`
- **Body**: raw JSON —
  ```json
  {"jsonrpc": "2.0", "id": "1", "method": "getblockcount", "params": []}
  ```

For cookie auth, `username` is the literal `__cookie__` and `password` is the portion of
`.cookie` after the colon. It changes on every node restart, so a saved Postman
environment goes stale — re-paste it, or configure `rpcauth` for static credentials.

Do not commit a Postman environment export containing real credentials.

## Node health check for monitoring

```bash
#!/usr/bin/env bash
# Exit 0 if the node is synced and reachable; non-zero otherwise.
set -euo pipefail
DATADIR="${BTC_DATADIR:-$HOME/Library/Application Support/Bitcoin}"

[ -f "$DATADIR/.cookie" ] || { echo "CRITICAL: node not running"; exit 2; }

resp=$(curl -sf --max-time 10 --user "$(cat "$DATADIR/.cookie")" \
  -H 'content-type: application/json' \
  --data-binary '{"jsonrpc":"2.0","id":"1","method":"getblockchaininfo","params":[]}' \
  http://127.0.0.1:8332/) || { echo "CRITICAL: RPC unreachable"; exit 2; }

code=$(jq -r '.error.code // empty' <<<"$resp")
if [ "$code" = "-28" ]; then echo "WARNING: node warming up"; exit 1; fi

jq -er '
  .result |
  if .initialblockdownload
  then "WARNING: IBD \(.verificationprogress * 100 | floor)%\n" | halt_error(1)
  else "OK: height \(.blocks)" end
' <<<"$resp"
```
