# Piece of PIE

Modern ELF systems can randomize the address at which shared libraries are
loaded. This is generally referred to as Address Space Layout Randomization, or
ASLR. Shared libraries are always position independent, which means that they
can be loaded at any address. Randomizing the load address makes it slightly
harder for attackers of a running program to exploit buffer overflows or
similar problems, because they have no fixed addresses that they can rely on.
ASLR is part of defense in depth: it does not by itself prevent any attacks,
but it makes it slightly more difficult for attackers to exploit certain kinds
of programming errors in a useful way beyond simply crashing the program.

Although it is straightforward to randomize the load address of a shared
library, an ELF executable is normally linked to run at a fixed address that
can not be changed. This means that attackers have a set of fixed addresses
they can rely on. Permitting the kernel to randomize the address of the
executable itself is done by generating a Position Independent Executable, or
PIE.

It turns out to be quite simple to create a PIE: a PIE is simply an executable
shared library. To make a shared library executable you just need to give it a
`PT_INTERP` segment and appropriate startup code. The startup code can be the
same as the usual executable startup code, though of course it must be compiled
to be position independent.

When compiling code to go into a shared library, you use the `-fpic` option.
When compiling code to go into a PIE, you use the `-fpie` option. Since a PIE
is just a shared library, these options are almost exactly the same. The only
difference is that since `-fpie` implies that you are building the main
executable, there is no need to support symbol interposition for defined
symbols. In a shared library, if function `f1` calls `f2`, and `f2` is globally
visible, the code has to consider the possibility that `f2` will be interposed.
Thus, the call must go through the PLT. In a PIE, `f2` can not be interposed,
so the call may be made directly, though of course still in a position
independent manner. Similarly, if the processor can do PC-relative loads and
stores, all global variables can be accessed directly rather than going through
the GOT.

Other than that ability to avoid the PLT and GOT in some cases, a PIE is really
just a shared library. The dynamic linker will ask the kernel to map it at a
random address and will then relocate it as usual.

This does imply that a PIE must be dynamically linked, in the sense of using
the dynamic linker. Since the dynamic linker and the C library are closely
intertwined, linking the PIE statically with the C library is unlikely to work
in general. It is possible to design a statically linked PIE, in which the
program relocates itself at startup time. The dynamic linker itself does this.
However, there is no general mechanism for this at present.

