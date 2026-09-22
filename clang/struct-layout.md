# Dumping struct layout with clang

clang can print the byte offset of every member of every struct it sees,
along with the struct's size and alignment, without compiling or running
anything:

```sh
clang -std=c11 -fsyntax-only -Xclang -fdump-record-layouts-complete foo.c
```

```
*** Dumping AST Record Layout
         0 | struct ObjMid
         0 |   ObjType type
         4 |   _Bool isMarked
         8 |   struct ObjMid * next
           | [sizeof=16, align=8]
```

The offsets on the left are the whole point: a gap between one member's
offset and the next is padding, stated rather than guessed.

## The flag without `-complete` will not show your struct

This is the part that misleads. There are two spellings, and the obvious one
is the wrong one.

Compiling a file that defines `struct ObjMid` and `struct ObjEnd`, uses both
through `sizeof` and `offsetof`, and includes `<stdio.h>`:

```sh
$ clang -std=c11 -fsyntax-only -Xclang -fdump-record-layouts layout.c \
    2>/dev/null | grep -E '^ +0 \| struct'
         0 | struct __sbuf
         0 | struct __sFILE

$ clang -std=c11 -fsyntax-only -Xclang -fdump-record-layouts-complete layout.c \
    2>/dev/null | grep -E '^ +0 \| struct'
         ... (eleven pthread and NSConstantString records) ...
         0 | struct __sbuf
         0 | struct __sFILE
         0 | struct ObjMid
         0 | struct ObjEnd
```

The plain flag dumped only two records, both from `stdio.h`, and neither of
the structs the file was written to inspect. `sizeof` and `offsetof` on them
was not enough to make them appear. Whatever the plain flag's criterion is,
it is not "used in this translation unit" — that was not investigated
further; `-complete` simply dumps every record that reaches a complete type,
so use `-complete` and filter.

The failure is silent: you get a dump, it is full of plausible-looking
records, and yours is not in it.

## `-Xclang` is required

`-fdump-record-layouts*` is a cc1 option, not a driver option, so it has to
be passed through:

```
$ clang -fsyntax-only -fdump-record-layouts-complete layout.c
clang: error: unknown argument '-fdump-record-layouts-complete';
       did you mean '-Xclang -fdump-record-layouts-complete'?
```

The error names the fix, so this one announces itself.

## Streams

The dump goes to **stdout**; driver diagnostics go to stderr. Confirmed by
redirecting each separately. That matters because in an environment that
injects linker flags — a nix devshell, for instance — `-fsyntax-only` makes
every one of them unused, and stderr fills with

```
clang: warning: argument unused during compilation: '-L/nix/store/...'
       [-Wunused-command-line-argument]
```

`2>/dev/null` clears it without touching the dump.

## Filtering a real project

Pass the include path and one source file, then grep with an anchor:

```sh
clang -std=c11 -I./include -fsyntax-only \
  -Xclang -fdump-record-layouts-complete src/object.c 2>/dev/null \
  | grep -A6 'struct Obj$'
```

```
         0 | struct Obj
         0 |   ObjType type
         4 |   _Bool isMarked
         8 |   struct Obj * next
           | [sizeof=16, align=8]
```

The `$` is load-bearing: without it the pattern also matches `struct
ObjString`, `struct ObjClosure` and every other name that starts the same
way. `-A<n>` is needed because each record is a block, not a line.

## What the offsets show: a byte costs 0 or 8

Two rules are directly readable off the dumps, and together they mean the
position of a new field decides its price.

1. A member sits at the next offset that is a multiple of its own alignment.
2. The struct's total size is rounded up to a multiple of its alignment.

Same three fields, two orders:

```
         0 | struct ObjMid                 0 | struct ObjEnd
         0 |   ObjType type                0 |   ObjType type
         4 |   _Bool isMarked              8 |   struct ObjEnd * next
         8 |   struct ObjMid * next       16 |   _Bool isMarked
           | [sizeof=16, align=8]           | [sizeof=24, align=8]
```

`struct { ObjType; void *; }` already wastes four bytes: the enum is four
bytes wide, the pointer needs offset 8. Putting the `bool` in that hole costs
**nothing** — 16 bytes before and after. Putting it after the pointer costs
**eight**: one byte at offset 16, then seven bytes of tail padding to round
17 up to 24.

So "I added one byte to the header" is not a statement about size. For a
struct that every heap object embeds, that is the difference between a free
flag and 8 bytes per object.

Measured on clang 21.1.8, arm64-apple-darwin, with `sizeof(ObjType) == 4`,
`sizeof(_Bool) == 1`, `sizeof(void *) == 8`. Enum width is implementation
defined rather than fixed at 4 — that part is reasoned from the standard, not
measured here, and `-fshort-enums` would change it. The dump is the check:
re-run it rather than carrying these numbers to another target.

## Compiler-independent fallback

`-Xclang -fdump-record-layouts*` is clang-specific. Where that is not
available, or to compare two variants of a struct side by side, print the
numbers instead:

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdio.h>

typedef enum { OBJ_CLOSURE, OBJ_FUNCTION } ObjType;
struct ObjMid { ObjType type; bool isMarked; struct ObjMid *next; };
struct ObjEnd { ObjType type; struct ObjEnd *next; bool isMarked; };

int main(void) {
  printf("ObjMid: size=%zu type@%zu isMarked@%zu next@%zu\n",
         sizeof(struct ObjMid), offsetof(struct ObjMid, type),
         offsetof(struct ObjMid, isMarked), offsetof(struct ObjMid, next));
  printf("ObjEnd: size=%zu type@%zu next@%zu isMarked@%zu\n",
         sizeof(struct ObjEnd), offsetof(struct ObjEnd, type),
         offsetof(struct ObjEnd, next), offsetof(struct ObjEnd, isMarked));
}
```

`%zu` for `size_t`. Include the project's real header and define only the
variant next to it, so the comparison runs against the actual struct without
editing it.
