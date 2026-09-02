Tests whether the model knows the Debug Console is not evidence of an HTTP
server. The user's stated reasoning ("RPC is clearly working") is the trap.

Required to pass — the response must state:

1. `bitcoin-qt` has the HTTP JSON-RPC server **disabled** by default, unlike
   `bitcoind`, and it is enabled with `server=1` in `bitcoin.conf` (or `-server`).
2. The Debug Console runs RPC methods **inside the process**, not over HTTP, so
   it working does not indicate that port 8332 is listening.
3. `bitcoin.conf` is read at startup, so the app must be restarted.

Strong responses also:

- Give the macOS config path `~/Library/Application Support/Bitcoin/bitcoin.conf`
  and note the file does not exist by default.
- Suggest verifying with `lsof -nP -iTCP:8332 -sTCP:LISTEN`.
- Mention cookie authentication for the script, and that the cookie is
  regenerated on each restart.

Fail the response if it:

- Diagnoses this primarily as a firewall, credential, or client-code problem.
- Recommends `rpcallowip`/`rpcbind` changes for a localhost client.
- Accepts the user's premise that the working Console proves HTTP RPC is enabled.
