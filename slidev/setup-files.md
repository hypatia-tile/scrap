# A Slidev setup file created after the dev server started never takes effect

Slidev lets a deck extend its build through files under `setup/` —
`setup/katex.ts`, `setup/shiki.ts`, `setup/unocss.ts` and a few others. Each
default-exports a function whose return value is merged into the
corresponding options.

These are read once, when the dev server starts. Slidev knows that, and
watches them so the server restarts when one changes. The misleading part is
that the watch only covers files that existed at startup. Create
`setup/katex.ts` for the first time while `slidev` is running and nothing
happens — not on creation, and not on any later edit either. The file looks
correct, the same file works in `slidev build`, and the running server keeps
serving output as though it were absent.

## Why the watch misses it

The restart list does include the setup files (`@slidev/cli` 52.19.1,
`dist/cli.mjs`):

```js
const FILES_CHANGE_RESTART = [
  "setup/shiki.ts",
  "setup/katex.ts",
  "setup/preparser.ts",
  "setup/transformers.ts",
  "setup/unocss.ts",
  "setup/vite-plugins.ts",
  "uno.config.ts",
  "unocss.config.ts",
  "vite.config.{js,ts,mjs,mts}",
];
```

but the watcher is built once, at startup, from those names joined onto each
root as concrete paths:

```js
const watcher = watch(
  roots.filter(i => !i.includes("node_modules"))
       .flatMap(root => FILES_CHANGE_RESTART.map(i => path.join(root, i))),
  { ignored: ["node_modules", ".git"], ignoreInitial: true, ... },
);
```

Handlers for `add`, `change` and `unlink` all call `restartServer()`, so the
intent is clearly to catch creation too. In practice, when neither the file
nor its parent `setup/` directory exists at that moment, the watch is never
established and does not recover for the lifetime of that server.

## Measured

With `setup/` moved away, the server started, and then the directory put
back with `katex.ts` inside:

```console
$ grep -c restarting server.log
0
```

Editing the file afterwards, in that same server, also produced nothing:

```console
$ printf '\n// probe\n' >> setup/katex.ts
$ sleep 8; grep -c restarting server.log
0
```

Starting the server with `setup/katex.ts` already present gives the
documented behaviour — editing it restarts:

```text
file /.../my-slides/setup/katex.ts changed, restarting...
```

## What to do

After creating a file under `setup/` for the first time, restart the dev
server by hand. Slidev prints its shortcuts on startup and `r` restarts in
place:

```text
shortcuts > restart | open | edit | quit
```

From then on, edits to that file restart the server on their own.

A useful confirmation that the file is genuinely being read is to check
`slidev build` output instead of the dev server: the build resolves setup
files fresh every time, so it is never affected by this.
