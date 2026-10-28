# Docker's detach keys swallow `C-p`

When a container runs with a terminal attached (`docker run -it`, and the
`exec`, `attach` and `start` subcommands, which all take `--detach-keys`), the
Docker CLI watches your keystrokes for its *detach sequence*: the keys that
leave the container running in the background and return you to the host
shell. The default sequence is `C-p C-q`.

## What it looks like

The sequence is two keys long, so the CLI holds back a `C-p` until it sees
the next key. The program in the container gets nothing, and it looks as if
it has stopped and is waiting for input:

- In Emacs, `C-p` (`previous-line`) does nothing at first. Observed with
  `docker run -it` on Docker 29.9.1 under Colima, with Emacs 30.1 in the
  container.
- Any other program that reads `C-p` is affected in the same way, for example
  shell history in bash or readline, or Vim's completion. This is reasoned
  from how the sequence is matched, not tested.

Nothing in the container is wrong, and nothing in the program's config
explains it. Because the key is caught on the host side, you won't find the
cause by debugging inside the container.

## Fix: choose a sequence the program does not use

```sh
docker run --rm -it --detach-keys=ctrl-^,ctrl-^ <image>
```

`docker run --help` lists `--detach-keys` but does not print the default;
`C-p C-q` comes from Docker's documentation. Docker 29.9.1 accepted
`ctrl-^,ctrl-^`.

`C-^` suits Emacs because `emacs -Q` leaves it unbound. Checked on 30.1:

```sh
emacs -Q --batch --eval '(princ (key-binding (kbd "C-^")))'   # => nil
```

A sequence that starts with a key the program *does* use brings the same
problem back with that key. Check the first key of whatever you choose.

## Not yet confirmed

- That `C-p` works normally with the new sequence in an interactive session.
  The flag was accepted, but nobody has pressed the keys since the change.
- Docker's docs also describe a `detachKeys` field in `~/.docker/config.json`
  that changes the default for every container. Not tested here, and it
  changes the default for every project, not just one.
