# pkg-stdlib-resolution

> When a stdlib or compiler change "doesn't take effect", suspect a stale `~/.zyl`: set `ZYL_HOME=$PWD/build/boot` or re-run `./install.sh`.

## Why It Matters

The compiler resolves the stdlib from its bundle directory: `$ZYL_HOME` if it holds `stdlib/`, else `~/.zyl` if it holds one, else the directory next to the compiler binary (`build/boot/`). An installed `~/.zyl` therefore **wins over the checkout** even when you run `build/boot/zyl-self`. `./boot.sh` exports `ZYL_HOME=build/boot` for this reason.

## Good

```bash
ZYL_HOME=$PWD/build/boot build/boot/zyl-self prog.zyl -o prog   # test against the checkout
./install.sh                                                     # refresh ~/.zyl after stdlib changes
```

## Notes

- `E_MODULE_NOT_FOUND` for a stdlib module: check the bundle directory's `stdlib/` exists and is current.
- User modules resolve next to the source file or within its package.
- No `ZYL_PATH`, no `zyl.toml`.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
- [tool-cli-arguments](tool-cli-arguments.md)
