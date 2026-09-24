# scrap

Notes on things that cost something to find out — general enough to be worth
keeping, and not tied to any one project.

One file per topic, so a note can be read back on its own.

## Notes

### clang

- [Dumping struct layout](clang/struct-layout.md) — printing every member's
  offset from the CLI, why `-fdump-record-layouts` leaves your own structs
  out, and why a one-byte field costs either 0 or 8 bytes.

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

### macOS

- [Symbolic hotkeys](macos/symbolic-hotkeys.md) — how system-wide shortcuts
  are stored, why an ID is missing until it is changed, and why writing the
  preference is not enough to change a binding.

### Slidev

- [Setup files](slidev/setup-files.md) — why a `setup/katex.ts` created
  while the dev server is running never takes effect, not even after a
  later edit.
