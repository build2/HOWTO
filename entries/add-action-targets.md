# How do I add action targets similar to `PHONY` targets in `make`?

> I am working on an embedded project and I would like to define the `upload`
> action that flashes the new system image to the device. In `make` I would
> use a `PHONY` target with an appropriate recipe. How can I do something
> similar in `build2`?

The equivalent of a `PHONY` target in `build2` is a target with the `alias{}`
target type. So to define the `upload` action we can do (using an executable
in place of a system image):

```
exe{image}: {hxx cxx}{*}

alias{upload}: exe{image}
{{
  diag upload $<
  echo "uploading $path($<)"
}}
```

See [Ad hoc recipes][adhoc-recipe] for background on `build2` recipes.

To trigger the upload action we have to specify the target type (without
the target type it will be treated as an operation name; see below for more
on that):

```
$ b alias{upload}
```

We can, however, prettify it a bit by defining a shorter and more meaningful
alias for the `alias{}` target type (no pun intended). For example:

```
define do: alias

do{upload}: exe{image}
{{
   diag upload $<
   echo "uploading $path($<)"
}}
```

```
$ b do{upload}
```


You might have noticed that this is not how built-in "actions" such as
`update`, `test`, or `install` work. In particular, there is no target type
and we can specify targets to perform the actions on, as in:

```
$ b test: exe{image}
```

The reason for this difference is that these built-in actions are not targets
but rather [_operations_][operations], a concept that does not exist in
`make`.

It is possible to define new operations in `build2` and we could add the
`upload` operation, which would allow us to do things like:

```
$ b upload: exe{image}
```

However, adding a new operation is quite involved and can only be done by a
build system module (implemented in C++). As a result, doing this for a small
project is most likely an overkill. The operation/module route would be
appropriate if, for example, you were the vendor of an embedded C++ SDK who
wished to make it easier for projects that use this SDK to setup their builds
(and who also had the time and expertise to implement and maintain the build
system module).

[adhoc-recipe]: https://build2.org/release/0.13.0.xhtml#adhoc-recipe
[operations]: https://build2.org/build2/doc/build2-build-system-manual.xhtml#intro-operations
