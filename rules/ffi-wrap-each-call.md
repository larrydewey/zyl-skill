# ffi-wrap-each-call

> Wrap every foreign function in one small, named Zyl function so the timeout, pinning and pointer conversion live in one place.

## Why It Matters

`ffi-call` has no declaration step and no type checking: the call *is* the declaration. Scattering raw calls multiplies the chances of a missing timeout, a float argument or an un-copied C string, and spreads capability requirements across a package.

## Good

```lisp
(defn c-strlen ((s String)) (ffi-call "strlen" s 1000))
(defn c-getenv ((name String))
  (let p (ffi-call "getenv" name 1000)
    (if (= p 0) None (Some (str-concat "" p)))))
(defn factorial (n) (ffi-call "zyl_factorial" n 1000))
```

## Notes

- libc is always linked: `strlen`, `abs`, `getenv`, `puts`, `free`, `strdup`...
- Runtime helpers reachable the same way: `zyl_actor_send_closure`, `zyl_actor_wait_all`, `zyl_cstr_from_int`, `zyl_now_ms`, `zyl_argc`/`zyl_arg_str`.
- In a package, every `ffi-call`/`ffi-pin`/`ffi-unpin` needs the `ffi` capability.

## See Also

- [ffi-timeout-always-last](ffi-timeout-always-last.md)
- [ffi-linking](ffi-linking.md)
