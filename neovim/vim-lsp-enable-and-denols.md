# Neovim `vim.lsp.enable` and a silent denols

On Neovim 0.11+, `vim.lsp.enable("name")` does **not** start a language
server. It registers the server from `vim.lsp.config` / `lsp/<name>.lua` so
that Neovim may attach it later, when a buffer's filetype matches and
`root_dir` (or `root_markers`) succeeds.

That split is easy to miss: enable once, and the server still only appears on
the buffers that deserve it.

## An attached client can still do nothing

The part that costs something: `vim.lsp.get_clients()` returning a client is
not evidence that hover or diagnostics work.

For Deno's built-in LSP (`deno lsp`, usually configured as `denols`), forcing
a start with the file's directory as `root_dir` when no Deno project marker
exists produces exactly that state — client attached, hover empty,
diagnostics empty.

Observed in a headless Neovim session (Deno 2.9.x, Neovim 0.13 nightly):

| Project layout | Client | Diagnostics | Hover on a binding |
| --- | --- | --- | --- |
| `deno.json` next to the `.ts` file | `denols` attached | type error reported | markdown hover returned |
| same file, no `deno.json` / `deno.jsonc` / `deno.lock`, started with the file's directory as root | `denols` still attached | none | empty result |

nvim-lspconfig's `lsp/denols.lua` already refuses to pick a root unless a
Deno marker wins over a non-Deno layout. Falling back to the file's directory
bypasses that gate and is what creates the silent client.

On macOS the broken case also showed a workspace URI under `file:///var/...`
while the buffer URI was `file:///private/var/...` (same directory via the
`/var` → `/private/var` symlink). Whether Deno rejects the document for that
mismatch alone was not isolated; the reliable fix is not to start without a
Deno project root.

## What to do instead

- Prefer `vim.lsp.enable("denols")` (plus settings in `after/lsp/denols.lua` or
  equivalent) over a `FileType` autocmd that calls `vim.lsp.start` with a
  directory fallback.
- Treat `deno.json`, `deno.jsonc`, or `deno.lock` as required for denols to
  be useful, not optional.
- When debugging "LSP does nothing", request hover or read diagnostics — do
  not stop at "a client is attached".
