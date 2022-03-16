# How do I handle auto-generated C/C++ headers?

> How do I handle auto-generated C/C++ headers in my project?

Most build systems deal with auto-generated headers either by making the user
explicitly specify their dependents or by generating them as part of a
separate pre-build step. Both of these approaches have drawbacks: explicit
dependency tracking does not scale while pre-generation leads to the loss of
parallelism and does not handle (or makes the user handle manually)
dependencies between generated headers.

One of `build2`'s main goals is to handle generated headers automatically and
within the main build step. However, real-world experience showed that the
C/C++ header model and, specifically, the header search paths (`-I`) mechanism
pose a number of fundamental obstacles (details below) for robust,
fully-automatic generated header support. It is not impossible to achieve if
the project follows a certain header inclusion scheme (discussed below) but
this is by no means a common practice and there are plenty of existing
projects that would be impractical to convert.

As a result, from version `0.15.0`, `build2` implements hybrid generated
header support: all headers listed as prerequisites of a library or executable
are by default pre-generated during the match phase. This should be sufficient
to address the needs of the majority of projects without any manual
intervention. Specifically, if all you have is an executable or a (utility)
library that has a few generated headers, such as `config.h`, then you are
covered. For example:

```
lib{hello}: {hxx cxx}{** -config} hxx{config}

hxx{config}: in{config}
```

In the above example, the generated `config.h` header will be pre-generated
before compiling any of the `lib{hello}`'s source files or any targets that
depend on it.

The pre-generation is still part of the main build step and should have
minimal impact on performance. In particular, header pre-generation happens in
parallel (if not prevented by dependency) and is interleaved with matching
other targets. But if you have a large number of expensive to generate headers
and would like to squeeze every last bit of performance, it is also possible
to disable pre-generation on a case by case basis.

There are also a few corner cases in the pre-generation approach that can
sometimes be found in more advanced projects and where a manual intervention
may be required.

The rest of this article covers these topics.


## Background

To understand the implications of the pre-generation approach we first need to
understand how `build2` handles auto-generated headers without pre-generation
and the fundamental obstacles posed by the header search paths.

For GCC and Clang, `build2` relies on the `-MG` option of the `-M*` option
family (used for the header dependency extraction) to detect auto-generated
headers.  Specifically, with this option, when the compiler encounters a
non-existent header, instead of issuing an error, it adds the header's name to
the dependency information and continues preprocessing. On the `build2` side
we then detect such an absent header, generate it, and restart the header
dependency extraction. For MSVC, `build2` uses the `/showIncludes` option to
extract the header dependency information and non-existent headers are
detected by analyzing the error diagnostics. This produces essentially the
same end result except that MSVC stops preprocessing on encountering the first
absent header.

This approach has two main problems. Firstly, when the header is absent, we
need to determine where exactly it should be generated. To put it another way,
given a list of header search paths (`-I` option values), we need to map the
header name as spelled in the `#include` directive to one of these paths. In
`build2` we use a heuristic that takes into account the header search paths,
the directory prefix of the header name, and the targets in `buildfiles`. This
has been repeatedly fine-tuned and now gives the expected result in most
situations.

The second problem is harder: the header may not be absent but rather found
and included. The header that has been found could be:

1. Outdated: an outdated header that we need to re-generate.

2. Wrong: a wrong header from the source directory of the in-source (or
   forwarded) build.

3. Installed: an installed header from one of the system search paths (for
   example, `/usr/include`).

4. Unrelated: an identically named header from an unrelated project.

Let's discuss each of these cases and see what problems they may cause.

The outdated header case (#1) is expected: the header may have already been
generated but is now out of date (for example, because one of its
prerequisites has changed) and should be re-generated. While `build2` handles
this case as expected when it gets the chance, the fact that some compilers
(namely GCC and Clang) will continue preprocessing after including an outdated
header may cause problems.

The typical example is a configuration file generated from an `.in` file that
defines a bunch of macros, for example, to `0` if the feature is enabled and
to `1` if it is disabled. The code that uses this configuration file checks
that the relevant macro is actually defined and issues an error (typically
with `#error`) if that's not the case. Adding a new feature and the
corresponding macro could then be problematic because the compiler will
include the outdated configuration header and continue preprocessing,
triggering the macro check. (It's easy to work around this problem locally but
think about others who will pull your changes while already having generated
the configuration file.)

Moving on, the wrong header case (#2) happens when you have both an in source
build (or a forwarded in source configuration with a backlinked generated
header) and an out of source build. For a project that uses generated headers,
the `-I` options would normally look like this:

```
cxx.poptions =+ "-I$out_root" "-I$src_root"
```

Which means that if the generated header does not yet exist in the
corresponding output directory, the next directory the compiler will look in
is the source directory. If the source directory is configured as an in source
build (or as a forwarded configuration with backlinked generated header), then
it may contain the header and which the compiler will include.

To resolve this, `build2` includes the remapping logic that detects the
output/source `-I` pairs like above and automatically remaps generated headers
that were mis-included. (Specifically, `build2` detect this situation,
generates the header in the output directory, and restarts the compiler at
which point it finds the correct header). Note, however, that this only works
if including the wrong header doesn't cause the macros issue described in case
#1.

The installed header case (#3) is a variant of the wrong header case where the
compiler includes an installed generated header of the same project, typically
from a location that is part of the built-in header search paths, for example
`/usr/include` or `/usr/local/include`.

Unlike for the wrong header, there is nothing `build2` can do for the
installed header case since it's not detectable without a substantial
performance penalty. However, it can be manually "reduced" to the wrong header
case by always keeping an empty dummy header with the same name in the
project's source directory, effectively preventing any built-in location
(which are examined last) from ever being considered. This technique is
described in detail in the "Dummy header technique" section below.

Finally, the unrelated header case (#4) happens when the compiler finds and
includes an identically named header in one of the header search paths. The
unrelated header can come from another project, whether installed or not, or
from the system via one of the built-in header search paths. And this case is
not limited to public or installed headers. All that's need is an include
search path pointing to the directory where an identically named header
happens to reside.

As an example, consider the common practice of including a generated
`config.h` header as `#include "config.h"`. With the `""` style inclusion the
compiler first looks in the directory of the including file, which is where
the author expects the configuration file to come from. But if there is no
`config.h` in this directory, the compiler continues looking in all the header
search paths, similar to the `<>` style inclusion. Imagine now an unrelated
project that is also part of the build and that happens to add a header search
path (via an exported `-I` option) pointing to a directory where it happens to
keep its `config.h`. In this case the compiler will include this unrelated
`config.h` file.

Similar to the installed case, there is nothing `build2` can do to rectify
this while the dummy header technique will only work for non-installed headers
where we can make sure our header search paths will always be considered
first. The better way to resolve this is to change the project's inclusion
scheme to use header names that are unlikely to clash. The easiest way to
achieve this in a consistent manner is to always use the `<>` style inclusion
and add the project name as a directory prefix to every header name (installed
or not), for example `#include <libhello/config.h>`. See [Canonical Project
Structure][proj-struct] for details.

Based on this analysis we can come up with the conditions that must be met for
an auto-generated header to be safe to use without pre-generation (we can work
around the last condition with the dummy header technique):

1. Not a "macro header".

2. Included with a name unlikely to clash.

3. Not installed.

It should now also be clear why pre-generation by default is the sensible
choice: the most widely used case of the generated header in today's C/C++
projects is likely the venerable `config.h` file. It is a macro header and
most projects include it as `"config.h"`.


## Header pre-generation semantics

Let's now discuss how exactly the header pre-generation is implemented in
`build2` and what are its implications.

Pre-generation is implemented in addition to rather than instead of the
"normal" generated header support. Specifically, without pre-generation, as
the compile rule is matched, it extracts the header dependency information
from each source file, (re)generating any missing or outdated headers. In
contrast, pre-generation is performed by the link rule. Specifically, before
synthesizing any object file dependencies for source files and then matching
them to the compile rule, the link rule updates all the headers that are
listed as prerequisites of a library or executable. The following example will
illustrate the sequence of steps. Let's say we have the auto-generated header
`hxx{config}` and the following `buildfile`:

```
exe{hello}: cxx{main} lib{hello}

lib{hello}: {hxx cxx}{hello} hxx{config}

hxx{config}: in{config}
```

The following steps will be performed when we match a rule to update
`exe{hello}` (for each step we list the target being "worked" on in []):

1. [`exe{hello}`] The link rule matches the `exe{hello}` target. As a first
   step, it matches all the libraries (`lib{hello}` in this case) and then
   updates (pre-generates) all the headers (none in this case) listed as
   prerequisites of the target.

2. [`exe{hello}`] Next the link rule synthesizes an object file dependency for
   `cxx{main}` with the result equivalent to the following `buildfile` fragment:

   ```
   exe{hello}: obje{main} lib{hello}

   obje{main}: cxx{main} lib{hello}
   ```

3. [`exe{hello}`] Finally, the link rule proceeds to matching the `obje{main}`
   and `lib{hello}` prerequisites.

4. [`obje{main}`] The `obje{main}` target is matched by the compile rule. As a
   first step, the compile rule matches all the library prerequisites. In this
   case it matches `lib{hello}`.

5. [`lib{hello}`] The `lib{hello}` target is matched by the link rule. As a
   first step, it matches all the libraries (none in this case) and then
   updates (pre-generates) all the headers (`hxx{config}` in this case) listed
   as prerequisites of the target (`hxx{hello}` is a static header and its
   update is a no-op).

6. [`lib{hello}`] Next the link rule synthesizes an object file dependency for
   `cxx{hello}` with the result equivalent to the following `buildfile`
   fragment:

   ```
   lib{hello}: hxx{hello} obj{hello} hxx{config}

   obj{hello}: cxx{hello}
   ```

7. [`lib{hello}`] Finally, the link rule proceeds to matching the `obj{hello}`
   prerequisite.

8. [`obj{hello}`] The `obj{hello}` target is matched by the compile
   rule. Since there are no library prerequisites, it can proceed directly to
   extracting header dependency information from `cxx{hello}`. If it happens
   to include `hxx{config}`, this header has already been pre-generated.

9. [`obje{main}`] The compile rule finishes matching the `obje{main}` target
   by extracting header dependency information from `cxx{main}`. If it happens
   to include `hxx{config}`, this header has already been pre-generated.

There are two key observations to note from this example: Firstly, both
pre-generation as well as the normal header (re)generation happen during the
match phase. This means that there is not as strong a "barrier" between header
generation and header inclusion as found in build system with a separate
pre-build step. While this results in better performance (since `build2` can
match other targets in parallel) it also means there is a greater chance of
things being done in the wrong order if we don't get the dependencies right.

Secondly, there are two alternative "pathways" that ensure pre-generation
happens before normal (re)generation: the first is via an explicit dependency
on a library that lists the generated header as its prerequisite
(`obje{main}`) and the second is as a "side effect" of the order in which the
link rule matches prerequisites (`obj{hello}`).

The first pathway is both robust (because it is based on explicit
dependencies) and what will naturally be used in the global context, for
example, when depending on a library from another project. But the second
pathway, as one would expect, has corner cases. The good news is that it is
limited to a fairly localized context (library or executable) and the corner
cases, if understood, a relatively easy to work around by explicitly arranging
for the first pathway.

Finally, in certain cases, it may be beneficial to disable pre-generation
to obtain better parallelism and thus performance. This can be done on the
header-by-header basis.

These two topics are discussed in the following sections.


## Header pre-generation corner cases

The corner cases of the "side effect" pathway discussed in the previous
section fall into two categories: "side effect sidestep" and "dependencies
between generated headers".

The side effect sidestep refers to the situation where the `obj*{}` target
(synthesized or not) that requires a generated header is matched via an
alternative dependency path that does not have the necessary side
effect. Consider the following `buildfile` as an example, assuming
`cxx{common}` includes generated `hxx{config}`.

```
exe{hello}: cxx{common hello} hxx{config}

exe{goodbye}: cxx{common goodbye}

hxx{config}: in{config}
```

Which, after synthesizing object file dependencies, will be equivalent to:

```
exe{hello}: obje{common hello} hxx{config}

exe{goodbye}: obje{common goodbye}

obje{common}: cxx{common}
obje{hello}: cxx{hello}
obje{goodbye}: cxx{goodbye}

hxx{config}: in{config}
```

In this example, whether `obje{common}` will be matched before or after
pre-generating `hxx{config}` will depend on which dependency path will trigger
its match first, `exe{hello} -> obje{common}` or `exe{goodbye} ->
obje{common}`. Which means it will be racy.

The are two ways to fix this. The first is to make sure all the dependency
paths have the necessary header pre-generation, for example:

```
exe{hello}: cxx{common hello} hxx{config}

exe{goodbye}: cxx{common goodbye} hxx{config}
```

The second way to fix this is to switch to the first, more robust
pre-generation pathway by introducing a utility library. For example:

```
exe{hello}: cxx{common hello} libue{hello-config}

exe{goodbye}: cxx{common goodbye} libue{hello-config}

libue{hello-config}: hxx{config}
```

Another variant of this corner case is a situation where the link rule is not
(or may not be) involved at all. For example, you may be only building object
files without linking them into a library or executable. For this case, the
first pathway is the only option (other than specifying an explicit
dependency on the generated header file, of course). For example:

```
obje{result}: cxx{result} libue{hello-config}

libue{hello-config}: hxx{config}
```

The second category of corner cases involves dependencies between generated
headers. While not very common, there are real-world examples of this
situation. For instance, the Qt project's `moc` compiler generates additional
source code for C++ classes that use certain Qt mechanisms. To achieve this in
a preprocessor-aware manner, `moc` takes into account included headers. While
its output is a source file, it's not uncommon for projects to include this
generated source file at the bottom of the file for which it was generated,
effectively turning it into a header. Here is an example of a `buildfile` that
illustrates this setup (we assume there is an ad hoc rule for `moc` that
generates `cxx{moc_gui}` from `hxx{gui}`):

```
exe{hello}: cxx{main} {hxx cxx}{gui}

exe{hello}: cxx{moc_gui}: include = adhoc  # Included by cxx{gui}.

cxx{moc_gui}: hxx{gui}
```

So far there are no issues in this setup: `cxx{moc_gui}` will be pre-generated
before `cxx{gui}` (which includes it) is matched (besides headers, the link
rule also pre-generates ad hoc source files).

But let's now add a generated configuration file which we will assume is
included by `hxx{gui}`:

```
exe{hello}: cxx{main} {hxx cxx}{gui} hxx{config}

exe{hello}: cxx{moc_gui}: include = adhoc

cxx{moc_gui}: hxx{gui}

hxx{config}: in{config}
```

Now we have a problem: because `hxx{gui}` includes `hxx{config}`, the latter
has to be generated before `cxx{moc_gui}`. But in our current setup the order
in which `cxx{moc_gui}` and `hxx{config}` are pre-generated is undefined
(and, in fact, is racy). (Note that falling back here to normal (re)generation
is not an option since `moc` simply ignores missing headers.)

To fix this we again switch to the first pathway by introducing a utility
library that establishes a proper order between the two pre-generation steps:

```
exe{hello}: cxx{main} {hxx cxx}{gui} libue{hello-config}

exe{hello}: cxx{moc_gui}: include = adhoc

cxx{moc_gui}: hxx{gui}

libue{hello-config}: hxx{config}

hxx{config}: in{config}
```

The way it works may not be immediately apparent: this relies on the fact that
the link rule will match the `libue{hello-config}` prerequisite of
`exe{hello}` before pre-generating `cxx{moc_gui}`. Matching
`libue{hello-config}` in turn triggers pre-generation of `hxx{config}`.


## Disabling header pre-generation

In certain situations you may obtain better parallelism and thus performance
by disabling pre-generation. The main bottleneck of pre-generation is the
"barrier" between pre-generating all the headers of a library or executable
and matching its other prerequisites, in particular source files. By disabling
pre-generation, we allow these two tasks to proceed concurrently.

An example of a situation where this could be beneficial is a large number of
generated headers that may take a while to produce, typically because they can
only be produced together with the corresponding source files.

In this case we can disable pre-generation provided that it's safe, that
is, these are not "macro headers", they are included with names unlikely
to clash, and they are not installed (see the Background section above
for details). For example, assuming the `codepage-*` header/source pairs
are expensive to generate:


```
exe{hello}: cxx{main}
exe{hello}: {hxx cxx}{codepage-utf8  \
                      codepage-utf16 \
		      codepage-utf32}: update = execute
```

Here the `update=execute` prerequisite-specific variable instructs the link
rule to use the default semantics (which is to update during the execute
phase) instead of updating the headers during match.

Note, however, that disabling pre-generation could just as likely hurt
performance since normal header (re)generation has its costs, namely potential
phase switch contention and having to restart the header dependency extraction
process after each (re)generated header. So it makes sense to measure the
performance of both approaches before deciding which one to use.


## Dummy header technique

The dummy header technique allows us to "reduce" the Installed header
mis-inclusion case to the Wrong header case for which `build2` has built-in
remapping support (see the Background section above for details). This is
achieved by maintaining a dummy header in the project's source directory that
will always be included before any installed header. Because we also have to
ship such a header in the project's distribution, this technique is only
applicable if it is sensible (or harmless) to distribute a pre-generated
header (there is currently no mechanism to override a distributed file's
contents). Good example where it is most likely neither is a configuration
file.

The dummy header technique involves the following steps. Here we use the
generated `version.hxx` header as an example (see the `version` module
documentation for details):

1. Create an empty `version.hxx` file in the project's source directory.
   Add it to `.gitignore` but also add it to the `git` repository:

   ```
   $ git add --force version.hxx .gitignore
   $ git commit -m "Add dummy version.hxx"
   ```

   After that, instruct `git` to ignore any changes to this file:

   ```
   $ git update-index --assume-unchanged version.hxx
   ```

   Anyone who clones your project will need to run this command so it probably
   makes sense to mention it in the relevant documentation.

2. In your `buildfile` arrange for this header to be distributed. It is also a
   good idea to disable its cleaning so that a cleaned state is identical to
   distributed. Also don't forget to disable pre-generation, which is the
   whole point of this exercise. For example:

   ```
   lib{hello}: {hxx cxx}{** -version}
   lib{hello}: hxx{version}: update = execute

   hxx{version}: in{version} $src_root/manifest
   {
     dist = true
     clean = ($src_root != $out_root)
   }
   ```

[proj-struct]: https://build2.org/build2-toolchain/doc/build2-toolchain-intro.xhtml#proj-struct
