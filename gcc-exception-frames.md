# GCC Exception Frames

When an exception is thrown in C++ and caught by one of the calling functions,
the supporting libraries need to unwind the stack. With gcc this is done using
a variant of DWARF debugging information. The unwind information is loaded at
runtime, but is not read unless an exception is thrown. That means that the
unwind library needs to have some way of finding the appropriate unwind
information at runtime.

On some systems, this is done by registering the exception frame information
when the program starts. The registration is done with a variant of the
handling of C++ constructors. This becomes interesting when one shared library
can throw an exception which is caught by another shared library. It is
possible for such a case to arise when the executable itself never throws
exceptions and therefore has no frames to register. Obviously the unwinder
needs to be able to find the unwind information for both shared libraries,
which means that both shared libraries need to use the same registration
functions. With gcc this is normally ensured by putting the unwind code in a
shared library, `libgcc_s.so`. Each shared library, and sometimes the
executable, will use `libgcc_s.so`. That ensures a single copy of the
registration and unwind functions, so the library will be able to reliably
unwind across shared libraries. With gcc the use of `libgcc_s.so` can be
controlled with the `-shared-libgcc` and `-static-libgcc` options. Normally the
right thing will happen by default.

That approach has a cost: there is an extra shared library, and there is a
small cost of registering the unwind information at program startup or library
load time (and unregistering it if a shared library is unloaded via dlclose).
There is now a better way, which requires linker support.

Both gold and the GNU linker support the command line option `--eh-frame-hdr`.
With this option, when the linker sees the `.eh_frame` sections used to hold
the unwind information, it automatically builds a header. This header is a
sorted array mapping program counter addresses to unwind information. The
header is recorded as a program segment of type `PT_GNU_EH_FRAME`. (This is a
little bit ugly since the `.eh_frame` sections are recognized only by name;
ideally they should have a special section type.)

At runtime, the unwind library can use the `dl_iterate_phdr` function to find
the program segments of the executable and all currently loaded shared
libraries.  It can use that to find the `PT_GNU_EH_FRAME` segments, and use the
sorted array in those segments to quickly find the unwind information.

This approach means that no registration functions are required. It also means
that it is not necessary to have a single shared library, since
`dl_iterate_phdr` is available no matter which shared library throws the
exception.

This all only works if you have a linker which supports generating
`PT_GNU_EH_FRAME` sections, if all the shared libraries and the executable are
linked by such a linker, and if you have a working `dl_iterate_phdr` function
in your C library or dynamic linker. I think that pretty much restricts this
approach to GNU/Linux and possibly other free operating systems. For those
scenarios, I hope that gcc will soon be able to stop using `libgcc_s.so` by
default.

