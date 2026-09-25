# fn-print-semantics

> `print` writes each argument on its own line and returns `Unit`; build one-line output with `str-concat` and `Show.show`.

## Why It Matters

`(print "Total:" n)` prints two lines, not one. `print` is a statement: its type is `Unit`, so using its value as a number is `E_TYPE_MISMATCH`, and a `main` ending in `print` does not type-check ([fn-main-and-exit-status](fn-main-and-exit-status.md)). Standard output is buffered while `PANIC:` messages go straight to stderr, so a panic line can appear **before** output printed earlier, and a crash by signal (SIGFPE, SIGSEGV) loses whatever was still buffered.

## Bad

```lisp
(print "total: " n)                 ; two lines
(let r (print x) (+ r 1))           ; E_TYPE_MISMATCH: arithmetic on Unit
(defn main () (print "done"))       ; E_TYPE_MISMATCH: main must return Int
```

## Good

```lisp
(defn label (s) (print (str-concat "result: " s)))
(label "disk full")                 ; one line: result: disk full

(print (str-concat "Total: " (Show.show n)))   ; one line, number included
```

## Notes

- Formats follow the static type: Int `%lld`, Float `%f` (six decimals), String `%s`, Bool as `1`/`0`, Unit as `0`. A value whose type has a `Show` impl prints `(Show.show v)`: `[1, 2]`, `Some(x)`, `{k: v}`, derived `Name(a, b)` for a variant and `Pt { x: 1, y: 2 }` for a struct ([trait-derive-show](trait-derive-show.md)). Inside a container, elements use `Show`, so Bools print `true`/`false`. A record or ADT without a `Show` impl prints its **address**; so does an Option/Result/List whose payload type has no `Show` impl (the container is printed raw). At the REPL, `=>` results print structurally.
- Number to text: `(Show.show n)` for Int and Float; compiler code uses `(ffi-call "zyl_int_text" n 1000)`.
- `(read-line)` reads one line from stdin and flushes stdout first; `(exit code)` flushes output and ends the process ([fn-unlowered-forms](fn-unlowered-forms.md)). File I/O: `file-open`/`file-read`/`file-write`/`file-close`.
- `print-int`, `print-float`, `print-string` and `print-bool` in the prelude take one argument of that type.
- `print` is not capability-gated in packages; `print` of a `Secret` is `E_SECRET_DEBUG`, but a record holding a secret in a `Secret` field prints with that field as `<secret>`.

## See Also

- [fn-types-drive-codegen](fn-types-drive-codegen.md) - print picks format by type
- [proj-file-io](proj-file-io.md) - file forms
