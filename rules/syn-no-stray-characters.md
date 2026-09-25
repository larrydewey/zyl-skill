# syn-no-stray-characters

> Never write `@`, `#`, `$`, `|`, `^`, `\`, a `.` that does not start `.name`, or non-ASCII bytes outside strings and comments: each is a located `E_INVALID_CHAR`. `'`, `` ` ``, `,` and `,@` are reader syntax, and `,`/`,@` outside a quasiquote or macro template is `E_MALFORMED_FORM`.

## Why It Matters

The lexer rejects any byte that cannot start a token with a located `error[E_INVALID_CHAR]` (file:line:col, caret, and a help line listing the characters that may appear only in strings and comments); a string left open is `E_UNTERMINATED_STRING`. (Before 2026-09-24 such a byte ended the input with no diagnostic and the rest of the file was dropped; an older installed compiler still does that.)

Since 2026-09-25 four characters that used to be stray are reader sugar ([syn-list-literals-and-quote](syn-list-literals-and-quote.md)): `'d` is `(quote d)`, `` `d `` is `(quasiquote d)`, `,e` is `(unquote e)` and `,@e` is `(unquote-splicing e)`. That changes how Lisp habits fail. A comma used as a separator no longer stops at the lexer: it reads as `unquote` and fails later, in an import list with an unlocated message naming an empty symbol. A backquoted macro template is read as quoted *data*, so every name in it is an error. `&` may now start an identifier (for `&rest`).

## Bad

```lisp
(defn main ()
  (begin
    (print 1)
    @            ; error[E_INVALID_CHAR]: unexpected character `@`
    (print 2)
    0))

(use text/view { view-of, view-len })  ; E_PKG_UNKNOWN_SYMBOL: "symbol  does not exist" (the comma)

(print (+ x ,y))                       ; E_MALFORMED_FORM: `,` outside a quasiquote or a macro template

#| block comment |#                    ; not a comment: E_INVALID_CHAR at `#`

(defmacro inc (x) `(+ ,x 1))           ; E_MALFORMED_FORM: malformed quasiquote: `+` is a name

(print .5)                             ; E_INVALID_CHAR at `.`: write 0.5

(print "unclosed)                      ; E_UNTERMINATED_STRING
```

## Good

```lisp
(defn main ()
  (begin
    (print 1)
    (print 2)
    0))

(use text/view { view-of view-len })   ; whitespace-separated

;; line comments only
(defmacro inc (x) (+ x 1))             ; the macro body is the template
(print 0.5)
```

## Notes

- Allowed token starters: whitespace, `( ) [ ] { }`, `"`, `'`, `` ` ``, `,`, `:`, `~`, `;`, digits, identifier characters `a-z A-Z _ - ? ! + / = < > * % &`, and `.` when a letter follows (the `.method` of `((expr).method args)`).
- Still `E_INVALID_CHAR`: `@` (except directly after `,`), `#`, `$`, `|`, `^`, `\`, a `.` not followed by a letter (`.5`, a dotted pair `(a . b)`), and any byte of 128 or above.
- `,@` is one token only when `@` follows the comma directly: `, @e` is a comma and then an invalid `@`.
- A leading UTF-8 BOM is an unrecognized character too: `E_INVALID_CHAR` at 1:1. Save files without a BOM.
- `E_UNTERMINATED_STRING` usually says only "reached end of input", with no line (the balance check finds it first): look for the last `"` that opens a string.
- Non-ASCII text is fine inside strings and comments. Identifiers are ASCII only.
- `'x` (a quoted name) is `E_MALFORMED_FORM`: there is no symbol type. Quoted data holds only numbers, strings, Bools and lists of them.

## See Also

- [syn-list-literals-and-quote](syn-list-literals-and-quote.md) - what `'`, `` ` ``, `,` and `,@` mean
- [macro-quasiquote-and-rest](macro-quasiquote-and-rest.md) - `,@` in macro templates
- [syn-brackets-and-balance](syn-brackets-and-balance.md) - what the balance checker does catch
