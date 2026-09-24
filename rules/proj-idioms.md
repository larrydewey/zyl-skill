# proj-idioms

> Write Zyl in its native style: small pure functions, recursion with accumulators, ADTs plus exhaustive `match`, state threaded through return values, short `let` chains, I/O at the edges.

## Why It Matters

This style avoids nearly every implementation gap at once: no `let-mut` means no capability errors; no deep nesting means fewer codegen corner cases; pure functions test without files or actors; `let`-bound call results avoid `E_MATCH_ARM_COMPLEX`.

## Good

```lisp
(use core/list)

;; tokenizer: index pair + accumulator, reversed at the end
(defn tokenize-h (s n i start acc sep)
  (if (>= i n)
    (if (> i start) (Cons (str-substring s start (- i start)) acc) acc)
    (if (> (str-eq (str-substring s i 1) sep) 0)
      (tokenize-h s n (+ i 1) (+ i 1) (Cons (str-substring s start (- i start)) acc) sep)
      (tokenize-h s n (+ i 1) start acc sep))))
(defn tokenize (s sep) (list-reverse (tokenize-h s (str-length s) 0 0 Nil sep)))

;; outcome ADT instead of sentinel values
(deftype ParseResult (Parsed LogEntry) (Unparsed String))

;; state threading: take a record, return a new one
(defn count-line (st line)
  (match (parse-line line)
    (Parsed e (count-entry st e))
    (Unparsed _ (count-skipped st))))

;; let-chains before combining calls
(defn message-offset (ts lvl svc)
  (let a (str-length ts)
    (let b (str-length lvl)
      (let c (str-length svc)
        (+ a b c 3)))))
```

## Notes

- `str-substring` returns a fresh heap copy, so tokens outlive the source slice.
- Keep an I/O shell (`process-file`) over a pure core (`scan-text`).

## See Also

- [test-program-library-tests-split](test-program-library-tests-split.md)
- [proj-file-io](proj-file-io.md)
