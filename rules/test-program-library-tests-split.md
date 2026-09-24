# test-program-library-tests-split

> Structure a project as a library module (no `main`), a program file (`use` + `main`), and a test file (`use` + tests + `run-tests`).

## Why It Matters

A file may not contain both tests and a `main`, and a `use`d module's top-level forms are spliced verbatim into the importer. Keeping logic in a `main`-free library lets both the program and the tests share it. Pure logic in the library is also the only thing you can test deterministically when actors are involved.

## Layout

```text
log-processor/
├── logstats.zyl             ; (use core/list) + types + functions; NO main
├── log-processor.zyl        ; (use logstats) + (defn main ...)
├── log-processor-tests.zyl  ; (use logstats) + (test ...) forms + (run-tests)
└── sample.log
```

`(use logstats)` finds `logstats.zyl` next to the file being compiled; `(use util/strings)` finds `util/strings.zyl`.

## Good

```lisp
; log-processor-tests.zyl
(use logstats)
(defn level-of (line)
  (match (parse-line line)
    (Parsed e (struct-get e "level"))
    (Unparsed _ "")))
(test "parse-reads-level"
  (assert-equal (str-eq (level-of "2024-01-15 [WARN] db slow query") "[WARN]") 1))
(run-tests)
```

## Notes

- Make I/O a thin shell over pure functions (`scan-text` takes a string; `process-file` reads the file and calls it) so tests need no files.
- Small helper functions keep each test body to one comparison.

## See Also

- [pkg-library-no-main](pkg-library-no-main.md)
- [pkg-use-what-you-construct](pkg-use-what-you-construct.md)
