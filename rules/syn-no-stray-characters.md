# syn-no-stray-characters

> Never write `'`, `` ` ``, `,`, `@`, `#`, `$`, `&`, `|`, `^`, `\` or non-ASCII bytes outside strings and comments.

## Why It Matters

The lexer treats any byte that cannot start a token as **end of input**, with no diagnostic. Everything after it is silently dropped. If the parentheses before the stray byte still balance, the program compiles and runs without the dropped code. The usual symptom is a missing function, or `undefined reference to _ZYL_main` when `main` was after the stray byte.

This also rules out Lisp habits: there is no `'quote`, no quasiquote/unquote (`` ` `` `,` `,@`), no `#| block comments |#`, and no commas in import lists.

## Bad

```lisp
(defn main ()
  (begin
    (print 1)
    @            ; lexer stops here -- (print 2) never compiled
    (print 2)
    0))

(use acme/greet { greet, greet-twice })   ; comma ends the token stream

#| block comment |#                        ; not a comment: truncates the file

(defmacro m (x) `(+ ,x 1))                ; backquote: rest of file vanishes
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
- A leading UTF-8 BOM is also an unrecognized character: the program becomes empty and linking reports a missing `main`.
- Non-ASCII text is fine inside strings and comments. Identifiers are ASCII only.
- `'foo` and `(quote foo)` produce no data; quoted data is not a runtime value in compiled programs.

## See Also

- [macro-template-no-quasiquote](macro-template-no-quasiquote.md) - macro bodies are plain templates
- [syn-brackets-and-balance](syn-brackets-and-balance.md) - what the balance checker does catch
