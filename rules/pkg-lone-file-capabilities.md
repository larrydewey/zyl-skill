# pkg-lone-file-capabilities

> A lone file (no `zyl.pkg`) has no capabilities until it declares them: start it with `(capabilities io ffi actor secret native unsafe)`, naming only the ones it uses, or every `file-open`, `ffi-call`, `spawn` or `math/secret` call is `E_PKG_CAPABILITY_VIOLATION`.

## Why It Matters

A program names what it may do, so a reader sees it at the top. Since 2026-10-02 a lone file is checked exactly like a package manifest, and an absent `(capabilities ...)` form means none. The check walks every expression form by name, including `test` bodies and `impl` methods, which it used to skip. The diagnostic is located at the gated call and its help line is the exact line to add:

```text
error[E_PKG_CAPABILITY_VIOLATION]: `file-open` needs the io capability, and this file declares none
  --> save.zyl:2:10
   = note: a program names what it may do, so a reader sees it at the top
   = help: add `(capabilities io)` at the top of the file
```

| Capability | What needs it |
|---|---|
| `io` | `file-open`, `file-read`, `file-write`, `file-close`, `read-line`, and any call into `io/*` or `core/io` |
| `ffi` | `ffi-call`, `ffi-pin`, `ffi-unpin` of a foreign symbol (a `zyl_*` runtime entry needs no grant), and `ffi/*` |
| `actor` | `spawn`, `chan`, `chan-send`, `chan-recv`, and `actor/*` |
| `secret` | any call into `math/secret/*` (a bare `Secret` annotation needs nothing) |
| `native`, `unsafe` | accepted names, meaningful in packages |

`print` needs no capability.

## Bad

```lisp
(defn save ((path String) (text String))
  (let h (file-open path "w")   ; E_PKG_CAPABILITY_VIOLATION: needs io
    (begin
      (file-write h text)
      (file-close h))))
(defn main () (begin (save "/tmp/zyl-cap-demo.txt" "hi\n") 0))
```

```lisp
(use actor/actor)
(test "worker joins" (assert-equal (actor-wait (spawn (fn () 1))) unit))   ; needs actor
(run-tests)
```

## Good

```lisp
(capabilities io)
(defn save ((path String) (text String))
  (let h (file-open path "w")
    (begin
      (file-write h text)
      (file-close h))))
(defn main () (begin (save "/tmp/zyl-cap-demo.txt" "hi\n") 0))
```

```lisp
(capabilities actor)
(use actor/actor)
(test "worker joins" (assert-equal (actor-wait (spawn (fn () 1))) unit))
(run-tests)
```

## Notes

- A name other than `io`, `ffi`, `actor`, `secret`, `native` and `unsafe` is `E_MALFORMED_FORM` (`` `network` is not a capability ``).
- The same form works in a REPL session: type `(capabilities io)` at the prompt.
- Inside a package the set lives in `zyl.pkg` instead ([pkg-capabilities](pkg-capabilities.md)); the root file of a manifested package does not repeat it.
- A file usually starts with two such lines: the capabilities and the numeric policy ([fn-numeric-policy](fn-numeric-policy.md)).
- Known gap: a top-level statement outside any `defn` or `test` is not walked. Such a file also does not link (only `test`/`run-tests` forms get an implicit `main`), so put work in `main`.
- The standard library holds every capability and is never checked.

## See Also

- [pkg-capabilities](pkg-capabilities.md)
- [ffi-extern-required](ffi-extern-required.md)
- [actor-channels-kahn](actor-channels-kahn.md)
- [secret-annotate-params](secret-annotate-params.md)
