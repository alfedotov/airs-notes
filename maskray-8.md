# Metadata sections, COMDAT and `SHF_LINK_ORDER`

## COMDAT

In C++, inline functions, template instantiations and a few other things can be
defined in multiple object files but need deduplication at link time. In the
dark ages the functionality was implemented by weak definitions: the linker
does not report duplicate definition errors and resolves the references to the
first definition. The downside is that unneeded copies remained in the linked
image.

In Microsoft PE file format, the section flag (`IMAGE_SCN_LNK_COMDAT`) marks a
section COMDAT and enables deduplication on a per-section basis. If a text
section needs a data section and deduplication is needed for both sections, two
COMDAT symbols are needed.

In the GNU world, `.gnu.linkonce.` was invented to duplicate groups with just
one member. `.gnu.linkonce.` has been long obsoleted in favor of section groups
but the usage has been daunting til 2020. Adhemerval Zanella removed the the
last live glibc use case for `.gnu.linkonce.`
[BZ #20543](http://sourceware.org/PR20543).

## ELF section groups

The ELF specification generalized this use case to allow an arbitrary number of
groups to be interrelated.

> Some sections occur in interrelated groups. For example, an out-of-line
> definition of an inline function might require, in addition to the section
> containing its executable instructions, a read-only data section containing
> literals referenced, one or more debugging information sections and other
> informational sections. Furthermore, there may be internal references among
> these sections that would not make sense if one of the sections were removed
> or replaced by a duplicate from another object. Therefore, such groups must
> be included or omitted from the linked object as a unit. A section cannot be
> a member of more than one group.

According to "such groups must be included or omitted from the linked object as
a unit", a linker's garbage collection feature must retain or discard the
sections as a unit.

The most common section group flag is `GRP_COMDAT`, which makes the member
sections similar to COMDAT in Microsoft PE file format, but can apply to
multiple sections. (The committee borrowed the name "COMDAT" from PE.)

> This is a COMDAT group. It may duplicate another COMDAT group in another
> object file, where duplication is defined as having the same group signature.
> In such cases, only one of the duplicate groups may be retained by the
> linker, and the members of the remaining groups must be discarded.

I want to highlight one thing GCC does (and Clang inherits) for backward
compatibility: the definitions relatived to a COMDAT group member are kept
`STB_WEAK` instead of `STB_GLOBAL`. The idea is that old toolchain which does
not recognize COMDAT groups can still operate correctly, just in a degraded
manner.

## Metadata sections

Many compiler options intrument text sections or annotate text sections, and
need to create a metadata section for (almost) every text section. Such
metadata sections have some characteristics:

* All relocations from the metadata section reference the associated text
  section.
* The metadata section is only referenced by the associated text section or not
  referenced at all.

Below is an example:

```
.section .text.foo,"ax",@progbits

.section .meta.foo,"a",@progbits
.quad .text.foo-.
```

Users want GC semantics for such metadata sections: if `.text.foo` is retained,
`.meta.foo` is retained. Note: the regular GC semantics are converse: if
`.meta.foo` is retained, `.text.foo` is retained.

To achieve the desired GC semantics on ELF platforms, we could use a non-COMDAT
section group. However, using a section group requires one extra section
(usually named `.group`), which requires 40 bytes on ELFCLASS32 platforms and
64 bytes on ELFCLASS64 platforms. Put it in another way, to represent the
metadata of a text section, we need two sections (the metadata section and the
section group), 128 bytes on ELFCLASS64 platforms. The size overhead is
concerning in many applications. (AArch64 and x86-64 define ILP32 ABIs and use
ELFCLASS32, but technically they can use ELFCLASS32 for small code model with
regular ABIs, if the kernel allows.)

In a generic-abi thread, Cary Coutant initially suggested to use a new section
flag `SHF_ASSOCIATED`. HP-UX and Solaris folks objected to a new generic flag.
Cary Coutant then discussed with Jim Dehnert and noticed that the existing
(rare) flag `SHF_LINK_ORDER` has semantics closer to the metadata GC semantics,
so he intended to replace the existing flag `SHF_LINK_ORDER`. Solaris had used
its own `SHF_ORDERED` extension before it migrated to the ELF simplification
`SHF_LINK_ORDER`. Solaris is still using `SHF_LINK_ORDER` so the flag cannot be
repurposed. People discussed whether `SHF_OS_NONCONFORMING` could be repurposed
but did not take that route: the platform already knows whether a flag is
unknown and knowing a flag is non-conforming does not help produce better
output. In the end the agreement was that `SHF_LINK_ORDER` gained additional
metadata GC semantics.

The new semantics:

> This flag adds special ordering requirements for link editors. The
> requirements apply to the referenced section identified by the sh_link field
> of this section's header. If this section is combined with other sections in
> the output file, the section must appear in the same relative order with
> respect to those sections, as the referenced section appears with respect to
> sections the referenced section is combined with.
>
> A typical use of this flag is to build a table that references text or data
> sections in address order.
>
> In addition to adding ordering requirements, `SHF_LINK_ORDER` indicates that
> the section contains metadata describing the referenced section. When
> performing unused section elimination, the link editor should ensure that
> both the section and the referenced section are retained or discarded
> together. Furthermore, relocations from this section into the referenced
> section should not be taken as evidence that the referenced section should be
> retained.

Actually, ARM EHABI has been using `SHF_LINK_ORDER` for index table sections
`.ARM.exidx*`. A `.ARM.exidx` section contains a sequence of 2-word pairs. The
first word is 31-bit PC-relative offset to the start of the region. The idea is
that if the entries are ordered by the start address, the end address of an
entry is implicitly the start address of the next entry and does not need to be
explicitly encoded. For this reason the section uses `SHF_LINK_ORDER` for the
ordering requirement. The GC semantics are very similar to the metadata
sections'.

So the updated `SHF_LINK_ORDER` wording can be seen as recognition for the
current practice (even though the original discussion did not actually notice
ARM EHABI).

However, in binutils, before 2.35, `SHF_LINK_ORDER` could be produced by ARM
assembly directives, but not specified by user-customized sections.

## C identifier name sections

A section whose name consists of pure C-like identifier characters (isalnum
characters in the C locale plus `_`) is considered as a GC root by ld
`--gc-sections`. The idea is that linker defined `__start_foo` and `__stop_foo`
are used to delimiter the output section foo. Even if input sections foo are
not referenced by other sections, `__start_foo`/`__stop_foo` is a signal that
foo should be retained.

The metadata use case requires an amendment of the rule: if `SHF_LINK_ORDER` is
set on foo, foo can be GCed (LLD r294592).

GNU ld does not implement this rule yet. https://sourceware.org/bugzilla/show_bug.cgi?id=27259

## Pitfalls

### Mixed unordered and ordered sections

If an output section consists of only non-`SHF_LINK_ORDER` sections, the rule is
clear: input sections are ordered in their input order. If an output section
consists of only `SHF_LINK_ORDER` sections, the rule is also clear: input
sections are ordered with respect to their linked-to sections.

What is unclear is how to handle an output section with mixed unordered and
ordered sections.

GNU ld had a diagnostic: . LLD rejected the case as well error:
`incompatible section flags for .rodata`.

When I implemented `-fpatchable-function-entry=` for Clang, I observed some GC
related issues with the GCC implementation. I reported them and carefully chose
`SHF_LINK_ORDER` in the Clang implementation if the integrated assembler is
used.

This was a problem if the user wanted to place such input sections along with
unordered sections, e.g.
`.init.data : { ... KEEP(*(__patchable_function_entries)) ... }`
(https://github.com/ClangBuiltLinux/linux/issues/953).

As a response, I submitted https://reviews.llvm.org/D77007 to allow ordered
input section descriptions within an output section.

This worked well for the Linux kernel. Mixed unordered and ordered sections
within an input section description was still a problem. This made it
infeasible to add `SHF_LINK_ORDER` to an existing metadata section and expect
new object files linkable with old object files which do not have the flag. I
asked how to resolve this upgrade issue and Ali Bahrami responded:

> The Solaris linker puts sections without `SHF_LINK_ORDER` at the end of the
> output section, in first-in-first-out order, and I don't believe that's
> considered to be an error.

So I went ahead and implemented a similar rule for LLD:
https://reviews.llvm.org/D84001 allowes arbitrary mix and places
`SHF_LINK_ORDER` sections before non-`SHF_LINK_ORDER` sections.

### If the associated section is discarded

We decided that the integrated assembler allows `SHF_LINK_ORDER` with
`sh_link=0` and LLD can handle such sections as regular unordered sections
(https://reviews.llvm.org/D72904).

### Other pitfalls

* During `--icf={safe,all}`, `SHF_LINK_ORDER` sections should not be separately
  considered.
* In relocatable output, `SHF_LINK_ORDER` sections cannot be combined by name.
* When comparing two input sections with different linked-to output sections,
  use vaddr of output sections instead of section indexes. Peter Smith fixed
  this in https://reviews.llvm.org/D79286.

## Miscellaneous

Arm Compiler 5 splits up DWARF Version 3 debug information and puts these
sections into comdat groups. On "monolithic input section handling", Peter
Smith commented that:

> We found that splitting up the debug into fragments works well as it permits
> the linker to ensure that all the references to local symbols are to sections
> within the same group, this makes it easy for the linker to remove all the
> debug when the group isn't selected.
>
> This approach did produce significantly more debug information than gcc did.
> For small microcontroller projects this wasn't a problem. For larger feature
> phone problems we had to put a lot of work into keeping the linker's memory
> usage down as many of our customers at the time were using 32-bit Windows
> machines with a default maximum virtual memory of 2Gb.

COMDAT sections have size overhead on extra section headers. Developers may be
tempted to decrease the overhead with `SHF_LINK_ORDER`. However, the approach
does not work due to the ordering requirement. Considering the following
fragments:

```
header [a.o common]
- DW_TAG_compile_unit [a.o common]
-- DW_TAG_variable [a.o .data.foo]
-- DW_TAG_namespace [common]
--- DW_TAG_subprogram [a.o .text.bar]
--- DW_TAG_variable [a.o .data.baz]
footer [a.o common]
header [b.o common]
- DW_TAG_compile_unit [b.o common]
-- DW_TAG_variable [b.o .data.foo]
-- DW_TAG_namespace [common]
--- DW_TAG_subprogram [b.o .text.bar]
--- DW_TAG_variable [b.o .data.baz]
footer [b.o common]
```

`DW_TAG_*` tags associated with concrete sections can be represented with
`SHF_LINK_ORDER` sections. After linking the sections will be ordered before the
common parts.

