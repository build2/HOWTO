# How do I sanitize the execution of my tests?

> I am converting a third-party project and its tests don't behave well. They
> write diagnostics to `stdout` instead of `stderr` as well as create
> temporary files in the current working directory and without cleaning after
> themselves. How can I sanitize the execution of such tests?

The easiest way to sanitize test execution is using [Testscript][testscript]
(an alternative would be an [ad hoc recipe][adhoc-recipe]). With Testscript
each test gets its own temporary working directory and there are mechanisms
for redirecting output streams, cleaning up temporary files, etc.

While normally each `testscript` file contains multiple tests for a single
executable, it's also possible to use the same `testscript` file to performs a
single test on multiple executables. For this we simply list the same
`testscript` as a prerequisite for multiple tests. For example, our
`buildfile` might look like this:

```
for t: cxx{test-*}
{
  ./: exe{$name($t)}: $t $libs testscript
}
```

The `testscript` file that redirects `stdout` to `stderr` and cleans up any
temporary files or directories created by the test would look like this:

```
# Redirect stdout to stderr (1>&2) and let stderr through (2>|).
# Cleanup anything the test might create, recursively (&***).
#
$* 1>&2 2>| &***
```

Other relevant things that can be accomplished with Testscript include:

* Setting (or unsetting) environment variables (see the [`env`][env]
  builtin).

* Imposing an execution timeout (see the [`env`][env] builtin).

* Arranging filesystem entries in the test working directory (see the
  [`ln`][ln] builtin).

As an example of the last case, if our tests require a certain file to be
present in the current working directory, then we can symlink it from the
source directory before running the test executable:

```
ln $src_base/test.config ./ ;
$*
```

Or if we want to combine this with the previous example (in this case
we disable the automatic symlink cleanup because it will be taken care
of by the recursive wildcard cleanup `&***`):

```
ln --no-cleanup $src_base/test.config ./ ;
$* 1>&2 2>| &***
```

[testscript]: https://build2.org/build2/doc/build2-testscript-manual.xhtml
[adhoc-recipe]: https://build2.org/release/0.13.0.xhtml#adhoc-recipe
[env]: https://build2.org/build2/doc/build2-testscript-manual.xhtml#builtins-env
[ln]: https://build2.org/build2/doc/build2-testscript-manual.xhtml#builtins-ln
