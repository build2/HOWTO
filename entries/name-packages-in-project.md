# How should I name packages when packaging third-party projects?

> I am trying to package a third-party project which provides both a library
> and an executable. I understand it makes sense to split them into separate
> packages but how should I name them?

Unless you have good reasons to deviate, you should call the library with the
`lib` prefix. To use a concrete example, there is the [`lz4` project][lz4]
which provides both the library and the command line utility. To package this
project, call the project (that is, the `git` repository) `lz4`, the library
package `liblz4`, and the utility package `lz4`.

Here are the [`bdep-new`][bdep-new] commands that establish this structure:

```
$ bdep new -t empty lz4
$ cd lz4

$ bdep new --package -l c -t lib liblz4
$ bdep new --package -l c -t exe lz4
```

Note that even if you are planning to start by only packaging the library, you
should still use the `lib` prefix since adding it later will be disruptive.
You may also want to establish a multi-package repository structure to make it
easier to add other packages later.

Why not always package libraries with the `lib` prefix, a la Debian? While it
would have been nice to be able to reliably deduce the package type from its
name and avoid any potential future conflicts with executables, it would have
also meant that the package name and the upstream name would often no longer
match (using the `lib` prefix is unfortunately far from the norm). While it
would have probably not been a big deal for the users (after all, Debian is
doing fine with this decision), it would have likely been a big deal for the
authors should they at some point consider switching to `build2` in upstream
(they would have had to rename their projects).

As a result, in `cppget.org` we have adopted the following policy: a library
is accepted without the `lib` prefix provided both of the following holds:

1. The upstream project does not provide an executable with the same name.

2. No other well-known project provides an executable with the same name
   (checked by searching the Debian package repository).

[bdep-new]: https://build2.org/bdep/doc/bdep-new.xhtml
[lz4]: https://github.com/lz4/lz4/
