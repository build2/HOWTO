# How do I make a header-only C/C++ library?

> My library doesn't have any sources, only headers. How do I handle this
> in `build2`?

In `build2` a header-only library (or a module interface-only library) is not
a different kind of library compared to static/shared libraries but is rather
a binary-less, or _binless_ for short, static or shared library. As a result,
it is possible to have a library that has a binless static and a binary-ful
(_binful_) shared variants or is binless on some platforms and binful on
others. Note also that in `build2` binless libraries can depend on binful
libraries and are fully supported where the `pkg-config` functionality is
concerned.

To make a library target binless, simply don't list any source files as its
prerequisites. For example:

```
lib{hello}: hxx{hello}
```

If you are creating a new binless library with [`bdep-new`][bdep-new] then you
can produce a simplified `buildfile` by specifying the `binless` option, for
example:

```
bdep new -l c++ -t lib,binless libhello
```

One counter-intuitive aspect of having a binless library that depends on a
system binful library, for example, `-lm` or [`-pthread`][link-pthread], is
that you still have to specify the system library in both `*.export.libs` and
`*.libs` because the latter is used when linking the static variant of the
binless library (see [Library Exportation and Versioning][intro-lib] for
background). For example, this is the correct setup:

```
lib{hello}: hxx{hello}

if ($cxx.target.class != 'windows')
{
  cxx.libs += -lm
  lib{hello}: cxx.export.libs = -lm
}
```

## Metadata libraries

One special case of a binless library is a _metadata library_, a library that
has no sources or headers and whose only purpose is to export dependency on
other, normally platform-specific system library or libraries. By convention,
such libraries are named by appending `-meta` (or `_meta`, `Meta`; depending
on the original library's naming convention) to the original library name
(it's also a good idea to check that the resulting name does not by any chance
clash with anything existing).

Here is a `buildfile` from [libpthread-meta][libpthread-meta], an example
metadata library for linking `-pthread`:

```
[rule_hint=c] lib{pthread-meta}:

c.libs = -pthread
lib{pthread-meta}: c.export.libs = -pthread
```

Note that because this library has neither sources nor headers, we have to
explicitly specify which rule (in this case, the C link rule) should build
this library.

[bdep-new]: https://build2.org/bdep/doc/bdep-new.xhtml
[link-pthread]: link-pthread.md
[intro-lib]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#intro-lib
[libpthread-meta]: https://github.com/build2-packaging/libpthread-meta
