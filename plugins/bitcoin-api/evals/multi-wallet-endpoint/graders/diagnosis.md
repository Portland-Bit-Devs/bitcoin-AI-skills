The response must correctly diagnose an environment change, not a code bug.

Required to pass — the response must state all of:

1. Error `-19` means more than one wallet is now loaded on the node (or, more
   generally, that the bare `/` endpoint cannot service a wallet call unless
   exactly one wallet is loaded).
2. The cause is a change on the node, which is consistent with the user not
   having deployed code.
3. The fix is to address the `/wallet/<walletname>` endpoint explicitly, e.g.
   `http://127.0.0.1:8332/wallet/mywallet`.

Strong responses also:

- Suggest `listwallets` to see which wallets are loaded.
- Recommend addressing `/wallet/<name>` unconditionally going forward, rather
  than relying on exactly one wallet being loaded.

Fail the response if it:

- Attributes the error to authentication, a stale cookie, or a network problem.
- Claims `-19` means the wallet does not exist (that is `-18`).
- Recommends unloading the second wallet as the primary fix without mentioning
  the endpoint.
