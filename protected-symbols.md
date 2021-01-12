# Protected symbols

Now for something really controversial: what’s wrong with protected symbols?

In an ELF shared library, an ordinary global symbol may be overridden if a
symbol of the same name is defined in the executable or in a shared library
which appears earlier in the runtime search path. This is called symbol
interposition. It is often used with functions such as `malloc`. A shared
library can define `malloc` and it can have code which calls `malloc`. If the
executable linked with the shared library defines `malloc` itself, then the
version in the executable will be used rather than the version in the shared
library. This permits the executable to control the memory allocation done by
the shared library, perhaps for debugging or logging purposes. In this regard,
shared libraries act much as static archives do.

This has a few consequences. One of them is that within a shared library, all
references to a global symbol must use the GOT and PLT, to make the overriding
possible. That means that all function calls and variable accesses are slightly
slower. Also, some compiler optimizations are forbidden: the compiler can not
inline a call to a global symbol, since that symbol might be overridden at run
time.

When building a shared library, you can provide a version script which
indicates that some symbols are actually not global. That can eliminate the GOT
and PLT accesses, but it does not permit the compiler optimizations, and you do
have to write that version script and keep it up to date.

When compiling code that goes into a shared library, you can set the visibility
of symbols. You can use hidden visibility, which means that the symbol is not
visible outside the shared library. You can use internal visibility, which is a
lot like hidden—I’ll skip the difference here. Or you can use protected
visibility. Protected visibility means that the symbol is visible outside of
the shared library, and can be accessed as usual. However, all references from
within the shared library will use the definition in the shared library. In
other words, the symbol acts more or less as usual, but it can not be
overridden. This means that accesses to the symbol avoid the GOT and PLT, and
it permits compiler optimizations.

So, what’s wrong with them? It turns out that protected symbols are slower at
dynamic link time, which means that programs which use the shared library start
up slower. This happens because of the C rule that two pointers to the same
function must compare as equal. Since protected symbols are globally visible,
you can get a pointer to a protected function in the main executable. You can
also get a pointer to that same function in the shared library, of course.
Those pointers have to be equal, or the C rule will break.

As noted, the access to the function in the shared library will not use the GOT
or PLT. The access in the main executable obviously will use the PLT. How can
we make those function pointers equal? We can’t. The executable will have a
direct reference to the PLT. The shared library will have a direct reference to
the function itself. In neither case will there be a relocation for the
reference. So there is no way to make the results equal. (This can work for
some targets, but not for ones with simple function references like the x86
targets.)

So, I must have lied. The lie was that there is a case where you need to use
the GOT for a protected symbol: when compiling position independent code for a
shared library, and taking the address of a protected function, you need to use
the GOT. Unfortunately, gcc for the x86_64 target, surely the most widely used
gcc target today, gets this wrong: http://gcc.gnu.org/PR19520. This generally
reveals itself as an error report when you go to create a shared library:
relocation R_X86_64_PC32 against protected symbol `NAME` can not be used when
making a shared object.

In any case, when the compiler gets it right, the dynamic linker has to fill in
that GOT entry. In order to make the function pointers compare as equal, it has
to fill in the entry with the address of the PLT in the executable (or the
earlier shared library). But remember, this is a protected symbol, and
protected symbols don’t support symbol interposition. So the dynamic linker
must only use the PLT of the executable if the reference in the executable
refers to the definition in the shared library. That means that when the
dynamic linker sees a reloc against a protected symbol in a shared library, it
has to do another walk through the executable and earlier shared libraries to
see if any of them have a definition for the symbol, in which case the GOT
entry must not be set to that earlier PLT entry but must instead be set to the
address of the symbol in the shared library itself. This check has to be done
for every symbol in the shared library.

Those extra symbol resolution passes means a slow down for every program which
uses the shared library, and that is what is wrong with protected symbols.

So how do you get the compiler and linker speedups available by avoiding symbol
interpositioning? Unfortunately, you have to give your symbols hidden
visibility, which means that they can not be accessed from other modules.
Assuming you do want them to be accessed, you need to define symbol aliases for
the ones which should be publicly visible. That means that you need to use
different names for the hidden symbols. This is awkward at best. Unfortunately
I have nothing better to offer. ELF is designed to support symbol
interpositioning, and there is no very good way to avoid that without causing
other consequences.

