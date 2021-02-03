# Copy relocations, canonical PLT entries and protected visibility

Background:

* `-fno-pic` can only be used by executables. On most platforms and
  architectures, direct access relocations are used to reference external data
  symbols.
* `-fpic` can be used by both executables and shared objects. Windows has
  `__declspec(dllimport)` but most other binary formats allow a default
  visibility external data to be resolved to a shared object, so generally
  direct access relocations are disallowed.
* `-fpie` was introduced as a mode similar to `-fpic` for ELF: the compiler can
  make the assumption that the produced object file can only be used by
  executables, thus all definitions are non-preemptible and thus
  interprocedural optimizations can apply on them.

For

```c
extern int a;
int *foo() { return &a; }
```

`-fno-pic` typically produces an absolute relocation (a PC-relative relocation
can be used as well). On ELF x86-64 it is usually `R_X86_64_32` in the position
dependent small code model. If a is defined in the executable (by another
translation unit), everything works fine. If a turns out to be defined in a
shared object, its real address will be non-constant at link time. Either
action needs to be taken:

* Emit a dynamic relocation in every use site. Text sections are usually
  non-writable. A dynamic relocation applied on a non-writable section is
  called a text relocation.
* Emit a single copy relocation. Copy relocations only work for executables.
  The linker obtains the size of the symbol, allocates the bytes in `.bss`
  (this may make the object writable. On LLD a readonly area may be picked.),
  and emit an `R_*_COPY` relocation. All references resolve to the new location.

Multiple text relocations are even less acceptable, so on ELF a copy relocation
is generally used. Here is a nice description from [Rich
Felker](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=55012): "Copy relocations
are not a case of overriding the definition in the abstract machine, but an
implementation detail used to support data objects in shared libraries when the
main program is non-PIC."

Copy relocations have drawbacks:

* Break page sharing.
* Make the symbol properties (e.g. size) part of ABI.
* If the shared object is linked with `-Bsymbolic` or `--dynamic-list` and
  defines a data symbol copy relocated by the executable, the address of the
  symbol may be different in the shared object and in the executable.

What went poorly was that `-fno-pic` code had no way to avoid copy relocations
on ELF. Traditionally copy relocations could only occur in `-fno-pic` code. A
GCC 5 change made this possible for x86-64. Please read on.

## x86-64: copy relocations and `-fpie`

`-fpic` using GOT indirection for external data symbols has cost. Making
`-fpie` similar to `-fpic` in this regard incurs costs if the data symbol turns
out to be defined in the executable. Having the data symbol defined in another
translation unit linked into the executable is very common, especially if the
vendor uses fully/mostly statically linking mode.

In GCC 5, ["x86-64: Optimize access to globals in PIE with copy
reloc"](https://gcc.gnu.org/git/?p=gcc.git&a=commit;h=77ad54d911dd7cb88caf697ac213929f6132fdcf)
started to use direct access relocations for external data symbols on x86-64 in
`-fpie` mode.

```c
extern int a;
int foo() { return a; }
```

* GCC&lt;5: `movq a@GOTPCREL(%rip), %rax; movl (%rax), %eax` (8 bytes)
* GCC&gt;=5: `movl a(%rip), %eax` (6 bytes)

This change is actually useful for architectures other than x86-64 but is never
implemented for other architectures. What went wrong: the change was
implemented as an inflexible configure-time choice (`HAVE_LD_PIE_COPYRELOC`),
defaulting to such a behavior if ld supports PIE copy relocations (most
binutils installations). Keep in mind that such a `-fpie` default [breaks
`-Bsymbolic` and `--dynamic-list` in shared objects](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65888).

Clang addressed the inflexible configure-time choice via an opt-in option
`-mpie-copy-relocations` (D19996).

I noticed that:

* The option can be used for `-fno-pic` code as well to prevent copy
  relocations on ELF. This is occasionally users want (if their shared objects
  use `-Bsymbolic` and export data symbols (usually undesired from API
  perspecitives but can avoid costs at times)), and they switch from `-fno-pic`
  to `-fpic` just for this purpose.
* The option name should describe the code generation behavior, instead of the
  inferred behavior at the linking stage on a partibular binary format.
* The option does not need to tie to ELF.
  * On COFF, the behavior is like always `-fdirect-access-external-data`.
    `__declspec(dllimport)` is needed to enable indirect access.
  * On Mach-O, the behavior is like `-fdirect-access-external-data` for
    `-fno-pic` (only available on arm) and the opposite for `-fpic`.
* H.J. Lu introduced `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` as GOT
  optimization to x86-64 psABI. This is great! With the optimization, GOT
  indirection can be optimized, so the incured cost is very low now.

So I proposed an alternative option `-f[no-]direct-access-external-data`:
https://reviews.llvm.org/D92633
https://gcc.gnu.org/bugzilla/show_bug.cgi?id=98112. My wish on the GCC side is
to drop `HAVE_LD_PIE_COPYRELOC` and (x86-64) default to GOT indirection for
external data symbols in `-fpie` mode.

Please keep in mind that `-f[no-]semantic-interposition` is for definitions
while `-f[no-]direct-access-external-data` is for undefined data symbols. GCC 5
introduced `-fno-semantic-interposition` to use local aliases for references to
definitions in the same translation unit.

## `STV_PROTECTED`

Now let's consider how `STV_PROTECTED` comes into play. Here is the generic ABI
definition:

> A symbol defined in the current component is protected if it is visible in
> other components but not preemptable, meaning that any reference to such a
> symbol from within the defining component must be resolved to the definition
> in that component, even if there is a definition in another component that
> would preempt by the default rules. A symbol with `STB_LOCAL` binding may not
> have `STV_PROTECTED` visibility. If a symbol definition with `STV_PROTECTED`
> visibility from a shared object is taken as resolving a reference from an
> executable or another shared object, the `SHN_UNDEF` symbol table entry
> created has `STV_DEFAULT` visibility.

A non-local `STV_DEFAULT` defined symbol is by default preemptible in a shared
object on ELF. `STV_PROTECTED` can make the symbol non-preemptible. You may
have noticed that I use "preemptible" while the generic ABI uses "preemptable"
and LLVM IR uses "`dso_preemptable`". Both forms work. "preemptible" is my
opition because it is more common.

### Protected data symbols and copy relocations

Many folks consider that copy relocations are best-effort support provided by
the toolchain. `STV_PROTECTED` is intended as an optimization and the
optimization can error out if it can't be done for whatever reason. Since copy
relocations are already oftentimes unacceptable, it is natural to think that we
should just disallow copy relocations on protected data symbols.

However, GNU ld 2.26 made a change which enabled copy relocations on protected
data symbols for i386 and x86-64.

A glibc change ["Add `ELF_RTYPE_CLASS_EXTERN_PROTECTED_DATA` to
x86"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=62da1e3b00b51383ffa7efc89d8addda0502e107)
is needed to make copy relocations on protected data symbols work.
["[AArch64][BZ #17711] Fix extern protected data handling"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=0910702c4d2cf9e8302b35c9519548726e1ac489)
and ["[ARM][BZ #17711] Fix extern protected data handling"](https://sourceware.org/git/gitweb.cgi?p=glibc.git;h=3bcea719ddd6ce399d7bccb492c40af77d216e42)
ported the thing to arm and aarch64.

Despite the glibc support, GNU ld aarch64 errors relocation
`R_AARCH64_ADR_PREL_PG_HI21` against symbol `foo` which may bind externally can
not be used when making a shared object; recompile with `-fPIC`.

powerpc64 ELFv2 is interesting: TOC indirection (TOC is a variant of GOT) is
used everywhere, data symbols normally have no direct access relocations, so
this is not a problem.

```c
// b.c
__attribute__((visibility("protected"))) int foo;
// a.c
extern int foo;
int main() { return foo; }
```

```
gcc -fuse-ld=bfd -fpic -shared b.c -o b.so
gcc -fuse-ld=bfd -pie -fno-pic a.c ./b.so
```

gold does not allow copy relocations on protected data symbols, but it misses
some cases: https://sourceware.org/bugzilla/show_bug.cgi?id=19823.

### Protected data symbols and direct accesses

If a protected data symbol in a shared object is copy relocated, allowing
direct accesses will cause the shared object to operate on a different copy
from the executable. Therefore, direct accesses to protected data symbols have
to be disallowed in `-fpic` code, just in case the symbols may be copy
relocated.  https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65248 changed GCC 5 to
use GOT indirection for protected external data.

```c
__attribute__((visibility("protected"))) int foo;
int val() { return foo; }
// -fPIC: GOT on at least aarch64, arm, i386, x86-64
```

This caused unneeded pessimization for protected external data. Clang always
treats protected similar to hidden/internal.

For older GCC (and all versions of Clang), direct accesses are produced in
`-fpic` code. Mixing such object files can silently break copy relocations on
protected data symbols. Therefore, GNU ld made the change
https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;a=commit;h=ca3fe95e469b9daec153caa2c90665f5daaec2b5
to error in `-shared` mode.

```
% cat a.s
leaq foo(%rip), %rax

.data
.global foo
.protected foo
foo:
```
```
% gcc -fuse-ld=bfd -shared a.s
/usr/bin/ld.bfd: /tmp/ccchu3Xo.o: relocation R_X86_64_PC32 against protected symbol `foo' can not be used when making a shared object
/usr/bin/ld.bfd: final link failed: bad value
collect2: error: ld returned 1 exit status
```

This led to a heated discussion
https://sourceware.org/legacy-ml/binutils/2016-03/msg00312.html. Swift folks
noticed this https://bugs.swift.org/browse/SR-1023 and their reaction was to
switch from GNU ld to gold.

GNU ld's aarch64 port does not have the diagnostic.

binutils commit ["x86: Clear `extern_protected_data` for
`GNU_PROPERTY_NO_COPY_ON_PROTECTED`"](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=73784fa565bd66f1ac165816c03e5217b7d67bbc)
introduced
`GNU_PROPERTY_NO_COPY_ON_PROTECTED`. With this property, `ld -shared` will not
error for relocation `R_X86_64_PC32` against protected symbol `foo` can not be
used when making a shared object.

The two issues above are the costs enabling copy relocations on protected data
symbols. Personally I don't think copy relocations on protected data symbols
are actually leveraged. GNU ld's x86 port can just (1) reject such copy
relocations and (2) allow direct accesses referencing protected data symbols in
`-shared` mode. But I am not really clear about the glibc case. I wish
`GNU_PROPERTY_NO_COPY_ON_PROTECTED` can become the default or be phased out in
the future.

### Protected function symbols and canonical PLT entries

```c
// b.c
__attribute__((visibility("protected"))) void *foo () {
  return (void *)foo;
}
```

GNU ld's aarch64 and x86 ports rejects the above code. On many other
architectures including powerpc the code is supported.

```
% gcc -fpic -shared b.c -fuse-ld=bfd b.c -o b.so
/usr/bin/ld.bfd: /tmp/cc3Ay0Gh.o: relocation R_X86_64_PC32 against protected symbol `foo' can not be used when making a shared object
/usr/bin/ld.bfd: final link failed: bad value
collect2: error: ld returned 1 exit status
% gcc -shared -fuse-ld=bfd -fpic b.c -o b.so
/usr/bin/ld.bfd: /tmp/ccXdBqMf.o: relocation R_AARCH64_ADR_PREL_PG_HI21 against symbol `foo' which may bind externally can not be used when making a shared object; recompile with -fPIC
/tmp/ccXdBqMf.o: in function `foo':
a.c:(.text+0x0): dangerous relocation: unsupported relocation
collect2: error: ld returned 1 exit status
```

The rejection is mainly a historical issue to make pointer equality work with
`-fno-pic` code. The GNU ld idea is that:

* The compiler emits GOT-generating relocations for `-fpic` code (in reality it
  does it for declarations but not for definitions).
* `-fno-pic` main executable uses direct access relocation types and gets a
  canonical PLT entry.
* glibc ld.so resolves the GOT in the shared object to the canonical PLT entry.

Actually we can take the interepretation that a canonical PLT entry is
incompatible with a shared `STV_PROTECTED` definition, and reject the attempt
to create a canonical PLT entry (gold/LLD). And we can keep producing direct
access relocations referencing protected symbols for `-fpic` code.
`STV_PROTECTED` is no different from `STV_HIDDEN`.

On many architectures, a branch instruction uses a branch specific relocation
type (e.g. `R_AARCH64_CALL26`, `R_PPC64_REL24`, `R_RISCV_CALL_PLT`). This is
great because the address is insignificant and the linker can arrange for a
regular PLT if the symbol turns out to be external.

On i386, a branch in `-fno-pic` code emits an `R_386_PC32` relocation, which is
indistinguishable from an address taken operation. If the symbol turns out to
be external, the linker has to employ a tricky called "canonical PLT entry"
(`st_shndx=0, st_value!=0`). The term is a parlance within a few LLD
developers, but not broadly adopted.

```c
// a.c
extern void foo(void);
int main() { foo(); }
```
```
% gcc -m32 -shared -fuse-ld=bfd -fpic b.c -o b.so
% gcc -m32 -fno-pic -no-pie -fuse-ld=lld a.c ./b.so

% gcc -m32 -fno-pic a.c ./b.so -fuse-ld=lld
ld.lld: error: cannot preempt symbol: foo
>>> defined in ./b.so
>>> referenced by a.c
>>>               /tmp/ccDGhzEy.o:(main)
collect2: error: ld returned 1 exit status

% gcc -m32 -fno-pic -no-pie a.c ./b.so -fuse-ld=bfd
# canonical PLT entry; foo has different addresses in a.out and b.so.
% gcc -m32 -fno-pic -pie a.c ./b.so -fuse-ld=bfd
/usr/bin/ld.bfd: /tmp/ccZ3Rl8Y.o: warning: relocation against `foo' in read-only section `.text'
/usr/bin/ld.bfd: warning: creating DT_TEXTREL in a PIE
% gcc -m32 -fno-pic -pie a.c ./b.so -fuse-ld=bfd -z text
/usr/bin/ld.bfd: /tmp/ccUv8wXc.o: warning: relocation against `foo' in read-only section `.text'
/usr/bin/ld.bfd: read-only segment has dynamic relocations
collect2: error: ld returned 1 exit status
```

This used to be a problem for x86-64 as well, until ["x86-64: Generate branch
with PLT32 relocation"](https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;h=bd7ab16b4537788ad53521c45469a1bdae84ad4a)
changed call/jmp foo to emit `R_X86_64_PLT32` instead of `R_X86_64_PC32`. Note:
(`-fpie`/`-fpic`) `call/jmp foo@PLT` always emits `R_X86_64_PLT32`.

The relocation type name is a bit misleading, `_PLT32` does not mean that a PLT
will always be created. Rather, it is optional: the linker can resolve `_PLT32`
to any place where the function will be called. If the symbol is preemptible,
the place is usually the PLT entry. If the symbol is non-preemptible, the
linker can convert `_PLT32` into `_PC32`. A function symbol can be either
branched or taken address. For an address taken operation, the function symbol
is used in a manner similar to a data symbol. `R_386_PLT32` cannot be used. LLD
and gold will just reject the link if text relocations are disabled.

On i386, my proposal is that branches to a default visibility function
declaration should use `R_386_PLT32` instead of `R_386_PC32`, in a manner
similar to x86-64. Originally I thought an assembler change sufficed:
https://sourceware.org/bugzilla/show_bug.cgi?id=27169. Please read the next
section why this should be changed on the compiler side.

### Non-default visibility ifunc and `R_386_PC32`

For a call to a hidden function declaration, the compiler produces an
`R_386_PC32` relocation. The relocation is an indicator that EBX may not be set
up.

If the declaration refers to an ifunc definition, the linker will resolve the
`R_386_PC32` to an IPLT entry. For `-pie` and `-shared` links, the IPLT entry
references EBX. If the call site does not set up EBX to be
`_GLOBAL_OFFSET_TABLE_`, the IPLT call will be incorrect.

GNU ld has implemented a diagnostic (["i686 ifunc and non-default symbol
visibility"](https://sourceware.org/bugzilla/show_bug.cgi?id=20515)) to catch
the problem. If we change `call/jmp foo` to always use `R_386_PLT32`, such a
diagnostic will be lost.

Can we change the compiler to emit `call/jmp foo@PLT` for default visibility
function declarations? If the compiler emits such a modifier but does not set
up EBX, the ifunc can still be non-preemptible (e.g. hidden in another
translation unit or `-Bsymbolic`) and we will still have a dilemma.

Personally, I think avoiding a canonical PLT entry is more useful than a ld
ifunc diagnostic. i386 ABI is legacy and the x86 maintainer will not make the
change, though.

## Summary

I hope the above give an overview to interested readers. Symbol interposition
is subtle. One has to think about all the factors related to symbol
interposition and the relevant toolchain fixes are like a whack-a-mole game. I
appreciate all the prior discussions and I believe many unsatisfactory things
can be fixed in a quite backward-compatible way.

Some features are inherently incompatible. We make the trade-off in favor of
more important features. Here are two things that should not work. However, if
`-fpie` or `-fno-direct-access-external-data` is specified, both limitations
will be circumvented.

* Copy relocations on protected data symbols.
* Canonical PLT entries on protected function symbols. With the `R_386_PLT32`
  change, this issue will only affect function pointers.

People sometimes simply just say: "protected visibility does not work." I'd
argue that Clang+gold/LLD works quite well.

The things on GCC+GNU ld side are inconsistent, though. Here is a list of
changes I wish can happen:

* GCC: add `-f[no-]direct-access-external-data`.
* GCC: drop `HAVE_LD_PIE_COPYRELOC` in favor of `-f[no-]direct-access-external-data`.
* GCC x86-64: default to GOT indirection for external data symbols in `-fpie`
  mode.
* GCC or GNU as i386: emit `R_386_PLT32` for branches to undefined function
  symbols.
* GNU ld x86: disallow copy relocations on protected data symbols. (I think
  canonical PLT entries on protected symbols have been disallowed.)
* GCC aarch64/arm/x86/...: allow direct access relocations on protected symbols
  in `-fpic` mode.
* GNU ld aarch64/x86: allow direct access relocations on protected data symbols
  in `-shared` mode.

The breaking changes for GCC+GNU ld:

* The "copy relocations on protected data symbols" scheme has been supported in
  the past few years with GNU ld on x86, but it did not work before circa 2015,
  and should not work in the future. Fortunately the breaking surface may be
  narrow: this scheme does not work with gold or LLD. Many architectures don't
  work.
* ld is not the only consumer of `R_386_PLT32`. The Linux kernel has code
  resolving relocations and it needs to be fixed (patch uploaded: https://github.com/ClangBuiltLinux/linux/issues/1210).

I'll conclude thie article with random notes on other binary formats:

Windows/COFF `__declspec(dllimport)` gives us a different perspecitive how
external references can be designed. The annotation is verbose but
differentiates the two cases (1) the symbol has to be defined in the same
linkage unit (2) the symbol can be defined in another linkage unit. If we lift
the "the symbol visibility is decided by the most constrained visibility"
requirement for protected-&gt;default, a COFF undefined/defined symbol is quite
like a protected undefined/defined symbol in ELF. `__declspec(dllimport)` gives
the undefined symbol default visibility (i.e. the LLVM IR `dllimport` is
redundant). `__declspec(dllexport)` is something which cannot be modeled with
the existing ELF visibilities.

For an undefined variable, Mach-O uses `__attribute__((visibility("hidden")))`
to say "a definition must be available in another translation unit in the same
linkage unit" but does not actually mark the undefined symbol anyway. COFF uses
`__declspec(dllimport)` to convey this. In ELF,
`__attribute__((visibility("hidden")))` additionally makes the undefined symbol
unexportable. The Mach-O notation actually resembles COFF: it can be exported
by the definition in another translation unit. From its behavior, I think it
would be more appropriately mapped to LLVM IR protected instead of hidden.

## Appendix

For a `STB_GLOBAL`/`STB_WEAK` symbol,

`STV_DEFAULT`: both compiler &amp; linker need to assume such symbols can be
preempted in `-fpic` mode. The compiler emits GOT indirection by default. GCC
`-fno-semantic-interposition` uses local aliases on defined non-weak function
symbols for x86 (unimplemented in other architectures). Clang
`-fno-semantic-interposition` uses local aliases on defined non-weak symbols
(both function and data) for x86.

`STV_PROTECTED`: GCC `-fpic` uses GOT indirection for data symbols, regardless
of defined or undefined. This pessimization is to make a misfeature "copy
relocation on protected data symbol" work
(https://maskray.me/blog/2021-01-09-copy-relocations-canonical-plt-entries-and-protected#protected-data-symbols-and-direct-accesses).
Clang code generation treats `STV_PROTECTED` the same way as `STV_HIDDEN`.

`STV_HIDDEN`: non-preemptible, regardless of defined or undefined. The compiler
suppresses GOT indirection, unless undefined `STB_WEAK`.

For defined symbols, `-fno-pic`/`-fpie` can avoid GOT indirection for
`STV_DEFAULT` (and GCC `STV_PROTECTED`). `-fvisibility=hidden` can change
visibility.

For undefined symbols, `-fpie`/`-fpic` use GOT indirection by default. Clang
`-fno-direct-access-external-data` (discussed in my article) can avoid GOT
indirection. If you `-fpic -fno-direct-access-external-data` &amp; `ld
-shared`, you'll need additional linker options to make the linker know defined
non-`STB_LOCAL` `STV_DEFAULT` symbols are non-preemptible.

