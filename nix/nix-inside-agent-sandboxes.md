# Running Nix inside a coding agent's sandbox (macOS)

On a multi-user Nix install, every `nix` command — `nix develop`, `nix build`,
even a cached evaluation — talks to the Nix daemon over a unix socket,
`/nix/var/nix/daemon-socket/socket`. Coding agents on macOS run shell commands
under a seatbelt sandbox, and their default profiles refuse that connection:

```
error: cannot connect to socket at '/nix/var/nix/daemon-socket/socket': Operation not permitted
```

So anything that shells out to Nix — a Git hook that runs `nix develop -c …`,
a `direnv` `use flake`, a build script — fails inside the sandbox even though
it works in a terminal. What it takes to let the socket through differs per
agent, and so does whether the agent could have committed in the first place.

Measured 2026-10 with Codex 0.156.1, cursor-agent 2026.09.18 and Nix 2.31.4.

## Testing without a model session

Both agents ship a way to run one command under their sandbox directly, which
makes this a measurement instead of a conversation with a model:

```sh
# Codex: built-in profiles are :read-only, :workspace, :danger-full-access
codex sandbox -P :workspace -C <dir> --log-denials -- <command>

# cursor-agent: the helper it spawns for every sandboxed command
<cursor-agent install dir>/cursorsandbox --policy policy.json -- <command>
```

A bare profile name (`-P workspace-write`) is looked up in your own
`config.toml` and fails with "default_permissions requires a `[permissions]`
table"; the built-in ones start with a colon. `--log-denials` prints every
seatbelt denial afterwards. To check a Git hook without committing, run it as
`git hook run pre-commit`.

## Codex

| Profile | Nix socket | `.git` writable |
| --- | --- | --- |
| `:read-only` | refused | no |
| `:workspace` (the workspace-write default) | refused | **no** |
| `:workspace` + `--allow-unix-socket /nix/var/nix/daemon-socket` | allowed | no |
| `:danger-full-access` | allowed | yes |

`--allow-unix-socket` is a narrow, per-path allowance: the socket opens and
nothing else does. Note the last column: under `:workspace` the working tree
is writable but `.git` is not (`touch .git/x` → `Operation not permitted`),
so a sandboxed Codex cannot commit at all. A commit has to run escalated,
outside the sandbox, where Nix works anyway. That escalated commands run
unsandboxed is Codex's documented behaviour, inferred here from the
`:danger-full-access` row rather than measured in a live session.

## cursor-agent

The opposite on both counts: **`.git` is writable inside its sandbox**, so it
*can* commit there, and **there is no socket-specific allowance**.

`cursorsandbox` takes `{"sandbox": {"type": "workspace_readwrite", "cwd":
"<dir>", …}, "networkPolicy": …, "networkPolicyStrict": …}`; `cwd` is
required. The fields of the `workspace_readwrite` variant, read from the
binary's serde strings: `cwd`, `readBoundary`, `hardcodedReadPaths`,
`additionalReadwritePaths`, `additionalReadonlyPaths`, `networkAccess`,
`disableTmpWrite`, `blockGitWrites`, `ignoreMapping`. Unknown keys are
ignored without an error, so a typo looks exactly like a setting that does
not help.

| Added to `workspace_readwrite` | Nix socket | `curl https://example.com` |
| --- | --- | --- |
| nothing | refused | blocked |
| `additionalReadwritePaths: ["/nix/var/nix/daemon-socket"]` | refused | blocked |
| `networkAccess: true` | **allowed** | **200** |
| `networkAccess: true` + `networkPolicy: {default: "deny", allow: []}` | refused | blocked |

A writable path is a file permission, and connecting to a socket is a network
permission, so the first does not help. Domain filtering is done by a proxy on
localhost: once a `networkPolicy` filters anything, the seatbelt profile admits
only localhost, and the unix socket is refused with the rest. **The socket is
reachable only with the network boundary removed entirely.** In that mode Nix
also cannot write its evaluation cache under `~/.cache/nix` (an "ignored"
SQLite error), so evaluation is uncached there as well.

The agent itself reads its user policy from `~/.cursor/sandbox.json`,
validated with the same field names. How that file's settings map onto
`networkAccess` was not tried in a session.
