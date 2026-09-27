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

## What to do

- `direnv reload` re-runs `.envrc`, which re-evaluates the flake.
- To make another file count, `watch_file lean-toolchain` in `.envrc`
  (direnv stdlib). Not tried here.
- To see what the flake says right now, regardless of any cache:
  `nix eval .#devShells.<system>.default.drvPath`. On a flake that builds
  during evaluation this is not free — see
  [what decides whether Nix builds](when-nix-rebuilds.md).
