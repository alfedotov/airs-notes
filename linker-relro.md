# Linker relro

gcc, the GNU linker, and the glibc dynamic linker cooperate to implement an
idea called read-only relocations, or relro. This permits the linker to
designate a part of an executable or (more commonly) a shared library as being
read-only after dynamic relocations have been applied.

This may be used for read-only global variables which are initialized to
something which requires a relocation, such as the address of a function or a
different global variable. Because the global variable requires a runtime
initialization in the form of a dynamic relocation, it can not be placed in a
read-only segment. However, because it is declared to be constant, and
therefore may not be changed by the program, the dynamic linker can mark it as
read-only after the dynamic relocation has been applied.

For some targets this technique may also be used for the PLT or parts of the
GOT.

Making these pages read-only helps catch some cases of memory corruption, and
making the PLT in particular read-only helps prevent some types of buffer
overflow exploits.

The first step is in gcc. When gcc sees a variable which is constant but
requires a dynamic relocation, it puts it into a section named `.data.rel.ro`
(this functionality unfortunately relies on magic section names). A variable
which requires a dynamic relocation against a local symbol is put into a
`.data.rel.ro.local` section; this helps group such variables together, so that
the dynamic linker may apply the relocations, which will always be `RELATIVE`
relocations, more efficiently, especially when using `combreloc`.

The linker groups `.data.rel.ro` and `.data.rel.ro.local` sections as usual.
The new step is that the linker then emits a `PT_GNU_RELRO` program segment
which covers these sections. If the PLT and/or GOT can be read-only after
dynamic relocations, they are put next to the `.data.rel.ro` sections and also
become part of the new segment. This segment will enclosed within a `PT_LOAD`
segment.  The `p_vaddr` field of the `PT_GNU_RELRO` segment gives the virtual
address of the start of the read-only after dynamic relocations code, and the
`p_memsz` field gives its length.

When the dynamic linker sees a `PT_GNU_RELRO` segment, it uses mprotect to mark
the pages as read-only after the dynamic relocations have been applied. Of
course this only works if the segment does in fact cover an entire page. The
linker will try to force this to happen.

Note that the current dynamic linker code will only work correctly if the
`PT_GNU_RELRO` segment starts on a page boundary. This is because the dynamic
linker rounds the `p_vaddr` field down to the previous page boundary. If there is
anything on the page which should not be read-only, the program is likely to
fail at runtime. So in effect the linker must only emit a `PT_GNU_RELRO`
segment if it ensures that it starts on a page boundary.

I see this as a relatively minor security benefit. It is not an optimization as
far as I can see. I am documenting it here as part of my general documentation
of obscure linker features. The current description of this feature in the GNU
linker manual is rather obscure.

