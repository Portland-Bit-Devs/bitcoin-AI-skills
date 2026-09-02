I'm running Bitcoin Core 31 as the desktop app on macOS and I'm fully synced.
The Debug Console inside the app answers `getblockcount` just fine.

But from my terminal this hangs and then fails with `Connection refused`:

```
curl --user foo:bar --data-binary '{"method":"getblockcount","params":[]}' http://127.0.0.1:8332/
```

Why can't I reach it?
