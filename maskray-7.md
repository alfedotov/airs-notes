# Everything I know about GNU toolchain

As mainly an LLVM person, I occasionally contribute to GNU toolchain projects.
This is sometimes for fun, sometimes for investigating why an (usually ancient)
feature works in a particular way, sometimes for pushing forward a toolchain
feature with the mind of both communities, or sometimes just for getting sense
of how things work with mailing list+GNU make.

For a debug build, I normally place my build directory `Debug` directly under
the project root.

## binutils

* Repository: https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git
* Mailing list: https://sourceware.org/pipermail/binutils
* Bugzilla: https://sourceware.org/bugzilla/
* Main tools: as (`gas/`, GNU assembler), ld (`ld/`, GNU ld), gold (`gold/`,
  GNU gold)

As of 2021-01, it has no wiki.

Target `all` builds targets `all-host` and `all-target`. When running
configure, by default most top-level directories binutils `gas gdb gdbserver ld
libctf` are all enabled. You can disable some components via `--disable-*`.
`--enable-gold` is needed to enable gold.

```sh
mkdir Debug; cd Debug
../configure --target=x86_64-linux-gnu --prefix=/tmp/opt --disable-gdb --disable-gdbserver
```

For cross compiling, make sure your have `$target-{gcc,as,ld}`.

For many tools (binutils, gdb, ld), `--enable-targets=all` will build every
supported architectures and binary formats. However, one gas build can only
support one architecture. ld has a default emulation and needs `-m` to support
other architectures (`aarch64 architecture of input file 'a.o' is incompatible
with i386:x86-64 output`). Many tests are generic and can be run on many
targets, but a `--enable-targets=all` build only tests its default target.

```sh
# binutils (binutils/*)
make -C Debug all-binutils
# gas (gas/as-new)
make -C Debug all-gas
# ld (ld/ld-new)
make -C Debug all-ld

# Build all enabled tools.
make -C Debug all
```

Build with Clang:

```sh
mkdir -p out/clang-debug; cd out/clang-debug
../../configure CC=~/Stable/bin/clang CXX=~/Stable/bin/clang++ CFLAGS='-O0 -g' CXXFLAGS='-O0 -g'
```

About security aspect, "don't run any of binutils as root" is sufficient advice
(Alan Modra).

## Test

GNU Test Framework DejaGnu is based on Expect, which is in turn based on Tcl.

To run tests:

```sh
make -C Debug check-binutils
# Find the result in (summary) Debug/binutils/binutils.sum and (details) Debug/binutils/binutils.log

make -C Debug check-gas
# Find the result in (summary) Debug/gas/testsuite/gas.sum and (details) Debug/gas/testsuite/gas.log

make -C Debug check-ld

# Test all enabled tools.
make -C Debug check-all
```

For ld, tests are listed in `.exp` files under `ld/testsuite`. A single test
normally consists of a `.d` file and several associated `.s` files.

To run the tests in `ld/testsuite/ld-shared/shared.exp`:

```sh
make -C Debug check-ld RUNTESTFLAGS=ld-shared/shared.exp
```

### Misc

* A bot updates bfd/version.h (`BFD_VERSION_DATE`) daily.
* Test coverage is low.

## gdb

gdb resides in the binutils-gdb repository. `configure` enables gdb and
gdbserver by default. You just need to make sure `--disable-gdb
--disable-gdbserver` is not on the configure line.

Run gdb under the build directory:

```sh
gdb/gdb -data-directory gdb/data-directory
```

To run the tests in `gdb/testsuite/gdb.dwarf2/dw2-abs-hi-pc.exp`:

```sh
make check-gdb RUNTESTFLAGS=gdb.dwarf2/dw2-abs-hi-pc.exp

# cd $build/gdb/testsuite/outputs/gdb.dwarf2/dw2-abs-hi-pc
```

## glibc

* Repository: https://sourceware.org/git/gitweb.cgi?p=glibc.git
* Wiki: https://sourceware.org/glibc/wiki/
* Bugzilla: https://sourceware.org/bugzilla/
* Mailing lists: `{libc-announce,libc-alpha,libc-locale,libc-stable,libc-help}@sourceware.org`

(Mostly) an implementation of the user-space side of standard C/POSIX functions
with Linux extensions.

A very unfortunate fact: glibc can only be built with `-O2`, not `-O0` or
`-O1`. If you want to have an un-optimized debug build, deleting an object file
and recompiling it with `-g` usually works. Another workaround is `#pragma GCC
optimize ("O0")`.

The `-O2` issue is probably related to (1) expected inlining and (2) avoiding
dynamic relocations.

Run the following commands to populate `/tmp/glibc-many` with toolchains.
Caution: please make sure the target file system has tens of gigabytes.

Preparation:

```sh
scripts/build-many-glibcs.py /tmp/glibc-many checkout --shallow
scripts/build-many-glibcs.py /tmp/glibc-many host-libraries

scripts/build-many-glibcs.py /tmp/glibc-many compilers aarch64-linux-gnu
scripts/build-many-glibcs.py /tmp/glibc-many compilers powerpc64le-linux-gnu
scripts/build-many-glibcs.py /tmp/glibc-many compilers sparc64-linux-gnu
```

* `--shallow` passes `--depth 1` to the git clone command.
* `--keep` all keeps intermediary build directories intact. You may want this
  option to investigate build issues.

The `glibcs` command will delete the glibc build directory, build glibc, and
run `make check`.

```sh
scripts/build-many-glibcs.py /tmp/glibc-many glibcs aarch64-linux-gnu
# Find the logs and test results under /tmp/glibc-many/logs/glibcs/aarch64-linux-gnu/

scripts/build-many-glibcs.py /tmp/glibc-many glibcs powerpc64le-linux-gnu

scripts/build-many-glibcs.py /tmp/glibc-many glibcs sparc64-linux-gnu
```

"On build-many-glibcs.py and most stage1 compiler bootstrap, gcc is build
statically against newlib. the static linked gcc (with a lot of disabled
features) is then used to build glibc and then the stage2 gcc (which will then
have all the features that rely on libc enabled) so the stage1 gcc *might* not
have the require started files"

During development, some interesting targets:

```sh
make -C Debug check-abi
```

Building with Clang is not an option.

* Clang does not support GCC nested functions [BZ #27220](https://sourceware.org/bugzilla/show_bug.cgi?id=27220)
* x86 `PRESERVE_BND_REGS_PREFIX`: integrated assembler does not support the
  `bnd` prefix.
* `sysdeps/powerpc/powerpc64/Makefile`: Clang does not support
  `-ffixed-vrsave -ffixed-vscr`

## GCC

* Mailing lists: `gcc-{patches,regression}@sourceware.org`

`--disable-bootstrap` is the most important, otherwise you will get a stage 2
build. It is not clear what make does when you touch a source file. It
definitely rebuilds stage1, but it is not clear to me how well stage2
dependency is handled. Anyway, touching a source file causes a total build is
not what you desire.

```sh
../configure --disable-bootstrap --enable-languages=c,c++ --disable-multilib
make -j 30

# Incremental build
make -C gcc cc1 cc1plus xgcc
make -C x86_64-pc-linux-gnu/libstdc++-v3
```

Use built libstdc++ and libgcc.

```sh
$build/gcc/xg++ -B $build/release/gcc forced1.C -Wl,-rpath,$build/x86_64-pc-linux-gnu/libstdc++-v3/src/.libs,-rpath,$build/x86_64-pc-linux-gnu/libgcc
```

### Misc

* A bot updates `ChangeLog` files daily. `Daily bump.`

## Unlisted

autotools, bison, m4, make, ...

### Contributing

[GNU Coding Standards](https://www.gnu.org/prep/standards/). Emacs has good
built-in support. clang-format's support is not as good.

Legally significant changes need [Copyright Papers](https://www.gnu.org/prep/maintain/html_node/Copyright-Papers.html).

