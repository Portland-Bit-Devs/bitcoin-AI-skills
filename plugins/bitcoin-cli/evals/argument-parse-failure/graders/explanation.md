Tests whether the model explains the actual mechanism rather than offering a
quoting superstition.

Required to pass — the response must convey:

1. `bitcoin-cli` knows each method's parameter *types* and JSON-parses the
   arguments that are non-string (numeric, boolean, array, object), while
   passing string arguments through untouched.
2. `getblockhash`'s argument is numeric, so `abc` is JSON-parsed and fails;
   `getblock`'s is a string, so it needs no quoting.
3. The real fix is to pass a number: `bitcoin-cli getblockhash 800000`.

Strong responses also:

- Note the error came from **bitcoin-cli itself**, not the node — nothing was
  sent — and that this is visible in the message form (`error: Error parsing
  JSON` vs. `error code: -N`).
- Recommend `bitcoin-cli help getblockhash` to see declared parameter types.
- Mention `-named` (or `bitcoin rpc`) as more robust.
- Note that JSON array/object arguments additionally need single quotes to
  survive the shell.

Fail the response if it:

- Says string arguments need quotes added, or gives a blanket "always quote
  arguments" rule that doesn't explain the difference.
- Claims the node rejected the argument.
- Attributes the failure to the shell.
