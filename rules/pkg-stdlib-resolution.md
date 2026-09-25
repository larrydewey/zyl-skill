# pkg-stdlib-resolution

> When a stdlib or compiler change "doesn't take effect", suspect a stale `~/.zyl`: set `ZYL_HOME=$PWD/build/boot`, or refresh the install with `./uninstall.sh && ./install.sh` (a verified `./boot.sh` does this for you).

## Why It Matters

The compiler resolves the stdlib (and `actor_runtime.c`) from its bundle directory, chosen by `cli-resolve-bundledir` in `selfhost/driver.zyl`:

1. If `ZYL_HOME` is set, it is the candidate; otherwise `$HOME/.zyl` is.
2. If the candidate holds a `stdlib/` directory, it is used.
3. Otherwise the directory of the compiler binary (`build/boot/` for `build/boot/zyl-self`) is used.

An installed `~/.zyl` therefore **wins over the checkout** even when you run `build/boot/zyl-self`. A `ZYL_HOME` that is set but has no `stdlib/` does not fall back to `~/.zyl`; it falls back to the binary's directory. `./boot.sh` and `run_regression_tests.sh` both export `ZYL_HOME=build/boot` for this reason, and a verified `./boot.sh` ends by refreshing an existing install (`uninstall.sh` then `install.sh`; `ZYL_NO_INSTALL_REFRESH=1` skips it, `ZYL_INSTALL_HOME` names another install).

## Good

```bash
ZYL_HOME=$PWD/build/boot build/boot/zyl-self prog.zyl -o prog   # test against the checkout
./uninstall.sh && ./install.sh                                   # refresh ~/.zyl by hand
```

## Notes

- `E_MODULE_NOT_FOUND` for a stdlib module: check the bundle directory's `stdlib/` exists and is current.
- The bundle directory also supplies the runtime the binary links with (`actor_runtime.o` if newer than `actor_runtime.c`, else the source), so a stale install links a stale runtime too.
- The compiler `chdir`s to the bundle directory before compiling; the source path and `-o` are resolved against the directory you ran it from first.
- User modules resolve next to the source file or within its package.
- No `ZYL_PATH`, no `zyl.toml`.

## See Also

- [boot-fixed-point-workflow](boot-fixed-point-workflow.md)
- [tool-cli-arguments](tool-cli-arguments.md)
