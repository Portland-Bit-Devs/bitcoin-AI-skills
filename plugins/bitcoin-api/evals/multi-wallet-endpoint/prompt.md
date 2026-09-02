My Python service calls `getbalance` against my Bitcoin Core node at
`http://127.0.0.1:8332/` and it worked fine for months. Since this morning every
call comes back with:

```json
{"result": null, "error": {"code": -19, "message": "Wallet file not specified"}, "id": "1"}
```

I haven't deployed any code. What's going on?
