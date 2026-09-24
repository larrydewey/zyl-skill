# secret-five-prohibitions

> Never branch on, index with, divide by, print, or send a `Secret`, and pass it to C only through `ffi-pin`.

## Why It Matters

Each prohibited use leaks through timing, cache or an output channel. They are compile-time errors:

| A secret may not… | Error |
|---|---|
| steer control flow: `if`/`while`/`for`/`cond` condition, subject of a `match` with **more than one arm** | `E_CT_VIOLATION` |
| address memory: index of `w-get`, `w-set`, `list-nth`, `alloc-read-int`, byte load/store offsets | `E_CT_VIOLATION` |
| be an operand of `/` or `mod`/`%` | `E_CT_VIOLATION` |
| reach `print`, an `error` message, or the text a `show` impl returns | `E_SECRET_DEBUG` |
| reach `spawn`, `send`, `file-write` | `E_SECRET_ESCAPE` |
| be a raw `ffi-call` argument | `E_FFI_PIN_REQUIRED` |
| be consumed into a public result without `zeroize` | `E_ZEROIZE_MISSING` (warning) |

## Bad

```lisp
(defn leak ((k Secret))
  (if (= k 0) 1 2))
;; error[E_CT_VIOLATION]: in `leak`: secret-dependent branch ...   (located: --> file:line:col)

(defn check ((k Secret))
  (error (str-concat "bad key " (ffi-call "zyl_int_text" k 1000))))   ; E_SECRET_DEBUG

(defstruct Login (user String) (pw Secret))
(impl Show Login (defn show (self) (str-concat self.user (Show.show self.pw))))  ; E_SECRET_DEBUG
```

## Good

```lisp
(use math/secret/secret)
(defn pick ((flag Secret) a b)
  (ct-select flag a b))             ; mask, no branch

(deftype Key (KeyW Int))
(impl Secret Key (defn wipe (self) 0))
(defn key-word ((k Key)) (match k (KeyW w (+ w 1))))   ; one-arm destructure: not a branch; `w` is secret
```

## Notes

- Allowed on secrets: `+ - *`, `bit-and bit-or bit-xor bit-not`, shifts — fixed, branchless instruction sequences.
- A single-arm `match` only destructures, so it is allowed on a secret; its binders are secret.
- Loop counts must come from public values (lengths, limb counts), never a secret's value.
- Impl method bodies are checked too (diagnostics name them, e.g. `in \`Dump.dump for L\``).
- A secret held in a `Secret` field does not taint the record: `(print login)` is allowed and shows `pw: <secret>`; `(print login.pw)` is still `E_SECRET_DEBUG` ([trait-derive-show](trait-derive-show.md)).
- `Secret` is deliberately not `Send`.

## See Also

- [secret-branchless-masks](secret-branchless-masks.md)
- [secret-declassify-explicitly](secret-declassify-explicitly.md)
