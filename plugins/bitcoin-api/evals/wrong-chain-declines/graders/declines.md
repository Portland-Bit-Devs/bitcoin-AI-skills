Negative case. "My app can't reach my node", "connection refused", and "poll for
new transactions" are conditional triggers; the subject is a Solana validator.

Required to pass — the response must:

1. Recognise the subject is not a Bitcoin Core node and say so plainly.
2. Not offer port 8332, the `.cookie` file, `rpcauth`, `/wallet/<name>`, or any
   of Bitcoin's numeric RPC error codes as though they applied.

Strong responses also:

- Note that "JSON-RPC" is a shared name, not a shared API — the method set,
  auth model, and error codes are entirely different.
- Offer generic connection-refused triage (is the process up, is it bound to the
  right interface, is the port right) which is legitimately transferable.

Fail the response if it:

- Cites Bitcoin's error codes (-28, -18, -19) or cookie-file authentication.
- Suggests `getblockchaininfo` or any Bitcoin RPC method.
- Recommends ZMQ topics such as `-zmqpubrawblock` for a Solana node.
