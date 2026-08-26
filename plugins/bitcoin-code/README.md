# bitcoin-code

Read and explain the Bitcoin Core source tree.

**Status:** stub — structure only, reference material not yet written.

```bash
claude plugin install bitcoin-code@bitcoin-ai-skills
```

Provides:

- the `bitcoin-code` skill — orientation in the Bitcoin Core tree
- the `bitcoin-source-reader` agent — a read-only explorer that searches
  `github.com/bitcoin/bitcoin` and answers with `file:line` citations at a pinned ref

The agent may clone the Bitcoin Core repository (a several-hundred-megabyte fetch) into a
scratch directory. It never writes to a Bitcoin Core checkout it did not create.

Licensed GPL-3.0. See the [repository root](../../README.md).
