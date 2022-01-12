# How do I handle projects that don't use semantic versioning?

> I am trying to package a third-party project for `build2` but it doesn't
> use semantic versioning. Instead, it uses a date, like `20210824`. How
> do I handle that?

There are two common strategies to deal with such projects: you can try to
"coerce" the upstream version to something resembling a semantic version
(semver) or you can use the upstream version as is. The following sections
describe each strategy as well as their pros and cons.

## Coerce upstream version to semver

With this approach we try to derive a semver-like version from the upstream
version. For example, if upstream uses `20210824`, then we could derive
`2021.8.24` from it.

The main drawback of this approach is that while the derived version is
syntactically semver, it is definitely not semantically: `2021` is not a major
version, `8` is not a minor version, and `24` is not a patch version. Which
means there are many opportunities for misuse, most notably bogus version
constraints (`^2021.8.24`). Note, however, that strictly speaking this problem
is not limited to cases where upstream version is not semver: there are plenty
of packages (for example, the Boost libraries) which have versions that are
syntactically semver but not semantically.

On the other hand, this approach is fairly straightforward: since the derived
version is syntactically semver, we can use the [`version`][version-module]
build system module and all the `bdep` commands as long as we (and the users
of such a project) are paying attention. For example, while we shouldn't use
[`bdep-release`][bdep-release] to manage the version components, we can still
use it to release revisions.

With this approach the only extra action we need to take (compared to
packaging a project that already uses semver) is to add the
[`upstream-version`][upstream-version] `manifest` value as documentation. For
example:

```
name: hello
version: 2021.8.24
upstream-version: 20210824
...
```

## Use upstream version as is

While semver is recommended for new projects, a `build2` package may use other
version formats as long as they conform to the generalized [Package
Version][package-version] specification. With this approach we use the
upstream version as is or with minor modifications to fit the
specification. For example, if upstream were using `2021-8-24`, we would need
to get rid of `-`, for example, by converting it to `20210824` (you can add
`upstream-version` with the original if you like).

The main advantage of this approach is that the package version matches
upstream and it's impossible to confuse it for semver. The main drawback is
that the preparation and management of such a package deviates from
semver-based packages. Specifically, we are not longer able to use the
[`version`][version-module] build system module nor most of the `bdep`
commands (both of these expect a project with semver). The rest of this
section discusses this approach in detail. For an example of a package that
uses this approach see [`byacc`][byacc].

Instead of using the `version` module to automatically extract the version
from `manifest` we have to extract it ourselves. For example, this is what our
`build/bootstrap.build` could look like:

```
project = hello
version = $process.run(sed -n -e 's/^version: (.+)/\1/p' $src_root/manifest)

dist.package = $project-$version

using config
using test
using install
using dist

```

While we cannot use `bdep` to manage such a package (we will have to use the
build system and optionally `bpkg` directly), we can use the special modes of
the [`bdep-ci`][bdep-ci] and [`bdep-publish`][bdep-publish] commands to test
and publish such a package. To be able to use these `bdep` commands we have to
configure an out of source build of our package and then configure the source
directory as forwarded to this out of source build (see [`b(1)`][b] for
details). For example, assuming our project is in the `hello/` directory:

```
$ ls
hello/

$ b configure: hello/@hello-gcc/ config.cxx=g++
$ b configure: hello/@hello-gcc/,forward
```

Now we can run the `bdep-ci` and `bdep-publish` commands from our projects
provided we pass the `--forward` option (for `bdep-publish` we also need
to explicitly specify the repository section since it cannot be deduced
from the version):

```
$ cd hello/

$ bdep ci --forward
$ bdep publish --forward --section=stable
```

[version-module]:   https://build2.org/build2/doc/build2-build-system-manual.xhtml#module-version
[bdep-release]:     https://build2.org/bdep/doc/bdep-release.xhtml
[bdep-ci]:          https://build2.org/bdep/doc/bdep-ci.xhtml
[bdep-publish]:     https://build2.org/bdep/doc/bdep-publish.xhtml
[upstream-version]: https://build2.org/bpkg/doc/build2-package-manager-manual.xhtml#manifest-package-version
[package-version]:  https://build2.org/bpkg/doc/build2-package-manager-manual.xhtml#package-version
[byacc]:            https://github.com/build2-packaging/byacc
[b]:                https://build2.org/build2/doc/b.xhtml
