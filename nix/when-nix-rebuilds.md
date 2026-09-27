# What decides whether Nix builds, fetches, or does nothing

A derivation is a recipe. Its output path — `/nix/store/<hash>-<name>` — is
computed from the recipe during **evaluation**, before anything is built.
Whether any work happens is then decided by that path alone. Nix does not
compare timestamps or file contents.

## The output path exists before the build

```
$ nix eval .#devShells.aarch64-darwin.default.drvPath
"/nix/store/hrnv…-nix-shell.drv"
```

Opening that `.drv` shows its inputs already named by output path, although
nothing has been built:

```
("nativeBuildInputs","/nix/store/840yiivrqfnj8k14f1d65w88gk4c3ss6-lean-stage1")
```

`nix derivation show <drv>` prints the same as JSON, under `outputs.out.path`.
The hash covers everything in the recipe — builder, arguments, environment,
and every input derivation, recursively. Equal hash means equal recipe. It
does not promise identical output bytes; it is what lets Nix skip work.

## Realisation: store, then cache, then build

Once the path is known, Nix looks in the local store. If
`ls -d /nix/store/840y…-lean-stage1` finds it, there is nothing to do. If not,
it asks each substituter for `<hash>.narinfo`. Only when every lookup misses
does it build. That order is Nix's documented behaviour; what was measured
here is the lookup itself:

```
bash-5.3p3 from nixpkgs                  → 200  fetched
lean-stage1 from a third-party overlay   → 404  built locally
```

cache.nixos.org holds what nixpkgs' own Hydra built. A package produced by
another flake's overlay is not there unless that project runs a cache of its
own. How to ask a cache yourself is in
[why a binary cache is not being used](binary-cache-not-being-used.md).

## Which inputs can change a hash

An input changes a hash only if something it provides ends up inside a
derivation.

**One that only arranges outputs does not.** flake-parts turns `perSystem`
into `devShells.<system>` and builds nothing. Swapping it for a revision nine
months older left the dev shell's derivation untouched:

```
flake-parts as locked (2026-08)  → /nix/store/w4j875jbdy3703b5q1i3mbhxry2ypmn8-nix-shell.drv
flake-parts at 2025-12           → /nix/store/w4j875jbdy3703b5q1i3mbhxry2ypmn8-nix-shell.drv
```

**nixpkgs does,** even under an unchanged package source: it supplies the
compiler and every C dependency. The consequence for caches is in
[why a binary cache is not being used](binary-cache-not-being-used.md). Not
measured here; overriding `nixpkgs` as below and comparing `drvPath` would
settle it.

## Asking "what if" without touching the lock

```sh
nix eval --raw --no-write-lock-file \
  --override-input flake-parts github:hercules-ci/flake-parts/<rev> \
  .#devShells.x86_64-linux.default.drvPath
```

- **A bare `github:owner/repo` resolves to the latest revision, which may be
  the one already locked.** The comparison then proves nothing, and prints two
  equal paths that look like a result. Pin a revision, or check what it
  resolved to:
  `nix flake metadata --no-write-lock-file --override-input … --json | jq -r '.locks.nodes["flake-parts"].locked.rev'`.
- **Another system's attributes can be evaluated without building them.** The
  x86_64-linux `drvPath` above evaluated on aarch64-darwin in about a second.

## Evaluation can build

Evaluation and realisation are separate phases in principle, but a flake that
imports from a derivation (IFD) needs build outputs before it can finish
evaluating. Evaluating a dev shell on darwin, where the overlay builds Lean
from source, printed this before returning any path:

```
building '/nix/store/…-lean-src.drv'...
building '/nix/store/…-lean-stage0.drv'...
```

An earlier evaluation of the same attribute that day returned instantly. Why
one built and the other did not was not determined. Treat `nix eval` on such
a flake as potentially expensive. For cheap comparisons, evaluate a system
whose path needs no IFD — here, the Linux one, which uses a prebuilt archive.
