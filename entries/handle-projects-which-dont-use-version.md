# How do I handle projects that don't use versions at all?

> I am trying to package a third-party project for `build2` but it doesn't use
> any versions at all. However, upstream may decide to start using some form
> of versioning in the future. How do I handle that?

A `build2` package must have a version so if upstream doesn't use any
versioning scheme, then we will need to come up with one. While adding a
version is not difficult, one aspect that we must consider is what happens
if/when upstream decides to start using versions. Ideally, if/when this
happens, we would want to switch to upstream versions while preserving the
correct ordering of existing releases. Taking this into account, there appears
to be two sensible options which are described in the following sections along
with their pros and cons.


## Use `0.0.Z` semver

With this option we start with semver in the hopes that if/when upstream
decides to start using versions, they will also pick semver.

To increase the likelihood of ending up with the correct ordering of existing
releases, it's a good idea to restrict ourselves to only use the patch
component keeping the major and minor components `0`. The thinking here is
that if/when upstream starts using versions, they will start either with
`0.1.0` or `1.0.0`, as is customary. And if that doesn't happen or if upstream
goes with something other than semver, then see the "Version epochs" section
below for plan B.

The main advantage of this approach is that we can use the
[`version`][version-module] build system module and all the `bdep` commands,
including [`bdep-release`][bdep-release], which expect semver.

The main drawback of this approach, if we stick to `0.0.Z`, is that our
versions are always alpha. But perhaps it is appropriate to keep a project
with upstream that doesn't use any form of versioning (and which to many
would signal immaturity) as alpha. And if the project is in fact mature, then
perhaps there is no reason to believe that upstream will adopt versioning and
it doesn't make sense to restrict ourselves to `0.0.Z`. However, in this case,
it may make more sense to use the second approach (date) since there probably
won't be any backwards-compatibility guarantees implied by semver components.

So, to sum up, this approach is more appropriate for immature projects that
are likely to adopt some form of versioning (hopefully semver) in the future.


## Use date as version

If the project is fairly mature and if upstream adopting versions is more
likely wishful thinking than a real possibility, then it may make sense to use
a date instead of semver, unless, perhaps, it's straightforward to map
upstream's backwards-compatibility guarantees back to semver (for example, if
upstream guarantees to never break binary or source backwards-compatibility,
then you could start with `1.0.0` and only ever increment the patch
component).

The date format that sorts appropriately as a version is `YYYYMMDD`, for
example, `20210824` (see [Package Version][package-version] for details).

The main advantage of this approach compared to semver is that you don't need
to think about which component you must increment for the next release. The
main drawback is that we are not longer able to use the
[`version`][version-module] build system module nor most of the `bdep`
commands (we can still use [`bdep-ci`][bdep-ci] and
[`bdep-publish`][bdep-publish] with a bit of effort). For details on
working with packages that don't use semver see the
"Use upstream version as is" section in ["How do I handle projects that
don't use semantic versioning?"](handle-projects-which-dont-use-semver.md)


## Version epochs

We can end up in a situation where we have started using one versioning scheme
(say, semver) but later upstream have started using a different scheme (say,
date). Or we could both end up using the same scheme but upstream have started
using version values that clash with those that we have released in the
past. To deal with such situations we have to use versioning epochs (see
[Package Version][package-version] for details). For example, if we currently
use semver:

```
name: libfoo
version: 0.0.3
```

To switch to upstream's date format:

```
name: libfoo
version: +2-20210824
```

Here `+2-` signifies the second versioning epoch (the first/default epoch is
`1`).

Note that the epoch can also be (explicit) `0` which signifies a "temporary"
versioning scheme before the first. Using the `0` epoch would be appropriate
in a situation where you know that upstream is going to start using versions
and you know that the scheme will be incompatible with what you will be using.
In other circumstances, `0` epoch is probably not worth the hassle.

[version-module]:   https://build2.org/build2/doc/build2-build-system-manual.xhtml#module-version
[bdep-release]:     https://build2.org/bdep/doc/bdep-release.xhtml
[bdep-ci]:          https://build2.org/bdep/doc/bdep-ci.xhtml
[bdep-publish]:     https://build2.org/bdep/doc/bdep-publish.xhtml
[upstream-version]: https://build2.org/bpkg/doc/build2-package-manager-manual.xhtml#manifest-package-version
[package-version]:  https://build2.org/bpkg/doc/build2-package-manager-manual.xhtml#package-version
