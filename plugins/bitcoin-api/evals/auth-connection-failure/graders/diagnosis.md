Tests whether the model knows that the GUI's Debug Console does not imply an
HTTP RPC server is listening.

Required to pass — the response must state:

1. `bitcoin-qt` has the JSON-RPC HTTP server **disabled** by default (unlike
   `bitcoind`, which enables it), and it must be turned on with `server=1` in
   `bitcoin.conf` or `-server` on the command line.
2. The Debug Console executes RPC methods internally, without going over HTTP,
   so it working tells the user nothing about port 8332 being open. (Wording may
   vary; the substance is that the console is not evidence the port is
   listening.)
3. The node must be restarted after the config change.

Strong responses also:

- Note that `--user foo:bar` will fail even once the server is on, and point to
  the `.cookie` file in `~/Library/Application Support/Bitcoin/` as the correct
  credential — e.g. `--user "$(cat "$HOME/Library/Application Support/Bitcoin/.cookie")"`.
- Mention that the cookie is regenerated on each restart.
- Suggest verifying with `lsof -nP -iTCP:8332 -sTCP:LISTEN`.
- Recommend `-H 'content-type: application/json'`.

Fail the response if it:

- Diagnoses this as primarily a firewall problem.
- Recommends setting `rpcallowip=0.0.0.0/0`, exposing the port publicly, or any
  remote-access configuration — the client here is on localhost.
- Claims the credentials are the cause of `Connection refused` (a refused
  connection happens before authentication; wrong credentials would give 401).
