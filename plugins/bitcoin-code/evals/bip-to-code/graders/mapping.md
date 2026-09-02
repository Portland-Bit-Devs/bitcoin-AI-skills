Tests whether the model uses Core's own BIP mapping rather than recalling
version numbers, and whether it avoids the deleted-deployment trap.

Required to pass — the response must:

1. Identify `doc/bips.md` in the Bitcoin Core tree as the authoritative mapping
   of BIP → implementing version → PR (or otherwise derive the version from the
   repository rather than asserting it from memory).
2. Name taproot as BIPs 340/341/342 — at minimum 341, ideally noting 340
   (Schnorr) and 342 (tapscript) are part of it.
3. Point to script-verification enforcement — `SCRIPT_VERIFY_TAPROOT` in
   `src/script/interpreter.h` / `interpreter.cpp`, and/or the flag set in
   `src/validation.cpp` — as where the rules actually live.

Strong responses also:

- Warn that searching for a BIP 9 deployment constant such as
  `DEPLOYMENT_TAPROOT` will find nothing, because activation machinery is
  removed once a fork is buried — and that this is not evidence taproot is
  absent.
- Note that `doc/bips.md` groups 340/341/342 in one entry, so grepping the
  literal string "BIP 341" can miss it.
- Distinguish "implemented in release X" from "activated on mainnet at height Y".
- State the ref the answer was read at, or offer to verify.

Fail the response if it:

- States a version number with no source and no offer to verify.
- Claims taproot activation is configured via a live BIP 9 deployment in current
  master.
- Confuses the BIP's own status with whether Core implements it.
