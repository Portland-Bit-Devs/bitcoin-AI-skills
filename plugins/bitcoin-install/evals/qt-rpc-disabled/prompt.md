I want my Python script to talk to my Bitcoin Core node. I'm running the desktop
app on macOS, installed via Homebrew, and it's fully synced.

I opened the app's Console window and `getblockcount` works there, so RPC is
clearly working. But my script gets `Connection refused` on port 8332.

What am I missing?
