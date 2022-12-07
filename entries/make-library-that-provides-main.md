# How do I make a C/C++ library that provides `main()`?

> I am dealing with a C++ testing framework library that optionally must
> provide an implementation of `main()`. How do I arrange for this?

Note: the following functionality is available in `build2` 0.16.0 and later.

It's common for C/C++ testing frameworks to include an additional library that
provides the default implementation of `main()`. However, `main()` is a
special function and providing its implementation in a library often requires
jumping through extra hoops.

While these extra hoops are reasonably well understood if `main()` is provided
by a static library (we need to link the library as "whole archive"), things
get platform-specific if it is provided by a shared library. As a result,
using shared libraries with `main()` is not recommended and below we focus on
the static library approach.

To illustrate the setup, let's develop a basic C++ testing framework called
`libmytest`. You can find the complete source code in the [`libmytest`
repository][libmytest] (see the commit history for the before/after `main()`
changes).

We start the project like this:

```
$ bdep new -t lib,no-version libmytest
$ cd libmytest
```

This gives us the standard `buildfile` in `libmytest/` that begins like this
(we've removed the library importation variables since our testing framework
won't have any dependencies):

```
lib{mytest}: {hxx ixx txx cxx}{*}

# Build options.
#
cxx.poptions =+ "-I$out_root" "-I$src_root"

{hbmia obja}{*}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
{hbmis objs}{*}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

# Export options.
#
lib{mytest}: cxx.export.poptions = "-I$out_root" "-I$src_root"

liba{mytest}: cxx.export.poptions += -DLIBMYTEST_STATIC
libs{mytest}: cxx.export.poptions += -DLIBMYTEST_SHARED
```

Next we implement our testing framework API (see [`mytest.hxx`][mytest.hxx]
for an example) and make sure it works with a user-provided `main()` (see
[`tests/basics/`][basics-pre]).

Once this is done, our next step is to provide `main()` as part of the
library. Specifically, we want to provide a separate library target, say,
`lib{mytest-main}` that users of our testing framework can choose to link
instead of `lib{mytest}` if they wanted to use our `main()` implementation.

As a first step, let's go ahead and add [`main.cxx`][main.cxx] in our
library source directory and exclude it from `lib{mytest}`:

```
lib{mytest}: {hxx ixx txx cxx}{* -main}
```

We've decided that we will be using a static library to provide `main()` and
the straightforward next step is to go ahead and add `liba{mytest-main}`:

```
./: lib{mytest}: {hxx ixx txx cxx}{* -main}
./: liba{mytest-main}: cxx{main} lib{mytest}

liba{mytest-main}:
{
  bin.whole = true
  cxx.export.libs = lib{mytest}
}
```

Notice the `bin.whole` variable which instructs `build2` to link this static
library in the "whole archive" mode.

Then in `tests/basics/buildfile` we change:

```
import libs = libmytest%lib{mytest}
```

To:

```
import libs = libmytest%liba{mytest-main}
```

Finally we get rid of our own `main()` in `tests/basics/driver.cxx`, try to
build and run the test, and it seems to work.

But if we look closer, we will notice an undesirable trait of this
straightforward approach: because we link `lib{mytest}` via
`liba{mytest-main}`, we always end up with static `liba{mytest}`. This is due
to `build2`'s library linking semantics: it links static libraries to static
and shared libraries \- to shared, which is a sensible default. Ideally, what
we would like is for the static/shared variant of `lib{mytest}` to be picked
as before, based on the project's executable linking preferences, while always
linking static `liba{mytest-main}`.

While this behavior can be achieved, the setup is a bit more elaborate. Here
are all the relevant changes to `libmytest/buildfile`:

```
src = cxx{* -main}

./: lib{mytest}: $src {hxx ixx txx}{*}
./: lib{mytest-main}

liba{mytest-main}: liba{mytest-main-static}
libs{mytest-main}: liba{mytest-main-shared}

liba{mytest-main-static}: obja{main-static} liba{mytest}
liba{mytest-main-shared}: obja{main-shared} libs{mytest}

obja{main-static}: cxx{main} liba{mytest}
obja{main-shared}: cxx{main} libs{mytest}

liba{mytest-main-static mytest-main-shared}: bin.whole = true

# Build options.
#
cxx.poptions =+ "-I$out_root" "-I$src_root"

# Note: we should not define any of these when compiling cxx{main}.
#
#{hbmia obja}{*}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
#{hbmis objs}{*}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

hbmia{*}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
hbmis{*}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

obja{$name($src)}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
objs{$name($src)}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

# Export options.
#
lib{mytest}: cxx.export.poptions = "-I$out_root" "-I$src_root"

liba{mytest}: cxx.export.poptions += -DLIBMYTEST_STATIC
libs{mytest}: cxx.export.poptions += -DLIBMYTEST_SHARED

liba{mytest-main}: cxx.export.libs = lib{mytest}
libs{mytest-main}: cxx.export.libs = liba{mytest-main-shared} lib{mytest}
```

The trick here is to use binless `liba{mytest-main}` and `libs{mytest-main}`
as selectors for the correct `lib{mytest}` variant. However, there are a few
other nuances so let's examine the key parts in more detail:

```
./: lib{mytest-main}

liba{mytest-main}: liba{mytest-main-static}
libs{mytest-main}: liba{mytest-main-shared}
```

This is binless `lib{mytest-main}` that depends on binful
`liba{mytest-main-static}` or `liba{mytest-main-shared}`. So we actually
have two static libraries that provide `main()`. We will see in a moment
why.

```
liba{mytest-main-static}: obja{main-static} liba{mytest}
liba{mytest-main-shared}: obja{main-shared} libs{mytest}

obja{main-static}: cxx{main} liba{mytest}
obja{main-shared}: cxx{main} libs{mytest}
```

The `liba{mytest-main-static}` and `liba{mytest-main-shared}` static libraries
are built from the same `main.cxx` but depend on `liba{mytest}` or
`libs{mytest}`, respectively. In particular, this means that when `main.cxx`
is compiled for `liba{mytest-main-shared}`, it will "see" symbols from
`lib{mytest}` as DLL-exported, while when compiled for
`liba{mytest-main-shared}` \- as ordinary. This is the reason for having two
static libraries that provide `main()`.


```
liba{mytest-main}: cxx.export.libs = lib{mytest}
libs{mytest-main}: cxx.export.libs = liba{mytest-main-shared} lib{mytest}
```

Here we make `lib{mytest}` an interface dependency of `lib{mytest-main}`.  We
also make `liba{mytest-main-shared}` an interface dependency of
`libs{mytest-main}` so that it gets linked to the users of the library. Note
that for `liba{mytest-main}`, `liba{mytest-main-static}` is linked
automatically as an implementation dependency.

Finally, we need to tweak the `cxx.poptions` assignment to make sure none
of the `LIBMYTEST_*_BUILD` macros apply when compiling `main.cxx`:

```
# Build options.
#
cxx.poptions =+ "-I$out_root" "-I$src_root"

# Note: we should not define any of these when compiling cxx{main}.
#
#{hbmia obja}{*}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
#{hbmis objs}{*}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

hbmia{*}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
hbmis{*}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD

obja{$name($src)}: cxx.poptions += -DLIBMYTEST_STATIC_BUILD
objs{$name($src)}: cxx.poptions += -DLIBMYTEST_SHARED_BUILD
```

Note also that the users of our testing framework will now import
`lib{mytest-main}` instead of `liba{mytest-main}`, which is also a good thing:

```
import libs = libmytest%lib{mytest-main}
```

To verify that our implementation links the correct library variants we can
use the following command lines:

```
b -v tests/basics/exe{driver}
b -v tests/basics/exe{driver} config.bin.exe.lib=static
```

[libmytest]: https://github.com/build2-packaging/libmytest
[mytest.hxx]: https://github.com/build2-packaging/libmytest/blob/master/libmytest/mytest.hxx
[main.cxx]: https://github.com/build2-packaging/libmytest/blob/master/libmytest/main.cxx
[basics-pre]: https://github.com/build2-packaging/libmytest/tree/ff09768f3301f9e5c4ebe72c9004fa137693102b/tests/basics
