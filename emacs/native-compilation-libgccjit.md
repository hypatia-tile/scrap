# Emacs native compilation needs gcc, not just libgccjit

Emacs built with `--with-native-compilation` compiles Lisp to native code
through libgccjit. On Homebrew that library is the `libgccjit` formula, and
installing it is not enough. libgccjit generates the code, but writing each
`.eln` ends in a real link, and that link wants `libemutls_w.a` from libgcc —
which only the `gcc` formula ships.

Homebrew does not connect the two:

```
$ brew deps libgccjit
gmp isl libmpc lz4 mpfr xz zstd
```

No `gcc`. And the formula installs only the JIT libraries themselves:

```
$ brew list libgccjit | grep lib/gcc
.../libgccjit/16.2.0/lib/gcc/16/libgccjit.dylib
.../libgccjit/16.2.0/lib/gcc/16/libgccjit.0.dylib
.../libgccjit/16.2.0/lib/gcc/current/libgccjit.dylib
.../libgccjit/16.2.0/lib/gcc/current/libgccjit.0.dylib
```

`libemutls_w.a` lives under the `gcc` formula instead, at
`Cellar/gcc/16.2.0/lib/gcc/current/gcc/aarch64-apple-darwin27/16/`.

## The failure is silent

This is the part that costs something: with `libgccjit` present and `gcc`
absent, nothing says native compilation is broken.

- `(native-comp-available-p)` still returns **`t`**. libgccjit loads fine; it
  is the link that fails.
- Byte compilation is untouched, so every package still works — just from
  `.elc`. Nothing misbehaves, no feature disappears.
- Each failed compilation leaves an `.eln.tmp` in `eln-cache/` that is never
  renamed into place. That residue is the only lasting trace.

Measured on one machine four weeks after `gcc` went missing: 33 `.eln.tmp`
files in `eln-cache/<version>/`, and no `.eln` newer than the day `gcc` was
removed. Nobody noticed until a newly installed package needed compiling.

Forcing one compilation shows the real error:

```
$ emacs --batch --eval '(native-compile "some-file.el" "/tmp/out.eln")'
ld: library 'emutls_w' not found
libgccjit.so: error: : error invoking gcc driver
Internal native compiler error: "failed to compile", "error invoking gcc driver"
```

So when native compilation is suspect, look for `.eln.tmp` in `eln-cache/`.
`native-comp-available-p` answers a different question than the one being
asked.

## Confirming the missing library is the whole of it

Pointing `LIBRARY_PATH` at any directory that holds a matching
`libemutls_w.a` makes the same compilation succeed with nothing else changed.

This was observed with a libgcc from a *different* GCC major (15) against
libgccjit 16 — it linked. That was a diagnostic, not a configuration worth
keeping: install the `gcc` whose major matches the installed `libgccjit`.

## No Emacs rebuild is needed to fix it

Homebrew's emacs-plus bakes the gcc library directory into the binary:

```
$ otool -l .../emacs-plus@30/30.2/bin/emacs-30.2 | grep -A2 LC_RPATH
         path /opt/homebrew/lib/gcc/16
```

The major in that path is whichever `gcc` was installed when Emacs was built,
so reinstalling the same major satisfies it as-is. libgccjit is referenced
through `/opt/homebrew/opt/libgccjit/lib/gcc/current/libgccjit.0.dylib`, which
carries no version, so upgrading libgccjit alone does not call for a rebuild
either. (The `bin/emacs` in that formula is a shell wrapper that `exec`s
`Emacs.app/Contents/MacOS/Emacs`; `otool` has to be pointed at the real
binary, `bin/emacs-<version>`.)

## How gcc goes missing in the first place

`gcc` is a declared dependency of emacs-plus, with the reason written into the
formula:

```ruby
# `libgccjit` and `gcc` are required when Emacs compiles `*.elc` files asynchronously (JIT)
depends_on "libgccjit"
depends_on "gcc"
```

So it arrives with the first install and is not something one forgets. It goes
away later, through something that uninstalls formulae it does not find
declared — `brew autoremove`, or `brew bundle cleanup` against a Brewfile.

What was observed afterwards: `gcc` stayed missing across repeated runs of
`brew bundle` even though the Brewfile declared emacs-plus and emacs-plus
declares `gcc`. The likely reason — `brew bundle` sees the formula already
installed, skips it, and so never re-resolves its dependencies — was not
isolated; watching a run's output would settle it. Either way the repair is
manual:

```
$ brew install gcc
```

A dependency deleted out from under an already-installed formula is worth
checking for directly, rather than assuming a later `bundle` will notice:

```
$ comm -23 <(brew deps <formula> | sort) <(brew list --formula | sort)
```
