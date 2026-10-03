# Lefthook passes when its config is missing

Lefthook is a Git hook dispatcher: a hook script calls `lefthook run <hook>`,
and Lefthook runs whatever `lefthook.yml` assigns to that hook. When there is
**no** config file, `lefthook run` does not fail — it reports that it found
none and exits 0:

```
$ git hook run pre-commit          # hook: exec … lefthook run pre-commit "$@"
│  No config files with names ["lefthook" ".lefthook" ".config/lefthook"] have been found in "<repo>"
$ echo $?
0
```

Observed with Lefthook 2.1.14. So a deleted, renamed or not-yet-checked-out
`lefthook.yml` silently turns every gate into a pass, and the commit goes
through. If the hooks are meant to fail closed, the hook script has to check
for the config itself before handing over:

```sh
#!/bin/sh
[ -f lefthook.yml ] || { echo "pre-commit: lefthook.yml is missing" >&2; exit 1; }
exec lefthook run pre-commit "$@"
```

## `lefthook run` may also touch hooks

One run printed `Skipping hook sync`: `run` checks whether the installed hooks
are in sync with the config and can rewrite them. In the runs observed it
never changed a hook script, and the condition that triggers a sync was not
established. If your hook scripts are hand-written and tracked (with
`core.hooksPath`), find that condition in the documentation of the Lefthook
you pin before relying on them staying as written.
