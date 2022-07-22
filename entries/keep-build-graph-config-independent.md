# How do I keep the build graph configuration-independent?

> While preparing a distribution of my project I get a warning like below.
> What does that mean and how do I fix it?

```
buildfile:32:3: warning: conditional dependency declaration may result in incomplete distribution
  buildfile:32:16: info: prerequisite declared here
  buildfile:28:1: info: conditional buildfile fragment starts here
  info: instead use 'include' prerequisite-specific variable to conditionally include prerequisites
  info: for example: <target>: <prerequisite>: include = (<condition>)
```

In `build2` the build graph, which consists of all the target-prerequisite
relationships, must be configuration-independent, that is, it should not
depend on the compiler, platform, project configuration, etc.

Currently, this is relied upon by the `dist` module in order to prepare an
identical distribution of our project regardless of the configuration. But
this could also be useful to other tools that need to have a
configuration-independent view of the project. For example, an IDE could use
`buildfiles` as a source of project definition. From the IDE's point of view a
source file still belongs to the project even if it's excluded from the build
on the current platform.

While a configuration-independent build graph is a useful property, it's not
without costs. After having written something like this a few times:

```
if ($cxx.target.class != 'windows')
{
  cxx.libs += -pthread
}
```

It is natural to then also write something like this:

```
if ($cxx.target.class != 'windows')
{
  exe{hello}: cxx{utility-posix}
}
else
{
  exe{hello}: cxx{utility-win32}
}
```

But this creates different build graphs depending on the configuration which
in turn will lead to different sets of files in the distribution depending on
which platform we are preparing it on. As a result, `build2` detects such
conditional dependency declarations and issues a warning like seen above.

The correct way to re-implement the above conditional dependency declaration
is by using the `include` prerequisite-specific variable. This variable has
the special semantics of including (`true`) or excluding (`false`)
prerequisites from the build (it can also be `adhoc` to include prerequisites
but in an ad hoc manner). For example:

```
exe{hello}: cxx{utility-posix}: include = ($cxx.target.class != 'windows')
exe{hello}: cxx{utility-win32}: include = ($cxx.target.class == 'windows')
```

While you may even find the correct version preferable (it does have a
declarative feel to it), handling more complex cases in a
configuration-independent manner may take some getting used to. The following
sections discuss a number of common scenarios.


## Platform-specific test

It's not uncommon to have a test that should be excluded on a specific
platform. For example (an incorrect version):

```
./: exe{test-file}: cxx{test-file} $libs

if ($cxx.target.class != 'windows')
  ./: exe{test-pipe}: cxx{test-pipe} $libs

```

And here is the correct variant:

```
./: exe{test-file}: cxx{test-file} $libs

./: exe{test-pipe}: include = ($cxx.target.class != 'windows')
exe{test-pipe}: cxx{test-pipe} $libs

```

## Conditional ad hoc recipes

A common scenario in code generators that use their own generated code is to
provide a bootstrap step that is used to re-generate such code. The bootstrap
step is normally only enabled in the development build. Here is a real example
from the `reflex` package (the incorrect version):

```
src = {h c}{* -parse -scan -initscan} {h c}{parse}

exe{reflex}: $src

if $config.reflex.develop
{
  # Bootstrap if in development mode.
  #
  exe{reflex}: c{scan}
  exe{reflex-boot}: $src c{initscan}
  {
    install = false
  }

  c{scan}: l{scan} exe{reflex-boot}
  {{
    diag lex ($<[0])
    ($<[1]) -L -o $path($>) $path($<[0])
  }}
}
else
{
  exe{reflex}: c{initscan}
  ./: l{scan} # Keep in the distribution.
}
```

Notice the last line in the `else` block that hacks around the distribution
issue. And here is the correct variant:

```
src = {h c}{* -parse -scan -initscan} {h c}{parse}

exe{reflex}: $src

# Bootstrap if in development mode.
#
exe{reflex}: c{scan}:     include = ( $config.reflex.develop)
exe{reflex}: c{initscan}: include = (!$config.reflex.develop)

exe{reflex-boot}: $src c{initscan}
{
  install = false
}

c{scan}: l{scan} exe{reflex-boot}
%
if $config.reflex.develop
{{
  diag lex ($<[0])
  ($<[1]) -L -o $path($>) $path($<[0])
}}
```

The tricky part here is the now conditional recipe for the `c{scan}` target.
Here it is again:

```
c{scan}: l{scan} exe{reflex-boot}
%
if $config.reflex.develop
{{
  ...
}}
```

What happens here is that while the target-prerequisite declaration is
unconditional, we do omit the recipe with the `if`-condition (yes, recipe
blocks can have conditions but that requires an explicit `%` recipe
header). We normally want to omit such recipes because while they will not be
executed in configurations that don't apply, they still have to be valid
in order to be parsed. While the above recipe probably would have no issues
even if left unconditional, consider the following hypothetical example:

```
# Only run lex if in development mode.
#
if $config.hello.develop
  import lex = reflex%exe{reflex}

exe{hello}: c{scan}:     include = ( $config.hello.develop)
exe{hello}: c{initscan}: include = (!$config.hello.develop)

c{scan}: l{scan} $lex
%
if $config.hello.develop
{{
  $lex -o $path($>) $path($<[0])
}}
```

The above recipe will actually not parse in the non-development mode because
in this case the `lex` variable will be empty.

Note also that the same considerations apply to ad hoc rules.
