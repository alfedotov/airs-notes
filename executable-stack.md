# Executable stack

The gcc compiler implements an extension to C: nested functions. A trivial example:

```c
int f() {
	int i = 2;
	int g(int j) { return i + j; }
	return g(3);
}
```

The function `f` will return 5. Note in particular that the nested function `g`
refers to the variable i defined in the enclosing function.

You can mostly treat nested functions as ordinary functions. In particular, you
can take the address of a nested function, and you can pass the resulting
function pointer to another function, that function can make a call through the
function pointer to the nested function, and the nested function will correctly
refer to variables in its caller’s stack frame. I’m not here going to go into
the details of how this is implemented. What I will say is that gcc currently
implements this by writing instructions to the stack and using a pointer to
those instructions. This requires that the stack be executable.

This approach was implemented many years ago, before computers were routinely
attacked. In the hostile Internet environment of today, an area of memory that
is both writable and executable is dangerous, because it gives an attacker
space to create brand new instructions to execute. Since the stack must be
writable, this means that we want to make the stack non-executable if possible.
Since very few programs use nested functions, this is normally possible. But we
don’t want to break those few programs either.

This is how the GNU tools do it on ELF systems such as GNU/Linux. The compiler
adds a new section to all code that it compiles. The section is named
`.note.GNU-stack`. It is empty and not allocated, which means that it takes up
no space at runtime. If the code being compiled does not require an executable
stack—the normal case—the compiler doesn’t set any flags for the section. If
the code does require an executable stack, the compiler sets the
`SHF_EXECINSTR` flag.

When the linker links a program, it checks each input object for a
`.note.GNU-stack` section. If there is no such section, the linker assumes that
the object must be old, and therefore may require an executable stack. If there
is such a section, the linker checks the section flags to see whether the code
requires an executable stack. The linker discards the `.note.GNU-stack`
sections, and creates a `PT_GNU_STACK` segment in the output executable. The
`PT_GNU_STACK` segment is empty and is not part of any `PT_LOAD` segment. The
segment flags `PF_R` and `PF_W` are always set. If the linker has determined
that the program requires an executable stack, it also sets the `PF_X` flag.

When the Linux kernel starts a program, it looks for a `PT_GNU_STACK` segment.
If it does not find one, it sets the stack to be executable (if appropriate for
the architecture). If it does find a `PT_GNU_STACK` segment, it marks the stack
as executable if the segment flags call for it. (It’s possible to override this
and force the kernel to never use an executable stack.) Similarly, the dynamic
linker looks for a `PT_GNU_STACK` in any executable or shared library that it
loads, and changes the stack to be executable if any of them require it.

When this all works smoothly, most programs wind up with a non-executable
stack, which is what we want. The most common reason that this fails these days
is that part of the program is written in assembler, and the assembler code
does not create a `.note.GNU_stack` section. If you write assembler code for
GNU/Linux, you must always be careful to add the appropriate line to your file.
For most targets, the line you want is:

```asm
.section .note.GNU-stack,"",@progbits
```

There are some linker options to control this. The `-z execstack` option tells
the linker to mark the program as requiring an executable stack, regardless of
the input files. The `-z noexecstack` option marks it as not requiring an
executable stack. The gold linker has a `--warn-execstack` option which will
cause the linker to warn about any object which is missing a `.note.GNU-stack`
option or which has an executable `.note.GNU-stack` option.

The execstack program may also be used to query whether a program requires an
executable stack, and to change its setting.

These days we could probably change the default: we could probably say that if
an object file does not have a `.note.GNU-stack` section, then it does not
require an executable stack. That would avoid the problem of files written in
assembler which do not create the section. It’s possible that this would cause
some programs to incorrectly get a non-executable stack, but I think that would
be quite unlikely in practice. An advantage of changing the default would be
that the compiler would not have to create an empty `.note.GNU-stack` section
in all object files.

By the way, there is one thing you can do with a normal function that you can
not do with a nested function: if the nested function refers to any variables
in the enclosing function, you can not return a pointer to the nested function
to the caller. If you do, the variable will disappear, so the variable
reference in the nested function will be dangling reference. It’s worth noting
here that the Go language supports nested function literals which may refer to
variables in the enclosing function, and when using Go this works correctly.
The compiler creates variables on the heap if necessary, so they do not
disappear until the garbage collector determines that nothing refers to them
any more.

Finally, I’ll mention that there are some plans to implement a different scheme
for nested functions in C, one which does not require any memory to be both
writable and executable, but these plans have not yet been implemented. I’ll
leave the implementation as an exercise for the reader.

