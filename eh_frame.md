# .eh_frame

When gcc generates code that handles exceptions, it produces tables that
describe how to unwind the stack. These tables are found in the `.eh_frame`
section. The format of the `.eh_frame` section is very similar to the format of
a DWARF `.debug_frame` section. Unfortunately, it is not precisely identical. I
don’t know of any documentation which describes this format. The following
should be read in conjunction with the relevant section of the DWARF standard,
available from http://dwarfstd.org.

The `.eh_frame` section is a sequence of records. Each record is either a CIE
(Common Information Entry) or an FDE (Frame Description Entry). In general
there is one CIE per object file, and each CIE is associated with a list of
FDEs. Each FDE is typically associated with a single function. The CIE and the
FDE together describe how to unwind to the caller if the current instruction
pointer is in the range covered by the FDE.

There should be exactly one FDE covering each instruction which may be being
executed when an exception occurs. By default an exception can only occur
during a function call or a throw. When using the `-fnon-call-exceptions` gcc
option, an exception can also occur on most memory references and floating
point operations. When using `-fasynchronous-unwind-tables`, the FDE will cover
every instruction, to permit unwinding from a signal handler.

The general format of a CIE or FDE starts as follows:

* Length of record. Read 4 bytes. If they are not `0xffffffff`, they are the
  length of the CIE or FDE record. Otherwise the next 64 bits holds the length,
  and this is a 64-bit DWARF format. This is like `.debug_frame`.
* A 4 byte ID. For a CIE this is 0. For an FDE it is the byte offset from this
  field to the start of the CIE with which this FDE is associated. The byte
  offset goes to the length record of the CIE. A positive value goes backward;
  that is, you have to subtract the value of the ID field from the current byte
  position to get the CIE position. This differs from `.debug_frame` in that
  the offset is relative rather than being an offset into the `.debug_frame`
  section.

A CIE record continues as follows:

* 1 byte CIE version. As of this writing this should be 1 or 3.
* NUL terminated augmentation string. This is a sequence of characters. Very
  old versions of gcc used the string “eh” here, but I won’t document that.
  This is described further below.
* Code alignment factor, an unsigned LEB128 (LEB128 is a DWARF encoding for
  numbers which I won’t describe here). This should always be 1 for `.eh_frame`.
* Data alignment factor, a signed LEB128. This is a constant factored out of
  offset instructions, as in `.debug_frame`.
* The return address register. In CIE version 1 this is a single byte; in CIE
  version 3 this is an unsigned LEB128. This indicates which column in the
  frame table represents the return address.

The next fields of the CIE depend on the augmentation string.

* If the augmentation string starts with ‘z’, we now find an unsigned LEB128
  which is the length of the augmentation data, rounded up so that the CIE ends
  on an address boundary. This is used to skip to the end of the augmentation
  data if an unrecognized augmentation character is seen.
* If the next character in the augmentation string is ‘L’, the next byte in the
  CIE is the LSDA (Language Specific Data Area) encoding. This is a
  `DW_EH_PE_xxx` value (described later). The default is `DW_EH_PE_absptr`.
* If the next character in the augmentation string is ‘R’, the next byte in the
  CIE is the FDE encoding. This is a `DW_EH_PE_xxx` value. The default is
  `DW_EH_PE_absptr`.
* The character ‘S’ in the augmentation string means that this CIE represents a
  stack frame for the invocation of a signal handler. When unwinding the stack,
  signal stack frames are handled slightly differently: the instruction pointer
  is assumed to be before the next instruction to execute rather than after it.
* If the next character in the augmentation string is ‘P’, the next byte in the
  CIE is the personality encoding, a `DW_EH_PE_xxx` value. This is followed by
  a pointer to the personality function, encoded using the personality
  encoding.  I’ll describe the personality function some other day.

The remaining bytes are an array of `DW_CFA_xxx` opcodes which define the
initial values for the frame table. This is then followed by `DW_CFA_nop`
padding bytes as required to match the total length of the CIE.

An FDE starts with the length and ID described above, and then continues as
follows.

* The starting address to which this FDE applies. This is encoded using the FDE
  encoding specified by the associated CIE.
* The number of bytes after the start address to which this FDE applies. This
  is encoded using the FDE encoding.
* If the CIE augmentation string starts with ‘z’, the FDE next has an unsigned
  LEB128 which is the total size of the FDE augmentation data. This may be used
  to skip data associated with unrecognized augmentation characters.
* If the CIE does not specify `DW_EH_PE_omit` as the LSDA encoding, the FDE
  next has a pointer to the LSDA, encoded as specified by the CIE.

The remaining bytes in the FDE are an array of `DW_CFA_xxx` opcodes which set
values in the frame table for unwinding to the caller.

The `DW_EH_PE_xxx` encodings describe how to encode values in a CIE or FDE. The
basic encoding is as follows:

* `DW_EH_PE_absptr = 0x00`: An absolute pointer. The size is determined by
  whether this is a 32-bit or 64-bit address space, and will be 32 or 64 bits.
* `DW_EH_PE_omit = 0xff`: The value is omitted.
* `DW_EH_PE_uleb128 = 0x01`: The value is an unsigned LEB128.
* `DW_EH_PE_udata2 = 0x02`, `DW_EH_PE_udata4 = 0x03`, `DW_EH_PE_udata8 = 0x04`:
  The value is stored as unsigned data with the specified number of bytes.
* `DW_EH_PE_signed = 0x08`: A signed number. The size is determined by whether
  this is a 32-bit or 64-bit address space. I don’t think this ever appears in
  a CIE or FDE in practice.
* `DW_EH_PE_sleb128 = 0x09`: A signed LEB128. Not used in practice.
* `DW_EH_PE_sdata2 = 0x0a`, `DW_EH_PE_sdata4 = 0x0b`, `DW_EH_PE_sdata8 = 0x0c`:
  The value is stored as signed data with the specified number of bytes. Not
  used in practice.

In addition the above basic encodings, there are modifiers.

* `DW_EH_PE_pcrel = 0x10`: Value is PC relative.
* `DW_EH_PE_textrel = 0x20`: Value is text relative.
* `DW_EH_PE_datarel = 0x30`: Value is data relative.
* `DW_EH_PE_funcrel = 0x40`: Value is relative to start of function.
* `DW_EH_PE_aligned = 0x50`: Value is aligned: padding bytes are inserted as
  required to make value be naturally aligned.
* `DW_EH_PE_indirect = 0x80`: This is actually the address of the real value.

If you follow all that, and also read up on `.debug_frame`, then you have
enough information to unwind the stack at runtime, e.g. to implement glibc’s
backtrace function. Later I’ll describe the LSDA and the personality function,
which work together to implement exception catching on top of stack unwinding.

