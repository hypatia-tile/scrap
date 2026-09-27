# Why a binary cache is not being used

A binary cache is added to Nix as a `substituters` entry plus a
`trusted-public-keys` entry. Both are **restricted settings**: on a multi-user
install, where builds go through the daemon, Nix accepts them only from a
trusted user. Everyone else's are dropped.

That is the whole difficulty, and almost every way of declaring a cache looks
like it worked when it did not.

## Which world you are in

```
$ nix store info --json | grep trusted
"trusted": 0
```

`0` means restricted settings you specify will be ignored. `trusted-users`
defaults to `root` alone, so this is the normal state for a daemon install
unless somebody changed it.

```
$ nix config show | grep -E '^(substituters|trusted-users) '
substituters = https://cache.nixos.org/
trusted-users = root
```

If a cache you declared is not in that first line, it is not going to be used,
whatever else happened.

## The three ways that silently do nothing

**A flake's `nixConfig`.** It prompts, which reads as consent being taken:

```
do you want to allow configuration setting 'extra-substituters' to be set to 'https://…' (y/N)? y
do you want to permanently mark this value as trusted (y/N)? y
warning: ignoring untrusted substituter 'https://…', you are not a trusted user.
```

Answering yes records the value in `~/.local/share/nix/trusted-settings.json`
and changes nothing else. Declining instead is *noisier*, not quieter — it
produces a `Using saved setting …` line and an `ignoring untrusted flake
configuration setting` warning per value. The only way to stop the warning is
to remove the `nixConfig` block, or to become a trusted user.

**The user's own `~/.config/nix/nix.conf`, and `--option` on the command
line.** Same restriction, same outcome.

**`/etc/nix/nix.custom.conf` — on upstream Nix.** This one leaves no trace at
all. It is a **Determinate Nix** feature; upstream Nix never reads the file and
does not warn that a config file went unread. Having `determinate-nixd`
installed does not mean the running Nix is Determinate's:

```
$ nix --version
nix (Nix) 2.31.4                                    # upstream
$ strings $(readlink -f $(which nix)) | grep nix.custom.conf
                                                    # no match: it is not read
```

## What works

Declare it at the system level, which is trusted by definition. On upstream Nix
that means `/etc/nix/nix.conf` and nothing else. Keeping the values in their own
file and including it leaves the installer-owned file almost untouched, and puts
them where Determinate would want them if it ever arrives:

```sh
sudo tee /etc/nix/nix.custom.conf >/dev/null <<'CONF'
extra-substituters = https://example.cachix.org
extra-trusted-public-keys = example.cachix.org-1:…
CONF
sudo tee -a /etc/nix/nix.conf >/dev/null <<'CONF'
!include /etc/nix/nix.custom.conf
CONF
```

`!include` does not fail when the file is missing.

One thing not to guess about while writing those lines: **an `extra-` setting
accumulates, it does not replace.** Two `extra-substituters` lines in one file
add up, exactly as putting both URLs on a single line does — so neither form
silently drops the other's caches. Measured:

```
$ NIX_CONFIG=$'extra-experimental-features = ca-derivations\nextra-experimental-features = fetch-closure' nix config show | grep ^experimental
experimental-features = ca-derivations fetch-closure fetch-tree flakes nix-command
```

The single-line form is a readability choice, nothing more. (A plain
`substituters = …`, without the prefix, *does* replace — that is what the prefix
is for.)

Then restart the daemon — **it**, not the client, performs substitution, and it
reads its configuration at startup:

```sh
sudo launchctl kickstart -k system/org.nixos.nix-daemon    # macOS
```

Adding yourself to `trusted-users` instead also works, and grants much more: any
flake's `nixConfig` could then name a substituter. Declaring the one cache is
the narrower change for the same benefit.

`/etc/nix/nix.conf` is installer-owned, so an upgrade can overwrite it and drop
the `include` line. When a cache stops being used for no apparent reason, check
that first.

## Proving it, and the trap in proving it

```
$ nix build --dry-run .#something
this path will be fetched (105.70 MiB download, 373.48 MiB unpacked):
  /nix/store/…
```

"will be fetched" is the proof. "will be built" means it is still not working.

The trap: if the path is **already in the local store**, `--dry-run` prints
nothing at all, and that looks like success. Delete it first, or the test proves
nothing:

```sh
nix store delete /nix/store/…        # refuses if something still refers to it
```

## A cache only hits when the derivation hash matches

Obvious in the abstract, easy to design around wrongly. If CI fills a cache by
building a flake's own output, and the consumer pulls that flake in with
`inputs.nixpkgs.follows = "nixpkgs"`, the consumer builds the package against
*its* nixpkgs while CI built against the flake's lock. Different inputs,
different hash, cache miss every time, with nothing to indicate why.

A flake whose builds are meant to be cached has to be consumed with its own pin
— which is also what stops the package moving every time the consumer's pin
does. It is how the widely-used overlay flakes are consumed.
