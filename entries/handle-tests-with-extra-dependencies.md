# How do I handle tests that have extra dependencies?

> I would like to use a testing framework in the tests of my library. But the
> testing framework is not needed for the library itself. What is the best way
> to arrange this?

If our tests (or examples, benchmarks, etc) depend on another package (such as
a testing or benchmarking framework) that is not needed by the library itself,
then it is recommended to factor the tests into a separate package (usually in
the same `git` repository). This way the extra dependency will only be built
if and when the users of our library wish to run the tests. This article
describes how to implement this setup.

By convention, such separate tests/examples/benchmarks packages use the main
package name with the `-tests`, `-examples`, or `-benchmarks` suffix. For
example, if our library is called `libhello`, then the separate tests package
would be called `libhello-tests`. While you can restructure your existing
repository manually, it's recommended to first create the desired structure
with [`bdep-new`][bdep-new] on the side and then use it as a guide, moving
individual pieces around as necessary.

Following with the `libhello` example, below are the `bdep-new` command
that initialize a `git` repository with two suitably-named packages:

```
$ bdep new -t empty libhello
$ cd libhello

$ bdep new --package -l c++ -t lib libhello
$ bdep new --package -l c++ -t exe libhello-tests
```

The contents of the `libhello` package can be replaced wholesale with your old
`libhello` package. The only two changes that we need to make here are to move
the tests to `libhello-tests` (see below) and add the following value to
`libhello/manifest` (use `examples` for examples and `benchmarks` for
benchmarks):

```
tests: libhello-tests == $
```

This value is what tells the `build2` package manager that tests for
`libhello` live in the separate `libhello-tests` package (see [`tests`,
`examples`, `benchmarks`][manifest-tests] for details).

For `libhello-tests` we used the standard executable project type (`-t exe`
above) but for tests we don't need everything. In particular, we can remove
the `install` module from `libhello-tests/build/bootstrap.build`. If your old
tests in `libhello` were in a subproject (as is customary), then you can
replace the rest in `libhello-tests` with that. You will also most likely want
to tweak `libhello-tests/manifest` (summary, license, etc).  Note that
normally the two packages use the same version with
[`bdep-release`][bdep-release] taking care of both automatically (see
[Developing Multiple Packages and Projects][guide-dev-multi] for details).

Finally, you can add the dependency on the testing framework in
`libhello-tests/manifest`. Note that while it may seem like a good idea to
also add a dependency on `libhello` here (after all, the tests need the
library), this is not necessary: the above `tests` manifest value implies such
a dependency.

Note also that when testing such a library with [`bdep-ci`][bdep-ci], you
submit the library, not the tests (all the separate tests/examples/benchmarks
will be pulled in automatically):

```
$ bdep ci -d libhello
```

[bdep-new]:         https://build2.org/bdep/doc/bdep-new.xhtml
[bdep-release]:     https://build2.org/bdep/doc/bdep-release.xhtml
[bdep-ci]:          https://build2.org/bdep/doc/bdep-ci.xhtml
[manifest-tests]:   https://build2.org/bpkg/doc/build2-package-manager-manual.xhtml#manifest-package-tests-examples-benchmarks
[guide-dev-multi]:  https://build2.org/build2-toolchain/doc/build2-toolchain-intro.xhtml#guide-dev-multi
