# The flake evaluation cache and a dirty Git tree

Nix caches flake evaluation results (`~/.cache/nix/eval-cache-v6/`), keyed on
the flake's locked source. For a flake in a Git working tree, the cache is
only used when the tree is **clean**: any modified tracked file — staged or
not — makes the source "dirty", and Nix evaluates from scratch every time.

This matters most for anything that runs *while* you are changing files —
above all a Git hook, which by definition runs with uncommitted changes.

## Measured

`nix develop --no-update-lock-file .#<shell> -c true` on a small repository
(about 3 MB), Nix 2.31.4, macOS, five runs each:

| Tree | Small shell | Larger shell (Neovim overlay) |
| --- | --- | --- |
| clean | 0.38–0.39 s | 0.41–0.43 s |
| one tracked file modified (staged or unstaged) | 1.27–1.34 s | 1.70 s |

A fresh clone at the same commit hit the cache on its first run (0.39 s): the
key is the source, not the checkout path. Repeating the dirty run did not get
faster — there is no cache to warm.

So a hook built as `exec nix develop .#shell -c <tool>` costs about 1.3 s of
evaluation per invocation at commit time, not the 0.4 s you measure on a clean
tree. Measure with a modified file, or you measure the wrong number.

## It evaluates the working tree, not the index

A partially staged commit is checked by a dev shell built from the unstaged
edits too. Nix warns `Git tree '…' is dirty` and reads tracked files as they
are on disk: an unstaged syntax error appended to `flake.nix` made the hook
fail although nothing broken was staged. This only matters when the flake
itself is among the edits.
