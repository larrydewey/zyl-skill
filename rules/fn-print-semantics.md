# fn-print-semantics

> `print` writes each argument on its own line and evaluates to 0; build one-line output with `str-concat`.

## Why It Matters

`(print "Total:" n)` prints two lines, not one. `print` returns 0, so a `main` ending in `print` exits 0 but an expression using its value gets 0. Standard output is buffered while `PANIC:` messages go straight to stderr, so a panic line can appear **before** output printed earlier.

## Bad

```lisp
(print "total: " n)                 ; two lines
(let r (print x) (+ r 1))           ; r is 0
```

## Good

```lisp
(defn label ((s String)) (print-string (str-concat "result: " s)))
(label "disk full")                 ; one line: result: disk full

(print "Total:")                    ; numbers: label and value on separate lines
(print n)
```

## Notes

- Formats: Int `%lld`, Float `%f` (six decimals), String `%s`, Bool as `1`/`0`. A struct/ADT prints its **address** (no derived `Show`). At the REPL, values print structurally.
- There is no number-to-string built-in in the core list of forms; compiler code uses `(ffi-call "zyl_cstr_from_int" arena n 1000)`.
- `read-line` is parsed but not lowered (evaluates to 0). `exit` and `close` are not lowered either. File I/O: `file-open`/`file-read`/`file-write`/`file-close`.
- `print` is not capability-gated in packages; `print` of a `Secret` is `E_SECRET_DEBUG`.

## See Also

- [fn-annotate-string-float-params](fn-annotate-string-float-params.md) - print picks format by kind
- [proj-file-io](proj-file-io.md) - file forms
