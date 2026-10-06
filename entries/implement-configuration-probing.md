# How do I implement `autoconf`-style configuration probing?

> I need to check something about the target environment by compiling (or even
> linking) a test program, similar to `autoconf`/CMake/etc. How can I do this
> in `build2` that doesn't have a separate configuration/project generation
> step?

It is possible to implement configuration probing in `build2` with a bit
of effort. However, before doing that, it's a good idea to understand the
main issues with this approach and how they can be mitigated in `build2`.

Configuration probing as implemented in `autoconf`/CMake/etc involves
compiling and linking a test program to determine whether a particular
feature, such as a function, is available in the environment being
targeted. For example, we may prefer to use the `strl*()` family of functions
in our codebase. However, these functions are not (yet) standard and are not
provided by all libc implementations. As a result, we may wish to detect
whether they are present and if not, provide fallback implementations or use
alternatives. One way to do this detection would be to compile and link a test
program that tries to use the functions we are interested in. If that
succeeds, then we conclude the functions are available.

While it sounds straightforward, there are a number of problems with such
probing:

1. It is wasteful: There is no need to keep compiling the `strl*()` probes on,
   say, FreeBSD, where these functions were available since eons. At the limit
   this becomes absurd, like keep probing for a feature while the latest
   target that doesn't have it would not be able to execute the probe, for
   example, due to having insufficient memory (see [A Generation Lost in the
   Bazaar][bazaar] for an illustration).

2. It is brittle: We decide that a feature is absent based on the failure to
   compile/link a test program. But a lot of other things can lead to a
   failure to compiler or link: mistakes in the test, misconfigured build,
   missing feature test macros such as `_GNU_SOURCE`, etc. A recent example
   that caused widespread false negatives were sloppily written probes that
   stopped compiling because both GCC and Clang stopped accepting certain
   long-deprecated C constructs.

3. It is slow: While compiling a single probe doesn't take long, compiling
   several hundreds is noticeable. `autoconf` and CMake both do it serially
   which exacerbates the problem.

4. Results are not change-tracked: Existing tools (`autoconf`, CMake) do not
   re-run the relevant probes when their inputs change. For example,
   `strl*()` were added in glibc 2.38. If your upgraded from 2.37, you
   would want all the already configured projects on your machine to
   detect the change and start using the new functions.

Solving the first problem (wastefulness) requires a completely different
approach. In `build2` we prefer the expectation-based configuration, where we
assume a feature should be available if certain conditions are met (e.g., the
libc is glibc and its version is greater or equal 2.38). This is supported in
the `autoconf`-compatible manner (including for the `strl*()` checks) by the
[`libbuild2-autoconf`][libbuild2-autoconf] build system module.

Assuming you still wish to do configuration probing for some reason or for
some special cases, let's see how to set it up in `build2` and solve, or at
least mitigate, the remaining problems. Unlike `autoconf`/CMake/etc which run
the probes during the separate project generation step, the `build2` approach
is to run them during the main build step. Because the resulting information
is typically needed in `buildfiles`, we do this during `buildfile` load using
the [update during load][udl] functionality; make sure you read and understand
the linked documentation before continuing.

The high level plan, therefore, is to implement an ad hoc build system rule
that implements the probing logic. That is, it tries to compile a probe and
writes the result to a generated `buildfile`. Then we write the necessary
probes, one per file, and use them as inputs for the generated
`buildfiles`. Finally, we generate the `buildfiles` during load and include
them into the main `buildfile`. Here is the outline of this plan as the main
`buildfile` fragment (we put the probes into the `probes/` subdirectory to
keep things organized):

```
# C probe rule.
#
buildfile{~'/have-(.*)/'}: c{~'/have-\1/'}
{{
  ...
}}

# Compile options that should be included in the probes.
#

# If targeting Linux define _GNU_SOURCE (used to enable strl*() in glibc,
# musl, etc).
#
if ($c.target.class == 'linux')
  c.poptions += -D_GNU_SOURCE

# Probes.
#
probes = have-strlcpy have-strlcat

for p: $probes
  probes/buildfile{$p}: probes/c{$p}

update probes/buildfile{$probes}
source $path(probes/buildfile{$probes})

./: probes/buildfile{$probes} # Make sure gets cleaned.

info have_strlcpy: $have_strlcpy
info have_strlcat: $have_strlcat

# The rest of the buildfile (make sure to exclude probes).
#
./: exe{autoprobe}: {h c}{** -strl* -probes/**}

exe{autoprobe}: {h c}{strlcpy}: include = (!$have_strlcpy)
exe{autoprobe}: {h c}{strlcat}: include = (!$have_strlcat)

c.poptions += ($have_strlcpy ? -DHAVE_STRLCPY : )
c.poptions += ($have_strlcat ? -DHAVE_STRLCAT : )

...
```

The implementation of the probe rule is not exactly trivial and we suggest
that you copy it from the [`autoprobe`][autoprobe] example. It provides
implementations for both C and C++ (the choice of the language is discussed
below). They are well commented and should be easy to follow and, if
necessary, customize.

Let's now discuss how we can solve, or at least mitigate, the remaining
problems of the configuration probing approach. The good news is that
#3 (slow) and #4 (lack of change-tracking) are already solved because
the probing is performed as part of the main build step and the full
support of the build system can be brought to bear. Specifically, the
probing rule keeps track of the changes in compile options, included
headers, etc., the same as during normal source file compilation. And
probes are all compiled in parallel (provided that they are all listed
in a single `update` directive), making the performance penalty bearable:
it takes about half a second to run 500 probes on modern hardware.

Solving problem #2 (brittleness) is more challenging. In a nutshell, we need to
distinguish the failure caused by the absence of the feature we are probing
from all other failures. Doing it directly would require analyzing compiler
diagnostics, which is hopeless. The next best thing we can try is to have a
"control" probe. The idea is to write a variant of the original probe that we
expect to fail in all the same circumstances except when the feature we are
interested in is absent. This control should mimic the original as close as
possible: it should include the same headers, use the same language
constructs, have the same logic, etc. In fact, it is best to have both
variants implemented in the same source file. Here is what a probe for
`strlcpy()` could look like (file `probes/have-strlcpy.c`):

```
#include <string.h>

size_t f (void)
{
  char dst[8];

#ifndef CONTROL
  size_t n = sizeof (dst);
  size_t r = strlcpy (dst, "strlcpy", n);
#else
  strcpy (dst, "strlcpy");
  size_t r = 7;
#endif

  return r;
}
```

The control probe can also serve an additional purpose: it can be used to
extract the header dependency information for change tracking (thus the
importance of including the same set of headers).

Putting it all together, the implementation of the probe rule would then
have the following high-lever logic (see the actual implementation for
details):

1. Compile the control probe passing through any diagnostics and failing if
   the compilation fails. At the same time extract the header dependency
   information.

2. Compile the actual probe ignoring any diagnostics. If the compilation
   succeeds, assume the feature is present, otherwise -- absent.

3. Write the outcome to the output `buildfile`.

While the control idea might seem like a clever solution, it's not without
drawbacks. The main one is that writing a good control might be challenging.
For `strlcpy()` it is pretty easy to implement a very close control using
`strcpy()`. This makes sure the headers we include are present and usable, the
language syntax and logic we use are correct, etc. In other situations writing
a close control might be more difficult.

One particular case where writing a valid control is pretty much impossible is
checking for the presence of a system header (or library). Thankfully, these
checks can be implemented in `build2` without probes using the
`${c,cxx}.find_system_{header,library}()` functions. For example:

```
have_string_h = ($c.find_system_header(string.h) != [null])
```

Next, let's discuss a few more relevant aspects of probing. The way `autoconf`
(and CMake) implement function probes is by compiling and linking a test
program. They also don't rely on the presence of the function declaration in
any header, rather declaring it themselves. In other words, what they really
check is the presence of the corresponding symbol in a library. This approach
has a long list of corner cases and drawbacks: The function might be inline or
a compiler builtin (and thus without a symbol). The symbol may be present but
the function declaration might not be enabled in the corresponding header. Or
the function signature might not match what we expect, rendering our call
sites invalid.

To give a concrete example, from glibc 2.38 a probe with its own `strlcpy()`
declaration links fine even if compiled without `_GNU_SOURCE`. But the
`strlcpy()` declaration in `<string.h>` is only enabled if this macro
is defined during compilation.

An alternative approach to checking for the presence of a library symbol would
be to obtain the declaration by including the standard header and check whether
the call site compiles. This approach doesn't have any of the corner cases
listed above. It also closely matches how the function will be used in the
actual code. It does require disabling implicit function declarations when
compiling C probes, but that's not difficult to do for modern C compilers.

As a result, our recommendation is to use the call site compilation approach,
which is what the probe rules in the `autoprobe` example implement.

The closely related question is which language to use to compile the probe.
While it may seem like a good idea to use C when checking, say, for
`strlcpy()` seeing that it's a C function, our recommendation is to use
the same language as the actual code that will be using the feature in
order to keep the probe and said code as similar as possible.

Finally, let's discuss dependencies between probe results. The motivating
example would be first checking for a header (e.g., `<string.h>`) and then, if
it's present, for some functions that it may or may not provide (e.g.,
`strlcpy()`).  The idea is that if the header is absent, then there is no
point in wasting time also checking for the functions.

The probe rule implementations in the `autoprobe` example provide support for
probe result dependencies. Specifically, you can list one or more generated
`buildfiles` as prerequisite of another generated `buildfile` target and the
probe rule will examine them and short-circuit the compilation if any of the
prerequisite results are `false`.

Let's see how we can use this to implement the above `<string.h>`/`strlcpy()`
scenario. As a first step, we need to produce the result of the
`<string.h>` header check as a generated `buildfile` rather than just
a variable assignment as was shown earlier. To make this easier, the
`autoprobe` example provides a special rule that does this automatically.
All we have to do is declare a `buildfile` target that ends with `-h` to
indicate we want a header presence check (you can override the header
name with the `header_name` target-specific variable to check for something
like `sys/types.h`):

```
probes/buildfile{have-string-h}: # Check for string.h.
```

Then we just list this target as a prerequisite of all the `strl*()` probes
we have. Putting it all together:

```
probes/buildfile{have-string-h}:
probes = have-string-h

for p: have-strlcpy have-strlcat
{
  probes/buildfile{$p}: probes/c{$p} probes/buildfile{have-string-h}
  probes += $p
}

update probes/buildfile{$probes}
source $path(probes/buildfile{$probes})
```

[bazaar]: https://queue.acm.org/doi/10.1145/2346916.2349257
[libbuild2-autoconf]: https://github.com/build2/libbuild2-autoconf/
[udl]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#directives-update
