# Version Scripts

I recently spent some time sorting through linker version script issues, so I’m
going to document what I discovered.

Linker symbol versioning was invented at Sun. The Solaris linker lets you use a
version script when you create a shared library. This script assigns versions
to specific named symbols, and defines a version hierarchy. When an executable
is linked against the shared library, the versions that it uses are recorded in
the executable. If you later try to dynamically link the executable with a
shared library which does not provide the required versions, you get a sensible
error message.

Sun’s scheme (as I understand it) only permits you to add new versions and new
symbols. Once a symbol has been defined at a specific version, you can not
change that in later releases. if you change the behaviour of a symbol, you
don’t change the version of the symbol itself, instead you add a new version to
the library even if it does not define any symbols. That is sufficient to
ensure that an executable will not be dynamically linked against a version of
the shared library which is too old.

Eric Youngdale and Ulrich Drepper introduced a more sophisticated symbol
versioning scheme in the GNU linker and the GNU/Linux dynamic linker. The GNU
linker permits symbols to have multiple versions, of which only one is the
default. These versions are specified in the object files linked together to
form the shared library. The assembler `.symver` directive is used to assign a
version to a symbol (the version is simply encoded in the name of the symbol).
This scheme permits using symbol versioning to actually change the behaviour of
a symbol; older executables will continue to use the old version. This also
permits deleting symbols, by removing the default version. The older versions
of the symbol remain but are inaccessible.

That is all fine. The problems come in with the extensions to the version
script language. First, the GNU linker permits wildcards in version scripts.
Second, the GNU linker permits symbols to match against demangled names, again
typically using wildcards. Third, the GNU linker permits the version script to
hide symbols which have explicit versions in input object files.

Every symbol can only have one version. When the linker asks for the version of
a symbol, there can only be one answer. The support for wildcards and matching
of demangled names in the GNU linker script means that there may not be a
unique answer for the version to use for a given name. The fact that the GNU
linker permits version scripts to hide symbols with explicit versions means
that in some cases you absolutely must list a symbol two times in a version
script (because you might have a `local: *;` entry which must not match your
symbol with an old version). This potential confusion means that using linker
scripts correctly with wildcards requires a clear understanding of exactly how
the linker parses a version script.

Unfortunately, this was never documented. Until now. Here are the rules which
the GNU linker uses to parse version scripts, as of 2010-01-11.

The GNU linker walks through the version tags in the order in which they appear
in the version script. For each tag, it first walks through the global patterns
for that tag, then the local patterns. When looking at a single pattern, it
first applies any language specific demangling as specified for the pattern,
and then matches the resulting symbol name to the pattern. If it finds an exact
match for a literal pattern (a pattern enclosed in quotes or with no wildcard
characters), then that is the match that it uses. If finds a match with a
wildcard pattern, then it saves it and continues searching. Wildcard patterns
that are exactly “*” are saved separately.

If no exact match with a literal pattern is ever found, then if a wildcard
match with a global pattern was found it is used, otherwise if a wildcard match
with a local pattern was found it is used.

This is the result:

* If there is an exact match, then we use the first tag in the version script
  where it matches.
  * If the exact match in that tag is global, it is used.
  * Otherwise the exact match in that tag is local, and is used.
* Otherwise, if there is any match with a global wildcard pattern:
  * If there is any match with a wildcard pattern which is not `*`, then we use
    the tag in which the last such pattern appears.
  * Otherwise, we matched `*`. If there is no match with a local wildcard
    pattern which is not `*`, then we use the last match with a global `*`.
    Otherwise, continue.
* Otherwise, if there is any match with a local wildcard pattern:
  * If there is any match with a wildcard pattern which is not `*`, then we use
    the tag in which the last such pattern appears.
  * Otherwise, we matched `*`, and we use the tag in which the last such match
    occurred.

As mentioned above, there is an additional wrinkle. When the GNU linker finds a
symbol with a version defined in an object file due to a `.symver` directive, it
looks up that symbol name in that version tag. If it finds it, it matches the
symbol name against the patterns for that version. If there is no match with a
global pattern, but there is a match with a local pattern, then the GNU linker
marks the symbol as local.

I want gold to be compatible, but I also want gold to be efficient. I’ve
introduced a hash table in gold to do fast lookups for exact matches. That
makes it impossible for gold to follow the exact rules when matching demangled
names. Currently gold does not do the final lookup to see if a symbol with an
explicit version should be forced local; I don’t understand why that is useful.
It is possible that I will be forced to add that to gold at some later date.

Here are the current rules for gold:

* If there is an exact match for the mangled name, we use it.
  * If there is more than one exact match, we give a warning, and we use the
    first tag in the script which matches.
  * If a symbol has an exact match as both global and local for the same
    version tag, we give an error.
* Otherwise, we look for an extern C++ or an extern Java exact match. If we
  find an exact match, we use it.
  * If there is more than one exact match, we give a warning, and we use the
    first tag in the script which matches.
  * If a symbol has an exact match as both global and local for the same
    version tag, we give an error.
* Otherwise, we look through the wildcard patterns, ignoring `*` patterns. We
  look through the version tags in reverse order. For each version tag, we look
  through the global patterns and then the local patterns. We use the first
  match we find (i.e., the last matching version tag in the file).
* Otherwise, we use the `*` pattern if there is one. We give a warning if there
  are multiple `*` patterns.

I hope for your sake that this information never actually matters to you.

