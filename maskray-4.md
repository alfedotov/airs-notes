# LLD and GNU linker incompatibilities

Subtitle: Is LLD a drop-in replacement for GNU ld?

The motivation for this article was someone challenging the "drop-in
replacement" claim on LLD's website (the discussion was about Linux-like ELF
toolchain):

> LLD is a linker from the LLVM project that is a drop-in replacement for
> system linkers and runs much faster than them. It also provides features that
> are useful for toolchain developers.

99.9% pieces of software work with LLD without a change. Some linker script
applications may need an adaption (such adaption is oftentimes due to brittle
assumptions: asking too much from GNU ld's behavior which should be fixed
anyway). So I defended for this claim.

Piotr Kubaj said that this is a probably more of a marketing term than a
technical term, the term tries to lure existing users into thinking "it's the
same you know, but better!". I think that this is fair in some senses: for many
applications LLD has achieved much faster speed and much lower memory usage
than GNU ld. A more important thing is that LLD adds a third choice to the
spectrum. It brings competitive pressure to both sides, gives incentive for
improvement, and makes for more standardized future features/extensions. One
reason that I am subscribed to the binutils mailing list is I want to
participate in its design processes (I am proud to say that I have managed to
find some early issues of various new things).

Anyway, I thought documenting the compatibility problems between the ELF ports
of LLD and GNU ld is useful, not only to others but also to my future self,
hence this article. I will try to describe GNU gold behaviors as well.

So here is the long list. Please keep in mind that many compatibility issues do
not really matter and a user may never run into such an issue. Many of them
just serve as educational purposes and my personal reference. There some some
user perceivable differences but quite a lot are WONTFIX on both GNU ld and
LLD. LLD, as a newer linker, has less legacy compatibility burden and can make
good default choices in some cases and say no to some unneeded
features/behaviors. A large number of features are duplicated in GNU ld's
various ports. It is also common that one thing behaves this way in port A and
another way in port B.

* GNU ld reports `gc-sections requires either an entry or an undefined symbol`
  in a -r --gc-section link. LLD doesn't error
  (https://reviews.llvm.org/D84131#2162411). I am unsure whether such a
  diagnostic will be useful (an uncommon use case where the GC roots are more
  than the explict linker options).
* The default image base for `-no-pie` links is different. For example, on
  x86-64, GNU ld defaults to 0x400000 while LLD defaults to 0x200000.
* GNU ld synthesizes a `STT_FILE` symbol when copying non-`STT_SECTION`
  `STB_LOCAL` symbols. LLD doesn't.
  * The `STT_FILE` symbol name is the input filename. For compiler driver
    specified startup files like `crti.o` and `crtn.o`, their absolute paths
    will end up in the linked image. This breaks local determinism (toolchain
    paths are leaked) for some users.
  * I filed https://bugs.llvm.org/show_bug.cgi?id=48023 and
    https://sourceware.org/bugzilla/show_bug.cgi?id=26822. From binutils 2.36
    onwards, the base name will be used.
* Text relocations.
  * In GNU ld, `-z notext`/`-z text`/unspecified are a tri-state. For
    `-z notext`/unspecified, the dynamic tags `DT_TEXTREL` and `DF_TEXTREL` are
    added on demand. If unspecified and GNU ld is configured with
    `--enable-textrel-check=warning`, a warning will be issued.
  * LLD has two states and add `DT_TEXTREL` and `DF_TEXTREL` if `-z notext` is specified.
  * GNU ld supports more relocation types as text relocations.
* Default library paths.
  * GNU ld has default library paths.
  * LLD doesn't. This is intentional so https://reviews.llvm.org/D70048
    (NetBSD) cannot be accepted.
* GNU ld supports grouped short options. This can sometimes cause surprising
  behaviors with misspelled or unimplemented options, e.g. `-no-pie` means
  `-n -o -pie` because GNU ld as of 2.35 has not implemented `-no-pie`. Nick
  Clifton committed `Update the BFD linker so that it deprecates grouped short
  options.` to deprecated the GNU ld feature. LLD never supports grouped short
  options.
* Mixed `SHF_LINK_ORDER` and non-`SHF_LINK_ORDER` input sections in an output
  section.
  * LLD performs sorting within an input section description and allows
    arbitrary mixes.
  * GNU ld does not allow mixed sections
    https://sourceware.org/bugzilla/show_bug.cgi?id=26256 (H.J. Lu has a patch)
* LLD defaults to `-z relro` by default. This is probably not a good default
  but it is difficult to change now. I have a comment
  https://bugs.llvm.org/show_bug.cgi?id=48549. GNU ld warns for `-z relro` and
  `-z norelro` for non Linux/FreeBSD BFD emulations (e.g. `-m aarch64elf`).
* Different archive member extraction semantics. See
  http://lld.llvm.org/ELF/warn_backrefs.html for details.
* LLD `--warn-backrefs` warns for `def.a ref.o def.so` if `def.a` cannot
  satisfy previous unresolved symbols. LLD resolves the definition to `def.a`
  while GNU linkers resolve the definition to `def.so`.
* GNU ld `-static` has traditionally been a synonym to `-Bstatic`. Recently on
  x86 it has been changed to behave a bit similar to gold `-static`, which
  disallows linking against shared objects. LLD `-static` is still a synonym to
  `-Bstatic`.
* GNU linkers have a default `--dynamic-linker`. LLD doesn't.
* GNU linkers warn for `.gnu.warning.*` sections. LLD doesn't. It is unclear
  the feature is useful. https://bugs.llvm.org/show_bug.cgi?id=42008
* GNU ld has architecture-specific rules for relocations referencing undefined
  weak symbols. I don't think the GNU ld behaviors can be summarized (even by
  maintainers!). LLD's are consistent.
* The conditions to create `.interp` are different. I believe GNU ld's is quite
  difficult to describe.
* `--no-allow-shlib-undefined` and `--rpath-link`
  * GNU ld traces all shared objects (transitive `DT_NEEDED` dependencies) and
    emulates the bheavior of a dynamic loader to warn more cases.
  * gold and LLD implement a simplified version. They warn for shared objects
    whose `DT_NEEDED` dependencies are all seen as input files.
* `--fatal-warnings`
  * GNU ld still reports warning: ....
  * LLD switches to error: ....
* `--no-relax`
  * GNU ld: disable `R_X86_64_[REX_]GOTPCRELX`
  * LLD: no-op (https://reviews.llvm.org/D81359)
* LLD places `.rodata` (among other `SHF_ALLOC` and
  non-`SHF_WRITE`-non-`SHF_EXECINSTR` sections) before .text (among other
  `SHF_ALLOC` and `SHF_EXECINSTR` sections).
* `.symtab`/`.shstrtab`/`.strtab` in a linker script.
  * Ignored by GNU ld, therefore `--orphan-handling=` does not warn/error.
  * Respected by LLD
* Whether `ADDR(.foo)` in a linker script can retain an empty output section.
  * GNU ld: no. Symbol assignments relative to such empty sections may have
    strange `st_shndx`.
  * LLD: yes.
* If an undefined symbol is referenced by both `R_X86_64_JUMP_SLOT` (lazy) and
  R_X86_64_GLOB_DAT (`non-lazy`)
  * GNU ld generates `.plt.got` with `R_X86_64_GLOB_DAT` relocations.
    `R_X86_64_JUMP_SLOT` can thus be omitted to decrease the number of dynamic
    relocations.
  * LLD does not implement this saving. This naturally requires more than one
    pass scanning relocations which LLD doesn't do at present. https://bugs.llvm.org/show_bug.cgi?id=32938
* GNU ld relaxes `R_X86_64_GOTPCREL` relocations with some forms (e.g.
  `movq foo@GOTPCREL(%rip), %reg` -&gt; `leaq foo(%rip), %reg`). LLD never
  relaxes `R_X86_64_GOTPCREL` relocations.
* GNU linkers give `.gnu.linkonce*` sections COMDAT section semantics. LLD
  simply ignores such sections. https://bugs.llvm.org/show_bug.cgi?id=31586
  tracks when the hack can be removed.
* GNU ld adds `PT_PHDR` and `PT_INTERP` together. A shared object usually does
  not have two program headers. In LLD, `PT_PHDR` is always added unless the
  address assignment makes is unsuitable to place program headers at all.
* The conditions to create the dynamic symbol table `.dynsym`.
  * LLD: there is an input shared object, `-pie`/`-shared`, or `--export-dynamic`.
  * GNU ld's is quite complex. `--export-dynamic` is not special, though.
* `--export-dynamic-symbol`
  * gold's implies `-u`.
  * GNU ld (from 2.35 onwards) and LLD's do not imply `-u`.
* In GNU ld, a defined `foo@v` can suppress the extraction of an archive member
  defining `foo@@v1`. LLD treats them two separate symbols and thus the archive
  member extraction still happens. This can hardly matter. See [All about symbol
  versioning](maskray-2.md) for details.
* Default program headers.
  * With traditional `-z noseparate-code`, GNU ld defaults to a `RX/R/RW`
    program header layout. With `-z separate-code` (default on Linux/x86 from
    binutils 2.31 onwards), GNU ld defaults to a `R/RX/R/RW` program header
    layout.
  * LLD defaults to `R/RX/RW(RELRO)/RW(non-RELRO)`. With `--rosegment`, LLD
    uses `RX/RW(RELRO)/RW(non-RELRO)`.
  * Placing all R before RX is preferable because it can save one program
    header and reduce alignment costs.
  * LLD's split of RW saves one maxpagesize alignment and can make the linked
    image smaller.
  * This breaks some assumptions that the (so-called) "text segment" precedes
    the (so-called) "data segment".
  * For example, certain programs expect `.text` is the first section of the
    text segment and specify `-Ttext=0` to place the `PF_R|PF_X` program header
    at `p_vaddr=0`. This is a brittle assumption and should be avoided. If
    `PT_PHDR` is needed, `--image-base=0` is a replacement. If `PT_PHDR` is not
    needed, `.text 0 : { *(.text .text.*) }` is a replacement.
* GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in
  `-pie` mode. glibc `csu/libc-start.c` needs it when statically linked, but
  not in the static pie mode. LLD does not distinguish `-no-pie`, `-pie` and
  `-shared`. https://bugs.llvm.org/show_bug.cgi?id=48674
* LLD uses `--no-apply-dynamic-relocs` by default. GNU ld and gold fill in the
  GOT entries with link-time values. GNU ld only supports
  `--no-apply-dynamic-relocs` for aarch64
  https://sourceware.org/bugzilla/show_bug.cgi?id=25891.
* When relaxing `R_X86_64_REX_GOTPCRELX`, GNU ld suppresses the relaxation if
  it would cause relocation overflow. LLD does not perform the check.
* GNU ld and gold allow `--exclude-libs=b` to hide `b.a`. LLD requires
  `--exclude=libs=b.a`.
* Whether to use executable stack if neither `-z execstack` nor `-z noexecstack`
  is specified. GNU ld and gold check whether an object file does not have
  `.note.GNU-stack`. LLD ignores `.note.GNU-stack` and defaults to `-z
  noexecstack`.

## Semantics of `--wrap`

GNU ld and LLD have slightly different `--wrap` semantics. I use "slightly"
because in most use cases users will not observe a difference.

In GNU ld, `--wrap` only applies to undefined symbols. In LLD, `--wrap` happens
after all other symbol resolution steps. The implementation is to mangle the
symbol table of each object file (`foo` -&gt; `__wrap_foo`; `__real_foo` -&gt;
`foo`) so that all relocations to foo or `__real_foo` will be redirected.

The LLD semantics have the advantage that non-LTO, LTO and relocatable link
behaviors are consistent. I filed
https://sourceware.org/bugzilla/show_bug.cgi?id=26358 for GNU ld.

```
# GNU ld: call bar
# LLD: call __wrap_bar
  call bar
.globl bar
bar:
```

## Relocation referencing a local relative to a discarded input section

* How to resolve a relocation referencing a STT_SECTION symbol associated to a
  discarded `.debug_*` input section.
  * GNU ld and gold have logic resolving the relocation to the prevailing
    section symbol.
  *  LLD does not have the logic. LLD 11 defines some tombstone values.

> A symbol table entry with `STB_LOCAL` binding that is defined relative to one
> of a group's sections, and that is contained in a symbol table section that
> is not part of the group, must be discarded if the group members are
> discarded. References to this symbol table entry from outside the group are
> not allowed.

ld.bfd/gold/lld error if the section containing the relocation is `SHF_ALLOC`.
`.debug*` do not have the `SHF_ALLOC` flag and those relocations are allowed.

lld resolves such relocations to 0. ld.bfd and gold, however, have some
`CB_PRETEND`/`PRETEND` logic to resolve relocations to the definitions in the
prevailing comdat groups. The code is hacky and may not suit lld.

https://bugs.llvm.org/show_bug.cgi?id=42030

## Canonical PLT entry for ifunc

How to handle a direct access relocation referencing a `STT_GNU_IFUNC`?

c.f. [GNU indirect function](maskray-6.md).

## `__rela_iplt_start`

GNU ld and gold define `__rela_iplt_start` in `-no-pie` mode, but not in `-pie`
mode.  LLD defines `__rela_iplt_start` regardless of `-no-pie`, `-pie` or
`-shared`.

Static pie and static no-pie relocation processing is very different in glibc.

* Static no-pie uses special code to process a magic array delimitered by
  `__rela_iplt_start`/`__rela_iplt_end`.
* Static pie uses self-relocation to take care of `R_*_IRELATIVE`. The above
  magic array code is executed as well. If `__rela_iplt_start`/`__rela_iplt_end`
  are defined (like what LLD does), we will get
  `0 < __rela_iplt_start < __rela_iplt_end` in `csu/libc-start.c`.
  `ARCH_SETUP_IREL` will crash when resolving the first relocation which has
  been processed.

nsz has a glibc patch that moves the self-relocation later so everything is set up for ifunc resolvers.

## Linker scripts

* Some linker script commands are unimplemented in LLD, e.g. `BLOCK()` as a
  compatibility alias for `ALIGN()`. `BLOCK` is documented in GNU ld as a
  compatibility alias and it is not widely used, so there is no reason to keep
  the kludge in LLD.
* Some syntax is not recognized by LLD, e.g. LLD recognizes
  `*(EXCLUDE_FILE(a.o) .text)` but not `EXCLUDE_FILE(a.o) *(.text)`
  (https://bugs.llvm.org/show_bug.cgi?id=45764)
  * To me the unrecognized syntax is misleading.
  * If we support one way doing something, and the thing has several
    alternative syntax, we may not consider the alternative syntax just for the
    sake of completeness.
* Different orphan section placement. GNU ld has very complex rules and certain
  section names have special semantics. LLD adopted some of its core ideas but
  made a lot of simplication:
  * output sections are given ranks
  * output sections are placed after symbol assignments At some point we should
    document it. https://bugs.llvm.org/show_bug.cgi?id=42327
* For an error detected when processing a linker script, LLD may report it
  multiple times (e.g. `ASSERT` failure). GNU ld has such issues, too, but
  probably much rarer.
* `SORT` commands
  * GNU ld: https://sourceware.org/binutils/docs/ld/Input-Section-Basics.html#Input-Section-Basics
    mentions the feature but its behavior is strange/unintuitive. I created
    `SORT` and multiple patterns in an input section description.
  * LLD performs sorting within an input section description.
    https://reviews.llvm.org/D91127
* In LLD, `AT(lma)` forces creation of a new `PT_LOAD` program header. GNU ld
  can reuse the previous `PT_LOAD` program header if LMA addresses are
  contiguous. `lma-offset.s`
* In LLD, non-`SHF_ALLOC` sections always get 0 `sh_addr`. In GNU ld you can
  have non-zero `sh_addr` but `STT_SECTION` relocations referencing such
  sections are not really meaningful.
* Dot assignment (e.g. `. = 4;`) in an output section description.
  * GNU ld: dot advances to 4 relative to the start. If you consider . on the
    right hand side and `ABSOLUTE(.)`, I don't think the behaviors are
    consistent.
  * LLD: move dot to address 0x4, which will usually trigger an unable to move
    location counter backward error. https://bugs.llvm.org/show_bug.cgi?id=41169

I'll also mention some LLD release notes which can demonstrate some GNU
incompatibility in previous versions. (For example, if one thing is supported
in version N, then the implication is that it is unsupported in previous
versions. Well, it could be that it worked in older versions but regressed at
some version. However, I don't know the existence of such things.)

LLD 12.0.0

* `-r --gc-sections` is supported.
* The archive member extraction semantics of COMMON symbols is by default
  (`--fortran-common`) compatible with GNU ld. You may want to read Semantics
  of a common definition in an archive for details. This is unfortunate.
* `.rel[a].plt` and `.rel[a].dyn` get the `SHF_INFO_LINK` flag. https://reviews.llvm.org/D89828

LLD 11.0.0

* LLD can discard unused symbols with `--discard-all`/`--discard-locals` when
  `-r` or `--emit-relocs` is specified. https://reviews.llvm.org/D77807
* `--emit-relocs --strip-debug` can be used. https://reviews.llvm.org/D74375
* `SHT_GNU_verneed` in shared objects are parsed, and versioned undefined
  symbols in shared objects are respected. Previously non-default version
  symbols could cause spurious `--no-allow-shlib-undefined` errors.
  https://reviews.llvm.org/D80059
* `DF_1_PIE` is set for position-independent executables. https://reviews.llvm.org/D80872
* Better compatibility related to output section alignments and LMA regions.
  [D75286](https://reviews.llvm.org/D75286) [D74297](https://reviews.llvm.org/D74297)
  [D75724](https://reviews.llvm.org/D75725) [D81986](https://reviews.llvm.org/D81986)
* `-r` allows `SHT_X86_64_UNWIND` to be merged into `SHT_PROGBITS`. This allows
  clang/GCC produced object files to be mixed together. https://reviews.llvm.org/D85785
* In a input section description, the filename can be specified in double
  quotes. archive:file syntax is added. https://reviews.llvm.org/D72517 https://reviews.llvm.org/D75100
* Linker script specified empty `(.init|.preinit|.fini)_array` are allowed with
  `RELRO`. https://reviews.llvm.org/D76915

LLD 10.0.0

* LLD supports `\` (treating the next character like a non-meta character) and
  `[!...]` (negation) in glob patterns. https://reviews.llvm.org/D66613

LLD 9.0.0

* The `DF_STATIC_TLS` flag is set for i386 and x86-64 when initial-exec TLS
  models are used.
* Many configurations of the Linux kernel's `arm32_7`, `arm64`, `powerpc64le`
  and `x86_64` ports can be linked by LLD.

LLD 8.0.0

* `SHT_NOTE` sections get very high ranks (they usually precede other
  sections). https://reviews.llvm.org/D55800

In the LLD 7.0.0 era, https://reviews.llvm.org/D44264 was my first meaningful
(albeit trivial) patch to LLD. Next I made contribution to `--warn-backrefs`.
Then I started to fix tricky issues like copy relocations of a versioned
symbol, duplicate `--wrap`, and section ranks. I have learned a lot from these
code reviews. In the 8.0.0, 9.0.0 and 10.0.0 era, I have fixed a number of
tricky issues and improved a dozen of other things and am confident to say that
other than MIPS ;-) and certain other ISA specific things I am familiar with
every corner of the code base. These are still challenges such as integration
of RISC-V style linker relaxation and post-link optimization, improvement to
some aspects of the linker script, but otherwise LLD is a stable and finished
part of the toolchain.

A few random notes:

* Symbol resolution can take 10%~20% time. Parallelization can theoretically
  improve the process but it is hard to overstate the challenge (if you
  additionally take into account determinism).
* Be wary of feature creep. I have learned a lot from ELF design discussions
  on generic-abi and from Solaris "linker aliens" in particular. I am sorry to
  say so but some development on LLD indeed belongs to such categories.
  Sometimes it is difficult to draw a line between unsupported legacy and
  legacy we have to support.
* LLD's adoption is now so large that sometimes a decision (like a default
  value for an option) cannot make everyone happy.

