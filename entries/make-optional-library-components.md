# How do I make components of a library optional?

> I have some optional components/features in my library. For example, I have
> JSON serialization support that is only used by some consumers of the
> library. What is the best way to handle this in `build2`?

The best way to handle this situation is to not make the components optional.
That is, unconditionally include them into the library. The reason for this is
that any kind of configurability, like all variability, increases complexity,
which means sooner or later someone will get confused and make a mistake. So
if a component is not huge and doesn't require any additional dependencies,
then the recommendation is to include it into your library unconditionally. If
your library is linked statically, then the linker will automatically drop the
unused components (provided they are placed into separate translation
units). And in the case of the shared library, you sidestep the issue of
some library builds being incompatible with some the consumers (because of
the missing components).

If you still think you must make a component optional, then the best approach
is to factor it into a separate library and, if it requires additional
dependencies, into a separate package. With this approach the consumers of the
library express the desire for optional components using the familiar and well
understood mechanisms: importing/linking an additional library and depending
on an additional package. This approach also composes well when shared
libraries are used. Note that if considering this approach feels like an
overkill, then it's a good indication that you should include the component
into the main library unconditionally.

If for some reason you are unable to factor optional components into separate
libraries/packages (for example, because you are packaging a third-party
library), then you can arrange for optional components in a single
library/package using configuration variables (see [Project
Configuration][proj-config] for background). Specifically, you would define a
boolean configuration variable for each optional component with the `false`
default value and then use it in your `buildfile` to include/exclude certain
translation units from the build. If an optional component requires additional
dependencies, then you would make them conditional on this configuration
variable being `true` (see [`depends` package `manifest`
value][manifest-package-depends] for details). The consumers of your library
will then need to specify the desired optional features as part of the package
dependency configuration (again, see [`depends` package `manifest`
value][manifest-package-depends] for details). You will also most likely want
to convey the configuration as part of the library metadata (see [How do I
convey additional information (metadata) with executables and C/C++
libraries?][metadata] for background). See [`libasio`][libasio] for a real
package that illustrates this arrangement.

Note also that the configuration variables approach can also be used to create
the "single package with multiple libraries" arrangement (or even a hybrid of
the two) in case you don't wish to factor separate libraries that require
extra dependencies into separate packages. See [`libevent`][libevent] for a
real package that illustrates this arrangement.

[proj-config]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#proj-config
[manifest-package-depends]: https://build2.org/stage/bpkg/doc/build2-package-manager-manual.xhtml#manifest-package-depends
[metadata]: convey-additional-information-with-exe-lib.md
[libasio]: https://cppget.org/libasio
[libevent]: https://cppget.org/libevent
