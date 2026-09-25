# tool-repl

> Use the REPL (`zyl repl`) to explore expressions and definitions; each name can be defined once per session.

## Why It Matters

REPL entries go through the real compiler phases and are evaluated by the ICNF interpreter, so values persist across entries and print structurally. That makes it easy to write code that works at the prompt and fails compiled (actors, division by zero).

## Meta commands

`:help`/`:h`, `:quit`/`:q`, `:history`/`:hist`, `:defs`/`:browse`, `:doc NAME`/`:d`, `:type EXPR`/`:t`, `:time EXPR`/`:tm`, `:load PATH`/`:l`, `:save PATH`/`:s`, `:reset`/`:r`, `:clear`/`:cls`.

## Session state

- Start-up: default modules (`core/core`, `core/list`, `core/option`, `core/result`, `allocator/allocator`), then `~/.zyl/replrc` (`$ZYL_REPLRC`), then `.zyl-session` in the start directory (rewritten after every changing entry).
- History: `~/.zyl/repl_history` (`$ZYL_REPL_HISTORY`, `$ZYL_STATE_DIR`).
- Non-tty stdin: script mode (no rc, session, history or banner).
- `def` bindings made at the prompt are live values, and a later `defn` can use them: `(def k 41)` then `(defn f (x) (+ x k))` then `(f 1)` gives 42 (Strings too). Each prompt def is emitted into the session program as a top-level def that reads a runtime table, so its expression is **not re-run** (side effects happen once); `:reset` clears the table.
- Redefining a name needs `:reset`: a second `(defn f ...)` is `E_DUPLICATE_DEFINITION`.
- Entries are type-checked like a compiled program: a type error is reported and the entry is not run. The location is in the generated session program (`<repl>:7:27`, with an expression shown wrapped as `(defn __zyl_repl_entry () ...)`), not the line you typed. `:type` prints the inferred type: `(f 2) : Int`, `f : (Int -> Int)`, `(str-concat "a" "b") : String`.
- Line editor: arrows/word motion, multi-line until the form closes, Ctrl-R, Tab completion, highlighting.

## See Also

- [tool-eval-differential](tool-eval-differential.md)
- [fn-toplevel-def](fn-toplevel-def.md)
