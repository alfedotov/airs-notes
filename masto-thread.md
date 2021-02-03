# Rough transcriptions of a thread on Mastodon

Here are some useful parts of posts I made on Mastodon, I haven't cleaned
them up too much.

## General structure of an ELF file

An ELF file starts with an ELF header (Ehdr), which contains offsets to the
program headers aka segments (Phdr), and the section headers (Shdr). also tells
you the entry point, architecture+bitsize, and which shdr is `.shstrtab`.

Shdrs and phdrs are explained [here](linkers-8.md). Both provide views on the
ELF file, but for different purposes.  Though its kinda not a good idea, I'll
give you that.

The stuff pointed to by the shdrs and phdrs are:

* `text`, `data`, `rodata`, `bss`, ... blobs
* string table blobs (`strtab`, `shstrtab`, `dynstr`)
* interpreter, comment, etc etc
* relocation tables (`Rel`, `Rela`) (phdrs don't know about this one)
* symbol tables (`Sym`) (phdrs don't know about this one)
* versioning info (`Versym`, `Verdef`, `Verneed`) (phdrs don't know about this one)
* dynamic table (Dyn), which also has entries for relocations, symbols, versioning

Yes, its true that phdrs, shdrs, dynamic, symtab, dynsym, ... could've just been
tables right after the Ehdr, but that was apparently not complicated enough for
Sun.

## What are all the different sections for?

`.hash, .gnu.hash`: hash tables for looking up symbols. both do the same but
are slihgtly different in implementation, `.hash` comes from SysV R4 and has
been deprecated for ages, no clue why its still there. `.gnu.hash` is made by
the GNU people because they thought the SysV one wasnt good enough.

`.comment` is just a string the toolchain inserts to tell people its built with
the toolchain, for some reason.

`.shstrtab` is the blob that contains the section names (so the actual ".text",
".data", ... strings), for some reason (elaborated on later) this is stored
separately from the other string tables (the '`sh_name`' field of an
`ElfXX_Shdr` is an offset into this table).

`.rel*` and `.rela*` contain relocation info, used both during
static/"compile-time" linking and runtime/dynamic linking. binaries contain
only runtime relocation info, linkable objects contain only static linking info
(the linker has to figure out which symbols and relocations need to get truned
into dynamic ones).

`.gnu.version` and `.gnu.version_r` contain versioning information of symbols,
glibc uses this a lot, and practically nothing else

There's also `.debug*` and `.dwarf*` stuff for debug info, that's yet another
rabbithole im *not* going into this time.

Usually, an ELF binary (not a non-linked object) has two symbol tables,
`.symtab` and `.dynsym`. the former contains all the 'internal' symbols (the
part you can strip away), the latter are the imported and exported ones

`.strtab` contains the symbol name strings, the `st_name` of the `ElfXX_Sym`
entries in `.symtab` is again an offset, `.dynstr` contains the names of the
names of the `.dynsym` entries

However, section headers don't actually have to be present at all in binaries
(executables *and* libraries), only in linkable object files. you can just, get
rid of them completely (patch out the shdr-related fields in the ELF header),
and things still work, which is why and how you can get rid of the `.symtab`,
`.strtab`, `.shstrtab`, etc (and the shdr table itself), and thats also why all
the string tables are separate.

But how would ld.so find `.dynsym` etc. if the shdrs that point to them are
gone?

That's where `.dynamic` is for: it contains a bunch of only half-related
offsets of the file into a table: a list of library dependencies, offsets into
the `.dynsym`, `.dynstr`, `.gnu.version`, `.rel(a)`, ... tables, misc flags and
settings, and so on (the entries are key/value pairs, see `ElfXX_Dyn`).

But then how does ld.so find `.dynamic`?

That's what the phdrs are for (not). Originally, those are meant for the kernel
to see where in memory an executable needs to be mapped, with offset+address,
alignment, permission, ... info. But as that's the table the kernel looks at,
that's also where they added the info about which interpreter should be used for
the binary, whether the stack should be mapped NX, and so on. there's also one
containing the offset of the `.dynamic` table. the kernel doesnt touch it, but
thats how ld.so can reliably find it.

The thing is, you can have most things "gone" by removing all the sections, but
many of these will still actually be present because they have an entry in the
`.dynamic` table. which is not very useful. so if you want to get rid of some
stuff (hash tables, versioning info, ...), you'll first have to remove the
entries from the dyn table, and only *then* remove the relevant shdrs, as that
will properly remove it from the binary

Then you can nuke the shdr table itself using a tool like `sstrip` (usually
packaged in `elf-kickers` or `elfkickers` or ...), binutils/objcopy won't let
you do this.

And thats why, if you want a *small* output file, you want to either write the
ELF headers manually, or use/write a custom linker that doesn't emit all this
stuff.

## Random notes

* `ld.so` has to be linked with `-static-pie`
* All symbol tables must start with a zeroed-out entry, because the standard
  says that symbol index 0 (when referencing a symbol elsewhere) means no
  symbol, instead of index -1 or so. It's not a sentinel value.
* ld.so will use the hash tables first to look up symbols that are defined in
  the binary before resorting to walking the symbol table manually. It probably
  actually needs at least one of these two to be present in a binary nowdays.
  `.hash` is provided as a fallback for when ld.so wouldn't know about
  `.gnu.hash`, but that practically never happens.
