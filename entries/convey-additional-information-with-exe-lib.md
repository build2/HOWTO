# How do I convey additional information (metadata) with executables and C/C++ libraries?

> I need to provide additional build-time information (AKA metadata) to
> projects that depend on my executable or C/C++ library. For example, I may
> need to convey the configuration information to my tests and the asset
> installation location to the users of my library.

Note: the following functionality is available in `build2` 0.15.0 and later.

In `build2` the unit of importation is a target and while we cannot import
individual variables, a target can set a number of target-specific variables
which we can then query with the `$(<target>: <variable>)` syntax. For
example, if we have the following in our library's `buildfile`:

```
lib{hello}:
{
  libhello.fancy = [bool] $config.libhello.fancy
  libhello.assets = [dir_path] $assets_dir
}
```

Then a project that uses our library could do something along these lines:

```
import lh = libhello%lib{hello}

if $($lh: libhello.fancy)
{
  ...
}

assets_dir = $($lh: libhello.assets)
```

As you may have noticed, this is essentially the same mechanism as what's used
by the C/C++ library metadata protocol and its `*.export.{poptions,libs}`
target-specific variables. In fact, the two can naturally be combined in the
same block, for example:

```
lib{hello}:
{
  cxx.export.poptions = "-I$out_root" "-I$src_root"
  cxx.export.libs = $intf_libs

  libhello.fancy = [bool] $config.libhello.fancy
  libhello.assets = [dir_path] $assets_dir
}
```

Target-specific variables set on imported targets is therefore the foundation
of the user metadata support in `build2`. In fact, the above example works out
of the box provided we are importing a development build of our library.
However, things will stop working as soon as we try to import an installed
version. This happens for two reasons: Firstly, in the installed case, the
`import` directive may not be able to find the library, falling back to the
rule-specific search (see [Target Importation][intro-import] for details). In
this case, the `lh` variable will contain the same project-qualified name we
tried to import and naturally using it to query a target-specific variable
will not work (because the target hasn't been found yet). But even if we
manage to get hold of an imported target (for example, because its installed
location was specified explicitly), there will be no target-specific variables
because the library's `buildfile` that sets them is not loaded in the
installed case (in fact, it's not even present in the installation).

To support the installed case, user metadata must be conveyed in the
installation and then somehow extracted during the importation. The way this
is achieved is specific to the target type. For C/C++ libraries, this
information is saved in the `pkg-config` files (`.pc`) along with the
compile/link flags from `*.export.{poptions,libs}`. For executables, this
information is saved in the executable itself and then extracted by executing
it.

The installed case poses another problem: the installation may have been
produced by another build system and could therefore lack the metadata. And,
in the case of executables, the executable name may not necessarily correspond
to the project we are expecting to import. For example, the executable could
be called `bison` but instead of being from the GNU `bison` project it could
be "impersonated" by, say, `byacc` (Berkeley `yacc`). To detect such
mismatches, the user metadata mechanism establishes a signaling protocol. The
following sections describe the necessary arrangements for C/C++ libraries and
executables in detail.

Note also that the user metadata mechanism is not limited to executables and
C/C++ libraries and can be implemented for other target types if a suitable
metadata storage and extraction can be provided for the installed case.
Support for C/C++ libraries, which is implemented by the `cc` module, can be
used as a template.


## C/C++ library metadata

To make the example shown above work for the installed case we will need to
make modifications both to the library and the user project. On the library
side all we need to do is add the `export.metadata` variable which signals
that the metadata is available, its version, and in which variables. For
example:

```
lib{hello}:
{
  export.metadata = 1 libhello

  libhello.fancy = [bool] $config.libhello.fancy
  libhello.assets = [dir_path] $assets_dir
}
```

The first element in `export.metadata` is the "metadata protocol version". It
is there to allow for potential future changes to the user metadata semantics
while preserving backwards compatibility. For now only version `1` is
recognized.

The second element in `export.metadata` is the metadata variable name prefix.
Only target-specific variables set on the target itself or its group and which
start with this prefix will be considered as user metadata and saved in the
`pkg-config` file during installation.

The recommended variable prefix is the project name itself. If you have
multiple libraries in a project that have different metadata, then you can
segregate them by using the target name the second variable name component,
for example (assuming the project is called `libab`):

```
lib{a}:
{
  export.metadata = 1 libab.liba

  libab.liba.fancy = [bool] $config.libab.fancy
}

lib{b}:
{
  export.metadata = 1 libab.libb

  libab.libb.assets = [dir_path] $assets_dir
}
```

Because the metadata variables are saved in the "foreign" `pkg-config` format,
there is a number of limitations on their values. They cannot be `null`
(`pkg-config` has no notion of NULL values), all the metadata values must be
typed, and only the following subset of the `buildfile` types is allowed:


```
bool  int64   uint64  string  path  dir_path
      int64s  uint64s strings paths dir_paths
```

Sometimes a library comes with additional files, often referred to as
"assets", and we may wish to communicate their location in the metadata.
Naturally, this location will differ between the development build and when
the library is installed. To support this distinction, the library metadata
mechanism recognizes the special `.for_install` variable name suffix. If both
variables (with and without this suffix) are set, then the value from the
`.for_install` variant is used when saving the metadata into the `pkg-config`
file.

For example, let's say our library has auto-generated assets that are placed
into the `$out_base/assets/` directory during the build and then installed
into the `data/assets/` installation location (for example,
`/usr/share/libhello/assets/`; see [Installing][intro-operations-install] for
details). This is how we can communicate these paths in the library metadata:

```
lib{hello}:
{
  export.metadata = 1 libhello

  libhello.assets = [dir_path] $out_base/assets/
  libhello.assets.for_install = ($install.root != [null] ? $install.resolve([dir_path] data/assets/) : )
}
```

For completeness, let's also briefly discuss the `import.metadata` variable.
This variable is set by `build2` in the export stub (similar to
`import.target`; see [Target Importation][intro-import]) and its presence
indicates that the importer has requested the metadata. If present, its value
is the maximum supported metadata protocol version, which is currently always
`1`. For now you can safely ignore this variable.

Let's now turn to the users of our library. To recap, the problem that we need
to solve here is the postponement of importation in the installed case to the
rule-specific search. The way we solve this is by asking for immediate
importation (`!` in `import!` below) and nominating the rule (`rule_hint`
attribute below) that should be used to search for the library in the
installed case. For C/C++ libraries the two rule options are `c.link` and
`cxx.link` and we should pick the same as what is used to link the user
project. For example:

```
import! [rule_hint=cxx.link] lh = libhello%lib{hello}
```

Now, with this variant of import we will always get a resolved target in `lh`.
The final touch is to request the user metadata with the `metadata` attribute.
For example:

```
import! [metadata, rule_hint=cxx.link] lh = libhello%lib{hello}
```

We have to do this explicitly because calculating or extracting the metadata
could be expensive in some cases. Also, the `metadata` attribute instructs the
`import` directive to verify the metadata is actually present.


## Executable metadata

The C/C++ library metadata discussed in the previous section is based on the
general mechanisms that could be replicated for other target types. For
executables, however, `build2` has a number of special extensions and
assumptions.

On the user side, while we still can nominate a rule to search for installed
executables with the `rule_hint` attribute, this will be uncommon. Instead,
`build2` provides built-in support for searching for installed executables in
the `PATH` environment variable. So importing an executable with metadata will
normally look like this:

```
import! [metadata] hello = hello%exe{hello}
```

On the executable side, the development build arrangement for the metadata is
the same as for libraries. All we need to do is to set the special
`export.metadata` variable, for example:

```
exe{hello}:
{
  export.metadata = 1 hello

  hello.fancy = [bool] $config.hello.fancy
}
```

The installed case is where things start to diverge. Unlike C/C++ libraries
with their accompanying `pkg-config` files, executables don't have a natural
metadata storage. Instead, `build2` expects the metadata to be embedded into
the executable itself and the special `--build2-metadata=<version>` option
recognized as a request to extract it. For example, here is what `main()` from
our `hello` executable could look like:

```c++
#include <cstring> // strncmp

int main (int argc, char* argv[])
{
  using namespace std;

  // Handle --build2-metadata (see also buildfile).
  //
  // The HELLO_FANCY macro corresponds to the config.hello.fancy value.
  //
  if (argc == 2 && strncmp (argv[1], "--build2-metadata=", 18) == 0)
  {
    cout << "# build2 buildfile hello"
         << "export.metadata = 1 hello"
         << "hello.fancy = [bool] " << (HELLO_FANCY ? "true" : "false") << '\n';

    return 0;
  }

  ...
}
```

While we can work the `--build2-metadata` option into our normal option
handling code, usually it's easier to just handle it ad hoc as in the example
above: it will always be the first and only option and it will always be
specified in the `--build2-metadata=` form.

If the `--build2-metadata` option is specified, the executable should print
the metadata to `stdout` and return with the `0` exit code. The printed
metadata is a `buildfile` fragment in the following form:

```
# build2 buildfile <key>
export.metadata = 1 <prefix>
<prefix>.<name> = ...
...
```

The first line in this fragment is a comment in this specific form that serves
as a metadata signature. The `build2 buildfile` part is used to detect the
expected format (because a stray executable can print gibberish in response to
`--build2-metadata`) while `<key>` is used to make sure the executable is not
being impersonated. It should be the target name as spelled inside `exe{...}`.

Next comes the `export.metadata` variable which we have already seen followed
by the metadata variables. Note that `<key>` and `<prefix>` are not
necessarily the same. For example, if our project name were `fancy-hello` and
the executable target \- `exe{hello-there}`, then the printed metadata would
look like this (notice the use of `_` instead of `-` in the variable prefix):

```
# build2 buildfile hello-there
export.metadata = 1 fancy_hello
fancy_hello.fancy = false
```

Note that unlike with C/C++ libraries, there is no automation in maintaining
the two copies of metadata in sync. As a result, it's helpful to add a comment
in both places as a reminder. For example:

```
exe{hello}:
{
  # Metadata (see also --build2-metadata in main.cxx).
  #
  export.metadata = 1 libhello

  hello.fancy = [bool] $config.hello.fancy
}
```

While one can import an executable for other reasons, most commonly this is
done so that it can be used during the build. A typical example is a source
code generator that is used in an ad hoc recipe or rule. If an executable is
used in such a way, then `build2` relies on a number of predefined executable
metadata variables to do a better job. These are:

```
<prefix>.name = [string]         # Stable name for diagnostics.
<prefix>.version = [string]      # Version for diagnostics.
<prefix>.checksum = [string]     # Checksum for change tracking.
<prefix>.environment = [strings] # Environment variables for change tracking.
```

The `<prefix>.name` value is used in the low-verbosity diagnostics. Normally
this is just the executable name but sometimes you may want to use a common
name identifying a family of tools. For example, there are several `yacc`
parser generator implementations but it's probably not useful to distinguish
exactly which one is being used in the low-verbosity diagnostics. So for the
`byacc` executable this name is just `yacc`.

The `<prefix>.version` value is the executable version. This value can be used
in diagnostics (for example, in the configuration report) or to verify the
version compatibility.

The `<prefix>.checksum` value is the executable checksum that is used by ad
hoc recipes and rules to detect changes to the executable in order to re-run
the recipe. Usually this is the same as version, which should be sufficiently
accurate if you use [continuous versioning][module-version] in the
executable project.

Finally, the `<prefix>.environment` value lists the environment variables that
could affect the executable output. These are used by ad hoc recipes and rules
to detect (or prevent, in case of hermetic configurations) changes. Note that
only the environment variables that could affect the output (for example,
generated source code) should be listed here. Many executables use environment
variables to control other aspects of their functionality, for example, the
location of temporary files (`TMP`) or diagnostics. Such variables should not
be listed.

As a concrete example, here are the values of these predefined variables for
the [ODB compiler][odb-compiler] (which is implemented as a GCC plugin):

```
# Target metadata, see also --build2-metadata in odb.cxx.
#
# While ODB itself doesn't use any environment variables, it uses GCC
# underneath which does (see "Environment Variables Affecting GCC").
#
exe{odb}:
{
  export.metadata = 1 odb
  odb.name = [string] odb
  odb.version = [string] $version
  odb.checksum = [string] $version
  odb.environment = [strings] CPATH CPLUS_INCLUDE_PATH GCC_EXEC_PREFIX COMPILER_PATH
}
```

[intro-import]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#intro-import
[intro-operations-install]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#intro-operations-install
[module-version]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#module-version
[odb-compiler]: https://cppget.org/odb
