Negative case. Every phrase here ("syncs really slowly", "do I need Docker for
this", Homebrew, macOS) is a trigger for this skill, but the subject is Geth —
an Ethereum client. Tests that the skill's Preconditions block fires.

Required to pass — the response must:

1. Recognise the subject is **not Bitcoin Core** and say so plainly, in about a
   sentence.
2. Not present Bitcoin Core's formula/cask distinction, `bitcoin.conf`, datadir
   paths, or `brew install bitcoin` as though they applied.

Strong responses also:

- Offer what it *can* legitimately do — general Homebrew or Docker help — rather
  than only refusing.
- Note the skill would apply if the user switches to Bitcoin Core.

Fail the response if it:

- Answers with Bitcoin Core install guidance, package names, or config paths.
- Translates the formula-vs-cask framing onto Geth as if the structure carried
  over.
- Silently answers about Bitcoin without noticing the mismatch.
