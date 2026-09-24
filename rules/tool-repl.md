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
- Redefining a name needs `:reset`.
- `:type` often answers *unresolved* for applications (a known inference gap).
- Line editor: arrows/word motion, multi-line until the form closes, Ctrl-R, Tab completion, highlighting.

## See Also

- [tool-eval-differential](tool-eval-differential.md)
- [fn-no-toplevel-def](fn-no-toplevel-def.md)
