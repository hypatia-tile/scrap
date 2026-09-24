# Install Biome as an exact-version toolchain

Biome is a single CLI for format, lint, and related assists. The supported
project install is a development dependency, then an optional config file:

```sh
pnpm add -D -E @biomejs/biome
pnpm exec biome init
```

`-E` is `--save-exact`. Other package managers have the same idea on Biome's
getting-started page. `biome init` writes `biome.json`; Biome can also run
with no config, but a generated file is the usual place to read defaults
before changing them.

The misleading part is treating Biome like an ordinary library dependency
with a caret range. Biome's own versioning guide says to save the **exact**
version in `package.json`, because fixes to lint rules or formatting can
make existing scripts fail. Even a patch release can change formatting or
lint results enough that CI and teammates disagree if they resolve different
versions.

Source: <https://biomejs.dev/internals/versioning/>

Install docs (including `-E`): <https://biomejs.dev/guides/getting-started/>

## `check` is the combined command

- `biome format` — formatting only
- `biome lint` — lint only
- `biome check` — format, lint, and import organization together

`--write` applies safe fixes. A common `package.json` pairing is a read-only
script (`biome check .`) and a write script (`biome check --write .`).

## Ranges still appear in the wild

Using `^` for `@biomejs/biome` is possible and will install; it does not
give the reproducibility Biome documents above. Whether a given repo pinned
with `-E` or left a range is a project choice — the durable fact is why the
install guide insists on an exact version.
