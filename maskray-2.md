# All about symbol versioning

In 1995, Solaris' link editor and ld.so introduced the symbol versioning
mechanism. Ulrich Drepper and Eric Youngdale borrowed Solaris symbol versioning
in 1997 and designed the GNU style symbol versioning for glibc.

When a shared object is updated, the behavior of a symbol changes (ABI changes
(such as changing the type of parameters or return values) or behavior
changes), traditionally a `DT_SONAME` bump is required. Otherwise a dependent
application/shared object built with the old version may run abnormally. This
can be inconvenient if the number of dependent applications is large.

Symbol versioning provides backward compatibility without changing `DT_SONAME`.

The following part describes the representation, and then describes the
behaviors from the perspectives of assembler, linker, and ld.so. One may wish
to skip the representation part when reading for the first time.

## Representation

In a shared object or executable file that uses symbol versioning, there are up
to three sections related to symbol versioning. `.gnu.version_r` and
`.gnu.version_d` among them are optional:

* `.gnu.version` (version symbol section). The `DT_VERSYM` tag in the dynamic
  table points to the section. Assuming there are N entries in `.dynsym`,
  `.gnu.version` contains N `uint16_t` values, with the i-th entry indicating
  the version ID of the i-th symbol. Put it another way, `.gnu.version` is a
  parallel table to `.dynsym`.
* `.gnu.version_r` (version requirement section). The `DT_VERNEED`/
  `DT_VERNEEDNUM` tags in the dynamic table delimiter this section. This
  section describes the version information used by the undefined versioned
  symbol in the module.
* `.gnu.version_d` (version definition section). The `DT_VERDEF`/`DT_VERDEFNUM`
  tags in the dynamic table delimiter this section. This section describes the
  version information used by the defined versioned symbols in the module.

```c
// Version definitions
typedef struct {
  Elf64_Half    vd_version;  // version: 1
  Elf64_Half    vd_flags;    // VER_FLG_BASE (index 1) or 0 (index != 1)
  Elf64_Half    vd_ndx;      // version index
  Elf64_Half    vd_cnt;      // number of associated aux entries, always 1 in practice
  Elf64_Word    vd_hash;     // SysV hash of the version name
  Elf64_Word    vd_aux;      // offset in bytes to the verdaux array
  Elf64_Word    vd_next;     // offset in bytes to the next verdef entry
} Elf64_Verdef;

typedef struct {
  Elf64_Word    vda_name;    // version name
  Elf64_Word    vda_next;    // offset in bytes to the next verdaux entry
} Elf64_Verdaux;

// Version needs
typedef struct {
  Elf64_Half    vn_version;  // version: 1
  Elf64_Half    vn_cnt;      // number of associated aux entries
  Elf64_Word    vn_file;     // .dynstr offset of the depended filename
  Elf64_Word    vn_aux;      // offset in bytes to vernaux array
  Elf64_Word    vn_next;     // offset in bytes to next verneed entry
} Elf64_Verneed;

typedef struct {
  Elf64_Word    vna_hash;    // SysV hash of vna_name
  Elf64_Half    vna_flags;   // usually 0; copied from vd_flags of the depended so
  Elf64_Half    vna_other;   // unused
  Elf64_Word    vna_name;    // .dynstr offset of the version name
  Elf64_Word    vna_next;    // offset in bytes to next vernaux entry
} Elf64_Vernaux;
```

Currently GNU ld does not set the `VER_FLG_WEAK` flag. [BZ24718#c15](https://sourceware.org/bugzilla/show_bug.cgi?id=24718#c15) proposed "set
`VER_FLG_WEAK` on version reference if all symbols are weak".

The advantage of using a parallel table for `.gnu.version` is that symbol
versioning is optional. ld.so implementations which do not support symbol
versioning can freely assume no symbol has a version. The behavior is that all
references as if bind to the default version definitions. musl ld.so falls into
this category.

### Version index values

Index 0 is called `VER_NDX_LOCAL`. The binding of the symbol will be changed to
`STB_LOCAL`. Index 1 is called `VER_NDX_GLOBAL`. It has no special effect and
is used for unversioned symbols. Index 2 to 0xffef are used for user defined
versions.

Defined versioned symbols have two forms:

* foo@@v2, the default version.
* foo@v2, a non-default version (hidden version). The `VERSYM_HIDDEN` bit of the
  version ID is set.

Undefined versioned symbols have only the `foo@v2` form.

Usually versioned symbols are only defined in shared objects, but executables
can have defined versioned symbols as well. (When a shared object is updated,
the old symbols are retained so that other shared objects do not need to be
relinked, and executable files usually do not provide versioned symbols for
other shared objects to reference.)

### Example

`readelf -V` can dump the symbol versioning tables.

In the `.gnu.version_d` output below:

* Version index 1 (`VER_NDX_GLOBAL`) is the filename (soname if shared object).
  The `VER_FLG_BASE` flag is set.
* Version index 2 is a user defined version. Its name is `LUA_5.3`.

In the `.gnu.version_r` output below, each of version indexes 3~10 represents a
version in a depended shared object. The name `GLIBC_2.2.5` appears thrice,
each for a different shared object.

The `.gnu.version` table assigns a version index to each `.dynsym` entry.

```
% readelf -V /usr/bin/lua5.3

Version symbols section '.gnu.version' contains 248 entries:
 Addr: 0x0000000000002af4  Offset: 0x002af4  Link: 5 (.dynsym)
  000:   0 (*local*)       3 (GLIBC_2.3)     4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)
  004:   5 (GLIBC_2.3.4)   4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)   4 (GLIBC_2.2.5)
  ...

Version definition section '.gnu.version_d' contains 2 entries:
 Addr: 0x0000000000002ce8  Offset: 0x002ce8  Link: 6 (.dynstr)
  000000: Rev: 1  Flags: BASE  Index: 1  Cnt: 1  Name: lua5.3
  0x001c: Rev: 1  Flags: none  Index: 2  Cnt: 1  Name: LUA_5.3

Version needs section '.gnu.version_r' contains 3 entries:
 Addr: 0x0000000000002d20  Offset: 0x002d20  Link: 6 (.dynstr)
  000000: Version: 1  File: libdl.so.2  Cnt: 1
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 9
  0x0020: Version: 1  File: libm.so.6  Cnt: 1
  0x0030:   Name: GLIBC_2.2.5  Flags: none  Version: 6
  0x0040: Version: 1  File: libc.so.6  Cnt: 6
  0x0050:   Name: GLIBC_2.11  Flags: none  Version: 10
  0x0060:   Name: GLIBC_2.14  Flags: none  Version: 8
  0x0070:   Name: GLIBC_2.4  Flags: none  Version: 7
  0x0080:   Name: GLIBC_2.3.4  Flags: none  Version: 5
  0x0090:   Name: GLIBC_2.2.5  Flags: none  Version: 4
  0x00a0:   Name: GLIBC_2.3  Flags: none  Version: 3
```

### Symbol versioning in object files

The GNU scheme allows `.symver` directives to label the versions of the symbols
in objec files. The symbol names residing in .o contain `@` or `@@`.

## Assembler behavior

GNU as and LLVM integrated assembler provide implementation.

* `.symver foo, foo@v1`
  * If foo is undefined, produce `foo@v1`
  * If foo is defined, produce `foo` and `foo@v1` with the same binding
    (`STB_LOCAL`, `STB_WEAK`, or `STB_GLOBAL`) and `st_other` value (i.e. the
    same visibility). Personally I think this behavior is a design flaw
    [{gas-copy}](). The proposed [V4 PATCH gas: Extend .symver directive](https://sourceware.org/pipermail/binutils/2020-April/110622.html)
    can address this problem.
* `.symver foo, foo@@v1`
  * If foo is undefined, error
  * If foo is defined, produce `foo` and `foo@v1` with the same binding and `st_other` value.
* `.symver foo, foo@@@v1`
  * If foo is undefined, produce `foo@v1`
  * If foo is defined, produce `foo@@v1`

Personal recommendation:

* To define a default version symbol: use `.symver foo, foo@@@v2` so that foo
  is not present.
* To define a non-default version symbol, add a suffix to the original symbol
  name (`.symver foo_v1, foo@v1`) to prevent conflicts with `foo`. This will
  however leave (usually undesirable) `foo_v1`. If you don't strip `foo_v1` from
  the object file, you may localize it with a local: pattern in the version
  script. With GNU as 2.35 ([PR25295](https://sourceware.org/bugzilla/show_bug.cgi?id=25295)),
  you can use `.symver foo_v1, foo@v1, remove`
* The version of an undefined symbol is usually bound at link time. It is
  usually unnecessary to set the version with `.symver`. If required, prefer
  `.symver foo, foo@@@v1` to `.symver foo, foo@v1`.

## Linker behavior

The linker enters the symbol resolution stage after reading in object files,
archive files, shared objects, LTO files, linker scripts, etc.

GNU ld uses indirect symbol to represent versioned symbols. There are
complicated rules, and these rules are not documented. The symbol resolution
rules that I personally derived:

* Defined `foo` resolves undefined `foo` (traditional unversioned rule)
* Defined `foo@v1` resolves undefined `foo@v1` (a non-default version symbol is
  like a separate symbol)
* Defined `foo@@v1` (default version) resolves both undefined `foo` and `foo@v1`

If there are multiple default version definitions (such as `foo@@v1 foo@@v2`),
a duplicate definition error should be issued even if one is weak. Usually a
symbol has zero or one default version (`@@`) definition, and an arbitrary
number of non-default version (`@`) definitions.

If the linker sees undefined `foo` and `foo@v1` first, it will treat them as
two symbols. When the linker see the definition `foo@@v1`, conceptually `foo`
and `foo@@v1` should be combined. If the linker sees `foo@@v2` instead,
`foo@@v2` should resolve `foo` and `foo@v1` should be a separate symbol.

* [Combining Versions](combining-versions.md) describes the problem.
* `gold/symtab.cc Symbol_table::define_default_version` uses a heuristic rule
  to solve this problem. It special cases on visibility, but I feel that this
  rule is unneeded.
* Before 2.26, GNU ld reported a bogus multiple definition error for defined
  weak `foo@@v1` and defined global `foo@v1` [PR ld/26978](https://sourceware.org/bugzilla/show_bug.cgi?id=26978)
* Before 2.26, GNU ld had a bug that the visibility of undefined `foo@v1` does
  not affect the output visibility of `foo@@v1`: [PR ld/26979](https://sourceware.org/bugzilla/show_bug.cgi?id=26979)
* I fixed the object file side problem of LLD 12.0 in https://reviews.llvm.org/D92259
  `foo` Archive files and lazy object files may still have incompatibility issues.

When LLD sees a defined `foo@@v`, it adds both `foo` and `foo@v1` into the
symbol table, thus `foo@@v1` can resolve both undefined `foo` and `foo@v1`.
After processing all input files, a pass iterates symbols and redirects
`foo@v1` to `foo@@v1`.  Becase LLD treats them as separate symbols during input
processing, a defined `foo@v` cannot suppress the extraction of an archive
member defining `foo@@v1`, leading to a behavior incompatible with GNU ld. This
probably does not matter, though.

GNU ld has another strange behavior: if both `foo` and `foo@v1` are defined, `foo`
will be removed. I strongly believe it is an issue in GNU ld but the maintainer
rejected [PR ld/27210](https://sourceware.org/bugzilla/show_bug.cgi?id=27210).

## Version script

To define a versioned symbol in a shared object or an executable, a version
script must be specified. If all versioned symbols are undefined, then the
version script can be omitted.

```
# Make all symbols other than foo and bar local.
{ global: foo; bar; local: *; };

# Assign version FBSD_1.0 to malloc and version FBSD_1.3 to mallocx,
# and make internal local.
FBSD_1.0 { malloc; local: internal; };
FBSD_1.3 { mallocx; };
```

A version script has three purposes:

* Define versions.
* Specify some patterns so that matched defined symbols (which do not have `@`
  in the name) are tied to the specified version.
* Scope reduction: for a defined unversioned symbol matched by a `local:`
  pattern, its binding will be changed to `STB_LOCAL` and will not be exported
  to the dynamic symbol table.

A version script can consist of one anonymous version tag (`{...};`) or a list of
named version tags (`v1 {...};`). If you use an anonymous version tag with other
version tags, GNU ld will error: `anonymous version tag cannot be combined with
other version tags`. A `local:` part can be placed in any version tag. Which
version tag is used does not matter.

If a defined symbol is matched by multiple version tags, the following
precedence rules apply (`binutils-gdb/bfd/linker.c:find_version_for_sym`):

* The first version tag with an exact pattern (i.e. there is no wildcard) wins.
* Otherwise, the last version tag with a non-`*` wildcard pattern wins.
* Otherwise, the first version tag with a `*` pattern wins.

The gotcha is that `**` is a wildcard pattern which matches any symbol but its
precedence is higher than `*`.

Most patterns are exact so gold and LLD iterate patterns instead of symbols to
improve performance.

## How a versioned symbol is produced

An undefined symbol can be assigned a version if:

* its name does not contain `@` (`.symver` is unused) and a shared object
  provides a default version definition.
* its name contains `@` and a shared object defines the symbol. GNU ld errors
  if there is no such a shared object. After https://reviews.llvm.org/D92260,
  LLD will report an error as well.

A defined symbol can be assigned a version if:

* its name does not contain `@` and it is matched by a pattern in a named version tag in a version script.
* its name contains `@`
  * If `-shared`, the version should be defined by a version script, otherwise
    GNU ld errors version node not found for symbol. This exception looks
    strange to me so I have filed [PR ld/26980](https://sourceware.org/bugzilla/show_bug.cgi?id=26980).
  * If `-no-pie` or `-pie`, a version definition is unneeded in GNU ld. This
    behavior is strange.

## ld.so behavior

/Linux Standard Base Core Specification, Generic Part/ describes the behavior
of ld.so. Kan added symbol versioning support to FreeBSD rtld in 2005.

The `DT_VERNEED` and `DT_VERNEEDNUM` tags in the dynamic table delimiter the
version requirement by a shared object/executable file: the requires versions
and required shared object names (`Vernaux::vna_name`).

For each Vernaux entry (a Verneed's auxilliary entry) without the
`VER_FLG_WEAK` bit, ld.so checks whether the referenced shared object has the
`DT_VERDEF` table.  If no, ld.so handles the case as a graceful degradation; if
yes and the table does not define the version, ld.so reports an error.
[verneed-check]

Usually a minor release does not bump soname. Suppose that libB.so depends on
the libA 1.3 (soname is libA.so.1) and calls an function which does not exist
in libA 1.2. If PLT lazy binding is used, libB.so may seem to work on a system
with libA 1.2, until the PLT of the 1.3 symbol is called. If symbol versioning
is not used and you want to solve this problem, you have to record the minor
version number (`libA.so.1.3`) in the soname. However, bumping soname is
all-or-nothing: all the dependent shared objects need to be relinked. If symbol
versioning is used, you can continue to use the soname `libA.so.1`. ld.so will
report an error if libA 1.2 is used, because the 1.3 version required by
libB.so does not exist.

In the symbol resolution stage:

* An undefined foo can be resolved to a definition of `foo` or `foo@@v2` (only
  the definitions with index number 1 (`VER_NDX_GLOBAL`) and 2 are used in the
  reference match).
* An undefined `foo@v1` can be resolved to a definition of `foo`, `foo@v1`, or
  `foo@@v1`.

Note (undefined `foo` resolving to `foo@v1`) is allowed by ld.so but not
allowed by the linker [{reject-non-default}](). This difference provides a
mechanism to refuse linking against old symbols while keeping compatibility
with unversioned old libraries. If a new version of a shared object needs to
deprecate an unversioned `bar`, you can remove bar and define `bar@compat`
instead. Libraries using `bar` are unaffected but new links against `bar` are
disallowed.

## Upgraded symbols in glibc

Note that GNU nm before binutils 2.35 does not display `@` or `@@`.

```
nm -D /lib/x86_64-linux-gnu/libc.so.6 | \
  awk '$2!="U" {i=index($3,"@"); if(i){v=substr($3,i); $3=substr($3,1,i-1); m[$3]=m[$3]" "v}} \
  END {for(f in m)if(m[f]~/@.+@/)print f, m[f]}'
```

The output on my x86-64 system:

```
pthread_cond_broadcast  @GLIBC_2.2.5 @@GLIBC_2.3.2
clock_nanosleep  @@GLIBC_2.17 @GLIBC_2.2.5
_sys_siglist  @@GLIBC_2.3.3 @GLIBC_2.2.5
sys_errlist  @@GLIBC_2.12 @GLIBC_2.2.5 @GLIBC_2.3 @GLIBC_2.4
quick_exit  @GLIBC_2.10 @@GLIBC_2.24
memcpy  @@GLIBC_2.14 @GLIBC_2.2.5
regexec  @GLIBC_2.2.5 @@GLIBC_2.3.4
pthread_cond_destroy  @GLIBC_2.2.5 @@GLIBC_2.3.2
nftw  @GLIBC_2.2.5 @@GLIBC_2.3.3
pthread_cond_timedwait  @@GLIBC_2.3.2 @GLIBC_2.2.5
clock_getres  @GLIBC_2.2.5 @@GLIBC_2.17
pthread_cond_signal  @@GLIBC_2.3.2 @GLIBC_2.2.5
fmemopen  @GLIBC_2.2.5 @@GLIBC_2.22
pthread_cond_init  @GLIBC_2.2.5 @@GLIBC_2.3.2
clock_gettime  @GLIBC_2.2.5 @@GLIBC_2.17
sched_setaffinity  @GLIBC_2.3.3 @@GLIBC_2.3.4
glob  @@GLIBC_2.27 @GLIBC_2.2.5
sys_nerr  @GLIBC_2.2.5 @GLIBC_2.4 @@GLIBC_2.12 @GLIBC_2.3
_sys_errlist  @GLIBC_2.3 @GLIBC_2.4 @@GLIBC_2.12 @GLIBC_2.2.5
sys_siglist  @GLIBC_2.2.5 @@GLIBC_2.3.3
clock_getcpuclockid  @GLIBC_2.2.5 @@GLIBC_2.17
realpath  @GLIBC_2.2.5 @@GLIBC_2.3
sys_sigabbrev  @GLIBC_2.2.5 @@GLIBC_2.3.3
posix_spawnp  @@GLIBC_2.15 @GLIBC_2.2.5
posix_spawn  @@GLIBC_2.15 @GLIBC_2.2.5
_sys_nerr  @@GLIBC_2.12 @GLIBC_2.4 @GLIBC_2.3 @GLIBC_2.2.5
nftw64  @GLIBC_2.2.5 @@GLIBC_2.3.3
pthread_cond_wait  @GLIBC_2.2.5 @@GLIBC_2.3.2
sched_getaffinity  @GLIBC_2.3.3 @@GLIBC_2.3.4
clock_settime  @GLIBC_2.2.5 @@GLIBC_2.17
glob64  @@GLIBC_2.27 @GLIBC_2.2.5
```

* `realpath@@GLIBC_2.3`: the previous version returns `EINVAL` when the second
  parameter is NULL
* `memcpy@@GLIBC_2.14` [BZ12518](https://sourceware.org/bugzilla/show_bug.cgi?id=12518):
  the previous version guarantees a forward copying behavior. Shockwave Flash
  at that time had a "memcpy downward" bug which required the workaround.
* `quick_exit@@GLIBC_2.24` [BZ20198](https://sourceware.org/bugzilla/show_bug.cgi?id=20198):
  the previous version copies the destructors of `thread_local` objects.
* `glob64@@GLIBC_2.27`: the previous version does not follow dangling symlinks.

## How to remove symbol versioning

Imagine that you want to build an application with a prebuilt shared object
which has versioned references, but you can only find shared objects providing
the unversioned definitions. The linker will helpfully error:

```
ld.lld: error: undefined reference to foo@v1 [--no-allow-shlib-undefined]
```

As the diagnostic suggests, you can add `--allow-shlib-undefined` to get rid of
the error. It is not recommended but the built application may happen to work.

For this case, an alternative hacky solution is:

```
# 32-bit
cp in.so out.so
r2 -wqc '/x feffff6f00000000 @ section..dynamic; w0 16 @ hit0_0' out.so
llvm-objcopy -R .gnu.version out.so

# 64-bit
cp in.so out.so
r2 -wqc '/x feffff6f @ section..dynamic; w0 8 @ hit0_0' out.so
llvm-objcopy -R .gnu.version out.so
```

With the removal of `.gnu.version`, the linker will think that `out.so`
references foo instead of `foo@v1`. However, llvm-objcopy will zero out the
section contents. At runtime, glibc ld.so will complain unsupported version 0
of Verneed record. To make glibc happy, you can delete `DT_VER*` tags from the
dynamic table. The above code snippet uses an r2 command to locate
`DT_VERNEED(0x6ffffffe)` and rewrite it to `DT_NULL`(a `DT_NULL` entry stops
the parsing of the dynamic table). The difference of the `readelf -d` output is
roughly:

```
  0x000000006ffffffb (FLAGS_1)            Flags: NOW
- 0x000000006ffffffe (VERNEED)            0x8ef0
- 0x000000006fffffff (VERNEEDNUM)         5
- 0x000000006ffffff0 (VERSYM)             0x89c0
- 0x000000006ffffff9 (RELACOUNT)          1536
  0x0000000000000000 (NULL)               0x0
```

## LLD

* If an undefined symbol is not defined by a shared object, GNU ld will report
  an error. LLD before 12.0 did not error (I fixed it in
  https://reviews.llvm.org/D92260).

## Remarks

GCC/Clang supports asm specifier and `#pragma redefine_extname` renaming a
symbol. For example, if you declare `int foo() asm("foo_v1");` and then
reference `foo`, the symbol in .o will be `foo_v1`.

For example, the biggest change in musl v1.2.0 is the time64 support for its
supported 32-bit architectures. musl adopted a scheme based on asm specifiers:

```c
// include/features.h
#define __REDIR(x,y) __typeof__(x) x __asm__(#y)

// API header include/sys/time.h
int utimes(cosnt char *, const struct timeval [2]);
__REDIR(utimes, __utimes_time64);

// Implementation src/linux/utimes.c
int utimes(const char *path, const struct timeval times[2]) { ... }

// Internal header compat/time32/time32.h
int __utimes_time32() __asm__("utimes");

// Compat implementation compat/time32/utimes_time32.c
int __utimes_time32(const char *path, const struct timeval32 times32[2]) { ... }
```

* In .o, the time32 symbol remains `utimes` and is compatible with the ABI
  required by programs linked against old musl versions; the time64 symbol is
  `__utimes_time64`.
* The public header redirects utimes to `__utimes_time64`.
  * cons: if the user declares utimes by themself, they will not link against
    the correct `__utimes_time64`.
* The "good-looking" name `utimes` is used for the preferred time64
  implementation internally and the "ugly" name `__utimes_time32` is used for
  the legacy time32 implementation.
  * If the time32 implementation is called elsewhere, the "ugly" name can make
    it stand out.

For the above example, here is an implementation with symbol versioning:

```c
// API header include/sys/time.h
int utimes(cosnt char *, const struct timeval [2]);

// Implementation src/linux/utimes.c
int utimes(const char *path, const struct timeval times[2]) { ... }

// Internal header compat/time32/time32.h
// Probably __asm__(".symver __utimes_time32, utimes@time32, rename"); if supported
__asm__(".symver __utimes_time32, utimes@time32");

// Implementation compat/time32/utimes_time32.c
int __utimes_time32(const char *path, const struct timeval32 times32[2])
{
  ...
}
```

Note that it is `@@@` cannot be used. The header is included in a defining
translation unit and `@@@` will lead to a default version definition while we
want a non-default version definition.

According to Assembler behavior, the undesirable `__utimes_time32` is present.
Be careful to use a version script to localize it.

So what is the significance of symbol versioning? I think carefully:

* Refuse linking against old symbols while keeping compatibility with
  unversioned old libraries. [{reject-non-default}]()
* No need to label declarations.
* The version definition can be delayed until link time. The version script
  provides a flexible pattern matching mechanism to assign versions.
* Scope reduction. Arguably another mechanism like `--dynamic-list` might have
  been developed if version scripts did not provide `local:`.
* There are some semantic issues in renaming builtin functions with asm
  specifiers in GCC and Clang (they do not know that the renamed symbol has
  built-in semantic). See [2020-10-15-intra-call-and-libc-symbol-renaming](https://maskray.me/blog/2020-10-15-intra-call-and-libc-symbol-renaming)
* [verneed-check]

For the first item, the asm specifier scheme uses conventions to prevent
problems (users should include the header); and symbol versioning can be forced
by ld.

Design flaws:

* `.symver foo, foo@v1` In foobehavior defined [{gas-copy}](): reserved symbol
  `foo`(redundant symbol has a link), binding / `st_other`sync (not convenient
  to set different binding / visibility)
* Verdaux is a bit redundant. In practice, one Verdef has only one auxilliary
  Verdaux entry.
* This is arguably a minor problem but annoying for a framework providing
  multiple shared objects. ld.so requires "a versioned symbol is implemented in
  the same shared object in which it was found at link time", which disallows
  moving definitions between shared objects. Fortunately, glibc 2.30 [BZ24741](http://sourceware.org/PR24741)
  relaxes this requirement, essentially ignoring `Vernaux::vna_name`.

Before that, glibc used a forwarder to move `clock_*` functions from librt.so
to libc.so:

```c
// rt/clock-compat.c
__typeof(clock_getres) *clock_getres_ifunc(void) asm("clock_getres");
__typeof(clock_getres) *clock_getres_ifunc(void) { return &__clock_getres; }
```

libc.so defines `__clock_getres` and `clock_getres`. librt.so defines an ifunc
called `clock_getres` which forwards to libc.so `__clock_getres`.

## Related links

* [Combining Versions](combining-versions.md)
* [Version Scripts](version-scripts.md)
* https://invisible-island.net/ncurses/ncurses-mapsyms.html

