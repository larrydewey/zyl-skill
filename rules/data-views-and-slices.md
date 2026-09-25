# data-views-and-slices

> Parse and window data without copying: `text/view` gives `StrView` (a substring) and `Cursor` (a parsing position), `collections/slice` gives `Slice` (a window on a Vec's storage); copy only at the end with `view-to-string` or `slice-to-vec`.

## Why It Matters

`str-substring` and building new Vecs copy every time. A view is an ordinary ADT holding its base, an offset and a length (`(StrViewC String Int Int)`, `(SliceC (Array T) Int Int)`), so making, splitting, trimming and sub-slicing copy nothing. Bounds are checked when a view is made from a String or Vec (`view-slice`, `slice-vec`, `slice-sub`, `slice-get`: `E_INDEX_OUT_OF_BOUNDS`, catchable); operations on an existing view stay inside it (`view-drop`, `view-take`, `slice-drop`, `slice-take` clamp; `view-byte-at` returns -1 outside). Because a view holds its base, region inference keeps the base alive as long as any view: a view of a string built in a callee can be returned safely.

## Good

```lisp
(use text/view)
(use collections/vec)
(use collections/slice)

(defn digit-p ((b Int)) (and (>= b 48) (<= b 57)))

(defn sum-fields ((parts (List StrView)))
  (match parts
    (Nil 0)
    (Cons p rest
      (+ (match (view-parse-int (view-trim p)) (Some n n) (None 0))
         (sum-fields rest)))))

(defn main ()
  (let line (view-of "  10, 20 ,x, 12  ")
    (let t (cursor-take-while (cursor-of "123abc") digit-p)
      (let v (vec-push (vec-push (vec-push (vec-create-default 4) 10) 20) 30)
        (let s (slice-vec v 1 2)
          (begin
            (print (sum-fields (view-split line 44)))          ; 42
            (print (view-trim line))                           ; 10, 20 ,x, 12
            (print (taken-view t))                             ; 123
            (print (cursor-pos (taken-rest t)))                ; 3
            (print (view-eq-str (view-slice "xab" 1 2) "ab"))  ; 1
            (print (slice-fold s (fn (a x) (+ a x)) 0))        ; 50
            (print s)                                          ; [20, 30]
            (print (try (slice-get s 2) (catch e -1)))         ; -1
            0))))))
```

## API

| `StrView` (`text/view`) | `Cursor` (`text/view`) | `Slice` (`collections/slice`) |
|---|---|---|
| `view-of s`, `view-slice s off len` | `cursor-of s`, `cursor-new view` | `slice-of-vec v`, `slice-vec v off len` |
| `view-sub v off len`, `view-drop v n`, `view-take v n` | `cursor-pos`, `cursor-view`, `cursor-rest`, `cursor-at-end` | `slice-sub s off len`, `slice-drop s n`, `slice-take s n` |
| `view-len`, `view-is-empty`, `view-byte-at v i` | `cursor-peek` (byte or -1), `cursor-advance c n` | `slice-len`, `slice-get s i`, `slice-get-or s i d` |
| `view-find v byte from`, `view-starts-with`, `view-ends-with` | `cursor-take-while c pred` returns `Taken` (`taken-view`, `taken-rest`) | `slice-fold s f init` |
| `view-split v byte` returns `(List StrView)`; `view-trim`, `view-trim-start`, `view-trim-end` | `cursor-skip-space`, `cursor-expect c "s"` returns `(Option Cursor)` | `slice-to-vec s arena` (copies) |
| `view-parse-int` returns `(Option Int)`; `view-eq`, `view-eq-str`, `view-compare` | | `Show` (`[20, 30]`) |
| `view-to-string` (the one copy); `Show`, `Debug`, `Eq`, `Ord`, `Hash` impls | | |

## Notes

- Compare views with `view-eq`, `view-eq-str`, `Eq.eq` or `Ord.compare`, never `==`: `==` is the generated structural equality over base, offset and length, so `(== (view-of "ab") (view-slice "xab" 1 2))` is false ([data-equality-structural](data-equality-structural.md)). `Hash.hash` of a view equals the hash of the String with its bytes.
- Views are byte-based: offsets, lengths and `view-byte-at` are bytes, not characters. Separators and predicates take byte values (`44` is `,`).
- A slice shares the Vec's storage: an element later written through a Vec that still uses that storage (`vec-set`, a `vec-push` onto an older version) is visible through the slice ([data-collections-persistent](data-collections-persistent.md)).
- The runtime accessors behind views (`zyl_view_byte`, `zyl_view_cmp`, `zyl_view_find`, `zyl_view_copy`) trust the bounds and are standard-library only: calling them from a program is `E_FFI_RESTRICTED`.

## See Also

- [data-collections-persistent](data-collections-persistent.md)
- [own-regions-status](own-regions-status.md)
