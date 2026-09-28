# tool-balance

> After every edit to a `.zyl` file, run `zyl balance <file>` (in a checkout, `build/boot/zyl-self balance`); never count parens by eye or with a script. It is the check every compile runs first, it follows the lexer exactly, and it catches a misplaced paren even when the count nets to zero.

## Why It Matters

Counting delimiters by hand, or with an ad hoc Python or shell script, gets strings, escapes and comments subtly wrong, and no count can see a missing `)` that an extra `)` elsewhere balances: that silently nests every following form inside the broken one. `zyl balance` is `compiler/sexp_balance.zyl`, the same check the compiler runs before reading any file:

- the lexer's own rules: a `;` comment to the end of the line, a string to its unescaped `"`, a backslash taking the next byte (a newline included); `regression/balance-agreement` holds it to the lexer's tokens on mutated sources;
- bracket kinds: `(` `)`, `[` `]`, `{` `}`, innermost first;
- the layout rule of spec §1.6: a top-level form starts in column 1 and no nested opener does, so an opener in column 1 while a form is still open is `E_UNBALANCED_UNCLOSED`, reported at the open form, with a fix-it naming the line where the indentation first contradicts the nesting;
- an unterminated string at its quote (`E_UNTERMINATED_STRING`), a NUL byte (`E_INVALID_CHAR`).

It checks a file or every `.zyl` file under a directory in well under a second, prints each fault located (`--error-format=json` for machines), and exits 1 when anything is unbalanced.

## Good

```bash
build/boot/zyl-self balance stdlib/compiler/parser.zyl    # one file
build/boot/zyl-self balance stdlib selfhost tests/regression   # trees
```

```
error[E_UNBALANCED_UNCLOSED]: this form is still open where a new top-level form starts at line 3
  --> main.zyl:1:1
   |
 1 | (defn g (x)
   | ^
   = help: insert ')' to close this form before line 3
```

## Notes

- In this repository a Claude Code hook (`.claude/settings.json`, `tools/hooks/balance-check.sh`) runs it after every Edit/Write of a `.zyl` file and after every shell command that leaves changed `.zyl` files, and returns the report; fix what it reports before going on.
- `tests/compile-fail/` holds deliberately unbalanced files; the hook skips them.
- The REPL uses the net check only (`sb-check-nets`), so a pasted body in column 1 still completes an entry.

## See Also

- [syn-brackets-and-balance](syn-brackets-and-balance.md)
- [boot-parens-per-file](boot-parens-per-file.md)
