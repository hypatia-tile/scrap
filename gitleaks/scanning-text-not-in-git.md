# Scanning text that is not in git with gitleaks

gitleaks is usually run over a repository's history, but it can also scan
arbitrary text on standard input — useful for checking an issue body, a PR
description or a pasted log *before* it is published, since none of those
ever pass through a commit.

## Without installing it

```sh
nix run nixpkgs#gitleaks -- stdin --no-banner --redact -v < draft.md
```

Observed with gitleaks 8.30.1, with a planted AWS-style key:

```
$ printf 'hello\nAWS_KEY=AKIA…\n' | gitleaks stdin --no-banner --redact   # a fake, full-length key
INF scanned ~35 bytes (35 bytes) in 23.6ms
WRN leaks found: 1                                # exit 1
$ printf 'clean text\n' | gitleaks stdin --no-banner --redact
INF no leaks found                                # exit 0
```

## The default output does not say what it found

Without `-v` the run reports only a count. To see which rule matched and
where, add `-v` (`--verbose`); `--redact` keeps the secret itself out of that
output, which matters when the output is going to be pasted somewhere too.

Exit 1 means something was found, exit 0 that nothing was — so it works as a
gate in a script.

## What it will not catch

gitleaks matches known secret shapes. Things that are sensitive without
looking like a credential — internal hostnames, private URLs, absolute paths
naming a user, environment variable values — pass silently, and still need a
read by eye before publishing.
