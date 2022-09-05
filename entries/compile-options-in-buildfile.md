# Which C/C++ compile/link options are OK to specify in a project's `buildfile`?

> I am trying to package a third-party project and I see that they specify
> all sorts of C/C++ compile and link options in their build definition
> files. Which ones should I copy into `build2` `buildfile`?

Most of the C/C++ build systems in use today were not designed with the
ecosystem of independent, reusable packages in mind. As a result,
thoughtlessly/faithfully copying all the options over is usually not the
correct approach.

As a general rule, a `buildfile` should not specify any options that
could:

1. Prevent the user from controlling the overall characteristics of the
   build, such as debug information, optimization, target architecture,
   diagnostics, etc.

2. Require changes to compilation/linking of other projects that depend
   on this project.

3. Interfere with the functioning of the build system, such us overriding
   output directories, parallelism, etc.

In other words, the options that you may specify in a `buildfile` should only
have local effect. Here are some common categories of such options:

1. Enable/disable warnings (`-Wno-XXX`, `/wdXXX`, `-D_{CRT,SCL}_SECURE_NO_WARNINGS`).

2. Enable/disable language features that only affect individual translation
   units, for example:

   * strict aliasing (`-fno-strict-aliasing`)
   * exceptions (`-fno-exceptions`, `/EHs- /EHc-`)
   * RTTI (`-fno-rtti`, `/GR-`)

3. Enable/disable standard library features that only affect individual
   translation units (`-D_GNU_SOURCE`, `-D_POSIX_C_SOURCE`, `-D_WIN32_WINNT`).

   Note: there should be no expectation that the consumers of the library are
   compiled with such options. For example, `-D_FILE_OFFSET_BITS` could
   create such an expectation if `off_t` is used in the library interface and
   thus violate rule #2 above.

4. Control visibility of symbols (`-fvisibility*`).

Note: if you need to link the `pthread` library, use `-pthread`, not
`-lpthread`; see [How do I link the `pthread` library?][link-pthread]

And here are some common examples of options that should normally not be
specified in a `buildfile` (with references to the above rules that they
violate).

* Turn warnings into errors (`-Werror`, `/WX`) [#1]

* Disable assertion checking (`-DNDEBUG`) [#1]

* Extra protection/verification (`-fstack-protector-all`, `-fsanitize*`, `-D_GLIBCXX_ASSERTIONS`) [#1]

* Enable SIMD (`-msse`, `-mavx`, `/arch:AVX`) [#1]

* Force position-independent code (`-fPIC`) [#3]

* Adjust parallelism (`/MP`, `-flto-jobs`) [#3]

* Adjust other compiler/linker behavior (`/FS`, `/MANIFEST:NO`) [#1, #3]

Note also that some options are passed by default (for example, `/EHsc`) and
repeating them explicitly is unnecessary. Similarly, you shouldn't explicitly
link system libraries that are linked by default (for example, `kernel32.lib`,
`advapi32.lib`).

Finally, a note on `*.export.poptions` and `*.export.loptions`: the rules here
are even stricter and you should not try to impose any "settings" on the
consumers of your library other than those that affect your library itself.
In particular, you should not try to disable any warnings (for example, via
`-D_{CRT,SCL}_SECURE_NO_WARNINGS`) or affect any standard library features
(for example, via `-D_GNU_SOURCE`, `-D_FILE_OFFSET_BITS`, etc). Note that
this is the reason why there is no `*.export.coptions`.

[link-pthread]: link-pthread.md
