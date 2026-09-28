# What makes direnv's `use flake` re-evaluate

`use flake` in an `.envrc` evaluates the flake's dev shell once and caches the
resulting environment. After that, direnv re-evaluates only when a **watched
file** changes. Anything else — including a file the flake itself reads — can
change, or break, without the environment noticing.

## What is watched

With nix-direnv (here through home-manager's integration), a shell where the
environment is loaded reports:

```
$ direnv status
Loaded watch: ".envrc"
Loaded watch: "…/direnv/allow/…"
Loaded watch: "…/direnv/deny/…"
Loaded watch: "../../../../.direnvrc"
Loaded watch: "../../../../.config/direnv/direnvrc"
Loaded watch: "flake.nix"
Loaded watch: "flake.lock"
Loaded watch: "devshell.toml"
Loaded watch: ".direnv/flake-profile-….rc"
Loaded watch: ".env"
```

`flake.nix` and `flake.lock` are there. A file the flake pulls in — say
`toolchain = ./lean-toolchain`, read with `builtins.readFile` at evaluation
time — is not.

## A broken input can look healthy

`lean-toolchain` was edited into a state the flake cannot evaluate. The shell
kept serving the old toolchain — `lake --version` unchanged — while forcing
evaluation failed at once:

```
$ nix eval .#devShells.aarch64-darwin.default.drvPath
error: attribute '"4.33.1"' missing
```

Nothing had re-read the file. The environment in use is a finished store path;
the edited file is only an input to the *next* evaluation, and no watch
triggered one.

## `direnv status` prints the watch list twice

Below the `Loaded watch:` lines comes a `Found watch:` section, and it is short:

```
Found watch: ".envrc"
Found watch: "…/direnv/allow/…"
Found watch: "…/direnv/deny/…"
```

Reading that section alone suggests `flake.nix` is not watched. That wrong
conclusion was once written down and relied on. The `Found` list appears to be
what direnv knows before `.envrc` runs, so the watches `use flake` adds are
missing from it — an inference from the output, not from direnv's source.

## The profile is not the dev shell's output

nix-direnv (3.2.0 here) calls `nix print-dev-env --profile …`, and the profile
it keeps in `.direnv/` does not point at the `pkgs.mkShell` output:

```
$ ls -l .direnv/ | rg flake-profile
flake-profile-… -> /nix/store/i8w3…-nix-shell-env
$ nix-store -q --deriver /nix/store/i8w3…-nix-shell-env
/nix/store/ix0d…-nix-shell-env.drv
```

That deriver is a sibling of the mkShell derivation, not the mkShell
derivation itself. It has the same builder and the same three input
derivations, with identical hashes. Only the arguments differ:

```
mkShell's nix-shell.drv      args: -e source-stdenv.sh default-builder.sh
nix-shell-env.drv            args: get-env.sh
```

`get-env.sh` runs the stdenv setup and writes out the resulting environment:

```
$ file /nix/store/i8w3…-nix-shell-env
JSON text data
$ jq -r '.variables.PATH.value' /nix/store/i8w3…-nix-shell-env | tr ':' '\n' | head -1
/nix/store/840y…-lean-stage1/bin
```

So the mkShell output (`…-nix-shell`) is never realised. Its path is written
into the `.drv` but is absent from the store, which is expected and not a
failure. mkShell's own `buildPhase` warns that "the existence of this path is
not guaranteed".

The mkShell `.drv` itself also disappeared from the store overnight, while the
env JSON and everything it references stayed. The likely reason is that the
profile is a GC root for the JSON's closure, and the mkShell `.drv` is not
the deriver of any live output. That is an inference; the collection itself
was not observed.

## What to do

- `direnv reload` re-runs `.envrc`, which re-evaluates the flake.
- To make another file count, `watch_file lean-toolchain` in `.envrc`
  (direnv stdlib). Not tried here.
- To see what the flake says right now, regardless of any cache:
  `nix eval .#devShells.<system>.default.drvPath`. On a flake that builds
  during evaluation this is not free — see
  [what decides whether Nix builds](when-nix-rebuilds.md).
