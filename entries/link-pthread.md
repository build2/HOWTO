# How do I link the `pthread` library?

> How do I link the `pthread` library?

> How do I deal with undefined `pthread_create` when using C++11 `<thread>`?

The short version: for modern `pthread`-based targets (Linux, FreeBSD, MacOS,
MinGW) add the `-pthread` option to `*.libs` (you can think of it as a special
alias for `-lpthread`). For example, in a C++ project:

```
if ($cxx.target.system != 'win32-msvc')
  cxx.libs += -pthread
```

Additionally, in case of a library, if the `pthread` API (or one of the C++11
multi-threading APIs found in `<thread>`, `<mutex>`, `<condition_variable>`,
or `<future>`) are used in the library's interface (for example, called from
inline/template functions), then also add `-pthread` to `*.export.libs`. For
example:

```
if ($cxx.target.system != 'win32-msvc')
{
  cxx.libs += -pthread

  lib{foo}: cxx.export.libs += -pthread
}
```

The rest of this article provides some background on the underlying problem
and shows how to handle legacy `pthread`-based targets.


## Background

In the old days, building multi-threaded applications required special actions
both during compilation and during linking. Specifically, during compilation
we would need to define a macro, typically `_REENTRANT`, to make sure that we
use thread-safe versions of the C/POSIX APIs. And during linking we would need
to link the `pthread` library, typically as `-lpthread`.

To help achieve this in a portable manner, the `-pthread` option was invented
with the expectation that the user will specify it both during compilation
(where it would be translated to the necessary macro definitions) and during
linking (where it would be translated to `-lpthread` or equivalent).

Two things have been happening since those old days: Firstly, most modern
operating systems have been moving to the always thread-safe C/POSIX APIs thus
no longer requiring any macros during compilation (see
[`__libc_single_threaded`][libc-single-threaded] for an example of an
alternative way to provide single-threaded optimizations).  And, secondly, the
`pthread` library were being merged into `libc` thus no longer requiring
`-lpthread` during linking.

The first change is complete, at least for the commonly-used general-purpose
operating systems: Linux with `glibc`, FreeBSD, MacOS, and MinGW. The second
change is still happening: MacOS were already there for some time, Linux is
[since `glibc` 2.34][glibc], while MinGW and FreeBSD still require
`-lpthread`, at least as of version FreeBSD 13 and MinGW GCC 11.

This background should make the motivation behind the above recommendation
clear: we normally no longer need to specify `-pthread` during compilation and
by using `-pthread` instead of `-lpthread` during linking we allow the
compiler to translate it into nothing if linking a separate library for
`pthread` is no longer necessary on the target platform.


## Legacy and "high touch" targets

There are situations where we may still need to specify `-pthread` during
compilation. These include:

1. Legacy targets which still have the single/multi-threaded C/POSIX APIs
   distinction and thus require macro definitions.

2. Legacy third-party libraries that follows the same approach for their
   APIs.

3. Targets where multi-threading requires a "high-touch" approach, such
   as Emscripten.

While it is tempting to think that we can support such targets seamlessly in
our libraries (for example, by specifying `-pthread` in `*.export.poptions`),
it's important to realize that for such targets *all* code that's going into
the application must be compiled with `-pthread`, not just the code that's
involved with multi-threading. As a result, the only sensible way to support
such targets is for the user to configure their entire build to use `-pthread`
by specifying it as a compiler mode option:

```
config.cxx="g++ -pthread"
```

[glibc]: https://developers.redhat.com/articles/2021/12/17/why-glibc-234-removed-libpthread
[libc-single-threaded]: https://www.gnu.org/software/libc/manual/html_node/Single_002dThreaded.html
