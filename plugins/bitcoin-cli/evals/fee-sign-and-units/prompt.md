I'm writing a script to total up what I've paid in fees. I'm doing this:

```bash
bitcoin-cli -rpcwallet=w1 gettransaction "$TXID" | jq '.fee'
```

and getting values like `-0.00001650`. Two things confuse me: why is it
negative, and can I just add these up to get my total fee in satoshis?
