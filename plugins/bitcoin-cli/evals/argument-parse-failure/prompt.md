I'm trying to get a block hash by height and bitcoin-cli won't cooperate:

```
$ bitcoin-cli getblockhash abc
error: Error parsing JSON: abc
```

But this works fine with no quotes at all:

```
$ bitcoin-cli getblock 0000000000000000000123456789abcdef...
```

Why does one need special treatment and the other doesn't?
