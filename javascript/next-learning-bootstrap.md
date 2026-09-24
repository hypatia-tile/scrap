# Bootstrapping a Next.js App Router learning project by hand

A learning project that uses Next.js does not need a working scaffold on day
one. What transfers across such projects is a small Phase 0: a reproducible
shell, a hand-written App Router skeleton, TypeScript `strict`, and one tool
for lint plus format. Versions are chosen when the project starts, not copied
from an older repo.

The misleading default is `create-next-app` (or any generator that leaves a
running app before you have touched `package.json`, `tsconfig`, or `app/`).
That path optimizes for a green terminal, not for knowing which files are
load-bearing. For a learning bootstrap, add those files yourself in order.

## Fix Node and pnpm in a flake; keep `.envrc` out of git

Put `nodejs` and `pnpm` in a Nix `devShell` and lock inputs with `flake.lock`.
Entering the shell (via `nix develop`, or direnv with `use flake`) is what
"the environment is ready" means before any `pnpm` command.

direnv's `.envrc` is a personal machine concern. Prefer not to commit it.
A practical pattern is to ignore `.env*` in `.gitignore`, write `use flake`
locally, and run `direnv allow`. Anyone cloning the repo either recreates
`.envrc` or calls `nix develop` each time.

This was checked in a learning repo whose `.gitignore` contains `.env*` and
whose tracked tree includes `flake.nix` but not `.envrc`; `git check-ignore`
reported `.envrc` matched by that `.env*` rule.

## Do not copy dependency versions from an old project

Pinning tools inside one repo (lockfiles, `flake.lock`, `devEngines`) is
right for that repo. Reusing the same Next / React / TypeScript / Biome
version numbers months later as a "recommended stack" is a different act:
it freezes a snapshot that may already be awkward with current peers.

At the start of a new learning repo, read the current Next.js App Router
docs and the chosen linter's install docs, then pick versions yourself.
What transfers is the checklist below, not a version table.

## Hand-written App Router skeleton with `strict`

Skip `create-next-app`. A minimal path that has been used successfully:

1. `pnpm init`
2. Add `next`, `react`, `react-dom`, and TypeScript-related dev dependencies
   at versions you just confirmed
3. Add scripts such as `dev` → `next dev`, `build` → `next build`,
   `start` → `next start`
4. Write `tsconfig.json` with `"strict": true` and the JSX / module settings
   Next expects for App Router (confirm against current Next TypeScript docs)
5. Write `app/layout.tsx` and `app/page.tsx` so the root route renders
6. Run `pnpm dev` and open the app

`next.config.*` is not always required for that first `dev` run. One learning
repo reached a working App Router tree with tracked `app/layout.tsx`,
`app/page.tsx`, `tsconfig.json`, and `package.json`, and no committed
`next.config` file. Add config when you need a setting, not to complete the
scaffold.

Treat `"strict": true` as part of the skeleton, not as an optional later
tighten. Explicit type annotations in application code are a per-project
learning policy; they do not belong in this general note.

## Unify lint and format; confirm the tool when you start

ESLint and Prettier parse the same source with separate pipelines. Overlap
in formatting opinions is why projects often add a mediation config that
turns off ESLint rules Prettier will own. A single tool that shares one
concrete syntax tree for lint and format removes that mediation step and
usually means one config file and one check command.

Biome is a workable default for that shape: install with an exact version
(`pnpm add -D -E @biomejs/biome`), run `biome init`, and wire scripts to
`biome check` / `biome check --write`. Why the exact pin matters is recorded
in `javascript/biome.md`. Domains such as Next-aware rules may activate from
`package.json` when `next` is present; treat that as something to read in the
generated config, not as magic.

Do not treat "always Biome" as eternal. The durable rule is **one tool for
lint and format**. At project start, verify that the candidate still installs
cleanly with the TypeScript and Next versions you chose. A previous attempt
to keep ESLint + Prettier beside TypeScript 7 needed npm alias workarounds;
that friction is why Biome was adopted in that project, not a proof that
ESLint is always impossible.

## Done when

Phase 0 for this kind of learning repo is done when:

1. The flake shell provides `node` and `pnpm`
2. `pnpm dev` serves a hand-written App Router page
3. `tsconfig` has `strict: true`
4. Lint and format run through one tool and are wired in `package.json`
   scripts

Learning process (issue cadence, review habits, agent roles) and later
topics (Docker, database, CSS framework) stay in each repository's own
roadmap. They are not part of this bootstrap.
