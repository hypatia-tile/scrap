# Bundle identifiers, and the two ways they surprise you

A macOS application is identified by the `CFBundleIdentifier` in its
`Info.plist` — `org.gnu.Emacs`, `net.kovidgoyal.kitty`. Anything that configures
behaviour *per application* keys on that string: an input method's per-app mode,
TCC permissions, `defaults` domains, `open -b`.

Two things about it are not obvious, and both bite quietly.

## A process started outside the bundle has no identifier at all

macOS gives a process the bundle identifier of the bundle it was **started
from** — not one looked up from the executable. Run the binary that lives inside
`Foo.app/Contents/MacOS/` and the process is `com.example.foo`; run a *different
copy* of the same program from a `bin/` directory and the process has no bundle
identifier, so nothing that keys on one can see it.

This matters wherever a program ships both a CLI entry point and an `.app`, and
it is invisible until something per-application fails. The two forms are easy to
mistake for each other:

| `bin/<prog>` is… | the process gets |
| --- | --- |
| a wrapper that `exec`s `<App>.app/Contents/MacOS/<prog>` | the app's bundle identifier |
| the program itself, sitting outside the bundle | no bundle identifier |

Emacs on macOS is a worked example. Homebrew's `emacs-plus` ships `bin/emacs` as
a shell wrapper that `exec`s the bundle's binary; nixpkgs' `emacs` ships
`bin/emacs` as the executable, a different file from the bundle's own (different
inode). Same program, same version, same `.app` beside it — but only the first
lets an input method register a per-application mode for a terminal-launched
Emacs. Observed with macSKK: 「Emacsで直接入力」simply never appears in the menu
until Emacs is started through the bundle, with `open -b org.gnu.Emacs`.

So when a per-app setting cannot see a running application, check how it was
launched before looking anywhere else.

## Several bundles can claim the same identifier, and version does not decide

Nothing stops two copies of an application from carrying the same
`CFBundleIdentifier`, and LaunchServices registers each **path** separately.
Which one the identifier resolves to is then a property of the database, not of
the applications: it is **not** the newest version, and not the one you last
used.

Ask, without launching anything:

```sh
osascript -e 'POSIX path of (path to application id "org.gnu.Emacs")'
```

Measured on one machine: an old 30.2.50 build won over an installed 31.1.

This accumulates wherever builds live at content-addressed paths. Under Nix each
build is its own store path, so every upgrade registers a new bundle while the
previous one stays registered for as long as any generation still roots it.

### Editing the database does not hold

Two obvious moves, neither of which works:

- `lsregister -gc` ("garbage collect old data") left the entry in place, even
  for an application already deleted from disk.
- `lsregister -u <path>` unregistered it, and the next rescan put it back with a
  fresh record id.

`lsregister -kill` is gone — *"removed because it was dangerous and no longer
useful"* — and `-delete` wants a reboot. As long as the bundle exists on disk the
registration comes back. **Removing the bundle is the only thing that holds.**
Under Nix that means letting go of whatever roots the store path:

```sh
nix-store --query --roots /nix/store/<path>     # what is holding it
nix store delete /nix/store/<path>             # once nothing does
```

Prefer `nix-collect-garbage --delete-older-than <n>d` over `-d` to drop the old
generations rooting it: `-d` takes every rollback target with it.

`lsregister` lives at
`/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister`.

## And a copy nobody manages outlives its source

A package manager removes what it installed. A copy made *from* that install is
not what it installed. Moving Emacs off Homebrew left `Emacs.app` and
`Emacs Client.app` in `/Applications`, put there by a script that had copied them
out of the Cellar; cleanup removed the Cellar and the copies stayed, claiming
`org.gnu.Emacs` while being unable to launch at all
(`Library not loaded: libtiff.6.dylib`, a dependency that left with the formula).

A broken bundle is still a registered bundle.
