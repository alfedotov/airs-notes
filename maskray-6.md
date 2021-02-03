# GNU indirect function

UNDER CONSTRUCTION.

GNU indirect function (ifunc) is a mechanism making a direct function call
resolve to an implementation picked by a resolver. It is mainly used in glibc
but has adoption in FreeBSD.

For some performance critical functions, e.g. memcpy/memset/strcpy, glibc
provides multiple implementations optimized for different architecture levels.
The application just uses `memcpy(...)` which compiles to call memcpy. The
linker will create a PLT for `memcpy` and produce an associated special dynamic
relocation referencing the resolver symbol/address. During relocation resolving
at runtime, the return value of the resolver will be placed in the GOT entry
and the PLT entry will load the address.

## Representation

ifunc has a dedicated symbol type `STT_GNU_IFUNC` to mark it different from a
regular function (`STT_FUNC`). The value 10 is in the OS-specific range (10~12).
`readelf -s` tell you that the symbol is ifunc if OSABI is `ELFOSABI_GNU` or
`ELFOSABI_FREEBSD`.

On Linux, by default GNU as uses `ELFOSABI_NONE` (0). If ifunc is used, the OSABI
will be changed to `ELFOSABI_GNU`. Similarly, GNU ld sets the OSABI to
`ELFOSABI_GNU` if ifunc is used. gold does not do this [PR17735](https://sourceware.org/bugzilla/show_bug.cgi?id=17735).

Things are loose in LLVM. The integrated assembler and LLD do not set
`ELFOSABI_GNU`. Currently the only problem I know is the `readelf -s` display.
Everything else works fine.

### Assembler behavior

In assembly, you can assign the type `STT_GNU_IFUNC` to a symbol via
`.type foo, @gnu_indirect_function`. An ifunc symbol is typically `STB_GLOBAL`.

In the object file, `st_shndx` and `st_value` of an `STT_GNU_IFUNC` symbol
indicate the resolver. After linking, if the symbol is still `STT_GNU_IFUNC`,
its `st_value` field indicates the resolver address in the linked image.

Assemblers usually convert relocations referencing a local symbol to reference
the section symbol, but this behavior needs to be inhibited for `STT_GNU_IFUNC`.

### Example

```
cat > b.s <<e
.global ifunc
.type ifunc, @gnu_indirect_function
.set ifunc, resolver

resolver:
  leaq impl(%rip), %rax
  ret

impl:
  movq $42, %rax
  ret
e

cat > a.c <<e
int ifunc(void);
int main() { return ifunc(); }
e

cc a.c b.s
./a.out  # exit code 42
```

GNU as makes transitive aliases to an `STT_GNU_IFUNC` ifunc as well.

```
.type foo,@gnu_indirect_function
.set foo, foo_resolver

.set foo2, foo
.set foo3, foo2
```

GCC and Clang support a function attribute which emits
`.type ifunc, @gnu_indirect_function; .set ifunc, resolver`:

```c
static int impl(void) { return 42; }
static void *resolver(void) { return impl; }
void *ifunc(void) __attribute__((ifunc("resolver")));
```

## Preemptible ifunc

A preemptible ifunc call is no different from a regular function call from the
linker perspective.

The linker creates a PLT entry, reserves an associated GOT entry, and emits an
`R_*_JUMP_SLOT` relocation resolving the address into the GOT entry. The PLT
code sequence is the same as a regular PLT for `STT_FUNC`.

If the ifunc is defined within the module, the symbol type in the linked image
is `STT_GNU_IFUNC`, otherwise (defined in a DSO), the symbol type is `STT_FUNC`.

The difference resides in the loader.

At runtime, the relocation resolver checks whether the `R_*_JUMP_SLOT`
relocation refers to an ifunc. If it does, instead of filling the GOT entry
with the target address, the resolver calls the target address as an indirect
function, with ABI specified additional parameters (hwcap related), and places
the return value into the GOT entry.

## Non-preemptible ifunc

The non-preemptible ifunc case is where all sorts of complexity come from.

First, the `R_*_JUMP_SLOT` relocation type cannot be used in some cases:

* A non-preemptible ifunc may not have a dynamic symbol table entry. It can be
  local. It can be defined in the executable without the need to export.
* A non-local `STV_DEFAULT` symbol defined in a shared object is by default
  preemptible. Using `R_*_JUMP_SLOT` for such a case will make the ifunc look
  like preemptible.

Therefore a new relocation type `R_*_IRELATIVE` was introduced. There is no
associated symbol and the address indicates the resolver.

```
R_*_RELATIVE: B + A
R_*_IRELATIVE: call (B + A) as a function
R_*_JUMP_SLOT: S
```

When an `R_*_JUMP_SLOT` can be used, there is a trade-off between
`R_*_JUMP_SLOT` and `R_*_IRELATIVE`: an `R_*_JUMP_SLOT` can be lazily resolved
but needs a symbol lookup. Currently powerpc can use `R_PPC64_JMP_SLOT` in some
cases [PR27203](https://sourceware.org/bugzilla/show_bug.cgi?id=27203).

A PLT entry is needed for two reasons:

* The call sites emit instructions like call foo. We need to forward them to a
  place to perform the indirection. Text relocations are usually not an option
  (exception: [{ifunc-noplt}]()).
* If the ifunc is exported, we need a place to mark its canonical address.

Such PLT entries are sometimes referred to as IPLT. They are placed in the
synthetic section .iplt. In GNU ld, `.iplt` will be placed in the output
section `.plt`. In LLD, I decided that `.iplt` is better
https://reviews.llvm.org/D71520.

On many architectures (e.g. AArch64/PowerPC/x86), the PLT code sequence is the
same as a regular PLT, but it could be different.

On x86-64, the code sequence is:

```
jmp *got(%rip)
pushq $0
jmp .plt
```

Since there is no lazy binding, `pushq $0; jmp .plt` are not needed. However,
to make all PLT entries of the same shape to simplify linker implementations
and facilitate analyzers, it is find to keep it this way.

## PowerPC32 `-msecure-plt` IPLT

As a design to work around the lack of PC-relative instructions, PowerPC32 uses
multiple GOT sections, one per file in `.got2`. To support multiple GOT
pointers, the addend on each `R_PPC_PLTREL24` reloc will have the offset within
`.got2`.

`-msecure-plt` has small/large PIC differences.
* `-fpic`/`-fpie`: `R_PPC_PLTREL24 r_addend=0`. The call stub loads an address
  relative to `_GLOBAL_OFFSET_TABLE_`.
* `-fPIC`/`-fPIE`: `R_PPC_PLTREL24 r_addend=0x8000`. (A partial linked object
  file may have an addend larger than 0x8000.) The call stub loads an address
  relative to `.got2+0x8000`.

If a non-preemptible ifunc is referenced in two object files, in
`-pie`/`-shared` mode, the two object files cannot share the same IPLT entry.
When I added non-preemptible ifunc support for PowerPC32 to LLD
https://reviews.llvm.org/D71621, I did not handle this case.

### `.rela.dyn` vs `.rela.plt`

LLD placed `R_*_IRELATIVE` in the `.rela.plt` section because many ports of GNU
ld behaved this way. While implementing ifunc for PowerPC, I noticed that GNU
ld powerpc actually places `R_*_IRELATIVE` in `.rela.dyn` and glibc powerpc
does not actually support `R_*_IRELATIVE` in `.rela.plt`. This makes a lot of
sense to me because `.rela.plt` normally just contains `R_*_JUMP_SLOT` which
can be lazily resolved. ifunc relocations need to be eagerly resolved so
`.rela.plt` was a misplace. Therefore I changed LLD to use `.rela.dyn` in
https://reviews.llvm.org/D65651.

## `__rela_iplt_start` and `__rela_iplt_end`

A statically linked position dependent executable traditionally had no dynamic
relocations.

With ifunc, these `R_*_IRELATIVE` relocations must be resolved at runtime. Such
relocations are in a magic array delimitered by `__rela_iplt_start` and
`__rela_iplt_end`. In glibc, `csu/libc-start.c` has special code processing the
relocation range.

GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in `-pie`
mode. LLD defines `__rela_iplt_start` regardless of `-no-pie`, `-pie` or
`-shared`.

In glibc, static pie uses self-relocation (`_dl_relocate_static_pie`) to take
care of `R_*_IRELATIVE`. The above magic array code is executed by static pie
as well. If `__rela_iplt_start`/`__rela_iplt_end` are defined, we will get
`0 < __rela_iplt_start < __rela_iplt_end` in `csu/libc-start.c`.
`ARCH_SETUP_IREL` will crash when resolving the first relocation which has been
processed.

I think the difference in the
`diff -u =(ld.bfd --verbose) =(ld.bfd -pie --verbose)` output is unneeded.
https://sourceware.org/pipermail/libc-alpha/2021-January/121755.html

## Address significance

A non-GOT-generating non-PLT-generating relocation referencing a
`STT_GNU_IFUNC` indicates a potential address-taken operation.

With a function attribute, the compilers knows that a symbol indicates an ifunc
and will avoid generating such relocations. With assembly such relocations may
be unavoidable.

In most cases the linker needs to convert the symbol type to `STT_FUNC` and
create a special PLT entry, which is called a "canonical PLT entry" in LLD.
References from other modules will resolve to the PLT entry to keep pointer
equality: the address taken from the defining module should match the address
taken from another module.

This approach has pros and cons:

* With a canonical PLT entry, the resolver of a symbol is called only once.
  There is exactly one `R_*_IRELATIVE` relocation.
* If the relocation appears in a non-`SHF_WRITE` section, a text relocation can
  be avoided.
* Relocation types which are not valid dynamic relocation types are supported.
  GNU ld may error relocation `R_X86_64_PC32` against `STT_GNU_IFUNC` symbol
  `ifunc` isn't supported
* References will bind to the canonical PLT entry. A function call needs to
  jump to the PLT, loads the value from the GOT, then does an indirect call.

For a symbolic relocation type (a special case of absolute relocation types
where the width matches the word size) like `R_X86_64_64`, when the addend is 0
and the section has the `SHF_WRITE` flag, the linker can emit an
`R_X86_64_IRELATIVE`. https://reviews.llvm.org/D65995 dropped the case.

For the following example, GNU ld linked `a.out` calls `fff_resolver` three
times while LLD calls it once.

```c
// RUN: split-file %s %t
// RUN: clang -fuse-ld=bfd -fpic %t/dso.c -o %t/dso.so --shared
// RUN: clang -fuse-ld=bfd %t/main.c %t/dso.so -o %t/a.out
// RUN: %t/a.out

//--- dso.c
typedef void fptr(void);
extern void fff(void);

fptr *global_fptr0 = &fff;
fptr *global_fptr1 = &fff;

//--- main.c
#include <stdio.h>

static void fff_impl() { printf("fff_impl()\n"); }
static int z;
void *fff_resolver() { return (char *)&fff_impl + z++; }

__attribute__((ifunc("fff_resolver"))) void fff();
typedef void fptr(void);
fptr *local_fptr = fff;
extern fptr *global_fptr0, *global_fptr1;

int main() {
  printf("local %p global0 %p global1 %p\n", local_fptr, global_fptr0, global_fptr1);
  return 0;
}
```

### Relocation resolving order

`R_*_IRELATIVE` relocations are resolved eagerly. In glibc, there used to be a
problem where ifunc resolvers ran before `GL(dl_hwcap)` and `GL(dl_hwcap2)`
were set up https://sourceware.org/bugzilla/show_bug.cgi?id=27072.

For the relocation resolver, the main executable needs to be processed the last
to process `R_*_COPY`. Without ifunc, the resolving order of shared objects can
be arbitrary.

For ifunc, if the ifunc is defined in a processed module, it is fine. If the
ifunc is defined in an unprocessed module, it may crash.

For an ifunc defined in an executable, calling it from a shared object can be
problematic because the executable's relocations haven't been resolved. The
issue can be circumvented by converting the non-preemptible ifunc defined in
the executable to `STT_FUNC`. GNU ld's x86 port made the change
[PR23169](https://sourceware.org/bugzilla/show_bug.cgi?id=23169).

## `-z ifunc-noplt`

Mark Johnston introduced `-z ifunc-noplt` for FreeBSD
https://reviews.llvm.org/D61613. With this option, all relocations referencing
`STT_GNU_IFUNC` will be emitted as dynamic relocations (if `.dynsym` is
created).  The canonical PLT entry will not be used.

## Miscellaneous

GNU ld has implemented a diagnostic (["i686 ifunc and non-default symbol
visibility"](https://sourceware.org/bugzilla/show_bug.cgi?id=20515)) to flag
`R_386_PC32` referencing non-default visibility ifunc in `-pie` and `-shared`
links. This diagnostic looks like the most prominent reason blocking my
proposal to use `R_386_PLT32` for `call/jump foo`. See [Copy relocations,
canonical PLT entries and protected visibility](maskray-5.md) for details.

https://sourceware.org/glibc/wiki/GNU_IFUNC misses a lot of information. There
are quite a few arch differences. I asked for clarification
https://sourceware.org/pipermail/libc-alpha/2021-January/121752.html

### Dynamic loader

In glibc, `_dl_runtime_resolver` needs to save and restore vector and floating
point registers. ifunc resolvers add another reason that `_dl_runtime_resolver`
cannot only use integer registers. (The other reasons are that ld.so has string
function calls which may use vectors and external calls to libc.so.)

