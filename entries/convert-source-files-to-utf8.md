# How do I convert source files to be in the UTF-8 encoding?

> While packaging a third-party library for `build2` I get errors like below
> because some of its source files are not in UTF-8. Is there an automated
> way to fix this?

```
hello.cxx: warning C4828: The file contains a character starting at offset 0x20 that is illegal in the current source character set (codepage 65001).
hello.cxx:4:6: error: source file is not valid UTF-8
hello.cxx:4:6: error: stray ‘\344’ in program
```

The majority of such errors are caused by non-ASCII characters in comments.
The `iconv(1)` utility available on POSIX systems can be used to convert the
character encoding of a file.

If you know (or can guess) the original encoding, then you should be able
to convert to UTF-8 without any loss of data. For example, if you know that
comments in `hello.cxx` are in Czech, you can guess that it's likely in the
ISO-8859-2 encoding and try to convert it using the following command:

```
iconv -f ISO-8859-2 -t UTF-8 hello.cxx >hello-utf8.cxx
```

To verify that you guessed the source encoding correctly, you can try to
Google-translate the spelling of some of the Czech words from the UTF-8
output.

If you are unable to guess the original encoding or if the source file
simply contains invalid UTF-8 sequences, then you can use the `-c` `iconv`
option to discard characters that cannot be converted. Naturally, to use
this method, you must make sure the discarded characters only appear in
comments. For example:

```
iconv -f UTF-8 -t UTF-8 hello.cxx >hello-utf8.cxx
diff -u hello.cxx hello-utf8.cxx
```

Note that you should also consider reporting such issues to the library
authors.
