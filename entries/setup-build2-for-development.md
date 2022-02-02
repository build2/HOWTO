# How do I setup the `build2` toolchain for development?

> I would like to have a `build2` build that is suitable for making changes
> to the toolchain.

The `build2` toolchain development is self-hosted, that is, we use the
toolchain for its own development. As a result, the development setup
is not as simple as `git clone && bdep init` because, well, there is
no `bdep` yet.

The recommended way to initialize the `build2` development setup is with the
`bootstrap` script found in the [`etc`][etc] repository. By default, it will
clone the necessary `git` repositories into the current working directory and
initialize the default build configurations. Later it can also be used to
initialize additional configurations.

NOTE: The `bootstrap` script has been tested on Linux and should also work on
other UNIX-like systems (Mac OS, FreeBSD) provided `bash` is available. While
it should be possible to recreate this setup on Windows, no automated way to
do this is currently provided.

Before running `bootstrap`, you will also need to perform the following
preliminary steps:

1. Install the latest [staged toolchain][stage] somewhere temporary, for
   example, `/tmp/build2-install`. Hint: use `--local` and `--no-modules` when
   building the staged toolchain to speed things up seeing that this is a
   throw-away installation. For example:

   ```
   $ sh build2-install-...-stage.sh --local --no-modules /tmp/build2-install
   ```

2. Install the [CLI][cli] compiler using the staged toolchain.

   If you don't need the development setup of CLI, you can build the compiler
   from a [package][cli] and install it into `/usr/local` or similar. See
   [Package Consumption][guide-consume-pkg] for details (note that you only
   need the compiler but it must be the version from the
   [stage.build2.org](https://stage.build2.org) repository).

   Alternatively, you can clone the CLI repository, create a development
   setup, and then symlink it in `/usr/local/bin/` (or make alternative
   arrangements, such as adding it to `PATH`). For example:

   NOTE: clone using the `git.codesynthesis.com:/var/scm/...` SSH URL if you
   have the read-write access.

   ```
   $ "$SHELL"
   $ export PATH="/tmp/build2-install/bin:$PATH"

   $ mkdir -p ~/work/cli
   $ cd ~/work/cli
   $ git clone --recursive https://git.codesynthesis.com/cli/cli.git
   $ cd cli
   # see README.md for further instructions
   # use config.cc.coptions="-Wall -Wextra -Werror -g -fsanitize=address -fsanitize=undefined -fno-omit-frame-pointer"
   $ b
   $ sudo ln -s "$(pwd)/cli/cli/cli" /usr/local/bin/cli
   $ which cli
   $ cli --version

   $ exit # Restore PATH.
   ```

3. Install the [ODB][odb] compiler using the staged toolchain.

   If you don't need the development setup of ODB, you can build the compiler
   from a [package][odb] and install it into `/usr/local` or similar. See
   [Package Consumption][guide-consume-pkg] and [Installing ODB with
   `build2`][odb-install-build2] for details (note that you only need the
   compiler but it must be the version from the
   [stage.build2.org](https://stage.build2.org) repository).

   Alternatively, you can clone the ODB repository, create a development
   setup, and then symlink it in `/usr/local/bin/` (or make alternative
   arrangements, such as adding it to `PATH`). For example:

   NOTE: clone using the `git.codesynthesis.com:/var/scm/...` SSH URL if you
   have the read-write access.

   ```
   $ "$SHELL"
   $ export PATH="/tmp/build2-install/bin:$PATH"

   $ mkdir -p ~/work/odb
   $ cd ~/work/odb
   $ git clone --recursive https://git.codesynthesis.com/odb/odb.git
   $ cd odb
   $ bdep init -C ../builds/gcc @gcc cc \
       config.cxx=g++                   \
       config.cc.coptions="-Wall -Wextra -Werror -g"
   $ b
   $ sudo ln -s "$(pwd)/odb/odb"    /usr/local/bin/odb
   $ sudo ln -s "$(pwd)/odb/odb.so" /usr/local/bin/odb.so
   $ which odb
   $ odb --version

   $ exit # Restore PATH.
   ```

Now you are ready to run the `bootstrap` script. Typically, you would create a
subdirectory in your home directory for `build2` development, for example,
`~/work/build2`:

NOTE: If you have the read-write access to the git repository, clone using the
`git.build2.org:/var/scm/...` SSH URL and pass the `--ssh` option to the
`bootstrap` script.

```
$ mkdir -p ~/work/build2
$ cd ~/work/build2
$ git clone --recursive https://git.build2.org/etc.git
$ PATH="/tmp/build2-install/bin:$PATH" etc/bootstrap
```

NOTE: In case of an error, you can re-try the initialization without
re-cloning the repositories by passing the `--clean` option to the
`bootstrap` script.

NOTE: After a successful bootstrap you can remove the staged toolchain.

By default the `bootstrap` script will clone the `build2` toolchain core
(`build2`, `bpkg`, and `bdep` plus the standard pre-installed modules) and use
a debug configuration with ASAN/UBSAN and `g++` as a compiler. This behavior,
however, can be customized with a number of the `bootstrap` script option
which are described in the script header.

After running `bootstrap`, the contents of this subdirectory will look like
this:

```
~/work/build2/
│
├── builds/                 -- build configurations
│   ├── gcc/                -- default toolchain configuration
│   └── gcc-libs/           -- default libraries configuration
│
├── bdep/                    -- project manager source (git repository)
├── bpkg/                    -- package manager source (git repository)
├── build2/                  -- build system source (git repository)
│
├── libbpkg/                 -- libraries source (git repositories)
├── libbutl/
│
├── libbuild2-hello/         -- hello build system module source (git repository)
├── libbuild2-hello-build/   -- hello build system module build configurations
│   ├── module/              -- module configuration
│   └── target/              -- target configuration
│
└── libbuild2-*/              -- other standard pre-installed build system modules
```

See [`libbuild2-hello`][libbuild2-hello] for details on build system modules
development setup.

Additionally, for convenience of development, the `bootstrap` script creates a
number of symlinks in `/usr/local/bin`. For example:

```
$ ls -l /usr/local/bin/
  b      -> ~/build2/builds/gcc/build2/build2/b
  b-boot -> ~/build2/build2/build2/b-boot
  bdep   -> ~/build2/builds/gcc/bdep/bdep/bdep
  bpkg   -> ~/build2/builds/gcc/bpkg/bpkg/bpkg
```

If you prefer not to create symlinks, you can pass the `--no-symlink` option
to the `bootstrap` script and make other arrangements (for example, add the
relevant directories to your `PATH`).

With this setup you should now be able to develop `build2` roughly as any
other project: you can either use `bdep` or run the build system directly,
including in the source directories (which are all configured as forwarded
to the default build configuration).

A few notes and tips on the development process: Because the toolchain is used
for its own development, you will sometimes find yourself in a situation where
one or more tools in the toolchain are no longer runnable or functioning
correctly but to fix this you need to be able to run those tools. This can
happen, for example, because of the linking errors or because the version (and
therefore the name) of one of the shared libraries has changed as a result of
updating another part of the toolchain (for instance, updating `bpkg` can
trigger an update of `libbpkg` and this could make `bdep`, which also depends
on `libbpkg`, no longer runnable).

NOTE: to minimize the chance of such breakage, the `bootstrap` script makes a
separate configuration (`builds/gcc-libs/` in the above listing) the default
for libraries (`libbutl`, `libpkg`, etc). This allows you to make changes
to libraries and run their tests without affecting the toolchain.

There are several mechanisms for recovering from such situations. If the build
system is functional, it itself and the rest of the toolchain can always be
update by first disabling the auto-synchronization hooks (which invoke `bdep`
and `bpkg`). For example, the following commands should get the toolchain back
into the fully-functional and synchronized state:

```
$ BDEP_SYNC=0 b build2/build2/ bpkg/bpkg/ bdep/bdep/
$             b build2/build2/ bpkg/bpkg/ bdep/bdep/
```

If the build system itself is not functional, it can always be rebuilt using
the bootstrapped build system (`b-boot`; built by this `bootstrap`
script). For example:

```
$ BDEP_SYNC=0 b-boot build2/build2/
$ BDEP_SYNC=0 b      build2/build2/
```

Note that the bootstrap build system can only be used to update the build
system project (`build2`). It also makes sense to rebuild it from time to time
to keep it reasonably up to date with the latest version. Normally this is
done when the latest version is assumed to be in a reasonably good shape. See
`build2/INSTALL` for details on the bootstrap build system.

While normally we use `bdep sync --upgrade` to upgrade dependencies, we cannot
do that in our case because as soon as we drop old dependencies, the
toolchain become no longer runnable (missing shared libraries, etc). To help
with that the [`etc`][etc] repository provides the `upgrade` script which
performs an elaborate dance of upgrading all the dependencies in the default
configurations (see the script source for details):

```
$ cd ~/work/build2
$ etc/upgrade
```

[etc]:   https://git.build2.org/cgit/etc/tree/
[stage]: https://build2.org/community.xhtml#stage
[cli]:   https://stage.build2.org/cli
[odb]:   https://stage.build2.org/odb
[guide-consume-pkg]: https://build2.org/build2-toolchain/doc/build2-toolchain-intro.xhtml#guide-consume-pkg
[odb-install-build2]: https://codesynthesis.com/products/odb/doc/install-build2.xhtml
[libbuild2-hello]: https://github.com/build2/libbuild2-hello/
