# How do I disable building of a shared library variant for a platform/compiler?

> My ideal solution would be to disable `libs{hello}` for Windows, log a
> warning and continue building only `liba{hello}`. At consumer side, access
> it as `lib{hello}` and get `liba{hello}` on Windows, on all other platforms,
> get what is set by the user. If the consumer forces `config.bin.lib=shared`
> then error out on Windows.

Firstly, this is not an ideal arrangement since it doesn't fit well into
`build2`'s library model. Ideally, a project should either provide only
`liba{}` or `libs{}` or provide `lib{}` and let the user decide which
variant(s) they want.

But if there is no other choice, you can achieve this by overriding `bin.lib`
in your `build/root.build`. For example, add something along these lines after
loading the `cxx` module (adjust the code accordingly if you are using `c`):

```
if ($cxx.target.class == 'windows')
{
  assert ($bin.lib != 'shared') 'only static variant can be built on Windows'
  bin.lib = static
}
```

For completeness you will also want to disable importing of the shared variant
by modifying your `build/export.build` along these lines:

```
$out_root/
{
  include libhello/
}

if ($($out_root/libhello/ cxx.target.class) == 'windows')
  assert ($import.target != libs{hello}) "only static variant can be imported on Windows"

export $out_root/libhello/$import.target
```

I wouldn't recommend issuing a warning (there is nothing the user of your
library can do about it), but if you want, you can modify the above code in
`build/root.build` along these lines:

```
if ($cxx.target.class == 'windows')
{
  switch $bin.lib
  {
    case 'shared'
      fail 'only static variant can be built on Windows'
    case 'both'
    {
      warn 'only static variant will be built on Windows'
      bin.lib = static
    }
  }
}
```
