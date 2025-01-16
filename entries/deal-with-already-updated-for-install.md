# How do I deal with `target already updated but [not] for install` errors?

> I get an error like this:
>
> ```
> error: incompatible exe{hello} build
>   info: target is already updated but for install
> ```
>
> Or:
>
> ```
> error: incompatible lib{hello} build
>   info: target is already updated but not for install
> ```
>
> What does it mean and how do I deal with it?

Real-world projects often need to build things slightly differently
_for-install_ (that is, as part of the `install` operation) compared to
_for-development_ (that is, during normal `update`). Typically, they may need
to adjust paths to external resources, such as data assets, plugins, etc.
Additionally, executables and shared libraries for POSIX targets are re-linked
without `rpath` (or with different `rpath`) when updated for-install.

As a result, `build2` provides support for customizing the build for-install
(see the `for_install` target-specific variable). However, there is a price to
pay: now the for-install build of, say, a library is not necessarily
compatible with the one for-development. So if you have two targets that link
a library and one wants it in the for-install flavor and the other -- in
for-development, then `build2` detects this and fails with the above
diagnostics.

This can happen in various situation but most commonly when you try to run
during the build an executable, such as a source code generator, that you are
updating for install. In this case you need it both for-install (to install)
and for-development (to execute) at the same time and you will get the above
error mentioning the executable target.

A variant of the same issue would be an executable that you again wish to run
during the build but this time it's not installed (for example, it's some
internal utility you need to run during the build). The executable itself
won't have a problem in this case since it's only needed to be updated
for-development. However, it may depend on a library and if that library is
installed, then you will again get the above error but this time mentioning
the library.

The for-install situation is not dissimilar to cross-compilation: if you are
building an executable for a target platform that differs from the host, then
you won't be able to execute it either. The solution to this problem is well
understood, if not exactly simple: you need a separate build of the executable
in the host configuration. The same overall approach should work for the
for-install problem: use a separate build configuration for installing where
you don't need to run the executable (for example, by reusing cached results
of its execution). For an example of this approach see the
[CLI](https://github.com/codesynthesis-com/cli) compiler which runs itself
during the build but only in the for-development setup.
