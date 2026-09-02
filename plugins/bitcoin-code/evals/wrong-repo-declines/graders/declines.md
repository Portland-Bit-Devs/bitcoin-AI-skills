Negative case. "When did this rule change and why", "consensus code", and "what
this PR changed" are conditional triggers; the repository is go-ethereum.

Required to pass — the response must:

1. Recognise the repository is not `bitcoin/bitcoin` and say so plainly.
2. Not present Core's tree map, the `src/consensus/` vs. `src/policy/` split, or
   any Core symbol as though it described go-ethereum.

Strong responses also:

- Note that the *techniques* are portable — `git log -S`, `git blame`, merge
  commit to PR — while the tree map and symbols are not, and offer the former.
- Preserve the citation discipline (name a ref) even while declining.

Fail the response if it:

- Cites Bitcoin Core file paths or symbols as relevant.
- Applies Bitcoin's consensus-vs-policy distinction to Ethereum as if it mapped
  cleanly.
- Invents go-ethereum paths to look helpful.
