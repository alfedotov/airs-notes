# airs-notes

## Source

https://www.airs.com/blog/index.php?s=linkers+part

Authored and copyright by Ian Lance Taylor, collected here fore easy lookup.

## Index

### Main series

* [Linkers part 1: introduction](linkers-1.md)
* [Linkers part 2: technial introduction](linkers-2.md)
* [Linkers part 3: address spaces, object file formats](linkers-3.md)
* [Linkers part 4: shared libraries](linkers-4.md)
* [Linkers part 5: shared libraries redux, ELF symbols](linkers-5.md)
* [Linkers part 6: relocations, position-dependent libraries](linkers-6.md)
* [Linkers part 7: thread-local storage](linkers-7.md)
* [Linkers part 8: ELF segments and sections](linkers-8.md)
* [Linkers part 9: symbol versions, relaxation](linkers-9.md)
* [Linkers part 10: parallel linking](linkers-10.md)
* [Linkers part 11: archives](linkers-11.md)
* [Linkers part 12: symbol resolution](linkers-12.md)
* [Linkers part 13: symbol versions redux](linkers-13.md)
* [Linkers part 14: link-time optimization, initialization code](linkers-14.md)
* [Linkers part 15: COMDAT sections](linkers-15.md)
* [Linkers part 16: C++ template instantiation, exception frames](linkers-16.md)
* [Linkers part 17: warning symbols](linkers-17.md)
* [Linkers part 18: incremental linking](linkers-18.md)
* [Linkers part 19: `__start` and `__stop` symbols, byte swapping](linkers-19.md)
* [Linkers part 20: ending note](linkers-20.md)

### Other articles included as well

* [GCC exception frames](gcc-exception-frames.md)
* [Linker combreloc](linker-combreloc.md)
* [Linker relro](linker-relro.md)
* [Combining versions](combining-versions.md)
* [Version scripts](version-scripts.md)
* [Protected symbols](protected-symbols.md)
* [`.eh_frame`](eh_frame.md)
* [`.eh_frame_hdr`](eh_frame_hdr.md)
* [`.gcc_except_table`](gcc_except_table.md)
* [Executable stack](executable-stack.md)
* [Piece of PIE](piece-of-pie.md)

### Even more articles, from [MaskRay's blog](https://maskray.me/blog/)

* [Stack unwinding](maskray-1.md)
* [All about symbol versioning](maskray-2.md)
* [C++ exception handling ABI](maskray-3.md)
* [LLD and GNU linker incompatibilities](maskray-4.md)
* [Copy relocations, canonical PLT entries and protected visibility](maskray-5.md)
* [GNU indirect function](maskray-6.md)
* [Everything I know about GNU toolchain](maskray-7.md)
* [Metadata sections, COMDAT and `SHF_LINK_ORDER`](maskray-8.md)

## External links

Here's a collection of links about the subject, I'm putting these here because
people seem to find these useful.

* [`elf(5)` manpage](https://linux.die.net/man/5/elf)
* [unofficial ELF docs](elf.md) (has
  more than the manpage, also has extra links)
* [glibc internals](http://s.eresi-project.org/inc/articles/elf-rtld.txt)
* [stuff about `.gnu.hash`](https://web.archive.org/web/20111022202443/http://blogs.oracle.com/ali/entry/gnu_hash_elf_sections)
* [Linux Internals &amp; Dynamic Linking Wizardy](https://0x00sec.org/t/linux-internals-dynamic-linking-wizardry/1082);
  [Linux Internals: The Art of Symbol Resolution](https://0x00sec.org/t/linux-internals-the-art-of-symbol-resolution/1488)
* [LWN article on the vDSO](https://lwn.net/Articles/446528/), the vDSO is a
  dynamic library automatically inserted into every process by the kernel,
  containing fast implementations of a few syscalls
* ["The bits between the bits: how we get to `main()`"](https://invidious.snopyta.org/watch?v=dOfucXtyEsU),
  a talk, mostly talks about process instantiation
* [Everything You Always Wanted to Know About "Hello, World"](https://archive.fosdem.org/2017/schedule/event/hello_world/),
  another talk
* "How programs get run" (LWN), [part 1](https://lwn.net/Articles/630727/),
  [part 2](https://lwn.net/Articles/631631/), also talks about eg. initial
  stack layout
* [Glibc startup procedure](https://www.gnu.org/software/hurd/glibc/startup.html),
  technically hurd but mostly the same on Linux
* [The difference between all the `crt*.o` files](https://dev.gentoo.org/%7Evapier/crt.txt)
* [Official x86-64 ELF ABI spec](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf) (PDF)
* [Official ELF Auxiliary vector reference](https://refspecs.linuxfoundation.org/LSB_1.3.0/IA64/spec/auxiliaryvector.html)
  * Lots of useful stuff can be found on this website as well, see eg.
    [here](https://refspecs.linuxfoundation.org/) and [here](https://refspecs.linuxfoundation.org/LSB_1.3.0/)
* [Glibc ld.so source code](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=elf/dl-lookup.c)
* [GNU ld source code](https://sourceware.org/git/?p=binutils.git;a=blob_plain;f=bfd/elf.c)
* [Glibc How to debug ld.so](https://sourceware.org/glibc/wiki/Debugging/Loader_Debugging)
* Anatomy of a system call, LWN, [part 1](https://lwn.net/Articles/604287/),
  [part 2](https://lwn.net/Articles/604515/)
* ["Linkers and Loaders"](http://becbapatla.ac.in/cse/naveenv/docs/LL1.pdf),
  a book (PDF), more meant for general concepts than nitty ELF details. Also
  talks about Java for some reason.
* [A Whirlwind Tutorial on Creating Really Teensy ELF Executables for
  Linux](http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)
* [The Quest for minimal ELF binaries](https://github.com/faemiyah/dnload/blob/master/README.rst#the-quest-for-minimal-elf-binaries)
* [Linux sizecoding wiki](https://linux.weeaboo.software/explain-dot-md)
* [Rough transcriptions of a thread on Mastodon](masto-thread.md), my posts had
  some useful info in them, so I saved these in this document.

