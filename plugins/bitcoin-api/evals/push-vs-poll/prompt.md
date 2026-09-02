I'm building a payment processor on top of my own Bitcoin Core node. Right now
it calls `getblockcount` every second, and when the number changes it fetches
the new block and looks for payments to our addresses.

It's hammering the node and we sometimes hit "Work queue depth exceeded". What's
the better design?
