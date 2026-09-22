# macOS symbolic hotkeys

System-wide keyboard shortcuts — the ones System Settings lists under
*Keyboard → Keyboard Shortcuts* — live in a single user-domain preference:

```sh
defaults read com.apple.symbolichotkeys AppleSymbolicHotKeys
```

The value is a dictionary keyed by a numeric **hotkey ID**, each holding
`enabled` and a `value` describing the chord. Taking a chord away from macOS
so an application can have it means turning the relevant ID off.

## An ID is absent until it is changed

This is the part that misleads.

macOS does not write an ID into the plist while it sits at its built-in
default. A freshly configured machine shows only the IDs that have ever been
touched — on the machine these notes were written on, 29 of them — and the
ones everybody uses every day, Spotlight among them, are **not there**.

So the plist is not an inventory of what is bound. Absence means "still at
the built-in default", and for most IDs that default is *enabled*. Reading a
missing ID as "not bound" is the mistake waiting here.

## Reading the chord

`value.parameters` is a three-element array:

```
[ ASCII character, key code, modifier bitmask ]
```

`65535` in the first slot means the key produces no ASCII character (a
function key, for instance). The modifier bitmask is a sum of:

| Modifier | Hex | Decimal |
| --- | --- | --- |
| Shift | `0x020000` | 131072 |
| Control | `0x040000` | 262144 |
| Option | `0x080000` | 524288 |
| Command | `0x100000` | 1048576 |

So Cmd+Opt+Space, whose key code is 49 and whose character is a space, is:

```
[ 32, 49, 1572864 ]      # 1572864 = 1048576 + 524288
```

Every value in that table was confirmed against real entries rather than
looked up: Ctrl+Space reads 262144, Ctrl+Opt+Space reads 786432, and
Shift+F8 reads 131072.

## IDs worth knowing

| ID | Shortcut | Default chord |
| --- | --- | --- |
| 60 | Select previous input source | Ctrl+Space |
| 61 | Select next input source | Ctrl+Opt+Space |
| 64 | Spotlight search | Cmd+Space |
| 65 | Finder search window | Cmd+Opt+Space |
| 79–82 | Move left / right a space | Ctrl+← / Ctrl+→ |
| 32 / 33 / 36 | Mission Control / App Windows / Show Desktop | F9 / F8 / F7 |

64 and 65 are the pair that matters when installing a launcher: a launcher
offered Cmd+Space collides with 64, and Cmd+Opt+Space collides with 65.
Leaving the OS side enabled means both fire on the same chord.

## Writing is not enough

A write to this preference does not take effect on its own. `WindowServer`
holds the copy it read at login, so the new value sits in the plist while the
old binding keeps working — which reads as "the change did nothing".

Either log out and back in, or poke the settings daemon:

```sh
/System/Library/PrivateFrameworks/SystemAdministration.framework/Resources/activateSettings -u
```

It is not on `PATH`; the full path is the only way to reach it.

This is what separates two different failures. If `defaults read` shows the
value you wrote but the key still behaves the old way, the write landed and
only the reload is missing. If `defaults read` does not show it, the write
itself never happened.

## The dictionary is replaced, not merged

Anything that writes this preference as a whole — `defaults write
com.apple.symbolichotkeys AppleSymbolicHotKeys '<plist>'`, which is the form
nix-darwin's `system.defaults` uses — **replaces the entire dictionary**.

Two consequences:

- Every ID the written dictionary omits reverts to its built-in default. A
  declarative config must therefore carry the complete set it cares about,
  and an ID left out is a decision, not an omission.
- A shortcut changed by hand in System Settings is erased the next time that
  write runs. Under configuration management, the hand-edit is not a
  shortcut to the declaration; it is a change with a countdown on it.

## Checking it worked

Three checks, in the order that isolates a failure:

1. `defaults read com.apple.symbolichotkeys AppleSymbolicHotKeys` — the value
   is what you intended, and the IDs you did *not* touch are still absent or
   unchanged.
2. `activateSettings -u` (full path above), or log out and back in.
3. Press the chord. Nothing before this is evidence about behaviour: the
   first two confirm the write and the reload, not the binding.
