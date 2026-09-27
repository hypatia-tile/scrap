# Stale `.elc` after an Emacs version change

A `.elc` is not a neutral cache. Byte compilation happens *after* macro
expansion, so a `.elc` carries the expansions of whichever Emacs compiled it.
Change the Emacs and every installed package's `.elc` was written against a
language that no longer quite exists.

Most of the time nothing happens, which is why this is easy to be wrong about.
It bites when a macro in Emacs's own libraries changes what it generates.

## What makes it confusing: the `.eln` is fresh

With native compilation on, the two artefacts beside each package have different
vintages after an upgrade:

| file | built by | expansion it carries |
| --- | --- | --- |
| `elpa/<pkg>/<pkg>.elc` | the **old** Emacs, at install time | old |
| `eln-cache/<version>-<hash>/<pkg>-*.eln` | the **new** Emacs, from `.el` source | new |

JIT native compilation reads the `.el`, so the `.eln` is correct for the running
Emacs while the `.elc` is not. Which definition answers then depends on load
order, and the disagreement surfaces somewhere unrelated to either file.

Worked example, 30.2 → 31.1. `define-globalized-minor-mode` renamed the variable
it generates:

```elisp
;; Emacs 30.x
(MODE-set-explicitly (intern (concat mode-name "-set-explicitly")))
;; Emacs 31.1
(MODE-set-explicitly (intern (concat mode-name "--set-explicitly")))
```

So `envrc.elc` defined `envrc-mode-set-explicitly` while the freshly built
`envrc.eln` used `envrc-mode--set-explicitly`, and the symptom was:

```
Error running timer: (void-variable envrc-mode--set-explicitly)
```

A symbol that appears in no source file anyone wrote, from a timer, naming
neither the package nor the upgrade.

## Reading the vintage

The compiler's version is in the header, in plain text:

```sh
$ head -c 120 ~/.emacs.d/elpa/envrc-*/envrc.elc | tr -d '\000' | sed -n '3p'
;;; in Emacs version 30.2
```

Across the whole tree, to see whether any stragglers are left:

```sh
head -c 120 ~/.emacs.d/elpa/*/*.elc | grep -o "in Emacs version [0-9.]*" | sort -u
```

## The fix

```sh
emacs --batch -l ~/.emacs.d/init.el --eval '(package-recompile-all)'
```

Then **restart Emacs** — a process that already loaded a stale `.elc` goes on
using it.

`package-recompile-all` is Emacs 29+. Deleting the `.elc` files by hand is not
equivalent: `package.el` does not rebuild them on its own, and a package loads
from source instead, slowly and silently.

Warnings during the recompile are not failures, and third-party packages produce
plenty — a missing `lexical-binding` cookie, an obsolete function call. They were
there before; the recompile is just the first time anything printed them.

## When to think of it

Any time Emacs's version changes, including when that arrives through a
dependency bump rather than a decision — a pinned flake or lockfile moving Emacs
underneath an `elpa/` tree that nothing recompiles.
