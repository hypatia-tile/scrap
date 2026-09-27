# Store path size versus closure size

A store path's **closure** is the path plus everything it references,
recursively. All of it must be present for the path to be usable, so
downloads, builds and disk usage are measured in closures. One package can be
tiny while its closure is enormous.

## `-S` is not "size"

```
$ nix path-info -sh /nix/store/…-lean-stage1
/nix/store/…-lean-stage1     5.8 MiB        # the path itself
$ nix path-info -Sh /nix/store/…-lean-stage1
/nix/store/…-lean-stage1     3.7 GiB        # its closure
$ nix-store -q --references /nix/store/…-lean-stage1 | wc -l
21                                          # direct references
$ nix path-info -r /nix/store/…-lean-stage1 | wc -l
2546                                        # paths in the closure
```

Lowercase `-s` is `--size`; uppercase `-S` is `--closure-size`. With `-r`,
`-S` prints every path in the closure with *its own* closure size, so the line
for the path you asked about just repeats the whole-closure figure. It is not
a total. `-h` gives human units.

## Reading a large closure

Most of those 2546 paths were single Lean modules (`Lean.Elab.Macro`,
`Lake.Config.LeanConfig`, …), each its own store path. That fits a
from-source build that took over an hour: thousands of derivations realised
one by one, not one big compile. The connection is inferred, not timed.
