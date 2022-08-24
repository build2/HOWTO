# How do I correctly use C/C++ `assert()` in tests?

> How do I correctly use C/C++ `assert()` in my tests?

While the `assert()` macro provided by the C/C++ `<assert.h>`/`<cassert>`
headers is pretty basic compared to the modern testing frameworks, it is often
adequate for simple tests and does not require any [extra
dependencies][tests-extra-dependencies].

However, using `assert()` correctly in tests requires some care. Specifically,
we need to make sure that the `NDEBUG` macro (which, if defined, disables the
assertion functionality) is not defined for our tests. The recommended way to
achieve this is to include `<assert.h>`/`<cassert>` last and undefine `NDEBUG`
just before that. For example:

```
#include <string>

#undef NDEBUG
#include <cassert>

int main ()
{
  assert (std::string ("test") == "test");
}
```

Note that the "include `<assert.h>`/`<cassert>` last" part is important
since we still want `NDEBUG` to have its normal effect in other headers
that we include.

[tests-extra-dependencies]: handle-tests-with-extra-dependencies.md
