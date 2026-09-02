Tests whether the model recommends ZMQ *and* correctly describes its delivery
guarantees. Recommending ZMQ while implying it is reliable is a fail — this is
the trap.

Required to pass — the response must state all of:

1. Recommend ZMQ notifications (`-zmqpubhashblock` / `-zmqpubrawblock`, and/or
   `-zmqpubsequence`) instead of polling.
2. State explicitly that ZMQ delivery is **not guaranteed** — messages can be
   dropped, there is no replay or acknowledgement.
3. Therefore recommend treating a notification as a hint and confirming the
   actual state via RPC or REST, rather than treating the ZMQ message as the
   authoritative record.

Strong responses also:

- Recommend `sequence` over `hashblock` for reorg awareness, noting that the
  `*block` topics only announce the new tip and do not report disconnections.
- Note the per-topic sequence number can be used to detect a gap.
- Warn that ZMQ is not compiled into every build, and suggest
  `getzmqnotifications` to check.
- Mention reconciling against the node on startup and after any detected gap.
- Note that for money-handling specifically, confirmation depth still governs
  settlement.

Fail the response if it:

- Presents ZMQ as a reliable delivery mechanism, or omits requirement 3.
- Only suggests raising `rpcworkqueue`/`rpcthreads` without changing the polling
  architecture.
- Recommends a third-party API or block explorer as the primary fix, given the
  user runs their own node.
