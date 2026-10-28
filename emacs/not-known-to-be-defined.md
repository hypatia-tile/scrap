# The native compiler warns that an autoloaded function is not defined

`Warning (native-compiler): … the function ‘NAME’ is not known to be defined`
is the byte compiler's unresolved-call check, printed while it native-compiles
one file. The function it names can exist, be autoloaded, and work at run
time. The check never looks at the package's autoload file unless the file
being compiled loads it.

A function counts as known when the file being compiled defines it, names it
in `declare-function`, or loads something that defines it before the check at
the end of the compilation. `;;;###autoload` only writes an autoload into the
generated autoloads file. `package.el` loads that file when it activates the
package, which is why the call succeeds in a running Emacs. Compiling the
`.el` on its own does not activate the package.

## Worked example: ddskk 20260329.1317

The warning, from a running Emacs:

```
Warning (native-compiler): ~/.emacs.d/elpa/ddskk-20260329.1317/skk-comp.el:311:33: Warning: the function ‘skk-num-compute-henkan-key’ is not known to be defined.
```

`skk-num-compute-henkan-key` is defined in `skk-num.el`, and that definition
carries `;;;###autoload`. `skk-comp.el` calls it (line 311 in the unpatched
file, and twice more below) and only does `(require 'skk)`. The same call,
with no `declare-function`, is in `skk.el`, `skk-cdb.el`, `skk-server.el`,
`skk-look.el`, `skk-server-completion.el`, and `skk-annotation.el`.

`skk.el` tries to feed the compiler the autoloads. Inside `eval-and-compile`:

```elisp
(require 'skk-autoloads nil 'noerror)
```

The comment above it says a `make` install ships `skk-autoloads.el`, which
provides the feature, and a MELPA install ships `ddskk-autoloads.el` with no
`provide`. On this MELPA tarball the second half of that comment is stale.
The only autoloads file present is `ddskk-autoloads.el`, and it ends with
`(provide 'ddskk-autoloads)`. There is no `skk-autoloads.el`. Measured in
`emacs --batch` with that directory on `load-path`:

```
require-skk-autoloads=nil
feature-skk-autoloads=nil
fboundp-after-failed-require=nil
require-ddskk-autoloads=ddskk-autoloads
feature-ddskk-autoloads=t
fboundp-after-ddskk-autoloads=t
```

`noerror` turns the missing file into nil. It does not load
`ddskk-autoloads.el`, because `require` looks up a feature's own file name.
The autoload that would make the function `fboundp` sits in the file the
package never requires.

Byte-compiling the unpatched `skk-comp.el` with only that directory on
`load-path` prints the warning at line 311, column 33. Two variations do not:

- a one-file probe whose first form is `(require 'ddskk-autoloads)` and which
  then calls `skk-num-compute-henkan-key`
- the same `skk-comp.el` after inserting
  `(declare-function skk-num-compute-henkan-key "skk-num" (key))`

So an autoload loaded at compile time is enough. The compiler does not need
the real `defun`. `declare-function` is the other way, and it does not load
`skk-num`. `skk-comp-do` let-binds `skk-num-list` under a comment that says
not having required `skk-num` can clobber a buffer-local value; this note
does not claim that a top-level `(require 'skk-num)` was tried.

`master` of [skk-dev/ddskk](https://github.com/skk-dev/ddskk), fetched
2026-10-29, still has those calls in `skk-comp.el` and no `declare-function`
for this function. A newer MELPA tarball was not installed, so this does not
say a later snapshot is unchanged — only that upstream source, that day, had
not added the declaration.

A `make` install's `skk-autoloads.el` was not built here. The comment says
that file would provide `skk-autoloads`; whether that install compiles
without the warning was not checked.
