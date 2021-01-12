# Combining versions

Sun introduced a symbol versioning scheme to use for the linker. Their
implementation is relatively simple: symbol versions are defined in a version
script provided when a shared library was created. The dynamic linker can
verify that all required versions are present. This is useful for ensuring that
an application can run with a specific version of the library.

In the Sun versioning scheme, when a symbol is changed to have an incompatible
interface, the library file name must change. This then produces a new
`DT_SONAME` entry, which leads to new `DT_NEEDED` entries, and thus manages
incompatibility at that level.

Ulrich Drepper and Eric Youngdale introduced a much more sophisticated symbol
versioning scheme, which is used by the glibc, the GNU linker, and gold. The
key differences are that versions may be specified in object files and that
shared libraries may contain multiple independent versions of the same symbol.
Versions are specified in object files by naming the symbol `NAME@VERSION` or
`NAME@@VERSION`. In the former case the symbol is a hidden version, available
only by specific request. In the latter case the symbol is a default version,
and references to `NAME` will be linked to `NAME@@VERSION`. Versions may also
be specified in version scripts.

This facility means that in principle it is never necessary to change the
library file name. The versioning scheme lets the dynamic linker direct each
symbol reference to the appropriate version. This in turn means that in a
complicated program with many shared libraries compiled against different
versions of the base library, only one instance of the base library needs to be
loaded.

However, this additional complexity leads to additional ambiguity. There are
now two possible sources of a symbol version: the name in the object file and
an entry in the version script. There is the possibility that two instances of
the same name will disagree on whether the name should be globally visible or
not–in fact, this is normal, as undefined references will always use
`NAME@VERSION`, not `NAME@@VERSION`. Symbol overriding can be confusing: if the
main executable defines `NAME` without a version, which versions should it
override in the shared library? Which version should be used in the program?
Symbol visibility adds an additional wrinkle to this.

The most important issue for the linker arises when it sees both NAME and
`NAME@VERSION`, and then sees `NAME@@VERSION`. At that time the linker has seen
two separate symbols and has to decide whether to merge them. The rules that
gold currently follows are these:

* If `NAME` is hidden, and `NAME@@VERSION` is in a shared object, they are two
  independent symbols, and we do not change `NAME` or its version.
* If `NAME` already has a version, because we earlier saw `NAME@@VERSION2`,
  then we produce two separate symbols, and leave `NAME@@VERSION2` as the
  default symbol.
* Otherwise, we change the version of `NAME` to `VERSION`, and do normal symbol
  resolution.

I recently fixed a bug in this code in gold, which was breaking symbol
overriding in a specific case. I wouldn’t be surprised if there are more bugs.
As far as I know nobody has worked through all the symbol combining issues and
defined what should happen.

