# Reading the source of a flake input

When an error points into a flake you depend on, read the revision your
`flake.lock` pins, not the repository's default branch — they can differ.
Every input is already in the store:

```
$ nix flake archive --dry-run --json | jq -r '.inputs["lean4-nix"].path'
/nix/store/j5l3…-source
```

Then read and grep it like any directory.

## Worked example

```
error: attribute '"4.33.1"' missing
```

The message names no package, tool or version file. It says one thing: an
attrset was indexed with a key it does not have. Work backwards from that:

1. Find the lookup: `rg -n 'getAttr' <path>/lib/overlay.nix` →
   `builtins.getAttr tag tags`, where `tag` is extracted from the toolchain
   file by a regex.
2. Find where the attrset comes from: `tags` is `manifests/default.nix`, whose
   keys are literal strings like `"v4.33.1"`.
3. Compare its keys with yours: the toolchain file said `4.33.1`, without the
   `v`.
