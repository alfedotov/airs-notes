# Linkers part 8

## ELF Segments

Earlier I said that executable file formats were normally the same as object
file formats. That is true for ELF, but with a twist. In ELF, object files are
composed of sections: all the data in the file is accessed via the section
table. Executables and shared libraries normally contain a section table, which
is used by programs like `nm`. But the operating system and the dynamic linker
do not use the section table. Instead, they use the segment table, which
provides an alternative view of the file.

All the contents of an ELF executable or shared library which are to be loaded
into memory are contained within a segment (an object file does not have
segments). A segment has a type, some flags, a file offset, a virtual address,
a physical address, a file size, a memory size, and an alignment. The file
offset points to a contiguous set of bytes which are the contents of the
segment, the bytes to load into memory. When the operating system or the
dynamic linker loads a file, it will do so by walking through the segments and
loading them into memory (typically by using the mmap system call). All the
information needed by the dynamic linker–the dynamic relocations, the dynamic
symbol table, etc.–are accessed via information stored in special segments.

Although an ELF executable or shared library does not, strictly speaking,
require any sections, they normally do have them. The contents of a loadable
section will fall entirely within a single segment.

The program linker reads sections from the input object files. It sorts and
concatenates them into sections in the output file. It maps all the loadable
sections into segments in the output file. It lays out the section contents in
the output file segments respecting alignment and access requirements, so that
the segments may be mapped directly into memory. The sections are mapped to
segments based on the access requirements: normally all the read-only sections
are mapped to one segment and all the writable sections are mapped to another
segment. The address of the latter segment will be set so that it starts on a
separate page in memory, permitting `mmap` to set different permissions on the
mapped pages.

The segment flags are a bitmask which define access requirements. The defined
flags are `PF_R`, `PF_W`, and `PF_X`, which mean, respectively, that the
contents must be made readable, writable, or executable.

The segment virtual address is the memory address at which the segment contents
are loaded at runtime. The physical address is officially undefined, but is
often used as the load address when using a system which does not use virtual
memory. The file size is the size of the contents in the file. The memory size
may be larger than the file size when the segment contains uninitialized data;
the extra bytes will be filled with zeroes. The alignment of the segment is
mainly informative, as the address is already specified.

The ELF segment types are as follows:

* `PT_NULL`: A null entry in the segment table, which is ignored.
* `PT_LOAD`: A loadable entry in the segment table. The operating system or
  dynamic linker load all segments of this type. All other segments with
  contents will have their contents contained completely within a `PT_LOAD`
  segment.
* `PT_DYNAMIC`: The dynamic segment. This points to a series of dynamic tags
  which the dynamic linker uses to find the dynamic symbol table, dynamic
  relocations, and other information that it needs.
* `PT_INTERP`: The interpreter segment. This appears in an executable. The
  operating system uses it to find the name of the dynamic linker to run for
  the executable. Normally all executables will have the same interpreter name,
  but on some operating systems different interpreters are used in different
  emulation modes.
* `PT_NOTE`: A note segment. This contains system dependent note information
  which may be used by the operating system or the dynamic linker. On
  GNU/Linux systems shared libraries often have a ABI tag note which may be
  used to specify the minimum version of the kernel which is required for the
  shared library. The dynamic linker uses this when selecting among different
  shared libraries.
* `PT_SHLIB`: This is not used as far as I know.
* `PT_PHDR`: This indicates the address and size of the segment table. This is
  not too useful in practice as you have to have already found the segment
  table before you can find this segment.
* `PT_TLS`: The TLS segment. This holds the initial values for TLS variables.
* `PT_GNU_EH_FRAME` (`0x6474e550`): A GNU extension used to hold a sorted table
  of unwind information. This table is built by the GNU program linker. It is
  used by gcc’s support library to quickly find the appropriate handler for an
  exception, without requiring exception frames to be registered when the
  program starts.
* `PT_GNU_STACK` (`0x6474e551`): A GNU extension used to indicate whether the
  stack should be executable. This segment has no contents. The dynamic linker
  sets the permission of the stack in memory to the permissions of this segment.
* `PT_GNU_RELRO` (`0x6474e552`): A GNU extension which tells the dynamic linker
  to set the given address and size to be read-only after applying dynamic
  relocations. This is used for const variables which require dynamic
  relocations.

## ELF Sections

Now that we’ve done segments, lets take a quick look at the details of ELF
sections. ELF sections are more complicated than segments, in that there are
more types of sections. Every ELF object file, and most ELF executables and
shared libraries, have a table of sections. The first entry in the table,
section 0, is always a null section.

ELF sections have several fields.

* Name.
* Type. I discuss section types below.
* Flags. I discuss section flags below.
* Address. This is the address of the section. In an object file this is
  normally zero. In an executable or shared library it is the virtual address.
  Since executables are normally accessed via segments, this is essentially
  documentation.
* File offset. This is the offset of the contents within the file.
* Size. The size of the section.
* Link. Depending on the section type, this may hold the index of another
  section in the section table.
* Info. The meaning of this field depends on the section type.
* Address alignment. This is the required alignment of the section. The program
  linker uses this when laying out the section in memory.
* Entry size. For sections which hold an array of data, this is the size of one
  data element.

These are the types of ELF sections which the program linker may see.

* `SHT_NULL`: A null section. Sections with this type may be ignored.
* `SHT_PROGBITS`: A section holding bits of the program. This is an ordinary
  section with contents.
* `SHT_SYMTAB`: The symbol table. This section actually holds the symbol table
  itself. The section contents are an array of ELF symbol structures.
* `SHT_STRTAB`: A string table. This type of section holds null-terminated
  strings. Sections of this type are used for the names of the symbols and the
  names of the sections themselves.
* `SHT_RELA`: A relocation table. The link field holds the index of the section
  to which these relocations apply. These relocations include addends.
* `SHT_HASH`: A hash table used by the dynamic linker to speed symbol lookup.
* `SHT_DYNAMIC`: The dynamic tags used by the dynamic linker. Normally the
  `PT_DYNAMIC` segment and the `SHT_DYNAMIC` section will point to the same
  contents.
* `SHT_NOTE`: A note section. This is used in system dependent ways. A loadable
  `SHT_NOTE` section will become a `PT_NOTE` segment.
* `SHT_NOBITS`: A section which takes up memory space but has no associated
  contents. This is used for zero-initialized data.
* `SHT_REL`: A relocation table, like `SHT_RELA` but the relocations have no
  addends.
* `SHT_SHLIB`: This is not used as far as I know.
* `SHT_DYNSYM`: The dynamic symbol table. Normally the `DT_SYMTAB` dynamic tag
  will point to the same contents as this section (I haven’t discussed dynamic
  tags yet, though).
* `SHT_INIT_ARRAY`: This section holds a table of function addresses which
  should each be called at program startup time, or, for a shared library, when
  the library is opened by `dlopen`.
* `SHT_FINI_ARRAY`: Like `SHT_INIT_ARRAY`, but called at program exit time or
  `dlclose` time.
* `SHT_PREINIT_ARRAY`: Like `SHT_INIT_ARRAY`, but called before any shared
  libraries are initialized. Normally shared libraries initializers are run
  before the executable initializers. This section type may only be linked into
  an executable, not into a shared library.
* `SHT_GROUP`: This is used to group related sections together, so that the
  program linker may discard them as a unit when appropriate. Sections of this
  type may only appear in object files. The contents of this type of section
  are a flag word followed by a series of section indices.
* `SHT_SYMTAB_SHNDX`: ELF symbol table entries only provide a 16-bit field for
  the section index. For a file with more than 65536 sections, a section of
  this type is created. It holds one 32-bit word for each symbol. If a symbol’s
  section index is `SHN_XINDEX`, the real section index may be found by looking
  in the `SHT_SYMTAB_SHNDX` section.
* `SHT_GNU_LIBLIST` (`0x6ffffff7`): A GNU extension used by the prelinker to
  hold a list of libraries found by the prelinker.
* `SHT_GNU_verdef` (`0x6ffffffd`): A Sun and GNU extension used to hold version
  definitions (I’ll take about symbol versions at some point).
* `SHT_GNU_verneed` (`0x6ffffffe`): A Sun and GNU extension used to hold
  versions required from other shared libraries.
* `SHT_GNU_versym` (`0x6fffffff`): A Sun and GNU extension used to hold the
  versions for each symbol.

These are the types of section flags.

* `SHF_WRITE`: Section contains writable data.
* `SHF_ALLOC`: Section contains data which should be part of the loaded program
  image. For example, this would normally be set for a `SHT_PROGBITS` section
  and not set for a `SHT_SYMTAB` section.
* `SHF_EXECINSTR`: Section contains executable instructions.
* `SHF_MERGE`: Section contains constants which the program linker may merge
  together to save space. The compiler can use this type of section for
  read-only data whose address is unimportant.
* `SHF_STRINGS`: In conjunction with `SHF_MERGE`, this means that the section
  holds null terminated string constants which may be merged.
* `SHF_INFO_LINK`: This flag indicates that the info field in the section holds
  a section index.
* `SHF_LINK_ORDER`: This flag tells the program linker that when it combines
  sections, this section must appear in the same relative order as the section
  in the link field. This can be used to ensure that address tables are built
  in the expected order.
* `SHF_OS_NONCONFORMING`: If the program linker sees a section with this flag,
  and does not understand the type or all other flags, then it must issue an
  error.
* `SHF_GROUP`: This section appears in a group (see `SHT_GROUP`, above).
* `SHF_TLS`: This section holds TLS data.

