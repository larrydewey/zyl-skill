# syn-string-literals

> Use only the supported escapes (`\n \t \r \0 \" \\ \e \xNN`); remember strings are NUL-terminated byte pointers.

## Why It Matters

An unknown escape such as `\q` is not reported: the whole literal decodes to a null string, which prints as an empty line. `\0` ends the string early because the runtime representation is a NUL-terminated pointer. `str-length` counts **bytes**, not characters.

## Bad

```lisp
(print "path\qfile")       ; unknown escape: prints an empty line
(print "abc\0def")         ; prints "abc"
(str-length "你好")         ; 6, not 2
```

## Good

```lisp
(print "line 1\nline 2")
(print "quote: \"x\"  backslash: \\")
(print "\e[1mbold\e[0m")   ; \e is ESC (0x1b)
(print "\x41")             ; A
```

## Notes

- A literal may span lines. No interpolation, no raw strings.
- An unterminated string is `E_UNTERMINATED_STRING`, reported before parsing.
- `;` inside a string is safe in the current lexer.
- String built-ins: `str-concat`, `str-length`, `str-substring s start len` (byte-indexed), `str-equal`/`str-eq` (1 or 0). There is no `+` for strings.
- New escapes are new syntax for the self-hosted compiler: see [boot-two-step-syntax](boot-two-step-syntax.md).

## See Also

- [fn-string-equality](fn-string-equality.md) - `==` vs `str-eq`
- [fn-types-drive-codegen](fn-types-drive-codegen.md) - printing strings
