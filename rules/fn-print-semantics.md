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
(defn label (s) (print (str-concat "result: " s)))
(label "disk full")                 ; one line: result: disk full

(print "Total:")                    ; numbers: label and value on separate lines
(print n)
```

## Notes

- Formats follow the inferred type: Int `%lld`, Float `%f` (six decimals), String `%s`, Bool as `1`/`0`. A value whose type has a `Show` impl prints `(Show.show v)`: `[1, 2]`, `Some(x)`, `{k: v}`, derived `Name(a, b)` ([trait-derive-show](trait-derive-show.md)). A record without a `Show` impl prints its **address**; so does an Option/Result/List whose payload type has no `Show` impl (the container is printed raw). At the REPL, `=>` results print structurally.
- Number to text: `(Show.show n)`; compiler code uses `(ffi-call "zyl_cstr_from_int" arena n 1000)`.
- `read-line` is parsed but not lowered (evaluates to 0). `exit` and `close` are not lowered either. File I/O: `file-open`/`file-read`/`file-write`/`file-close`.
- `print` is not capability-gated in packages; `print` of a `Secret` is `E_SECRET_DEBUG`, but a record holding a secret in a `Secret` field prints with that field as `<secret>`.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md) - print picks format by kind
- [proj-file-io](proj-file-io.md) - file forms
