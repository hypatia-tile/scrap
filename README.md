# scrap

Notes on things that cost something to find out — general enough to be worth
keeping, and not tied to any one project.

One file per topic, so a note can be read back on its own.

## Notes

### clang

- [Dumping struct layout](clang/struct-layout.md) — printing every member's
  offset from the CLI, why `-fdump-record-layouts` leaves your own structs
  out, and why a one-byte field costs either 0 or 8 bytes.

### macOS

- [Symbolic hotkeys](macos/symbolic-hotkeys.md) — how system-wide shortcuts
  are stored, why an ID is missing until it is changed, and why writing the
  preference is not enough to change a binding.
