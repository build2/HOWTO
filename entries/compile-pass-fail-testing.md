# How do I do compile pass/fail testing?

> How do I test that certain template instantiations or other constructs compile, or fail to compile?

When leveraging the power of static type safety in C++, we often rely on the compiler to reject certain constructs—instantiations or operations—that we consider invalid.

For example, we might have a type template that should only be instantiated with integral types:

```cpp
template <typename T>
class foo
{
  static_assert(std::is_integral_v<T>);
};
```

We would like to be confident that our assumptions about this type hold, and that it really does compile successfully only when `T` is an integral type. In other words, we would like to test that:

1.  `foo<T>` compiles when `T` is an integral type; *and*
2.  `foo<T>` *fails* to compile when `T` is *not* an integral type.

Generally, there are situations where we would like the compiler to act as a safety guard, allowing compilation to succeed only when certain conditions are true, and failing to compile otherwise. And we would like to *ensure* that it actually works that way, via testing. In other words, we want some to way to test that certain code compiles—that is, compile *pass* testing—… and that certain code reliably *fails* to compile—that is, compile *fail* testing.

Compile pass/fail testing is surprisingly tricky. The language itself does not concern itself with code that’s “supposed to fail”. If code is ill-formed, that’s the end of the C++ standard’s interest in it. There is no way, in general, to merely *check* that some code is ill-formed, yet still end up with well-formed code.

With so little help from the language, we are forced to rely on tooling to detect compilation successes and failures… but even *that* is trickier than you’d think. Build systems are generally not in the business of accepting failure. They are designed to be given a set of steps (or to generate those steps from a computed dependency graph), and then execute them, and if there are any failures at any point… they’re done; they just stop, and report that the whole build task failed.

To get compile pass/fail testing to work, programmers have often resorted to going beyond their build system, [using external scripts](https://youtu.be/hMn_dCae00g), or other such skulduggery. We will attempt to do compile pass/fail testing using only `build2`.

But before we get into how to do that, there are some alternatives and dangers to be aware of.

## Use the language wherever possible

Wherever possible, one should use the language to facilitate testing, if only to ensure that the tests can be carried out portably. Unfortunately, for compile pass/fail testing, this is not possible in the general case. However, there are some things we can do.

For example, the code above could be refactored as:

```cpp
template <typename T>
  requires std::is_integral_v<T> // or std::integral<T>
class foo
{
  // You can optionally leave the original static assert in place, as a
  // "just-to-make-sure" or "fuse" check.
  static_assert(std::is_integral_v<T>);
};
```

Now our “compile pass/fail tests” can be done using `requires` expressions:

```cpp
// Note: Using std::declval() ensures we're testing *just* the validity
// of the type, and not of default construction.
TEST(requires { std::declval<foo<int>>(); });
TEST(requires { std::declval<foo<unsigned int>>(); });
TEST(not requires { std::declval<foo<float>>(); });
TEST(not requires { std::declval<foo<std::string>>(); });
```

Unfortunately, this kind of refactorization is not always possible, practical, or desirable. Sometimes the code being tested cannot be modified. Sometimes the requirements cannot be easily expressed in a class-level `requires` clause, or cannot be expressed at all. Sometimes it’s simply impractical. Imagine, for example, a class that has multiple dependent types:

```cpp
template <typename T>
  // If we had a requires clause here, it would have to be the logical
  // conjugation of *ALL* of the requires clauses for all of the member
  // variables... *PLUS* any additional requirements of the type itself.
  //
  // This would impose a significant maintenance burden, as you would
  // have to continually ensure that *ALL* of the requirements for *ALL*
  // of the member types (and any base classes, etc.) stay in sync.
struct foo
{
  bar<T> a;
  baz<T> b;
  qux<T> c;
};
```

And of course, there are other things that we cannot use `requires` to check, like whether something can be done at compile time.

Where we can no longer leverage the language to detect when something will fail to compile, we are forced to rely on the build system (or other shenanigans, but let’s focus on what we can do with `build2`).

## Be especially careful with fail testing

Compile *pass* testing is… sorta-kinda “easy”. If some code fragment is supposed to compile successfully, one can simply include it in any source code file, and if that source file does compile, that is a successful compile-pass test. Granted, there is a lot of hand-waving going on in the previous sentence. For one, it is assumed that is *possible* to simply embed that code fragment in that source code file without otherwise impacting the way that file’s source code is interpreted. That will not always be true. There is also the challenge of determining, if the compilation fails, whether it failed due to the specific code fragment, and not for some other reason. But these complications are merely challenges, not show-stoppers.

Compile *fail* testing, however, is extremely difficult.

The primary challenge with compile fail testing is ensuring that the failure is due to the reasons expected. For example, suppose we want to test that the `foo` class template above will not compile with a `float` template parameter. We might think to write a test source code file that looks like this:

```cpp
#include <libfoo/foo.hxx>

auto f = foo<float>{};
```

In theory, the only thing that should make this source file fail to compile is the fact that `foo` cannot be instantiated with `float`. If you replace `float` with `int`, then it should compile. But the “should” there is doing a lot of lifting. Here are just some of the reasons why that source code file may fail to compile:

*   The header `<libfoo/foo.hxx>` may not be found (or the wrong header is found).
*   The template class `foo` may not be found (maybe we forgot the namespace?).
*   We may have forgotten other template arguments.
*   `foo<float>` may be instantiable, but it may not have a default constructor.
*   Maybe we made a typo and wrote `fo<float>` or `foo<flaot>`.
*   We may have misconfigured the build system (maybe we forgot a required compiler flag).
*   We may have triggered an internal compiler error.
*   There may be something wrong with the environment (maybe we ran out of disk space).
*   Maybe the stars aren’t aligned correctly, or we misspoke when chanting “klaatu barada nikto”, or… who knows what else?

The reality is that there is nothing we can do to *ensure* that when a source code file fails to compile, that it is due to the reason we *expect* it to fail. Thus, even a compile-fail tests *succeeds* (that is, it fails to compile, as expected), we can’t be sure that it proves what we were trying to verify.

Note that this problem is not specific to compile-fail tests. This problem exists for *all* tests. Even the simplest compile-pass test does not necessarily prove what is being tested. Suppose `auto f = foo<int>{};` compiles, as it should… except that it only compiled to a compiler bug, and would otherwise have failed.

The moral is that no test, and no amount of testing, can *ever* prove with absolute, 100% certainty that code is correct. However, every well-written and executed test can increase confidence, getting you asymptotically *closer* to 100%.

While compile-fail tests *on their own* cannot provide much confidence, when coupled with compile-pass testing—and all other testing—they can still be a very useful tool. So with that said, here are some tips to help make compile-fail testing somewhat more reliable, to help increase the amount of confidence it can provide:

*   Compile-fail tests should be *absolutely minimal*. There should be *nothing* in a compile-fail source code file other than the code that should fail, and the bare minimum of supporting code that would allow it to compile otherwise (if it were not going to fail).

    Compile-pass tests should *also* be absolutely minimal, so that, should they fail, one can be reasonably certain that the code-under-test is the cause, and not something else. This also goes hand-in-hand with the next tip.

*   Where possible, compile-fail tests should be paired with a corresponding compile-pass test(s), which is identical *except* for the key thing being tested that makes the compile-fail code invalid. This will help increase confidence that the cause for failure is the expected reason.

    As implied in the tip above, if you have two or more near-identical, absolutely minimal source code files, and some compile while the others do not, then it is *probably* due to the differences between the files. If the only differences are the things being tested, then that will increase your confidence that the tests are testing what they should be, and that the results mean what you want them to mean.

*   The compiler output—the error messages—of a compile-fail test should be examined from time-to-time, to verify that the failure is occurring for the reasons expected.

    This should only be needed on rare occasions, such as when the test is first written (or first expected to be valid), before major releases, or during audits. Assuming a well-written compile-fail test (absolutely minimal, and paired with compile-pass tests), then once it is found to be valid, it can be assumed to remain valid unless something major changes (such as a new version of C++ being used).

## Implementing compile pass/fail testing in build2

The easiest way to do compile pass/fail testing is to dedicate entire directories, or specific file name patterns, to the purpose. Either way will allow you to use wildcards to select multiple source code files.

Once you can select all your compile-pass and compile-fail test files, you can use ad-hoc rules to do the actual tests.

For example, you could create two directories `compile-pass` and `compile-fail` as subdirectories of the `tests` directory. (For an executable project that doesn’t have a `tests` directory by default, you could simply create one, and add a `buildfile` containing only the line `./: */`.)

```
lib<name>
├── 📁 build
├── 📁 lib<name>
├── 📂 tests
│   ├── 📁 basics
│   ├── 📁 build
│   ├── 📂 compile-fail
│   │   ├── 📄 buildfile
│   │   ├── 📄 fail-test-1.cxx
│   │   ├── 📄 fail-test-2.cxx
│   │   └── 📄 <… more tests …>
│   ├── 📂 compile-pass
│   │   ├── 📄 buildfile
│   │   ├── 📄 pass-test-1.cxx
│   │   ├── 📄 pass-test-2.cxx
│   │   └── 📄 <… more tests …>
│   └── 📄 buildfile
├── 📄 buildfile
├── 📄 manifest
├── 📄 README.md
└── 📄 repositories.manifest
```

The `buildfile` for compile-pass testing simply has to attempt to compile each test source code file. If that succeeds, the test succeeds.

```
# <root>/tests/compile-pass/buildfile

# When there are no tests, build2 will try to "build" an empty
# directory, which will trigger an error.
#
# The following line will silence the error by trying to "build" all the
# subdirectories (even if there are none).
./: */

# For each source file in this directory...
for t: cxx{*}
{
  # ... create an alias named after the file, dependent on the file.
  ./: alias{$name($t)}: $t
  % test # This ad hoc recipe will only run during the test command.
  {{
    src = $path($<)
    out = $out_base/$name($<).o
    opts = $cxx.poptions $cc.poptions $cc.coptions $cxx.coptions $cxx.mode

    # NOTE: There is an ugly hack here:
    #   "-I $src_base/../.."
    # This is necessary for the compiler to be able to find any package
    # includes. (Unfortunately, it will *only* find the package
    # includes... it won't use any installed headers.)
    #
    # There are two steps up because this buildfile is assumed to be in
    # `<root>/tests/compile-pass`, so we need to go up two directories
    # to get to the package root. If you organize your code any
    # differently, you may need to adjust.
    hack = -I $src_base/../..

    # Try the compile!
    $cxx.path $opts $hack -o $out -c $src

    # Compile succeeded! Now we have to clean up after ourselves.
    rm $out_base/$name($<).o

    # Print a pretty message on success.
    echo "compile-pass-test $name($<) passes"
  }}
}
```

Each compile-pass test should be the smallest fragment of code that has the thing being tested, and that can compile. For example:

```cpp
// <root>/tests/compile-pass/any-test-file.cxx

#include <libfoo/foo.hxx>

auto f = foo<int>{};
```

You could have multiple tests, where `int` is replaced by `unsigned int`, `short`, `unsigned char`, etc..

The `buildfile` for compile-*fail* testing is very similar to the one for compile-pass testing, except:

1.  The compile command should be excepted to *fail* (return non-zero exit status).
2.  We should prevent the compiler error messages from being printed, as they would just be noise; just the fact that compilation failed is the point of the test.
3.  We need to remove the object file generated if compilation *succeeds* (that is, the test fails).
4.  We should change the pretty message.

```
# <root>/tests/compile-fail/buildfile

./: */

for t: cxx{*}
{
  ./: alias{$name($t)}: $t
  % test
  {{
    src = $path($<)
    out = $out_base/$name($<).o
    opts = $cxx.poptions $cc.poptions $cc.coptions $cxx.coptions $cxx.mode

    # NOTE: Ugly hack:
    hack = -I $src_base/../..

    if $cxx.path $opts $hack -o $out -c $src 2>- == 0
      rm $out_base/$name($<).o
      exit "compile-fail-test $name($<) failed"
    else
      echo "compile-fail-test $name($<) passes"
    end
  }}
}
```

And, of course, compile-pass tests should be minimally compilable:

```cpp
// <root>/tests/compile-fail/any-test-file.cxx

#include <libfoo/foo.hxx>

// Should fail to compile, because double is not integral.
auto f = foo<double>{};
```

Now when you run the `test` operation (for example, with `b test`), the compile-pass and compile-fail tests will be run. If they all succeed, you will see output like:

```
$ b test
c++ ../lib<name>-default/lib<name>/tests/compile-pass/alias{pass-test-1}
c++ ../lib<name>-default/lib<name>/tests/compile-fail/alias{fail-test-2}
c++ ../lib<name>-default/lib<name>/tests/compile-pass/alias{pass-test-2}
c++ ../lib<name>-default/lib<name>/tests/compile-fail/alias{fail-test-1}
test ../lib<name>-default/lib<name>/tests/basics/exe{driver}
compile-fail-test fail-test-1 passes
compile-pass-test pass-test-2 passes
compile-fail-test fail-test-2 passes
compile-pass-test pass-test-1 passes
$ 
```

If a compile-pass test fails, you will see the compiler error output. If a compile-fail test fails—which means compilation succeeded—there will be no compiler error output, but build diagnostics will be printed identifying the failing test.

Happy testing!
