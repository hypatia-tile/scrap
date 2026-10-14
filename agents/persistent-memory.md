# Persistent memory in coding agents

A coding agent can carry context from one session to the next in two layers.
**Instruction files** are written by you: `CLAUDE.md`, `AGENTS.md`, Cursor
Rules. **Memory** is written by the agent itself, from what it noticed while
working. The second layer is the one that misleads. It is usually stored
outside the repository, so it does not change when the repository does. It
goes stale silently, and it is still loaded into every session as if it were
true.

## Claude Code writes memory by default, outside the repository

From Claude Code's memory documentation (`code.claude.com/docs/en/memory`,
read on 2026-10-15):

- Auto memory is **on by default in local sessions**. Claude decides what to
  save. The only sign is a line like `Saved 2 memories`.
- It is **machine-local**, under `~/.claude/projects/<project>/memory/`, with a
  `MEMORY.md` index. Every worktree of one Git repository shares the same
  directory.
- Every conversation loads the first 200 lines or 25 KB of `MEMORY.md`,
  whichever limit it reaches first.
- Both `CLAUDE.md` and auto memory are "context, not enforced configuration".
  To block an action whatever Claude decides, the documentation points to a
  `PreToolUse` hook, not to memory.

Two settings control it:

```json
{ "autoMemoryEnabled": false }
```

turns it off for one project when it is in that project's settings. The other,
`autoMemoryDirectory`, moves the store elsewhere and "is read from any
settings scope". Pointing it at a tracked path would make memory changes show
up in `git diff`. That follows from the documentation, but has **not been
tested** here.

## What it looked like after a refactor

This is what was observed in one repository after about five weeks of use:
24 memory files. The repository then replaced its mechanism for placing
config files. Several memories now said the opposite of the truth:

- One said config files were read-only copies that do not follow branch
  checkouts. They had become symlinks, which do follow checkouts.
- Another told the agent never to propose a design. The repository had just
  adopted that design.

Nothing reported this. Memory is not in the repository, so the change never
touched it, no diff showed it, and the timestamps in the files' frontmatter
said nothing about whether an entry was still true. It was found only by
reading every file and checking it against the repository.

A second problem came out of the same read. Several working procedures, such
as how to verify a dependency-update pull request, existed **only** in that
memory. Other agents used in the same repository (Cursor, Codex) never see
another agent's memory. To them, that knowledge did not exist.

## Memory is an attack surface

OWASP's Top 10 for Agentic Applications lists *ASI06: Memory & Context
Poisoning*. Its entry lead described a case against Claude Code (Cisco's
"MemoryTrap",
[OWASP GenAI blog, 2026-05-13](https://genai.owasp.org/2026/05/13/memory-is-a-feature-it-is-also-an-attack-surface/)):

- The workflow was ordinary: clone a repository, let the agent help, approve a
  dependency install.
- The payload then reached persistent memory and the global hooks
  configuration, and kept influencing behaviour "across sessions, projects,
  and even reboots".
- After the disclosure, Claude Code v2.1.50 removed user memories from the
  system prompt. Memory itself is still loaded.

The general point is that memory is persistent, the agent trusts it
implicitly, and the agent can write to it without asking. Any path with those
three properties lets a prompt injection outlive the session it came from.

## Other agents choose differently

From OpenAI's documentation on Codex memories
(`developers.openai.com/codex/memories`, read on 2026-10-15):

- Local Codex memories are **off by default**.
- `memories.disable_on_external_context = true` keeps chats that used MCP
  tools or web search out of memory generation.
- The page says to keep required guidance in `AGENTS.md` or checked-in
  documentation, to treat memories as "a helpful recall layer", and to treat
  the files under `~/.codex/memories/` as generated state rather than editing
  them by hand.

The following were reported in a literature review but **not re-checked**
against their sources:

- Cursor removed its auto-generated Memories in 2.1.x (late 2025) and
  pointed users to Rules.
- GitHub Copilot Memory is on by default. It checks each memory's citations
  against the current code before use and deletes memories that have not
  been used for 28 days. GitHub reports that the pull-request merge rate rose
  from 83% to 90% with it; this is the vendor's own A/B test, not replicated.
- Gemini CLI's Auto Memory is experimental and off by default. It queues
  proposed changes for approval.
- Irrelevant or near-miss context lowers model accuracy (Chroma's "Context
  Rot", 2025). This is why a stale memory that resembles the current task is
  worse than none.

## A rule that fits

The evidence does not say that memory an agent writes is bad. Verified,
scoped and expiring memory measured well, in the Copilot result above. What
matters is not who writes an entry, but where it lives and whether anything
ever invalidates it:

- **Instructions are code.** Anything meant to steer the agent's behaviour
  lives in a reviewed, versioned file in the repository.
- **Learned facts are a cache.** Accept them only if they can be inspected,
  are scoped to one project, and either expire or are checked against the
  source before use. Claude Code's default auto memory offers none of this,
  except that the files can be read.
- Hard boundaries belong in a hook, not in either kind of memory.

Whatever stays enabled, read it now and then against the repository. The
files are plain Markdown, and reading them is the only check there is.
