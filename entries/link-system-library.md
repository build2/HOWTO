# How do I link a system library like `-lm`, `-ldl`, or `advapi32.lib`?

> My project requires linking `-lm` and I tried importing `libm%lib{m}` but
> then I get strange errors like there not being a static variant available.

System libraries like `-lm` and `-ldl` (which are essentially extensions of
`libc`) or Windows Platform SDK libraries like `advapi32.lib` and `ws2_32.lib`
should not be imported. Instead, you should list them in the `*.libs`
variables (`c.libs`, `cxx.libs`, or `cc.libs`). For example:

```
if ($c.target.class != 'windows')
  c.libs += -lm
```

Or:

```
if ($cxx.target.class == 'windows')
{
  if ($cxx.target.system == 'mingw32')
    cxx.libs += -ladvapi32
  else
    cxx.libs += advapi32.lib
}
```

See also [How do I link the pthread library?][link-pthread]

[link-pthread]: link-pthread.md
