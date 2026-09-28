# scrap

Notes on things that cost something to find out — general enough to be worth
keeping, and not tied to any one project.

One file per topic, so a note can be read back on its own.

## Notes

### clang

- [Dumping struct layout](clang/struct-layout.md) — printing every member's
  offset from the CLI, why `-fdump-record-layouts` leaves your own structs
  out, and why a one-byte field costs either 0 or 8 bytes.

### Emacs

- [Native compilation needs gcc, not just libgccjit](emacs/native-compilation-libgccjit.md)
  — why `native-comp-available-p` still says `t` while every `.eln` fails to
  link, and why `.eln.tmp` in `eln-cache/` is the only trace.
- [Stale `.elc` after an Emacs version change](emacs/elc-after-a-version-change.md)
  — why a `.elc` belongs to the Emacs that compiled it, why the `.eln` beside it
  disagrees, and why the symptom names a symbol you never wrote.
- [Why a Flymake backend is not running](emacs/flymake-backend-not-running.md) —
  why a backend that signalled once is never retried, what `Flymake:!` actually
  means, how eglot assigns over the backend list, and why a nixpkgs-built Emacs
  disables every backend as untrusted content when no other Emacs does.

### JavaScript

- [Biome install and version pin](javascript/biome.md) — why `@biomejs/biome`
  should be saved with an exact version (`-E`), and how `check` differs from
  `format` / `lint`.
- [npm and pnpm lockfiles](javascript/package-lockfiles.md) — why lockfiles
  describe installation models rather than only exact versions, and how npm's
  hoisted tree differs from pnpm's shared store and linked dependency graph.
- [Next.js learning bootstrap](javascript/next-learning-bootstrap.md) — Phase 0
  for a hand-written App Router learning repo: flake shell, no
  `create-next-app`, `strict`, and one lint+format tool, without freezing old
  version numbers.

### KaTeX

- [Equation numbering](katex/equation-numbering.md) — why the number is
  absent from KaTeX's own output, why there is no `\eqref`, and what `\tag`
  and macros do instead.

### Nix

- [Why a binary cache is not being used](nix/binary-cache-not-being-used.md) —
  why a substituter named in a flake's `nixConfig`, in user config or on the
  command line is dropped unless you are a trusted user, why
  `/etc/nix/nix.custom.conf` leaves no trace on upstream Nix, why
  `--dry-run` printing nothing is not success, and how to ask a cache for one
  path directly.
- [What decides whether Nix builds, fetches, or does nothing](nix/when-nix-rebuilds.md)
  — why the output path is known before the build, which flake inputs can
  change a hash, how to test "what if" without touching `flake.lock`, and why
  `nix eval` can start a build.
- [What makes direnv's `use flake` re-evaluate](nix/direnv-use-flake-watches.md)
  — why a file the flake reads can break without the shell noticing, why
  the `Found watch` list in `direnv status` makes `flake.nix` look unwatched,
  and why the dev shell's own output is never in the store.
- [Store path size versus closure size](nix/closure-size.md) — why
  `nix path-info -S` is the closure and not the package, and why a 6 MiB
  package can need 3.7 GiB.
- [Reading the source of a flake input](nix/flake-input-source.md) — finding
  the pinned revision in the store, and working back from an
  `attribute '…' missing` error to the line that failed.

### macOS

- [Bundle identifiers](macos/bundle-identifiers.md) — why a program run from
  `bin/` has no bundle identifier while the same program in an `.app` does, why
  two bundles can claim one identifier with version deciding nothing, and why
  editing the LaunchServices database does not hold.
- [Symbolic hotkeys](macos/symbolic-hotkeys.md) — how system-wide shortcuts
  are stored, why an ID is missing until it is changed, and why writing the
  preference is not enough to change a binding.

### Neovim

- [`vim.lsp.enable` and a silent denols](neovim/vim-lsp-enable-and-denols.md) —
  why `enable` does not start a server, and why an attached `denols` client
  can still return empty hover and diagnostics without a Deno project root.

### Slidev

- [Setup files](slidev/setup-files.md) — why a `setup/katex.ts` created
  while the dev server is running never takes effect, not even after a
  later edit.
