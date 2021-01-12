# Linker combreloc

The GNU linker has a `-z combreloc` option, which is enabled by default (it can
be turned off via `-z nocombreloc`). I just implemented this in gold as well.
This option directs the linker to sort the dynamic relocations. The sorting is
done in order to optimize the dynamic linker.

The dynamic linker in glibc uses a one element cache when processing relocs: if
a relocation refers to the same symbol as the previous relocation, then the
dynamic linker reuses the value rather than looking up the symbol again. Thus
the dynamic linker gets the best results if the dynamic relocations are sorted
so that all dynamic relocations for a given dynamic symbol are adjacent.

Other than that, the linker sorts together all relative relocations, which
don’t have symbols. Two relative relocations, or two relocations against the
same symbol, are sorted by the address in the output file. This tends to
optimize paging and caching when there are two references from the same page.

This may seem like a micro-optimization, but it can have a real effect on
program startup time, especially if the program has lots of shared libraries.
I’ve seen a case where a program starts up 16% faster because the relocations
were sorted.

