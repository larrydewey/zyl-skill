# syn-string-literals

> Use only the supported escapes (`\n \t \r \0 \" \\ \e \xNN`); remember strings are NUL-terminated byte pointers.

## Why It Matters

An unknown or incomplete escape such as `\q` or `\x4` is a located `E_INVALID_ESCAPE` (the help line lists the valid escapes); before 2026-09-24 the whole literal silently decoded to a null string. `\0` still ends the string early because the runtime representation is a NUL-terminated pointer. `str-length` counts **bytes**, not characters.

## Bad

```lisp
(print "path\qfile")       ; E_INVALID_ESCAPE
(print "\x4")              ; E_INVALID_ESCAPE: \x takes two hex digits
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
- String built-ins: `str-concat`, `str-length`, `str-substring s start len` (byte-indexed), `str-equal`/`str-eq` (both return a Bool). There is no `+` for strings: `(+ "a" "b")` is `E_TYPE_MISMATCH` ("arithmetic on String").
- For zero-copy substrings and hand-written scanners use `text/view` (`StrView`, `Cursor`) instead of repeated `str-substring`.
- New escapes are new syntax for the self-hosted compiler: see [boot-two-step-syntax](boot-two-step-syntax.md).

## See Also

- [fn-string-equality](fn-string-equality.md) - `=` compares String contents
- [fn-print-semantics](fn-print-semantics.md) - printing strings
