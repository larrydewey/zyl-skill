# proj-idioms

> Write Zyl in its native style: small pure functions, recursion with accumulators, ADTs plus exhaustive `match`, state threaded through return values, zero-copy views for parsing, short `let` chains, I/O at the edges.

## Why It Matters

This style avoids nearly every implementation gap at once: no `let-mut` means no capability errors; no deep nesting means fewer codegen corner cases; pure functions test without files or actors; `let`-bound call results avoid `E_MATCH_ARM_COMPLEX`. Views (`text/view`: `StrView`, `Cursor`) make tokenizing and parsing allocation-free: taking a view, a sub-view, splitting or trimming never copies bytes, where every `str-substring` allocates a fresh copy.

## Good

```lisp
(use core/list)
(use text/view)

;; tokenizer over views: splitting copies nothing
(defn tokenize ((s String) (sep Int))
  (view-split (view-of s) sep))                 ; (List StrView)

;; outcome ADT instead of sentinel values
(defstruct LogEntry (level String) (msg String))
(deftype ParseResult (Parsed LogEntry) (Unparsed String))

(defn parse-line ((line String))
  (let v (view-trim (view-of line))
    (if (view-starts-with v "[")
      (let close (view-find v 93 0)             ; 93 = ]
        (if (< close 0)
          (Unparsed line)
          (Parsed (make-LogEntry (view-to-string (view-take v (+ close 1)))       ; copy only what is kept
                                 (view-to-string (view-trim (view-drop v (+ close 1))))))))
      (Unparsed line))))

;; state threading: take a record, return a new one
(defstruct Stats (warns Int) (skipped Int))
(defn count-entry ((st Stats) (e LogEntry))
  (if (str-eq e.level "[WARN]") (make-Stats (+ st.warns 1) st.skipped) st))
(defn count-line ((st Stats) (line String))
  (match (parse-line line)
    (Parsed e (count-entry st e))
    (Unparsed _ (make-Stats st.warns (+ st.skipped 1)))))

;; recursion with an accumulator
(defn scan-lines (st lines)
  (match lines
    (Nil st)
    (Cons l rest (scan-lines (count-line st (view-to-string l)) rest))))

;; let-chains before combining calls
(defn message-offset (ts lvl svc)
  (let a (str-length ts)
    (let b (str-length lvl)
      (let c (str-length svc)
        (+ a b c 3)))))

(defn main ()
  (let st (scan-lines (make-Stats 0 0) (tokenize "[WARN] slow\n[INFO] ok\ngarbage\n[WARN] again" 10))
    (begin
      (print st.warns)                          ; 2
      (print st.skipped)                        ; 1
      0)))
```

## Notes

- `str-eq` is a `Bool` predicate: use it directly as a condition (`(if (str-eq a b) ...)`); the old `(> (str-eq a b) 0)` idiom is now `E_TYPE_MISMATCH`.
- A view keeps its base string alive (region inference places the base wherever the view goes), so returning views from a parser is safe. `view-to-string` is the one operation that copies; call it only for what you keep.
- `Cursor` (`cursor-of`, `cursor-peek`, `cursor-advance`, `cursor-take-while`, `cursor-skip-space`, `cursor-expect`) is the view for hand-written parsers; `collections/slice` (`Slice`, `slice-of-vec`, `slice-sub`, `slice-fold`) is the same idea for `Vec`.
- List literals make test data short: `[1 2 3]` or `(list 1 2 3)` builds a `Cons` list.
- Keep an I/O shell (`process-file`) over a pure core (`scan-text`).

## See Also

- [test-program-library-tests-split](test-program-library-tests-split.md)
- [proj-file-io](proj-file-io.md)
