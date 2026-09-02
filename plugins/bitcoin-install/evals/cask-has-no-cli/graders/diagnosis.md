Tests the core distinction this skill exists to teach.

Required to pass — the response must state all of:

1. The install did **not** fail, and reinstalling the cask will not help.
2. The Homebrew **cask** `bitcoin-core` ships only the GUI app and contains no
   command-line binaries.
3. The fix is to additionally install the Homebrew **formula**: `brew install bitcoin`.

Strong responses also:

- Note that the two packages coexist and can both be installed.
- Note that the formula provides `bitcoind`, `bitcoin-cli`, `bitcoin-tx`,
  `bitcoin-util`, `bitcoin-wallet`, and `bitcoin`.
- Mention that `bitcoin-cli` will then work against the already-running GUI node
  via the shared datadir and cookie — **provided** `server=1` is set, since
  bitcoin-qt disables the HTTP RPC server by default.
- Offer `brew list --cask` / `brew list --formula` to confirm what is installed.

Fail the response if it:

- Says the installation is broken or corrupted.
- Recommends only a PATH fix, symlinking into the app bundle, or
  `brew reinstall --cask bitcoin-core` as the solution.
- Claims the cask does include CLI tools.
