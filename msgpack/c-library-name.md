# The msgpack-c library is `-lmsgpack-c`, not `-lmsgpackc`

msgpack-c is the C implementation of MessagePack. Code samples, blog posts
and answers online link it with `-lmsgpackc`. With a current msgpack-c that
fails. The header still compiles, and only the link step breaks:

```
$ clang ipc_msgpack.c -o ipc_msgpack -lmsgpackc
ld.bfd: cannot find -lmsgpackc: No such file or directory
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Because `#include <msgpack.h>` works, it looks as if the package is installed
but the linker cannot see it. In fact the linker sees it fine. There is just
no file named `libmsgpackc.*`.

## What the library is called now

In msgpack-c 7.0.2 (nixpkgs, 2026-11), the files in `lib/` are:

```
libmsgpack-c.a  libmsgpack-c.so  libmsgpack-c.so.2  libmsgpack-c.so.2.0.0
```

and pkg-config says so too:

```
$ pkg-config --list-all | grep -i msgpack
msgpack-c MessagePack - Binary-based efficient object serialization library
$ pkg-config --cflags --libs msgpack-c
-I/nix/store/…-msgpack-c-7.0.2/include -L/nix/store/…-msgpack-c-7.0.2/lib -lmsgpack-c
```

The rename is in upstream's `CHANGELOG.md`, under 6.0.0 (2023-03-02):

> Change CMake package name of C library to msgpack-c (#1053)
> Unified all C package, library, cmake, tarball name become msgpack-c.

5.0.0 had just changed the CMake package name to `msgpackc`, so a tutorial
from 2023 can already be wrong.

Only 7.0.2 was checked directly. That releases before 6.0 install
`libmsgpackc` comes from the changelog and the old samples. It was not
verified by installing one.

## Link through pkg-config

Ask pkg-config for the flags instead of writing the library name by hand:

```make
MSGPK := $(shell pkg-config --cflags --libs msgpack-c)
```

It also supplies the `-I` path. A compilation database (`bear -- make`)
generated from a build without it leaves clangd unable to find `msgpack.h`.

## Finding this kind of failure

`cannot find -lNAME` means no `libNAME.so` or `libNAME.a` in any search
path. It says nothing about whether the package is installed. To tell the
two apart, list the library directory and look at the real file name. In a
Nix shell the directories are the `-L` entries in `$NIX_LDFLAGS`:

```sh
echo $NIX_LDFLAGS | tr ' ' '\n' | sed -n 's/^-L//p'
```
