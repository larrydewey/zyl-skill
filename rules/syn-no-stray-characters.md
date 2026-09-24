# syn-no-stray-characters

> Never write `'`, `` ` ``, `,`, `@`, `#`, `$`, `&`, `|`, `^`, `\` or non-ASCII bytes outside strings and comments: each is a located `E_INVALID_CHAR`.

## Why It Matters

The lexer rejects any byte that cannot start a token with a located `error[E_INVALID_CHAR]` (file:line:col, caret, and a hint that Zyl has no quote, quasiquote or comma syntax); a string left open is `E_UNTERMINATED_STRING`. (Before 2026-09-24 such a byte ended the input with no diagnostic and the rest of the file was dropped; an older installed compiler still does that.) The error is easy to fix, but it rules out Lisp habits: there is no `'quote`, no quasiquote/unquote (`` ` `` `,` `,@`), no `#| block comments |#`, and no commas in import lists.

## Bad

```lisp
(defn main ()
  (begin
    (print 1)
    @            ; error[E_INVALID_CHAR]: unexpected character `@`
    (print 2)
    0))

(use acme/greet { greet, greet-twice })   ; E_INVALID_CHAR at the comma

#| block comment |#                        ; not a comment: E_INVALID_CHAR

(defmacro m (x) `(+ ,x 1))                ; backquote: E_INVALID_CHAR

(print "unclosed)                          ; E_UNTERMINATED_STRING
```

## Good

```lisp
(defn main ()
  (begin
    (print 1)
    (print 2)
    0))

(use acme/greet { greet greet-twice })    ; whitespace-separated

;; line comments only
(defmacro m (x) (+ x 1))                   ; the body is the template
```

## Notes

- Allowed token starters: whitespace, `( ) [ ] { }`, `"`, `:`, `~`, `;`, digits, and identifier characters `[a-zA-Z_-?!+/=<>*%]`.
- A leading UTF-8 BOM is an unrecognized character too: `E_INVALID_CHAR` at 1:1. Save files without a BOM.
- `E_UNTERMINATED_STRING` usually says only "reached end of input", with no line (the balance check finds it first): look for the last `"` that opens a string.
- Non-ASCII text is fine inside strings and comments. Identifiers are ASCII only.
- `'foo` and `(quote foo)` produce no data; quoted data is not a runtime value in compiled programs.

## See Also

- [macro-template-no-quasiquote](macro-template-no-quasiquote.md) - macro bodies are plain templates
- [syn-brackets-and-balance](syn-brackets-and-balance.md) - what the balance checker does catch
