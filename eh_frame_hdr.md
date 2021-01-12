# .eh_frame_hdr

If you followed my last post, you will see that in order to unwind the stack
you have to find the FDE associated with a given program counter value. There
are two steps to this problem. The first one is finding the CIEs and FDEs at
all. The second one is, given the set of FDEs, finding the one you need.

The old way this worked was that gcc would create a global constructor which
called the function `__register_frame_info`, passing a pointer to the
`.eh_frame` data and a pointer to the object. The latter pointer would indicate
the shared library, and was used to deregister the information after a dlclose.
When looking for an FDE, the unwinder would walk through the registered frames,
and sort them. Then it would use the sorted list to find the desired FDE.

The old way still works, but these days, at least on GNU/Linux, the sorting is
done at link time, which is better than doing it at runtime. Both gold and the
GNU linker support an option `--eh-frame-hdr` which tell them to construct a
header for all the .eh_frame sections. This header is placed in a section named
.eh_frame_hdr and also in a PT_GNU_EH_FRAME segment. At runtime the unwinder
can find all the `PT_GNU_EH_FRAME` segments by calling `dl_iterate_phdr`.

The format of the `.eh_frame_hdr` section is as follows:

* A 1 byte version number, currently 1.
* A 1 byte encoding of the pointer to the exception frames. This is a
  `DW_EH_PE_xxx` value. It is normally `DW_EH_PE_pcrel | DW_EH_PE_sdata4`,
  meaning a 4 byte relative offset.
* A 1 byte encoding of the count of the number of FDEs in the lookup table.
  This is a `DW_EH_PE_xxx` value. It is normally `DW_EH_PE_udata4`, meaning a 4
  byte unsigned count.
* A 1 byte encoding of the entries in the lookup table. This is a
  `DW_EH_PE_xxx` value. It is normally `DW_EH_PE_datarel | DW_EH_PE_sdata4`,
  meaning a 4 byte offset from the start of the `.eh_frame_hdr` section. That
  is the only encoding that gcc’s current unwind library supports.
* A pointer to the contents of the `.eh_frame` section, encoded as indicated by
  the second byte in the header. This pointer is only used if the format of the
  lookup table is not supported or is for some reason omitted..
* The number of FDE pointers in the table, encoded as indicated by the third
  byte in the header. If there are no FDEs, the encoding can be `DW_EH_PE_omit`
  and this number will not be present.
* The lookup table itself, starting at a 4-byte aligned address in memory.
  Assuming the fourth byte in the header is `DW_EH_PE_datarel | DW_EH_PE_sdata4`,
  each entry in the table is 8 bytes long. The first four bytes are an offset
  to the initial PC value for the FDE. The last four byte are an offset to the
  FDE data itself. The table is sorted by starting PC.

Since FDEs do not overlap, this table is sufficient for the stack unwinder to
quickly find the relevant FDE if there is one.

