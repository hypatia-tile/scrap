# Which layer put a tool on PATH (nix-darwin)

On a Mac built with nix-darwin, a command can come from the system profile,
from Home Manager, from a user Nix profile, from Homebrew or from macOS
itself — and each has a different owner and a different way to change it.
`command -v` alone does not tell them apart, because most of them are
symlinks into `/nix/store`. Resolve the link instead, and look at where the
*first* path sits, not where it ends up.

## Resolve, then read the prefix

```
$ command -v nvim
/etc/profiles/per-user/<user>/bin/nvim
$ readlink -f "$(command -v nvim)"
/nix/store/…-neovim-unwrapped-…/bin/nvim
$ command -v zsh
/run/current-system/sw/bin/zsh
$ command -v git
/usr/bin/git
$ command -v brew
/opt/homebrew/bin/brew
```

Every Nix-provided tool ends in `/nix/store`, so the store path says nothing
about who declared it. The `PATH` entry it was found through does:

| Found under | Put there by |
|---|---|
| `/run/current-system/sw/bin` | the nix-darwin system configuration (`environment.systemPackages`) |
| `/etc/profiles/per-user/$USER/bin` | Home Manager running as a nix-darwin module (`home.packages`) |
| `~/.nix-profile/bin` | a user profile — `nix profile install`, or standalone Home Manager |
| `/opt/homebrew/bin` | Homebrew — declared only if nix-darwin's `homebrew` module manages it |
| `/usr/bin`, `/bin` | macOS itself |

The table was read off one nix-darwin host with Home Manager as a module
(observed 2026-10). The first two rows are what that setup produces; the
third is inferred from the profile's name, because on that host it was empty
(next section).

## `~/.nix-profile` can exist and hold nothing

With Home Manager as a nix-darwin module, packages go to
`/etc/profiles/per-user/$USER`, not to `~/.nix-profile`. On the host above,
`~/.nix-profile` was still a symlink, still on `PATH` — and dangling:

```
$ ls -l ~/.nix-profile
~/.nix-profile -> ~/.local/state/nix/profiles/profile
$ ls -a ~/.local/state/nix/profiles/
.  ..
```

So on such a host, anything that *does* resolve through `~/.nix-profile` was
put there by hand (`nix profile install`) and is in no declaration. A
`PATH` entry existing is not evidence that anything manages it.

## `darwin-version` does not say which flake built the system

`darwin-version --json` reports nix-darwin's and nixpkgs' revisions, not your
configuration's:

```
$ darwin-version --json
{ "darwinLabel": "26.11.4cff07d",
  "darwinRevision": "4cff07de…",
  "nixpkgsRevision": "34ca302a…" }
$ darwin-version --configuration-revision
darwin-version: configuration commit hash is unknown     # exit 1
```

The configuration revision is whatever `system.configurationRevision` was set
to, and nothing sets it by default — on that host it evaluated to `null`.
Setting it from the flake (`self.rev or self.dirtyRev or null`) is the
intended fix; that part was not tried. Without it, the practical check that a
host is built from a given flake is that its name (`scutil --get
LocalHostName`) is one of the flake's `darwinConfigurations`:

```sh
nix eval --no-update-lock-file <flake>#darwinConfigurations --apply builtins.attrNames
```

That shows the flake *can* build this host, not that the running system *was*
built from it.
